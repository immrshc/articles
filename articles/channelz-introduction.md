---
title: "gRPCのChannelzの概要"
emoji: "🍇"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["grpc", "go"]
published: true
---

# 概要

クライアントとサーバー間で通信が失敗するとどのサーバーで失敗したのか、そもそもコネクションは確立されたのか、どこまで処理が成功していたのかなどを調査する必要があります。gRPCによる通信の場合は、Channelzという通信状況をデバッグできるツールが用意されています。具体的なChannelzを使ったデバッグ方法は[こちらの記事](https://grpc.io/blog/a-short-introduction-to-channelz/)で詳しく説明されています。

今回はgrpc-goを例にChannelzの実現できることとその仕組みについて調べた内容を整理しました。

主に以下のモチベーションを持った方を対象に順を追って説明していきます。

- Channelzの概要を知りたい
- 実際に動作確認して試してみたい
- 具体的にどう実装されているか知りたい

# Channelzの概念

様々な実装はそれぞれ詳細を持っているので抽象化した概念で表現することでChannelzによる通信状況の確認を実現しています。
まず、ここでは基本的な概念を整理します。[こちらのProposal](https://github.com/grpc/proposal/blob/master/A14-channelz.md)でより詳しく説明されています。


**channel**

まずchannelは一つのRPCを抽象化したものです。そしてchannelは非巡回有向グラフを形成します。つまり、向きが巡回しないようにchannelが複数のchannelもしくはsubchannel（後述）を持ち、それぞれのchannelが同じように続いていきます。そして最後の末端はsocket（後述）を持ちます。

https://github.com/grpc/proposal/blob/master/A14-channelz.md#channels-and-subchannels
> Channels and Subchannels, or descendent channels, are hierarchically organized into a DAG structure. The union of all channels and subchannels may not contain a cycle. A descendent channel may have any number of descendent channels. Each descendent channel may also have any number of sockets. However, a given descendent channel cannot have heterogeneous children. That is, a channel or subchannel may have descendent channels, or have sockets, but not both.

c.f. https://github.com/grpc/proposal/blob/master/A14-channelz.md#channelz-data

grpc-goでは[channelz.ChannelMetric](https://pkg.go.dev/google.golang.org/grpc/internal/channelz#ChannelMetric)で表現されています。
そこでは子となるchannelやsocketを持ち、Traceでは発生したイベントログが格納されます。

```go
type ChannelMetric struct {
	// ID is the channelz id of this channel.
	ID int64
	// RefName is the human readable reference string of this channel.
	RefName string
	// ChannelData contains channel internal metric reported by the channel through
	// ChannelzMetric().
	ChannelData *ChannelInternalMetric
	// NestedChans tracks the nested channel type children of this channel in the format of
	// a map from nested channel channelz id to corresponding reference string.
	NestedChans map[int64]string
	// SubChans tracks the subchannel type children of this channel in the format of a
	// map from subchannel channelz id to corresponding reference string.
	SubChans map[int64]string
	// Sockets tracks the socket type children of this channel in the format of a map
	// from socket channelz id to corresponding reference string.
	// Note current grpc implementation doesn't allow channel having sockets directly,
	// therefore, this is field is unused.
	Sockets map[int64]string
	// Trace contains the most recent traced events.
	Trace *ChannelTrace
}
```

三つ目のフィールドの型である[channelz.ChannelInternalMetric](https://pkg.go.dev/google.golang.org/grpc/internal/channelz#ChannelInternalMetric)はそのchannel自体の情報を持ちます。そして以下のように、clientとserverの通信状態（[Channel state](https://github.com/grpc/grpc/blob/master/doc/connectivity-semantics-and-api.md)）やRPCの成功、失敗した数などが保持されています。

```go
type ChannelInternalMetric struct {
	// current connectivity state of the channel.
	State connectivity.State
	// The target this channel originally tried to connect to.  May be absent
	Target string
	// The number of calls started on the channel.
	CallsStarted int64
	// The number of calls that have completed with an OK status.
	CallsSucceeded int64
	// The number of calls that have a completed with a non-OK status.
	CallsFailed int64
	// The last time a call was started on the channel.
	LastCallStartedTimestamp time.Time
}
```

**subchannel**

subchannelはchannelの通信がロードバランスされ、複数のサーバーとコネクションを確立した場合の一つ一つを表現しています。channel同様に複数のchannelもしくはsubchannelを持ちます。

https://github.com/grpc/proposal/blob/master/A14-channelz.md#channelz-service
> A "subchannel" represents an abstraction that is load balanced over by an owning channel. A subchannel may have channels and subchannels. 

**socket**

トランスポート層でTCPコネクションが確立され、そこにはHTTP/2のstreamが多重化されることになります。socketはTCPコネクションごとに複数のstreamに関する情報を保持しています。

https://github.com/grpc/proposal/blob/master/A14-channelz.md#sockets-1
> Conceptually, sockets are the equivalent of a file descriptor. They have a local and remote address, as well as some concept of security detail. Sockets keep track of "streams" while Channels and Servers keep track of "calls."

grpc-goでは[Channelz.SocketMetric](https://pkg.go.dev/google.golang.org/grpc/internal/channelz#SocketMetric)が対応しています。

# 実装による確認

## Channelz用のサーバーの仕組み

channelzの情報はchannelz用のAPI（[ChannelzServer](https://github.com/grpc/grpc-go/blob/v1.32.x/channelz/grpc_channelz_v1/channelz.pb.go#L3029-L3046)インターフェース）を実装した構造体をサーバー（[grpc.Server](https://pkg.go.dev/google.golang.org/grpc@v1.33.0#Server)）に登録する必要があります。

```go
// ChannelzServer is the server API for Channelz service.
type ChannelzServer interface {
	// Gets all root channels (i.e. channels the application has directly
	// created). This does not include subchannels nor non-top level channels.
	GetTopChannels(context.Context, *GetTopChannelsRequest) (*GetTopChannelsResponse, error)
	// Gets all servers that exist in the process.
	GetServers(context.Context, *GetServersRequest) (*GetServersResponse, error)
	// Returns a single Server, or else a NOT_FOUND code.
	GetServer(context.Context, *GetServerRequest) (*GetServerResponse, error)
	// Gets all server sockets that exist in the process.
	GetServerSockets(context.Context, *GetServerSocketsRequest) (*GetServerSocketsResponse, error)
	// Returns a single Channel, or else a NOT_FOUND code.
	GetChannel(context.Context, *GetChannelRequest) (*GetChannelResponse, error)
	// Returns a single Subchannel, or else a NOT_FOUND code.
	GetSubchannel(context.Context, *GetSubchannelRequest) (*GetSubchannelResponse, error)
	// Returns a single Socket or else a NOT_FOUND code.
	GetSocket(context.Context, *GetSocketRequest) (*GetSocketResponse, error)
}
```

そして、channelzの情報自体はchannelz用のサーバーを起動した上で、同じmain関数内でgRPCのクライアントもしくはサーバーの実行をすることで取得できるようになります。

内部的には、[channelMap](https://github.com/grpc/grpc-go/blob/v1.32.x/internal/channelz/funcs.go#L316-L329)構造体でchannelzに関する情報を保持し、その変数（`db`）をロックを取りながら、クライアントもしくはサーバーの処理に応じて更新していくようになっています。channelz用のサーバーはその変数から値を取り出して返しています。したがって、同じ変数を共有して更新と取得を行うため一つのmain関数の範囲内でchannelzの情報は取得できることが分かります。

https://github.com/grpc/grpc-go/blob/v1.32.x/internal/channelz/funcs.go
```go
var (
	db    dbWrapper
  ...
)

// dbWarpper wraps around a reference to internal channelz data storage, and
// provide synchronized functionality to set and get the reference.
type dbWrapper struct {
	mu sync.RWMutex
	DB *channelMap
}

// channelMap is the storage data structure for channelz.
// Methods of channelMap can be divided in two two categories with respect to locking.
// 1. Methods acquire the global lock.
// 2. Methods that can only be called when global lock is held.
// A second type of method need always to be called inside a first type of method.
type channelMap struct {
	mu               sync.RWMutex
	topLevelChannels map[int64]struct{}
	servers          map[int64]*server
	channels         map[int64]*channel
	subChannels      map[int64]*subChannel
	listenSockets    map[int64]*listenSocket
	normalSockets    map[int64]*normalSocket
}
```

## クライアント側の実装

実際にクライアントとサーバーの実装をしてChannelzの情報を確認してみます。

[A short introduction to Channelz](https://grpc.io/blog/a-short-introduction-to-channelz/)で紹介されている、[クライアント](https://gist.github.com/lyuxuan/515fa6da7e0924b030e29b8be56fd90a)と[サーバー](https://gist.github.com/lyuxuan/81dd08ca649a6c78a61acc7ab05e0fef)の実装を参考にさせていただきました。今のgrpc-goのバージョンだと利用できないメソッド呼び出しなどの修正や、デバッグ用に`gRPC Server Reflection`の設定などを追加で行っています。

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net"
	"os"
	"os/signal"
	"time"

	"google.golang.org/grpc/balancer/roundrobin"

	"google.golang.org/grpc/resolver"
	"google.golang.org/grpc/resolver/manual"

	"google.golang.org/grpc"
	"google.golang.org/grpc/channelz/service"
	pb "google.golang.org/grpc/examples/helloworld/helloworld"
	"google.golang.org/grpc/reflection"
)

func main() {
	// channelzのRPC用のサーバーを起動する
	lis, err := net.Listen("tcp", ":50050")
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()
	service.RegisterChannelzServiceToServer(s)
	// 確認用にgRPC Serviceの情報を返せるようにする
	// c.f. https://github.com/grpc/grpc-go/blob/master/Documentation/server-reflection-tutorial.md
	reflection.Register(s)
	go s.Serve(lis)
	defer s.Stop()

	// 三つのサーバーにラウンドロビンするための名前解決の設定
	r, cleanup := manual.GenerateAndRegisterManualResolver()
	defer cleanup()
	state := resolver.State{Addresses: []resolver.Address{{Addr: ":10001"}, {Addr: ":10002"}, {Addr: ":10003"}}}
	r.InitialState(state)
	// サーバーへのコネクションを設定する
	conn, err := grpc.Dial(
		r.Scheme()+":///test.server",
		grpc.WithInsecure(),
		grpc.WithDefaultServiceConfig(fmt.Sprintf(`{"LoadBalancingPolicy": "%s"}`, roundrobin.Name)),
	)
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	defer conn.Close()

	// サーバーへRPCするクライアントの設定
	c := pb.NewGreeterClient(conn)
	// 100回RPCし、150msをタイムアウトの閾値とする
	for i := 0; i < 100; i++ {
		ctx, cancel := context.WithTimeout(context.Background(), 150*time.Millisecond)
		defer cancel()
		r, err := c.SayHello(ctx, &pb.HelloRequest{Name: "world"})
		if err != nil {
			log.Printf("could not greet: %v", err)
		} else {
			log.Printf("Greeting: %s", r.Message)
		}
	}

	// CTRL+Cでexitするまで待つことで、channelzの情報を保持しておける
	ch := make(chan os.Signal, 1)
	signal.Notify(ch, os.Interrupt)
	<-ch
}
```

## サーバー側の実装

```go
package main

import (
	"context"
	"log"
	"math/rand"
	"net"
	"os"
	"os/signal"
	"time"

	"google.golang.org/grpc/reflection"

	"google.golang.org/grpc"
	"google.golang.org/grpc/channelz/service"

	pb "google.golang.org/grpc/examples/helloworld/helloworld"
)

var (
	ports = []string{":10001", ":10002", ":10003"}
)

type server struct {
	pb.UnimplementedGreeterServer
}

func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	return &pb.HelloReply{Message: "Hello " + in.Name}, nil
}

type slowServer struct {
	pb.UnimplementedGreeterServer
}

func (s *slowServer) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	// 100~200msの間レスポンスを待つ
	time.Sleep(time.Duration(100+rand.Intn(100)) * time.Millisecond)
	return &pb.HelloReply{Message: "Hello " + in.Name}, nil
}

func main() {
	// channelzのRPC用のサーバーを起動する
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatal(err)
	}
	defer lis.Close()
	s := grpc.NewServer()
	service.RegisterChannelzServiceToServer(s)
	// 確認用にgRPC Serviceの情報を返せるようにする
	// c.f. https://github.com/grpc/grpc-go/blob/master/Documentation/server-reflection-tutorial.md
	reflection.Register(s)
	go s.Serve(lis)
	defer s.Stop()

	// 三つのサーバーを起動させ、一つをレスポンスがクライアント側で設定したタイムアウトを超えるサーバーにする
	var listeners []net.Listener
	var svrs []*grpc.Server
	for i := 0; i < 3; i++ {
		lis, err := net.Listen("tcp", ports[i])
		if err != nil {
			log.Fatalf("failed to listen: %v", err)
		}
		listeners = append(listeners, lis)
		s := grpc.NewServer()
		svrs = append(svrs, s)
		if i == 2 {
			pb.RegisterGreeterServer(s, &slowServer{})
		} else {
			pb.RegisterGreeterServer(s, &server{})
		}
		go s.Serve(lis)
	}

	// CTRL+Cでexitするまで待つことで、channelzの情報を保持しておける
	ch := make(chan os.Signal, 1)
	signal.Notify(ch, os.Interrupt)
	<-ch

	for i := 0; i < 3; i++ {
		svrs[i].Stop()
		listeners[i].Close()
	}
}
```

## Channelzの内容を確認する

それぞれのChannelzサーバーのポートは以下のように指定しています。

- クライアント: 50050
- サーバー: 50051

まずクライアント側のChannelzサーバーにRPCしてクライアント側保持している情報を確認します。ここでは、[channlez.GetTopChannels](https://pkg.go.dev/google.golang.org/grpc/internal/channelz#GetTopChannels)を呼び出しています。

```shell
{
  "channel": [
    {
      "ref": {
        "channelId": "2",
        "name": "c69t4j8m1o1k:///test.server"
      },
      "data": {
        "state": {
          "state": "READY"
        },
        "target": "c69t4j8m1o1k:///test.server",
        "trace": {
          "numEventsLogged": "11",
          "creationTimestamp": "2020-10-11T05:20:03.881251Z",
          "events": [
            {
              "description": "Channel Created",
              "severity": "CT_INFO",
              "timestamp": "2020-10-11T05:20:03.881444Z"
            },
            {
              "description": "parsed scheme: \"c69t4j8m1o1k\"",
              "severity": "CT_INFO",
              "timestamp": "2020-10-11T05:20:03.881488Z"
            },
            {
              "description": "ccResolverWrapper: sending update to cc: {[{:10001  <nil> 0 <nil>} {:10002  <nil> 0 <nil>} {:10003  <nil> 0 <nil>}] <nil> <nil>}",
              "severity": "CT_INFO",
              "timestamp": "2020-10-11T05:20:03.881522Z"
            },
            {
              "description": "Resolver state updated: {Addresses:[{Addr::10001 ServerName: Attributes:<nil> Type:0 Metadata:<nil>} {Addr::10002 ServerName: Attributes:<nil> Type:0 Metadata:<nil>} {Addr::10003 ServerName: Attributes:<nil> Type:0 Metadata:<nil>}] ServiceConfig:<nil> Attributes:<nil>} (resolver returned new addresses)",
              "severity": "CT_INFO",
              "timestamp": "2020-10-11T05:20:03.881537Z"
            },
            {
              "description": "ClientConn switching balancer to \"round_robin\"",
              "severity": "CT_INFO",
              "timestamp": "2020-10-11T05:20:03.881543Z"
            },
            {
              "description": "Channel switches to new LB policy \"round_robin\"",
              "severity": "CT_INFO",
              "timestamp": "2020-10-11T05:20:03.881546Z"
            },
            {
              "description": "Subchannel(id:4) created",
              "severity": "CT_INFO",
              "timestamp": "2020-10-11T05:20:03.881580Z",
              "subchannelRef": {
                "subchannelId": "4"
              }
            },
            {
              "description": "Subchannel(id:5) created",
              "severity": "CT_INFO",
              "timestamp": "2020-10-11T05:20:03.881600Z",
              "subchannelRef": {
                "subchannelId": "5"
              }
            },
            {
              "description": "Subchannel(id:6) created",
              "severity": "CT_INFO",
              "timestamp": "2020-10-11T05:20:03.881609Z",
              "subchannelRef": {
                "subchannelId": "6"
              }
            },
            {
              "description": "Channel Connectivity change to CONNECTING",
              "severity": "CT_INFO",
              "timestamp": "2020-10-11T05:20:03.881753Z"
            },
            {
              "description": "Channel Connectivity change to READY",
              "severity": "CT_INFO",
              "timestamp": "2020-10-11T05:20:03.882701Z"
            }
          ]
        },
        "callsStarted": "100",
        "callsSucceeded": "81",
        "callsFailed": "19",
        "lastCallStartedTimestamp": "2020-10-11T05:20:08.591615Z"
      },
      "subchannelRef": [
        {
          "subchannelId": "4"
        },
        {
          "subchannelId": "5"
        },
        {
          "subchannelId": "6"
        }
      ]
    }
  ],
  "end": true
}
```

Channelの作成から、名前解決、SubChannelの作成、それとの接続などがtraceから確認できます。また100回呼び出して81回成功（19回失敗）していることが分かります。

今度はサーバー側のChannelzサーバーの[channelz.GetServers](https://pkg.go.dev/google.golang.org/grpc/internal/channelz#GetServers)を呼び出してみます。
Channelzにおいて、serverとはRPCのエンドポイントであり、いくつかのsocketを持つものを意味します。

https://github.com/grpc/proposal/blob/master/A14-channelz.md#channelz-service
> A "server" represents the entry point for RPCs. A server may have one or more listening sockets, and has a collection of "services". Unlike clients, servers are not hierarchical. A server may only have sockets.

```shell
$ echo {} | evans -r -p 50051 cli call grpc.channelz.v1.Channelz.GetServers | jq .
{
  "server": [
    {
      "ref": {
        "serverId": "1"
      },
      "data": {
        "callsStarted": "4",
        "callsSucceeded": "2",
        "lastCallStartedTimestamp": "2020-10-11T05:22:11.234225Z"
      },
      "listenSocket": [
        {
          "socketId": "5",
          "name": "[::]:50051"
        }
      ]
    },
    {
      "ref": {
        "serverId": "2"
      },
      "data": {
        "callsStarted": "33",
        "callsSucceeded": "33",
        "lastCallStartedTimestamp": "2020-10-11T05:20:08.590413Z"
      },
      "listenSocket": [
        {
          "socketId": "6",
          "name": "[::]:10001"
        }
      ]
    },
    {
      "ref": {
        "serverId": "3"
      },
      "data": {
        "callsStarted": "33",
        "callsSucceeded": "33",
        "lastCallStartedTimestamp": "2020-10-11T05:20:08.591172Z"
      },
      "listenSocket": [
        {
          "socketId": "8",
          "name": "[::]:10002"
        }
      ]
    },
    {
      "ref": {
        "serverId": "4"
      },
      "data": {
        "callsStarted": "34",
        "callsSucceeded": "18",
        "callsFailed": "16",
        "lastCallStartedTimestamp": "2020-10-11T05:20:08.591976Z"
      },
      "listenSocket": [
        {
          "socketId": "7",
          "name": "[::]:10003"
        }
      ]
    }
  ],
  "end": true
}
```

Channelz用のサーバーと残り三つのサーバーが存在し、100回中84回成功（16回失敗）していることが分かります。

また、クライアント側では19回の失敗だったが、サーバー側では16回の失敗となっており、3回はサーバー側では成功と扱われていることが分かります。ここでは載せていませんが、socketの内容も同じ回数にそれぞれなっていました。gRPCの場合はクライアントとサーバー側の成否は独立しているためだと思われます。この辺りは別途詳しく見ていきたいと思っています。

https://grpc.io/blog/deadlines/
> In gRPC, both the client and server make their own independent and local determination about whether the remote procedure call (RPC) was successful. This means their conclusions may not match! An RPC that finished successfully on the server side can fail on the client side. For example, the server can send the response, but the reply can arrive at the client after their deadline has expired. The client will already have terminated with the status error DEADLINE_EXCEEDED.


# ソースコードから仕組みを知る

grpc-go（ここでのバージョンは1.32）の実装を見てみると以下のタイミングで、Channelzに関する情報が登録されていました。

## Server

- grpc.NewServerでserverが登録される
  - https://github.com/grpc/grpc-go/blob/v1.32.x/server.go#L522
  - https://github.com/grpc/grpc-go/blob/v1.32.x/internal/channelz/funcs.go#L246
- TCPコネクションをacceptするとsocketが登録される
  - https://github.com/grpc/grpc-go/blob/v1.32.x/server.go#L704
  - https://github.com/grpc/grpc-go/blob/v1.32.x/internal/channelz/funcs.go#L261
- RPCの呼び出し後、結果をバッファに書き込みしてchannelの状態を更新する
  - https://github.com/grpc/grpc-go/blob/v1.32.x/server.go#L1088-L1090  

## Client

- grpc.ClientConnを作成するとchannelが登録される
  - https://github.com/grpc/grpc-go/blob/v1.32.x/clientconn.go#L151
- grpc.ClientConnを作成時にResolverState.Addressesごとにsubchannelを登録する
  - https://github.com/grpc/grpc-go/blob/v1.32.x/clientconn.go#L307
  - https://github.com/grpc/grpc-go/blob/v1.32.x/balancer/base/balancer.go#L107
  - https://github.com/grpc/grpc-go/blob/v1.32.x/balancer_conn_wrappers.go#L140
  - https://github.com/grpc/grpc-go/blob/revert-3938-release_version_1.33.0/clientconn.go#L744
