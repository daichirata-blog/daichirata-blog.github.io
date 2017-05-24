---
title: Go Data Race Detectorによる競合検知
---

こんにちは。開発部の平田です。

今回は、Golang に標準で組み込まれているデータ競合の検知の仕組みである race detector と、どのようなケースで検知してくれるのかについて [https://golang.org/doc/articles/race_detector.html](https://golang.org/doc/articles/race_detector.html) を元に簡単に紹介したいと思います。

Golang には goroutine という並行処理のための機構があるので、気軽に goroutine を起動して並行処理を書くことが出来ます。その為、意図しない所でレースコンディションが発生してしまうことが時々あります。極力そういったことが起きないように基本的には channel を使って値をコピーして渡したりするのですが、思わぬ所で発生してしまうデータレースに対して検知する仕組みが標準で組み込まれています。

## 使い方

使い方は、各種コマンドに `-race` flag を追加します。

``` shell
$ go test -race mypkg
$ go run -race mysrc.go
$ go build -race mycmd
$ go install -race mypkg
```

データレースが見つかった場合には、以下のように出力されます。

```
==================
WARNING: DATA RACE
Write at 0x00c420078090 by goroutine 6:
  runtime.mapassign1()
      /usr/local/Cellar/go/1.7.1/libexec/src/runtime/hashmap.go:442 +0x0
  main.main.func1()
      /src/github.com/xxx/xxx/main.go:11 +0x86

Previous write at 0x00c420078090 by main goroutine:
  runtime.mapassign1()
      /usr/local/Cellar/go/1.7.1/libexec/src/runtime/hashmap.go:442 +0x0
  main.main()
      /src/github.com/xxx/xxx/main.go:14 +0x13e

Goroutine 6 (running) created at:
  main.main()
      /src/github.com/xxx/xxx/main.go:13 +0xd4
==================
```

また、幾つかのオプションを環境変数 `GORACE` 経由で渡すことが出来ます。

* log_path (default stderr)
* exitcode (default 66)
* strip_path_prefix (default "")
  * 指定したパスのログを取り除く
* history_size (default 1)
  * groutine のメモリアクセス履歴。32K * 2**history_size
* halt_on_error (default 0)
  * データレースが発見した段階でプログラムを止める

例えばデータレースが見つかった段階で exit code 1 で終了させたい場合には

``` shell
GORACE="exitcode=1,halt_on_error=1" go test -race
```

といったように指定します。

## Data Race

具体的に検知されるデータレースにはどのようなものがあるかを簡単に紹介します。

### Mapへの同時書き込み

``` golang
package main
import (
    "fmt"
)
func main() {
	c := make(chan bool)
	m := make(map[string]string)
	go func() {
		m["first"] = "a" //
		c <- true
	}()
	m["second"] = "b" //
	<-c
	for k, v := range m {
		fmt.Println(k, v)
	}
}
```

この例では、`m` を main と goroutine の両方から排他制御をせずに書き込んでいる為、データレースの警告が表示されます。(このケースでは検出されませんでしたが、go 1.6 からは map への書き込み中に別の groutine からアクセスすると panic してします)

``` shell
$ go run -race main.go
==================
WARNING: DATA RACE
Write at 0x00c420078090 by goroutine 6:
  runtime.mapassign1()
      /usr/local/Cellar/go/1.7.1/libexec/src/runtime/hashmap.go:442 +0x0
  main.main.func1()
      /src/github.com/xxx/main.go:11 +0x86

Previous write at 0x00c420078090 by main goroutine:
  runtime.mapassign1()
      /usr/local/Cellar/go/1.7.1/libexec/src/runtime/hashmap.go:442 +0x0
  main.main()
      /src/github.com/xxx/main.go:14 +0x13e

Goroutine 6 (running) created at:
  main.main()
      /src/github.com/xxx/main.go:13 +0xd4
==================
2 b
1 a
Found 1 data race(s)
exit status 66
```

これを対応するためには、書き込みの前後を排他的に制御する必要があります。実際にこの様に対応するとは無いかもしれませんが、今回は素直に前後にロックを取得するようにします。

``` golang
var mu sync.Mutex
go func() {
	mu.Lock()
	m["first"] = "a"
	mu.Unlock()
	c <- true
}()
mu.Lock()
m["second"] = "b"
mu.Unlock()
```

これで、race flag 付きの実行でも警告が出なくなります。注意する点として、map は実体への参照を保持する構造体なので、関数へ値渡しするだけでは対応できません。

``` golang
go func(m map[string]string) {
	m["1"] = "a" // これでは対応できない
	c <- true
}(m)
```

## ループカウンター

``` golang
package main
import (
    "sync"
    "fmt"
)
func main() {
	var wg sync.WaitGroup
	wg.Add(5)
	for i := 0; i < 5; i++ {
		go func() {
			fmt.Println(i)
			wg.Done()
		}()
	}
	wg.Wait()
}
```

`i` がループカウンタ用の変数として定義されていますが、その値を goroutine から読み込んでいます。本来であれば、 01234 という出力を期待していますが、実際に実行してみるとそのようにはならないことがわかると思います。

``` shell
$ go run -race main.go
==================
WARNING: DATA RACE
Read at 0x00c420074188 by goroutine 6:
  main.main.func1()
      /src/github.com/xxx/main.go:31 +0x3f

Previous write at 0x00c420074188 by main goroutine:
  main.main()
      /src/github.com/xxx/main.go:29 +0x120

Goroutine 6 (running) created at:
  main.main()
      /src/github.com/xxx/main.go:33 +0xf6
==================
1
2
3
4
5
Found 1 data race(s)
exit status 66
```

今回のケースでは、`i` が int で primitive な型なので、関数の引数として値コピーしてあげることで対応できます。

``` golang
go func(i int) {
	fmt.Println(i)
	wg.Done()
}(i)
```

## グローバルな変数への書き込み

``` golang
var types map[string]int

func AddType(t string, i int) {
	types[t] = i
}

func GetType(t string) int {
	return types[t]
}
```

この場合も最初の例と同様に、map の前後を排他的に制御する必要があります。

``` golang
var (
	types map[string]int
	mu    sync.Mutex
)

func AddType(t string, i int) {
	mu.Lock()
	defer mu.Unlock()
	types[t] = i
}

func GetType(t string) int {
	mu.Lock()
	defer mu.Unlock()
	return types[t]
}
```

map 以外のデータ型の場合でも同様に、排他制御を行う必要があります。

## Primitive な値の読み込み

``` golang
package main
import (
	"os"
	"time"
)
type Connection struct {
	last int64
}
func (c *Connection) KeepAlive() {
	c.last = time.Now().UnixNano()
}
func (c *Connection) Start() {
	go func() {
		for {
			time.Sleep(time.Second)
			// c.lastの読み込みと、KeepAliveでの書き込みが競合
			if c.last < time.Now().Add(-10*time.Second).UnixNano() {
				os.Exit(1)
			}
		}
	}()
}
```

このケースの場合、`KeepAlive` は別の goroutine から呼び出される想定です。その場合 c.last の読み込みと、KeepAliveによる c.last への書き込みが競合してしまいます。 last は primitive な型なので一見問題無さそうにも思えますが、メモリへのアクセスはアトミックでは無いため、コンパイラの最適化やメモリアクセスの並び替えなどで、デバッグが非常に難しい問題が発生する場合があります。

このケースでも、mutex か channel で対応することが出来ますが、int の場合には `sync/atomic` パッケージでアトミックな操作が行えるので、今回はこちらを使って対応します。

``` golang
func (c *Connection) KeepAlive() {
	atomic.StoreInt64(&c.last, time.Now().UnixNano())
}
func (c *Connection) Start() {
	go func() {
		for {
			time.Sleep(time.Second)
			if atomic.LoadInt64(&c.last) < time.Now().Add(-10*time.Second).UnixNano() {
				os.Exit(1)
			}
		}
	}()
}
```

似たようなケースで、例えば bool によってフラグを管理しているケースなどでも応用できます。

``` golang
var Debug int32
func SetDebug(b bool) {
	if b {
		atomic.StoreInt32(&Debug, 1)
	} else {
		atomic.StoreInt32(&Debug, 0)
	}
}
func IsDebug() bool {
	return atomic.LoadInt32(&Debug) == 1
}
func main() {
	if IsDebug() {
		// ...
	}
}
```

goroutine safe ではない bool を0/1で代用することによって、atomicに操作できます。

## まとめ

上記以外にも、データレースは様々なケースで発生します。どうしてもパフォーマンスがほしい所以外では基本的には channel を使って通信によるメモリの共有を行うのが一番良いとは思いますが、想定外の所で発生してしまうケースはあると思います。レースコンディションを起こさないように正しくプログラミング・レビューを行うことはとても難しいので、こういった仕組みが公式に用意されているのはとても助かります。

もしまだ導入していないのであれば、テストの実行に是非検討してみては如何でしょうか。

