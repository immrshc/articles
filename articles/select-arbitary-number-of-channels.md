---
title: "reflect.Selectを使って任意の個数のchannelと通信する"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# 概要

可変数のどれかのchannelと通信するには、reflect.Selectを利用することで実現できます。
ここでは、その使い方とどのような状況で利用すべきかについて説明します。

# 通常の方法と課題

まず通常、複数のchannelと通信するにはselectを利用することで実現できます。

selectはcaseに通信を記述しどれかが通信可能になるまで待機します。通信可能になった時点でそのcaseの処理を実行し、selectを抜けます。

```go
func main() {
  select {
  case <-ch1:
    ...
  case <-ch2:
    ...
  case ch2 <- x:
    ...
  }
}
```

しかし、上記の例のように明示的にcaseでchannelとそれとの通信（送受信）を指定する必要があります。したがって、可変数のchannelを受け取って、そのうちのどれかと通信できるまで待つことがこれだとできません。

# reflect.Selectの利用
可変数のchannelのうちどれかと通信できるまで待つには、[reflect.Select](https://golang.org/pkg/reflect/#Select)を利用することで実現できます。

```go
func Select(cases []SelectCase) (chosen int, recv Value, recvOK bool)
```

channelとの通信のリストを引数に少なくとも一つの通信がされるまで待機し、通信後に選ばれたリストのインデックス、受信した値、チャンネルが閉じられていないかどうかを返します。

引数には、[reflect.SelectCase](https://golang.org/pkg/reflect/#SelectCase)のスライスが必要です。これは、selectにおけるcaseを表現したものです。

```go
type SelectCase struct {
    Dir  SelectDir // direction of case
    Chan Value     // channel to use (for send or receive)
    Send Value     // value to send (for send)
}
```

Dirでcaseにおける通信の種類（[SelectDir](https://golang.org/pkg/reflect/#SelectDir)）を指定します。つまり、デフォルト、送信、受信があります。

```go
const (
    SelectSend    SelectDir // case Chan <- Send
    SelectRecv              // case <-Chan:
    SelectDefault           // default
)
```

そして、Chanで送受信に利用されるchannelと、Sendで送信時に送られる値を指定します。受信時はSendはゼロ値である必要があります。

以下のように、任意の個数のchannelを受け取って、最初に通信できたchannelの値を標準出力できます。
```go
func send(ch chan<- int) {
	ch <- rand.Int()
}

func printRecv(chs []<-chan int) {
	cases := make([]reflect.SelectCase, len(chs))
	// 共にgoroutineから送信されるので受信用のSelectCaseを作成する
	for i, ch := range chs {
		cases[i] = reflect.SelectCase{Dir: reflect.SelectRecv, Chan: reflect.ValueOf(ch)}
	}
	// 通信があるまで待機する
	chosen, value, ok := reflect.Select(cases)
	if ok {
		fmt.Printf("chosen: %d, value: %v\n", chosen, value)
	}
}

func main() {
	ch1 := make(chan int)
	ch2 := make(chan int)
	go send(ch1)
	go send(ch2)
	
	chs := []<-chan int{ch1, ch2}
	printRecv(chs)
}

// $ go run main.go
// > chosen: 1, value: 5577006791947779410
```

# reflect.Selectを使って複数のサーバーの起動を管理する

具体的なユースケースとして、複数のサーバーを起動させてどれか一つが終了すると全て終了させるようなことがreflect.Selectを使って実現できます。

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"reflect"
	"strings"
	"time"
)

type muxErrs struct {
	errs []error
}

func (m *muxErrs) Error() string {
	messages := make([]string, 0)
	for _, err := range m.errs {
		if err != nil {
			messages = append(messages, err.Error())
		}
	}
	return strings.Join(messages, ", ")
}

type server interface {
	Start() error
	ErrChan() <-chan error
}

type mux struct {
	servers []server
}

func (m *mux) Serve() error {
	errs := make([]error, len(m.servers))
	cases := make([]reflect.SelectCase, len(m.servers))
	for i, s := range m.servers {
		if err := s.Start(); err != nil {
			errs[i] = err
			break
		}
		cases[i] = reflect.SelectCase{Dir: reflect.SelectRecv, Chan: reflect.ValueOf(s.ErrChan())}
	}
	// どれかからエラーを受信するまで待つ
	chosen, value, ok := reflect.Select(cases)
	if ok {
		if err, ok := value.Interface().(error); ok {
			errs[chosen] = err
		}
	}
	return &muxErrs{
		errs: errs,
	}
}

type webServer struct {
	Addr         string
	SurvivalTime time.Duration
	errCh        chan error
}

func (w *webServer) Start() error {
	mux := http.NewServeMux()
	server := &http.Server{Addr: w.Addr, Handler: mux}

	w.errCh = make(chan error, 1)
	go func() {
		fmt.Printf("----ListenAndServe %s----\n", w.Addr)
		// 起動し終わるとエラーを送信する
		w.errCh <- server.ListenAndServe()
		close(w.errCh)
	}()
	go func() {
		time.Sleep(w.SurvivalTime)
		fmt.Printf("-------Shutdown %s-------\n", w.Addr)
		// 指定時間を待ってからサーバーを終了させる
		if err := server.Shutdown(context.Background()); err != nil {
			w.errCh <- err
			close(w.errCh)
		}
	}()
	return nil
}

func (w *webServer) ErrChan() <-chan error {
	return w.errCh
}

func main() {
	s1 := &webServer{Addr: ":8080", SurvivalTime: 10 * time.Second}
	s2 := &webServer{Addr: ":8081", SurvivalTime: 5 * time.Second}
	m := mux{
		servers: []server{s1, s2},
	}
	if err := m.Serve(); err != nil {
		log.Printf("main exited because %s", err)
	}
}
```

実行すると以下のように8081ポートのサーバーが終了するとmainを抜けることがわかります。
```shell
$ go run main.go
----ListenAndServe :8080----
----ListenAndServe :8081----
-------Shutdown :8081-------
2020/09/26 14:08:46 main exited because http: Server closed
```
