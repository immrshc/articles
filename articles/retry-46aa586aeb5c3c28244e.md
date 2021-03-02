---
title: "Goのリトライ処理で考慮すること"
emoji: "🔁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

# 概要
## 動機
サービス間通信でリトライ処理をする必要があります。なぜなら、一時的な通信先の不具合など、しばらくしてから再実行することで成功する場合があるからです。しかし、Goの標準パッケージからはリトライ機構は提供されていないので、自身で実装するか他のパッケージを利用しなければなりません。

ここでは具体的な実装例ではなく、net/httpパッケージの実装（主に、`http.Client`と`http.Transport`）を踏まえた上で、何がリトライ処理で考慮されているべきかを整理しました。

## 要約

リトライ処理を実装する場合は以下の観点を考慮する必要があります。

- リクエストの内容（`Request.Body`）をリトライ前に巻き戻す
- `Request.Context()`の終了を確認する
- リトライ前に`Response.Body`を全て読み切ってから閉じる
- デフォルトの`Transport`を使ってコネクションプールを管理すべきか考える

# 基本的な要件
まず、以下の要件を満たしたリトライ処理を行う前提とします。

- リトライ上限の指定
  - 指定回数以上はリトライしない
- リトライ可否の判定
  - レスポンスとエラーに応じてリトライを続けるか判定できる
- リトライ間隔の調整（バックオフ）
  - Exponential Backoffなどリトライ間隔を指定のアルゴリズムに応じて調整できる

以下がその実装例ですが、これから説明する留意点は全て実装されていません。

```go
type CheckRetry func(*http.Response, error) bool
type Backoff func(attemptNum int) time.Duration

type Client struct {
	RetryMax   int
	HTTPClient *http.Client
	CheckRetry CheckRetry
	Backoff    Backoff
}

func (c *Client) Do(req *http.Request) (*http.Response, error) {
	var attemptNum int
	for {
		attemptNum++
		res, err := c.HTTPClient.Do(req)
		shouldRetry := c.CheckRetry(res, err)
		if !shouldRetry {
			return res, err
		}
		if c.RetryMax < attemptNum {
			return nil, errors.New("retry max exceeded")
		}
		wait := c.Backoff(attemptNum)
		time.Sleep(wait)
	}
}
```

# リクエストの内容を巻き戻す
`http.Request`の`Body`フィールドは、`io.ReadCloser`インターフェース型です。複数回リトライされる場合は、複数回`io.ReadCloser`が実行されることになります。

https://github.com/golang/go/blob/go1.16/src/net/http/request.go#L167-L181
```go
type Request struct {
	...

	// Body is the request's body.
	//
	// For client requests, a nil body means the request has no
	// body, such as a GET request. The HTTP Client's Transport
	// is responsible for calling the Close method.
	//
	// For server requests, the Request Body is always non-nil
	// but will return EOF immediately when no body is present.
	// The Server will close the request body. The ServeHTTP
	// Handler does not need to.
	//
	// Body must allow Read to be called concurrently with Close.
	// In particular, calling Close should unblock a Read waiting
	// for input.
	Body io.ReadCloser

	...
}
```

しかし、`io.ReadCloser`を満たす具象型によっては冪等な操作にならない場合があります。例えば、`io.ReadCloser`を満たす`bytes.Buffer`構造体などは読み取った位置を内部で保持しています。

https://github.com/golang/go/blob/go1.16/src/bytes/buffer.go#L18-L24
```go
// A Buffer is a variable-sized buffer of bytes with Read and Write methods.
// The zero value for Buffer is an empty buffer ready to use.
type Buffer struct {
	buf      []byte // contents are the bytes buf[off : len(buf)]
	off      int    // read at &buf[off], write at &buf[len(buf)]
	lastRead readOp // last read operation, so that Unread* can work correctly.
}
```

したがって、一つの`Request`をもとに複数回リクエストする場合は、`Request.Body`を初期状態に戻す必要があります。

## net/httpのリクエストの巻き戻し
ここではnet/httpの実装を参考にします。net/httpパッケージの`Client`構造体はHTTPクライアントであり、内部で`Transport`構造体を持ちます。この`Transport`が実際にはコネクション確立やコネクションプールの管理など行っています。そして、冪等なリクエストがネットワークエラーになった特定の場合のみリトライするようになっています。

https://golang.org/pkg/net/http/#Transport
> Transport only retries a request upon encountering a network error if the request is idempotent and either has no body or has its Request

実装としては、リクエスト時に呼び出される`Trasport.roundTrip`メソッド内のループ処理でリトライ可否を判定しています。
https://github.com/golang/go/blob/go1.16/src/net/http/transport.go#L604

そこでは以下のように、`rewindBody`関数により`Requst.Body`を巻き戻しています。
https://github.com/golang/go/blob/go1.16/src/net/http/transport.go#L615
https://github.com/golang/go/blob/go1.16/src/net/http/transport.go#L653-L674
```go
func (t *Transport) roundTrip(req *Request) (*Response, error) {
	...
  
	for {
		...
		pconn, err := t.getConn(treq, cm)
    
		...
		resp, err = pconn.roundTrip(treq)

		// Failed. Clean up and determine whether to retry.
		...
 
		// Rewind the body if we're able to.
		req, err = rewindBody(req)
		...
	}
}

func rewindBody(req *Request) (rewound *Request, err error) {
	...
	if !req.Body.(*readTrackingBody).didClose {
		req.closeBody()
	}
	if req.GetBody == nil {
		return nil, errCannotRewind
	}
	body, err := req.GetBody()
	if err != nil {
		return nil, err
	}
	newReq := *req
	newReq.Body = &readTrackingBody{ReadCloser: body}
	return &newReq, nil
}
```

主な処理の流れとしては、`Request.Body`を閉じた後に`Request.GetBody`から`Requst.Body`のコピーを取り出して新しい`Request`にセットしています。`Request.GetBody`は`Request.Body`を返す関数で、リクエスト作成時（`NewRequestWithContext`）に`Request.Body`の具象型が`*bytes.Buffer`, `*bytes.Reader`, `*strings.Reader`の場合はセットされます。

https://github.com/golang/go/blob/go1.16/src/net/http/request.go#L183-L189
```go
type Request struct {
	...
	// GetBody defines an optional func to return a new copy of
	// Body. It is used for client requests when a redirect requires
	// reading the body more than once. Use of GetBody still
	// requires setting Body.
	//
	// For server requests, it is unused.
	GetBody func() (io.ReadCloser, error)

	...
}
```

https://github.com/golang/go/blob/go1.16/src/net/http/request.go#L887-L929
```go
func NewRequestWithContext(ctx context.Context, method, url string, body io.Reader) (*Request, error) {
	...
	req := &Request{
		...
	}
	if body != nil {
		switch v := body.(type) {
		case *bytes.Buffer:
			...
			req.GetBody = func() (io.ReadCloser, error) {
				r := bytes.NewReader(buf)
				return io.NopCloser(r), nil
			}
		case *bytes.Reader:
			...
			req.GetBody = func() (io.ReadCloser, error) {
				r := snapshot
				return io.NopCloser(&r), nil
			}
		case *strings.Reader:
			...
			req.GetBody = func() (io.ReadCloser, error) {
				r := snapshot
				return io.NopCloser(&r), nil
			}
		default:
			...
		}
		...
	}
	return req, nil
}
```

## 実装例
これらを参考に以下のように実装することができます。`Trasport.roundTrip`の実装と異なり、`Request.GetBody`が無い場合は、元の内容を読み取って返す関数をセットしてあげます。[`io.NopCloser`](https://golang.org/pkg/io/#NopCloser)を使うことで、何もしない`io.Closer`インターフェースを実装することができます。

```go
func rewindBody(req *http.Request) (*http.Request, error) {
	if req.Body == nil || req.Body == http.NoBody {
		return req, nil
	}
	if req.GetBody == nil {
		buf, err := io.ReadAll(req.Body)
		if err != nil {
			return nil, err
		}
		req.GetBody = func() (io.ReadCloser, error) {
			return io.NopCloser(bytes.NewReader(buf)), nil
		}
	}
	if err := req.Body.Close(); err != nil {
		return nil, err
	}
	body, err := req.GetBody()
	if err != nil {
		return nil, err
	}
	newReq := *req
	newReq.Body = body
	return &newReq, nil
}
```

これで、リトライ処理は以下のように変更されました。
```go
func (c *Client) Do(req *http.Request) (*http.Response, error) {
	var attemptNum int
	for {
		attemptNum++
		req, err := rewindBody(req)
		if err != nil {
			return nil, err
		}
		res, err := c.HTTPClient.Do(req)
		...
	}
}
```

# context.Contextの終了を確認する
`context.Context`はgoroutine間でタイムアウトやキャンセルを伝播させる仕組みです。`Request`も内部で`Context`を保持します。リトライに関わらずループ処理をする際は、呼び出し元で`Context`がキャンセルされている場合があるので確認する必要があります。そうしないと、呼び出し元のgoroutineが終了しているにも関わらず処理が継続することになるためです。

先ほどの、`Trasport.roundTrip`のリクエスト時も同様に`Context`の終了を確認しています。
https://github.com/golang/go/blob/go1.16/src/net/http/transport.go#L561-L563

以下のように実装を追加します。

```go
func (c *Client) Do(req *http.Request) (*http.Response, error) {
	var attemptNum int
	ctx := req.Context()

	for {
		...
 
		wait := c.Backoff(attemptNum)
		select {
		case <-ctx.Done():
			return nil, ctx.Err()
		case <- time.After(wait):
		}
	}
}
```

# http.Response.Bodyを読み切ってから閉じる
レスポンスの内容は`io.ReadCloser`インターフェースである`Response.Body`から読み取れます。しかし、呼び出し側で最後まで読み取ってから`Close`する必要があります。なぜなら、そうしないとkeep-aliveのTCPコネクションが再利用されないからです。
https://golang.org/pkg/net/http/#Response
> It is the caller's responsibility to close Body. The default HTTP client's Transport may not reuse HTTP/1.x "keep-alive" TCP connections if the Body is not read to completion and closed.

## http.Response.Bodyが読み取られた時の挙動
より詳しく理解するためにnet/httpパッケージの実装を確認しました。`http.DefaultClient.Do`を呼び出してリクエストすると、内部では`persistConn`構造体のメソッドが三つのgoroutineに分かれて実行されます。`persistConn`は`net.Conn`を包んでおり、逆に`Transport`構造体の内部でTCPコネクションを抽象化したデータとして管理されています。

https://github.com/golang/go/blob/go1.16/src/net/http/transport.go#L2524
https://github.com/golang/go/blob/go1.16/src/net/http/transport.go#L2048
https://github.com/golang/go/blob/go1.16/src/net/http/transport.go#L2379

![](https://storage.googleapis.com/zenn-user-upload/r41r4xi494iyzgul9eti7crfl1i3)

`readLoop`では実際のレスポンスの内容を、`bodyEOFSignal`構造体の`body`フィールドにセットして、`Response.Body`として渡しています。
この`bodyEOFSignal`構造体の`fn`フィールドの関数が呼ばれると、`waitForBodyRead`チャンネルにエラーが送信されます。そして、そのエラーが`io.EOF`だった場合は最後のselect文で`tryPutIdleConn`が呼び出されます。この`tryPutIdleConn`は`Transport`内部でTCPコネクションを再利用するための関数になります。

```go
func (pc *persistConn) readLoop() {
	...

	alive := true
	for alive {
		...
   
		rc := <-pc.reqch

		var resp *Response
		if err == nil {
			resp, err = pc.readResponse(rc, trace)
		} else {
			...
		}
		...
 
		waitForBodyRead := make(chan bool, 2)
		body := &bodyEOFSignal{
			body: resp.Body,
			...
			fn: func(err error) error {
				isEOF := err == io.EOF
				waitForBodyRead <- isEOF
				...
				return err
			},
		}

		resp.Body = body
		...
  
		select {
		case rc.ch <- responseAndError{res: resp}:
		case <-rc.callerGone:
			return
		}

		...
		select {
		case bodyEOF := <-waitForBodyRead:
			...
			alive = alive &&
				bodyEOF &&
				!pc.sawEOF &&
				pc.wroteRequest() &&
				replaced && tryPutIdleConn(trace)
			...
		}
		...
	}
}
```

そして、`bodyEOFSignal`構造体の`fn`フィールドの関数は`Response.Body.Read`が呼び出されると読み取り結果のエラーを引数に呼び出されます。
https://github.com/golang/go/blob/go1.16/src/net/http/transport.go#L2764-L2771
```go
func (es *bodyEOFSignal) Read(p []byte) (n int, err error) {
	...
	n, err = es.body.Read(p)
	if err != nil {
		...
		err = es.condfn(err)
	}
	return
}

func (es *bodyEOFSignal) condfn(err error) error {
	...
	err = es.fn(err)
	...
}
```

また、`Respose.Body.Close`が呼び出された場合もその際のエラーが`fn`フィールドの関数に渡されます。したがって、`Response.Body.Close`する前に`Response.Body.Read`で`io.EOF`が返されると、`readLoop`でTCPコネクションが再利用されるという仕組みになっています。そして、`Respose.Body.Close`が実行されないと、こちら側で明示的にTCPコネクションを閉じることができないため、余計にファイルディスクリプタを利用することになります。

## 実装例
これらを踏まえて、以下のようにリトライ処理の実装ではリトライ実行前に`Response.Body`を必ず読み切ってから閉じる必要があります。

```go
func drainBody(body io.ReadCloser) {
	io.Copy(io.Discard, body)
	body.Close()
}

func (c *Client) Do(req *http.Request) (*http.Response, error) {
	...
	for {
		...
		res, err := c.HTTPClient.Do(req)
		...
		drainBody(res.Body)
	}
}
```

ちなみに`io.Copy`は内部で読み取り時に`io.EOF`が返されても、エラーを返さない（`err == nil`）ようになっています。
https://golang.org/pkg/io/#Copy
> A successful Copy returns err == nil, not err == EOF. Because Copy is defined to read from src until EOF, it does not treat an EOF from Read as an error to be reported.

# 要件に応じてコネクションプールを管理する
## TransportでのTCPコネクションの管理
`Transport`の内部では、`connectMethodKey`（リクエストの宛先）単位でidle状態のTCPコネクションや確立待ちのキューが管理されています。
https://github.com/golang/go/blob/go1.16/src/net/http/transport.go#L1846-L1861
```go
type Transport struct {
	...
	idleConn     map[connectMethodKey][]*persistConn // most recently used at end
	idleConnWait map[connectMethodKey]wantConnQueue  // waiting getConns
	...
}

type connectMethodKey struct {
	proxy, scheme, addr string
	onlyH1              bool
}
```

特に指定することなく`Client`を使った場合は、内部の`Transport`はグローバル変数である`DefaultTransport`が利用されます。
https://golang.org/pkg/net/http/#Client
> A Client is an HTTP client. Its zero value (DefaultClient) is a usable client that uses DefaultTransport.

https://github.com/golang/go/blob/go1.16/src/net/http/transport.go#L37-L53
```go
var DefaultTransport RoundTripper = &Transport{
	Proxy: ProxyFromEnvironment,
	DialContext: (&net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
	}).DialContext,
	ForceAttemptHTTP2:     true,
	MaxIdleConns:          100,
	IdleConnTimeout:       90 * time.Second,
	TLSHandshakeTimeout:   10 * time.Second,
	ExpectContinueTimeout: 1 * time.Second,
}
```

上にあるようにデフォルトでは、`IdleConnTimeout`が90秒で指定されており、これはidle状態で保持されるTCPコネクションの時間です。つまり、90秒間はリクエストが終わってからもTCPコネクションは確保されることになります。もちろん無制限ではなく、`MaxIdleConns`や`MaxIdleConnsPerHost`を超えないようにコネクションプールの数は制限されます。デフォルトの`MaxIdleConnsPerHost`は`DefaultMaxIdleConnsPerHost`の２です。
https://github.com/golang/go/blob/go1.16/src/net/http/transport.go#L191-L198
```go
type Transport struct {
	...
	// MaxIdleConns controls the maximum number of idle (keep-alive)
	// connections across all hosts. Zero means no limit.
	MaxIdleConns int
  
	// MaxIdleConnsPerHost, if non-zero, controls the maximum idle
	// (keep-alive) connections to keep per-host. If zero,
	// DefaultMaxIdleConnsPerHost is used.
	MaxIdleConnsPerHost int
	...
}
```

そして、コネクションプールは`CloseIdleConnections`で一括で閉じることができます。
https://golang.org/pkg/net/http/#Client.CloseIdleConnections
> CloseIdleConnections closes any connections on its Transport which were previously connected from previous requests but are now sitting idle in a "keep-alive" state. It does not interrupt any connections currently in use.

## リトライ処理におけるTransportの選択

これらの`Transport`におけるTCPコネクションの管理から考えると、専用に`Transport`を用意すべきか、グローバルな`Transport`を利用すべきか判断する必要があります。

例えば、短時間で何度も同じ宛先にリクエストする場合はデフォルトの`MaxIdleConnsPerHost`（同一ホストあたりidle状態で確保するコネクションの最大数:２）が少ない場合があります。その場合、専用の`Trasport`を用意して`MaxIdleConnsPerHost`を変えてあげることで、再利用しやすくできます。そして、リトライ終了後にその`Transport`が利用されないのであれば、`IdleConnTimeout`の間だけ無駄にTCPコネクションが確保されているのでリソースを消費した状態になります。この場合はリトライ処理の終了後に`CloseIdleConnections`を呼び出すことですぐ閉じることができます。

リトライ機能を提供するパッケージである[go-retryablehttp](https://github.com/hashicorp/go-retryablehttp)は`MaxIdleConnsPerHost`を増やした専用の`Transport`を使ってリトライ処理直後に閉じるような実装になっていました。

実際に、`DefalutlClient`（`MaxIdleConnsPerHost`: 2）を使って一つのgoroutineのループ内で1秒間隔で30回リクエストした結果を確認しました。
まず、TCPコネクションの確立時のハンドシェイクは最初だけであり、一つを使い回せていることが分かります。

![](https://storage.googleapis.com/zenn-user-upload/i459rkrwtx6622ql3b98k7v5yy2e)

このことから、一つのループ内で適度なバックオフでリトライする場合、`MaxIdleConnsPerHost`がデフォルト値でも十分だということが分かります。

また、リクエスト終了後もkeep-aliveでTCPコネクションが確立されていることが分かります。keep-aliveの確認は15秒間隔でされ、90秒後に終了しています。これは、`IdleConnTimeout`のデフォルト値と一致します。

![](https://storage.googleapis.com/zenn-user-upload/9q855fdvdrlxfozzlxayg0ln09kg)

一方で、最後に`CloseIdleConnections`を呼び出すようにすると、即座に終了していることが分かります。

![](https://storage.googleapis.com/zenn-user-upload/vfv5s4e3ubsdrtr0sieftqpy49t7)

このことから、リトライ処理後に確保したコネクションをすぐに解放したい場合は、専用の`Transport`を利用する必要も想定できます。

