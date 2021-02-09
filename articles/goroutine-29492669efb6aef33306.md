---
title: "原理原則から適切なgoroutineの数を考える"
emoji: "☕️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

# 概要
## 動機

goroutineを使ってパフォーマンスを改善する際に、どれくらの数で並行処理すればいいのか分かりませんでした。そこで、そもそもどのような仕組みなのか調べ、どのような性質の仕事が改善されるのか計測して、適切な数を決めるための観点を整理しました。

## 要約

goroutineはカーネルスレッドとM:Nの関係になっています。そしてカーネルスレッドごとにgoroutineのキューがあり、Goのスケジューラが順次実行していきます。
IO-Boundな処理は、netpollerが別のカーネルスレッドで非同期でシステムコールを実行するので他のgoroutineをブロックしないようになっています。

goroutineの使用時には以下の観点を留意する必要が計測から分かりました。

- goroutineを使う場合はコンテキストスイッチのコストとトレードオフになる
- CPU-Boundなgoroutineは並列処理の恩恵を受ける場合がある
- IO-Boundなgoroutineは並行処理の恩恵を受ける場合がある

環境変数`GODEBUG`を使うことで、Goのスケジューラの状態を確認して仮説を検証することができます。検証すると以下の観点を留意する必要が分かりました。

- IO-Boundなgoroutineの並行数を増やしてもnetpollerのスレッド数がボトルネックになる
- 単純にGOPAXPROCSの値を増やしてもIO-Boundなgoroutineのパフォーマンスが上がるわけではない

# goroutineの仕組み

CPUコアは複数のカーネルスレッド（以降は単にスレッドと表現する）を切り替えて（コンテキストスイッチ）処理を進めます。

一方、いくつかのgoroutineは一つのスレッドとして実行されます。具体的にはGoのスケジューラ（P, Processor）ごとにgoroutine（G）のキューがあり、そこから実行対象のgoroutineを取り出しスレッド（M, Machine）に割り当てます。したがって、Goのruntimeで実行対象のgoroutineを切り替えても、OSからはスレッドのコンテキストスイッチが発生していないように見せることができます。
c.f. https://en.wikipedia.org/wiki/Thread_(computing)#User_threads

そして、PにはLRQ(Local Run Queue)というGのキューが割り当てられ、GoのスケジューラはそこからGをコンテキストスイッチしてMに割り当てます。またGRQ（Global Run Queue）というキューは、LRQが枯渇した際にMにGを提供し、仕事を続けさせるために存在します。

![](https://storage.googleapis.com/zenn-user-upload/tp2cxxvurmgdzy0t298159pk0y2f)

上の図のように、Goのプログラムからはスレッドが仮想的なCPUコアとして扱われることが分かります。そして、CPUのコア数に対して複数のスレッドが対応するように、スレッドとgoroutineもM:Nの関係になります。

# IO-Boundなgoroutineの扱い

goroutineの仕事にはCPU-BoundとIO-Boundの二種類があります。

CPU-Boundな仕事は待機状態にならず一定の処理を続けることができます。IO-Boundな仕事はネットワークごしの通信やシステムコールなどを伴い待機状態になることがあります。

IO-Boundなgoroutineは待機状態になった時に別の実行可能状態なgoroutineに切り替えること（コンテキストスイッチ）でスレッドの待ち時間を減らせます。つまり、待機状態のGがMをブロックすることを回避できます。逆にCPU-Boundなgoroutineはそれだけで処理を継続することができるので、コンテキストスイッチをしてもオーバーヘッドだけ発生することになります。

Goのスケジューラは以下の実行時にコンテキストスイッチを行うべきか判断します。

- goroutineの作成
- ガーベージコレクション
- システムコール
- goroutineの同期

ネットワーク通信を伴うシステムコールはnetpollerという仕組みで別のスレッドで非同期に処理を進めることができます。具体的には、goroutineはノンブロッキングモードでファイルディスクリプタに書き込み、準備ができていない場合はエラーが返ります。そして、netpollerを呼び出し、netpollerはI/O処理が可能になったタイミングで元のgoroutineに通知し、実行を再開します。そうすることで、netpollerはgoroutineに非同期I/Oのような仕組みを提供します。

https://morsmachine.dk/netpoller
> Whenever a goroutine tries to read or write to a connection, the networking code will do the operation until it receives such an error, then call into the netpoller, telling it to notify the goroutine when it is ready to perform I/O again. 

![](https://storage.googleapis.com/zenn-user-upload/zg5i1i51yw5lo8avk5f3iw4szbqo)

そして、netpollerで実行を終えたGは元のLRQに戻されます。

netpollerで非同期に処理できない場合は、そのGがMをブロックすることになるので、Goのスケジューラは別のMを残りのLRQのために割り当てます。

![](https://storage.googleapis.com/zenn-user-upload/406s4y6oiac9brlz2qg37wxnrg9y)

# パフォーマンスの計測
## CPU-Boundなgoroutineの場合

まずCPU-Boundなgoroutineのパフォーマンスを計測します。
CPU-Boundな仕事として単純にスライスの数字を合計するプログラムを用意します。

```go
// 受け取ったスライスの数字を合計する
func Sum(nums []int) int {
	var s int
	for _, n := range nums {
		s += n
	}
	return s
}

// 受け取ったスライスの数字を指定の数のgoroutineごとに計算して合計する
func SumConcurrently(nums []int, cncrtNum int) int {
	totalNum := len(nums)
	numPerGrtne := totalNum / cncrtNum

	var s int64
	var wg sync.WaitGroup
	for i := 0; i < cncrtNum; i++ {
		wg.Add(1)
		go func(i int) {
			defer wg.Done()
			start := i * numPerGrtne
			end := start + numPerGrtne
			if i == cncrtNum-1 {
				end = totalNum
			}
			w := nums[start:end]
			atomic.AddInt64(&s, int64(Sum(w)))
		}(i)
	}
	wg.Wait()
	return int(s)
}
```

`go test`コマンドの`-cpu`フラグで環境変数`GOMAXPROCS`に値をセットすることができます。
この環境変数はユーザーレベルのGoのコードを実行するスレッド数を指定することができます。具体的にはPの数を指定することができます。そして、netpollerのスレッド数は影響を受けません。

https://golang.org/pkg/runtime/
> The GOMAXPROCS variable limits the number of operating system threads that can execute user-level Go code simultaneously. There is no limit to the number of threads that can be blocked in system calls on behalf of Go code; those do not count against the GOMAXPROCS limit. 

したがって、`-cpu=1`にすることでCPU-Boundなgoroutineを並行に処理できます。
長さが100万のスライスを渡して、実行した結果が以下になります。

```go
$ go test -bench=. -cpu=1 ./tmp/grtne
BenchmarkSum                        1888            555984 ns/op
BenchmarkSumConcurrently            1974            562925 ns/op
```

スレッド数が限られている場合は並行処理をしてもパフォーマンスが改善しないことが分かります。

次にPを2, 4, 8にして並列で実行した結果が以下になります。

```go
$ go test -bench=. -cpu=2,4,8 ./tmp/grtne
BenchmarkSum-2                      1890            564212 ns/op
BenchmarkSum-4                      1998            533496 ns/op
BenchmarkSum-8                      2098            529612 ns/op
BenchmarkSumConcurrently-2          3339            328858 ns/op
BenchmarkSumConcurrently-4          8176            145778 ns/op
BenchmarkSumConcurrently-8          9716            120419 ns/op
```

並列で実行するとパフォーマンスが改善されました。

しかし、並列で実行したからといって必ずしもパフォーマンスが改善されるとは限りません。
スライスの長さを1000にして並列で実行した結果が以下です。

```go
$ go test -bench=. -cpu=2,4,8 ./tmp/grtne
BenchmarkSum-2                   2883750               413 ns/op
BenchmarkSum-4                   2944737               407 ns/op
BenchmarkSum-8                   2934186               400 ns/op
BenchmarkSumConcurrently-2        389985              3144 ns/op
BenchmarkSumConcurrently-4        333583              3718 ns/op
BenchmarkSumConcurrently-8        331477              3589 ns/op
```

これは、並列処理による仕事の分担による恩恵よりもスレッドのコンテキストスイッチのオーバーヘッドの方が大きくなるためかと思われます。

## IO-Boundなgoroutineの場合

今度はIO-Boundなgoroutineのパフォーマンスを計測します。
IO-Boundな仕事としてHTTPリクエストをするプログラムを用意します。

```go
func request() {
	res, err := http.Get("http://www.google.com/robots.txt")
	if err != nil {
		log.Fatal(err)
	}
	if _, err := ioutil.ReadAll(res.Body); err != nil {
		log.Fatal(err)
	}
	if err = res.Body.Close(); err != nil {
		log.Fatal(err)
	}
}

// 指定数だけ一つのgoroutineでリクエストする
func Do(num int) {
	for i := 0; i < num; i++ {
		request()
	}
}

// 指定数のgoroutineで一回ずつリクエストする
func DoConcurrently(cncrtNum int) {
	var wg sync.WaitGroup
	for i := 0; i < cncrtNum; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			request()
		}()
	}
	wg.Wait()
}
```

goroutineの数を20として並行処理した結果が以下になります。

```go
$ go test -bench=. -cpu=1,2,4,8 ./tmp/grtne
BenchmarkDo                            2         848504372 ns/op
BenchmarkDo-2                          2         853469266 ns/op
BenchmarkDo-4                          2         853167418 ns/op
BenchmarkDo-8                          2         842226196 ns/op
BenchmarkDoConcurrently               19          60890016 ns/op
BenchmarkDoConcurrently-2             19          63518738 ns/op
BenchmarkDoConcurrently-4             19          64801823 ns/op
BenchmarkDoConcurrently-8             16          67286300 ns/op
```

上述の通り`-cpu`フラグでnetpollerの数は制御できないので、`-cpu`フラグの値（Pの数）による変化はほぼありませんでした。このことから、netpollerのスレッドの並列数がIO-Boundな処理に大きく影響を与えることが考えられます。

そして、`BenchmarkDo`と`BenchmarkDoConcurrently`を比べるとgoroutineを使った並行処理によりパフォーマンスが改善したことが分かります。

![](https://storage.googleapis.com/zenn-user-upload/lbh73matcrx11pxjo8kb577e043h)
上の図のように、並行処理されるgoroutineの数が100くらいになるまでリクエストあたりの時間は短くなりました。しかし、それを超えると速度は改善しませんでした。おそらくgoroutineやnetpollerのスレッドのコンテキストスイッチのオーバーヘッドが大きくなったためかと思われます。

## 計測から分かること

これまでの計測から分かることは以下になります。

- goroutineを使う場合はコンテキストスイッチのコストとトレードオフになる
- CPU-Boundなgoroutineは並列処理の恩恵を受ける場合がある
- IO-Boundなgoroutineは並行処理の恩恵を受ける場合がある

# スケジューラの挙動

環境変数`GODEBUG`に名前と値をセットするとこと様々なデバッグ情報を出力できます。
今回は`schedtrace`を使って、指定ミリ秒ごとにスケジューラの状態を出力します。また、`scheddetail`と併用するとP, M, Gの状態も詳しく見れます。

https://golang.org/pkg/runtime/
> scheddetail: setting schedtrace=X and scheddetail=1 causes the scheduler to emit detailed multiline info every X milliseconds, describing state of the scheduler, processors, threads and goroutines.
> schedtrace: setting schedtrace=X causes the scheduler to emit a single line to standard error every X milliseconds, summarizing the scheduler state.

`GOPAXPROCS=1`で`DoConcurrently`を実行した結果が以下になります。

```go
// DoConcurrently(20)
$ GOMAXPROCS=1 GODEBUG=schedtrace=50 go run tmp/grtne/grtne.go
SCHED 0ms: gomaxprocs=1 idleprocs=0 threads=4 spinningthreads=0 idlethreads=1 runqueue=0 [0]
SCHED 51ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=2 [16]
SCHED 105ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=1 [13]
SCHED 161ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=0 [1]
SCHED 220ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=1 [9]
SCHED 277ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=1 [9]
SCHED 328ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=2 [7]
// DoConcurrently(50)
$ GOMAXPROCS=1 GODEBUG=schedtrace=50 go run tmp/grtne/grtne.go
SCHED 0ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=0 [0]
SCHED 50ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=9 [2]
SCHED 110ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=2 [8]
SCHED 162ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=1 [2]
SCHED 218ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=1 [9]
SCHED 275ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=1 [7]
SCHED 332ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=2 [7]
// DoConcurrently(100)
$ GOMAXPROCS=1 GODEBUG=schedtrace=50 go run tmp/grtne/grtne.go
SCHED 0ms: gomaxprocs=1 idleprocs=0 threads=4 spinningthreads=0 idlethreads=1 runqueue=0 [0]
SCHED 51ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=2 [14]
SCHED 106ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=2 [14]
SCHED 163ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=1 [1]
SCHED 222ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=3 [8]
SCHED 278ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=3 [6]
SCHED 334ms: gomaxprocs=1 idleprocs=0 threads=5 spinningthreads=0 idlethreads=1 runqueue=1 [1]
```

各実行時ごとの項目の意味は以下になります。

- idleprocs: idle状態なProcessor（P）の数
- threads: runtimeが実行できるスレッド数
- spinningthreads: 実行できるgoroutineを見つけられないスレッド数
- idlethreads: idle状態なスレッド数
- runqueue: GRQに積まれたgoroutineの数
- \[n, m, ...]: 各LRQに積まれたgoroutineの数

確かに`GOMAXPROCS=1`によりLRQの数は一つになっているのが分かります。

また、goroutineの数を増やしても、全体のスレッド数（threads）は変わっていないので、goroutineの数がnetpollerのスレッド数に影響を与えないと考えられます。したがって、IO-Boundなgoroutineの並行数を増やしてもパフォーマンスに効果があるのは限度があることが推測されます。また先ほどgoroutineの数を増やし過ぎてパフォーマンスが下がったのは、スレッドではなくGoのスケジューラのコンテキストスイッチのコストに起因すると考えられます。

今度は、`GOMAXPROCS=4`とすると以下のようになりました。

```go
// DoConcurrently(20)
$ GOMAXPROCS=4 GODEBUG=schedtrace=50 go run tmp/grtne/grtne.go
SCHED 0ms: gomaxprocs=4 idleprocs=3 threads=6 spinningthreads=0 idlethreads=3 runqueue=0 [0 0 0 0]
SCHED 57ms: gomaxprocs=4 idleprocs=2 threads=10 spinningthreads=0 idlethreads=4 runqueue=0 [0 0 0 0]
SCHED 114ms: gomaxprocs=4 idleprocs=0 threads=10 spinningthreads=1 idlethreads=3 runqueue=0 [0 0 0 0]
SCHED 170ms: gomaxprocs=4 idleprocs=2 threads=12 spinningthreads=0 idlethreads=6 runqueue=1 [0 0 0 0]
// DoConcurrently(50)
$ GOMAXPROCS=4 GODEBUG=schedtrace=50 go run tmp/grtne/grtne.go
SCHED 0ms: gomaxprocs=4 idleprocs=2 threads=6 spinningthreads=1 idlethreads=2 runqueue=0 [0 0 0 0]
SCHED 54ms: gomaxprocs=4 idleprocs=3 threads=10 spinningthreads=0 idlethreads=5 runqueue=0 [0 0 0 0]
SCHED 113ms: gomaxprocs=4 idleprocs=1 threads=10 spinningthreads=1 idlethreads=4 runqueue=0 [0 0 0 0]
SCHED 169ms: gomaxprocs=4 idleprocs=0 threads=11 spinningthreads=0 idlethreads=3 runqueue=1 [1 2 0 0]
// DoConcurrently(100)
$ GOMAXPROCS=4 GODEBUG=schedtrace=50 go run tmp/grtne/grtne.go
SCHED 0ms: gomaxprocs=4 idleprocs=3 threads=7 spinningthreads=0 idlethreads=3 runqueue=0 [0 0 0 0]
SCHED 59ms: gomaxprocs=4 idleprocs=3 threads=10 spinningthreads=0 idlethreads=5 runqueue=0 [0 0 0 0]
SCHED 116ms: gomaxprocs=4 idleprocs=2 threads=10 spinningthreads=0 idlethreads=4 runqueue=0 [0 0 0 0]
SCHED 170ms: gomaxprocs=4 idleprocs=0 threads=10 spinningthreads=0 idlethreads=3 runqueue=1 [0 0 0 0]
```

GOMAXPROCS(=P)が増え、threads（スレッド数）が増えたことが分かります。
しかし、`threads - GOMAXPROCS`の値はあまり変わらないので、ユーザーレベルのコード以外を実行するスレッドには影響がなさそうだと分かります。つまりGOMAXPROCSの数がnetpollerのスレッド数に影響を与えるとは考えづらいことが分かります。そして、idlethreadsも増えているので、IO-Boundなgoroutineの場合は単純にGOMAXPROCSを増やしてもあまり意味がないことが分かります。

# 参考
- goroutineの仕組みに関して
  - [Scheduling In Go : Part I - OS Scheduler](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)
  - [Scheduling In Go : Part II - Go Scheduler](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html)
  - [Scheduling In Go : Part III - Concurrency](https://www.ardanlabs.com/blog/2018/12/scheduling-in-go-part3.html)
  - [The Go scheduler](https://morsmachine.dk/go-scheduler)
  - [The Go netpoller](https://morsmachine.dk/netpoller)
- スケジューラの挙動に関して
  - [Scheduler Tracing In Go](https://www.ardanlabs.com/blog/2015/02/scheduler-tracing-in-go.html)
  - [Debugging performance issues in Go* programs](https://software.intel.com/content/www/us/en/develop/blogs/debugging-performance-issues-in-go-programs.html)
  - [Go runtime package](https://github.com/golang/go/blob/go1.15.8/src/runtime/proc.go#L54-L63)

