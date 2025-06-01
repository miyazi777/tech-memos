```embed
title: "GitHub - ovechkin-dm/go-dyno: Dynamic proxy for golang"
image: "https://opengraph.githubassets.com/1f20503a970c80335359446ac8f23275ccbd51e8183858e1286f9c94d2bfe06f/ovechkin-dm/go-dyno"
description: "Dynamic proxy for golang. Contribute to ovechkin-dm/go-dyno development by creating an account on GitHub."
url: "https://github.com/ovechkin-dm/go-dyno"
aspectRatio: "50"
```

# 概要
横断的な処理を任意のinterfaceやstructに対して追加し、機能追加することが可能になる。

dynoが提供するproxyにより、Handleメソッド内で定義された内容を実行して、任意のタイミングでオリジナルのメソッドが実行されるようにできる。
イメージとしてはserverのmiddlewareと同じようなことを任意のinterfaceやstructに対して実施できるようになる。

# 使い方
公式のサンプルとほぼ同じだが、少しコメントなどをつけ加えている。

今回の場合であれば、Greet1とGreet2の2つのメソッドが実行される前後で、それぞれ”Before〜"と"After〜"が表示されている。

main.go
```go
package main

import (
	"fmt"
	"reflect"

	"github.com/ovechkin-dm/go-dyno/pkg/dyno"
)

// target interface
type Greeter interface {
	Greet1()
	Greet2()
}

// target
type SimpleGreeter struct{}

func (g *SimpleGreeter) Greet1() {
	fmt.Println("Hello1!")
}

func (g *SimpleGreeter) Greet2() {
	fmt.Println("Hello2!")
}

// proxy
type ProxyHandler[T any] struct {
	Impl T
}

func (p *ProxyHandler[T]) Handle(m reflect.Method, values []reflect.Value) []reflect.Value {
	fmt.Println("Before process: method call:", m.Name)
	ret := reflect.ValueOf(p.Impl).MethodByName(m.Name).Call(values)
	fmt.Println("After process: method called:", m.Name)
	return ret
}

func NewDynamicGreeter() Greeter {
	greeter := &SimpleGreeter{}
	proxyHandler := &ProxyHandler[Greeter]{Impl: greeter}
	dynamicGreeter, _ := dyno.Dynamic[Greeter](proxyHandler.Handle)
	return dynamicGreeter
}

func main() {
	dynamicGreeter := NewDynamicGreeter()
	dynamicGreeter.Greet1()
	dynamicGreeter.Greet2()
}

// 結果
// Before process: method call: Greet1
// Hello1!
// After process: method called: Greet1
// Before process: method call: Greet2
// Hello2!
// After process: method called: Greet2
```


これを使うことで横断的に処理したい内容（例えばロギングとかOpenTelemetolyのスパンをとか）を元の処理には手を加えず、コンストラクタに手を加えるだけで差し挟むことができるようになる。

ただ、メソッド呼び出しだけとはいえ、リフレクションでの呼び出しになるのでその分、パフォーマンスは犠牲になる。
犠牲になるのはどのぐらいか？
調べてみた。

# ベンチマーク
最初のプログラムをさらに改造してベンチマークに適した形にし、ベンチマーク用のコードを追加し、性能を検証してみた。

修正内容は以下のとおり。
* ベンチマークには邪魔になるのでfmt.Printlnでの表示を削除し、Greet関数は文字列を返却するだけに修正
* NewDynamicGreeter関数ではgo-dynoのproxy経由でのインスタンスを提供
* NewStaticGreeter関数ではgo-dynoを経由しないインスタンスを提供
* ベンチマーク用のbenchmark_test.goを追加

main.go
```go
package main

import (
	"reflect"

	"github.com/ovechkin-dm/go-dyno/pkg/dyno"
)

// target interface
type Greeter interface {
	Greet() string
}

// target
type SimpleGreeter struct{}

func (g *SimpleGreeter) Greet() string {
	return "Hello!"
}

// proxy
type ProxyHandler[T any] struct {
	Impl T
}

func (p *ProxyHandler[T]) Handle(m reflect.Method, values []reflect.Value) []reflect.Value {
	return reflect.ValueOf(p.Impl).MethodByName(m.Name).Call(values)
}

func NewDynamicGreeter() Greeter {
	greeter := &SimpleGreeter{}
	proxyHandler := &ProxyHandler[Greeter]{Impl: greeter}
	dynamicGreeter, _ := dyno.Dynamic[Greeter](proxyHandler.Handle)
	return dynamicGreeter
}

func NewStaticGreeter() Greeter {
	return &SimpleGreeter{}
}

func main() {
	dynamicGreeter := NewDynamicGreeter()
	dynamicGreeter.Greet()
}

```

benchmark_test.go
```go
package main

import "testing"

func BenchmarkToDynamicGreeter(b *testing.B) {
	greeter := NewDynamicGreeter()
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		greeter.Greet()
	}
}

func BenchmarkToStaticGreeter(b *testing.B) {
	greeter := NewStaticGreeter()
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		greeter.Greet()
	}
}

```

## ベンチマーク結果
一番、右側が1回呼び出し辺りの処理時間。

go-dynoを使った方を平均650ns、使わない方を平均0.25nsとすると約2600倍、go-dynoを使った方が遅いということになる（雑なベンチだけども・・・）
```
❯ go test -bench=. -benchtime=10000x -count=3
goos: linux
goarch: amd64
pkg: github.com/miyazi777/go-dyno-sample1
cpu: 11th Gen Intel(R) Core(TM) i7-11700K @ 3.60GHz
BenchmarkToDynamicGreeter-16               10000               624.0 ns/op
BenchmarkToDynamicGreeter-16               10000               670.2 ns/op
BenchmarkToDynamicGreeter-16               10000               645.5 ns/op
BenchmarkToStaticGreeter-16                10000                 0.2243 ns/op
BenchmarkToStaticGreeter-16                10000                 0.2302 ns/op
BenchmarkToStaticGreeter-16                10000                 0.2295 ns/op
PASS
ok      github.com/miyazi777/go-dyno-sample1    0.024s

```


go-dynoを使った方が遅いだろうな、というのは予想どおりではあるものの、想像より差があった。

# まとめ
横断的な処理を差し挟みたい場合は以下のように判断して良さそう。

* テストコードでの使用、ツールでの使用、そこまで処理性能を重視していない・上記のようなオーバーヘッドが許容できるのであれば、go-dynoは良い選択肢になると思う
* 処理性能を重視したい場合で横断的な処理を挟みたい場合はgo-wrapなどコード生成で対応するのが良さそう

```embed
title: "GitHub - hexdigest/gowrap: GoWrap is a command line tool for generating decorators for Go interfaces"
image: "https://opengraph.githubassets.com/34e8444a81c68bbde6b2bd4653dd8ed51b963d6466664aa4a812e2cb408002e9/hexdigest/gowrap"
description: "GoWrap is a command line tool for generating decorators for Go interfaces - hexdigest/gowrap"
url: "https://github.com/hexdigest/gowrap"
aspectRatio: "50"
```
