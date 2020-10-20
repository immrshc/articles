---
title: "ソースコードを読んでcontextを理解する"
emoji: "🧊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

# 概要

contextパッケージは、生成したgoroutineの実行をキャンセルし、リソースを解放するための仕組みを提供しています。また、リクエストスコープの値を保持させることもできます。

ここでは、contextパッケージのソースコードから、どのようにgoroutineの実行がキャンセルされるかを見ていきます。

具体的には、基本的な以下のような使い方をした場合に何が行われているのかを確認していきます。

```go
import (
	"context"
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	wg.Add(1)

	ctx, cancel := context.WithCancel(context.Background())

	go func(ctx context.Context) {
		select {
		case <-ctx.Done():
			fmt.Println("----done----")
			wg.Done()
			return
		}
	}(ctx)

	cancel()
	wg.Wait()
}
// $ go run context.go 
// ----done----
```

これは、goroutineの生成側で`cancel`を実行し、`ctx.Done()`が返すchannelをcloseしています。そうすることで、実行中のgoroutineでそのchannelから受信することができ、selectを抜けます。なぜなら、閉じられたchannelからはゼロ値を受信することができるためです。

```go
func main() {
	ch := make(chan struct{})
	close(ch)
	fmt.Println(<-ch)
}
// $ go run context.go 
// {}
```

# contextパッケージを読む
## 派生したContextを作成する

まず、`Context`は[インターフェース](https://golang.org/pkg/context/#Context)です。そしてデフォルトの`Context`は以下の二つが用意されています。これらは、キャンセルすることもDeadlineを指定することもできません。主にmain関数から渡される最初の`Context`として利用されます。

https://github.com/golang/go/blob/release-branch.go1.15/src/context/context.go#L199-L218
```go
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

func Background() Context {
	return background
}

func TODO() Context {
	return todo
}
```

これをキャンセル可能にするには、[`context.WithCancel`](https://golang.org/pkg/context/#WithCancel)に`Context`を渡します。

https://github.com/golang/go/blob/release-branch.go1.15/src/context/context.go#L232-L239
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```

内部ではまず、`newCancelCtx`で`cancelCtx`構造体を作ります。これは元の`Context`が埋め込まれます。

https://github.com/golang/go/blob/release-branch.go1.15/src/context/context.go#L242-L244
```go
func newCancelCtx(parent Context) cancelCtx {
	return cancelCtx{Context: parent}
}
```

この`cancelCtx`構造体は以下のように、`children`フィールドを持ち、キャンセル用のインターフェースを持つ`Context`（`canceler`）をmapで保持しています。これは後で確認するように、派生した`Context`を表現するために用いられます。

https://github.com/golang/go/blob/release-branch.go1.15/src/context/context.go#L344-L351
```go
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```

そして次に、`propagateCancel`に元の`Context`と`cancelCtx`が渡されます。

https://github.com/golang/go/blob/release-branch.go1.15/src/context/context.go#L232-L239
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	...
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	...
}
```

以下のように、`propagateCancel`は親の`Context`が`*cancelCtx`型であるか否かで処理が分かれます。

- `Context`が`*cancelCtx`型の場合は、その`children`に子を登録する
- `Context`が`*cancelCtx`型でないの場合は、終了すると子をキャンセルするgoroutineを起動する

どちらにせよ、親の`Context`が終了すると、子の`Context`が終了できるような準備をしています。

https://github.com/golang/go/blob/release-branch.go1.15/src/context/context.go#L250-L286
```go
func propagateCancel(parent Context, child canceler) {
	...

	// Contextインターフェースのparentを*cancelCtx型にキャストする
	if p, ok := parentCancelCtx(parent); ok {
		p.mu.Lock()

		...
		// *cancelCtxの場合はchildrenにchildを登録する
		p.children[child] = struct{}{}
		...

		p.mu.Unlock()
	} else {
		// *cancelCtxでない場合はparentのchannelがcloseされるとchildをcancelするgoroutineを起動する
		atomic.AddInt32(&goroutines, +1)
		go func() {
			select {
			case <-parent.Done():
				child.cancel(false, parent.Err())
			case <-child.Done():
			}
		}()
	}
}
```

最後に、生成した`Context`（実際は`*cancelCtx`型）と、それをキャンセルする関数を返します。

https://github.com/golang/go/blob/release-branch.go1.15/src/context/context.go#L232-L239
```go
// Canceled is the error returned by Context.Err when the context is canceled.
var Canceled = errors.New("context canceled")

func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	...
	c := newCancelCtx(parent)
	...
	return &c, func() { c.cancel(true, Canceled) }
}
```

## Contextをキャンセルする

次に先ほどの、`cancelCtx.cancel`が呼び出された場合を見ていきます。

https://github.com/golang/go/blob/release-branch.go1.15/src/context/context.go#L394-L419
```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	...
	c.mu.Lock()
	...

	// 通信用channelを閉じる
	close(c.done)

	// 子のcancelCtxを全てキャンセルする
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```

内部ではまず、自身の終了を告げるchannelをcloseします。そうすることで、この`Context`に対して`ctx.Done()`から受信することができるようになります。

また、この`Context`だけなく`children`に格納されている`Context`を再起的にキャンセルしていきます。こうすることで、以下のような親の`Context`（`c1`）から何度も派生した`Context`（`c2`, `c3`）も、親が終了すると終了できるようになります。

```go
import (
	"context"
	"fmt"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	wg.Add(1)

	c1, can1 := context.WithCancel(context.Background())

	go func(ctx context.Context) {
		c2, _ := context.WithCancel(ctx)

		go func(ctx context.Context) {
			c3, _ := context.WithCancel(ctx)
			select {
			case <-c3.Done():
				fmt.Println("----done----")
				wg.Done()
				return
			}
		}(c2)
	}(c1)

	can1()
	wg.Wait()
}
```

そして最後に`removeFromParent`がtrueの場合は、`removeChild`を呼び出し親の`Context`が`*cancelCtx`型の場合は、`children`から自身を除きます。

https://github.com/golang/go/blob/release-branch.go1.15/src/context/context.go#L394-L419
```go
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	...

	if removeFromParent {
		// newCancelCtxでは、親のContextはcancelCtx.Contextフィールドに格納されている
		removeChild(c.Context, c)
	}
}
```

https://github.com/golang/go/blob/release-branch.go1.15/src/context/context.go#L316-L326
```go
func removeChild(parent Context, child canceler) {
	p, ok := parentCancelCtx(parent)
	if !ok {
		return
	}
	p.mu.Lock()
	if p.children != nil {
		delete(p.children, child)
	}
	p.mu.Unlock()
}
```

## context.WithDeadlineの場合  

`context.WithDeadline`も基本的に`context.WithCancel`と同じことをやっています。
違いとしては、[`time.AfterFunc`](https://golang.org/pkg/time/#AfterFunc)を利用して指定時間を過ぎるとキャンセルするようにしている点くらいです。

https://github.com/golang/go/blob/release-branch.go1.15/src/context/context.go#L430-L456
```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
	...
	c := &timerCtx{
		cancelCtx: newCancelCtx(parent),
		deadline:  d,
	}
	...

	c.timer = time.AfterFunc(dur, func() {
		c.cancel(true, DeadlineExceeded)
	})

	...
	return c, func() { c.cancel(true, Canceled) }
}
```

# 分かったこと
## 親から子へ再起的にキャンセルされる

既に見てきたように、親の`Context`がキャンセルされると、派生した子である`Context`は再起的にキャンセルされます。逆に、キャンセルされた`Context`の派生元である親や親から派生した他の`Context`はキャンセルされません。つまりA1がキャンセルされてもAはキャンセルされず、したがって他のA2, A3はキャンセルされません。キャンセルする場合にはどの階層の`Context`に対応したキャンセルなのかを意識する必要があります。

```
ctx A
 ├ ctx A1 <- cancel
 ├ ctx A2
 └ ctx A3
```
 
## Context Leak

既に見たように、使われなくなった`Context`をキャンセルしないと、`removeChild`が呼ばれずに親の`Context`に残り続けることになります。したがって、余分にメモリを利用した状態になってしまいます。

