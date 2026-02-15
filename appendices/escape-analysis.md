<div align="right">
<img src="https://img.shields.io/badge/AI-ASSISTED_STUDY-3b82f6?style=for-the-badge&labelColor=1e293b&logo=bookstack&logoColor=white" alt="AI Assisted Study" />
</div>

# なぜ変数がヒープに「逃げる」のか

## はじめに

[04-stack-heap](../04-stack-heap.md) で、Go は<strong>エスケープ解析</strong>によってスタックとヒープを自動的に選択すると学びました

```go
func example() *int {
    x := 10        /* 関数内でのみ使用 → スタック */
    y := x + 10    /* 関数外に出る(エスケープする) → ヒープ */
    return &y      /* y のアドレスを返す */
}
```

しかし、なぜ `y` はヒープに配置されるのでしょうか？

コンパイラはどのような基準で判断しているのでしょうか？

このページでは、エスケープ解析の仕組みと、実際の判断基準を深掘りします

---

## 目次

1. [エスケープとは何か](#エスケープとは何か)
2. [なぜエスケープ解析が必要か](#なぜエスケープ解析が必要か)
3. [エスケープの判断基準](#エスケープの判断基準)
4. [Go のエスケープ解析](#go-のエスケープ解析)
5. [JVM のエスケープ解析](#jvm-のエスケープ解析)
6. [エスケープを避ける書き方](#エスケープを避ける書き方)
7. [用語集](#用語集)
8. [参考資料](#参考資料)

---

## エスケープとは何か

### 定義

変数が<strong>エスケープする</strong>とは、その変数が定義されたスコープを超えて参照される可能性があることを意味します

```go
func escape() *int {
    x := 42
    return &x  /* x のアドレスを返す → エスケープ */
}
```

この例では、`x` は関数 `escape` 内で定義されていますが、そのアドレスが戻り値として返されます

関数が終了した後も `x` にアクセスできる必要があるため、`x` はスタックではなくヒープに配置されます

### エスケープしない例

```go
func noEscape() int {
    x := 42
    return x  /* 値を返す → エスケープしない */
}
```

この例では、`x` の値のコピーが返されます

関数終了後に `x` にアクセスする必要がないため、`x` はスタックに配置できます

---

## なぜエスケープ解析が必要か

### スタックとヒープの違い

[04-stack-heap](../04-stack-heap.md) で学んだように、スタックとヒープには大きな違いがあります

| 特性 | スタック               | ヒープ     |
| ---- | ---------------------- | ---------- |
| 速度 | 非常に高速             | 比較的遅い |
| 管理 | 自動（関数終了で解放） | GC が必要  |
| 寿命 | 関数スコープ           | 任意       |

スタックに配置できれば、以下のメリットがあります

- 割り当てが高速（ポインタを動かすだけ）
- 解放が高速（関数終了で自動的に解放）
- GC の対象にならない（GC の負荷が減る）

### コンパイラによる最適化

エスケープ解析は、コンパイラが行う最適化の一つです

プログラマが明示的に「スタックに置いて」と指定するのではなく、コンパイラが自動的に判断します

Go では、`new()` や `make()` で作成したオブジェクトでも、エスケープしなければスタックに配置されます

```go
func example() {
    p := new(int)  /* new でもエスケープしなければスタック */
    *p = 42
    fmt.Println(*p)
}
```

---

## エスケープの判断基準

### 主なエスケープパターン

以下のパターンでは、変数はエスケープします

<strong>1. アドレスを返す</strong>

```go
func escape1() *int {
    x := 42
    return &x  /* エスケープ */
}
```

<strong>2. グローバル変数に代入</strong>

```go
var global *int

func escape2() {
    x := 42
    global = &x  /* エスケープ */
}
```

<strong>3. チャネルに送信</strong>

```go
func escape3(ch chan *int) {
    x := 42
    ch <- &x  /* エスケープ */
}
```

<strong>4. クロージャにキャプチャ</strong>

```go
func escape4() func() int {
    x := 42
    return func() int {
        return x  /* エスケープ */
    }
}
```

<strong>5. インターフェースに変換</strong>

```go
func escape5() {
    x := 42
    var i interface{} = x  /* エスケープする可能性 */
    fmt.Println(i)
}
```

<strong>6. スライスに追加（容量超過の可能性）</strong>

```go
func escape6() {
    s := make([]int, 0)
    s = append(s, 1, 2, 3)  /* 再割り当てでエスケープの可能性 */
}
```

### エスケープしないパターン

<strong>1. 値を返す</strong>

```go
func noEscape1() int {
    x := 42
    return x  /* エスケープしない */
}
```

<strong>2. ローカルでのみ使用</strong>

```go
func noEscape2() {
    x := 42
    y := x + 10
    fmt.Println(y)  /* x はエスケープしない */
}
```

<strong>3. 関数内で完結するポインタ</strong>

```go
func noEscape3() {
    x := 42
    p := &x
    *p = 100
    fmt.Println(x)  /* p は関数外に出ない */
}
```

---

## Go のエスケープ解析

### エスケープ解析の確認

Go では、`-gcflags="-m"` オプションでエスケープ解析の結果を確認できます

```bash
$ go build -gcflags="-m" main.go
./main.go:5:2: moved to heap: x
./main.go:10:13: ... argument does not escape
```

`-m` を増やすと、より詳細な情報が表示されます

```bash
$ go build -gcflags="-m -m" main.go
```

### 具体例

```go
package main

import "fmt"

func escapeExample() *int {
    x := 42
    return &x
}

func noEscapeExample() int {
    x := 42
    return x
}

func main() {
    p := escapeExample()
    fmt.Println(*p)

    v := noEscapeExample()
    fmt.Println(v)
}
```

```bash
$ go build -gcflags="-m" main.go
./main.go:6:2: moved to heap: x    # x はヒープに移動
./main.go:11:2: x does not escape  # x はエスケープしない
```

### インライン化との関係

Go のコンパイラは、小さな関数を<strong>インライン化</strong>することがあります

インライン化されると、呼び出し元の関数内に展開されるため、エスケープの判断が変わることがあります

```go
func small() *int {
    x := 42
    return &x
}

func main() {
    p := small()  /* small がインライン化されると... */
    fmt.Println(*p)
}
```

`small` がインライン化されると、`x` は `main` 内の変数として扱われます

`main` 内で `x` がエスケープしなければ、スタックに配置される可能性があります

---

## JVM のエスケープ解析

### 歴史的背景

エスケープ解析は、Java の HotSpot JVM で広く知られるようになりました

「オブジェクトがメソッドの外に逃げない」ことが分かれば、ヒープではなくスタックに割り当てられます

これにより GC の負荷を減らせます

### JVM での最適化

JVM でも、JIT コンパイラがエスケープ解析を行います

エスケープしないオブジェクトには、以下の最適化が適用されます

<strong>スタック割り当て</strong>

ヒープではなくスタックに割り当て

<strong>スカラー置換（Scalar Replacement）</strong>

オブジェクトを個別のフィールドに分解

```java
/*
 * 元のコード
 */
Point p = new Point(x, y);
double dist = Math.sqrt(p.x * p.x + p.y * p.y);

/*
 * スカラー置換後（概念的）
 */
double p_x = x;
double p_y = y;
double dist = Math.sqrt(p_x * p_x + p_y * p_y);
```

オブジェクト自体が消え、フィールドだけが残ります

<strong>ロック省略（Lock Elision）</strong>

エスケープしないオブジェクトへの同期は不要なため、ロックを省略

### JVM でのエスケープ解析の確認

JVM では、以下のオプションでエスケープ解析の状況を確認できます

```bash
$ java -XX:+PrintEscapeAnalysis -XX:+PrintEliminateAllocations ...
```

ただし、これらは診断用オプションであり、通常の使用では必要ありません

---

## エスケープを避ける書き方

### パフォーマンスを意識した書き方

GC の負荷を減らしたい場合、エスケープを意識した書き方が有効です

<strong>1. ポインタを返す代わりに値を返す</strong>

```go
/* 避ける */
func newPoint(x, y int) *Point {
    return &Point{x, y}  /* エスケープ */
}

/* 推奨 */
func newPoint(x, y int) Point {
    return Point{x, y}  /* エスケープしない */
}
```

<strong>2. バッファを引数で受け取る</strong>

```go
/* 避ける */
func readData() []byte {
    buf := make([]byte, 1024)
    /* ... */
    return buf  /* エスケープ */
}

/* 推奨 */
func readData(buf []byte) int {
    /* buf に書き込む */
    return n  /* buf は呼び出し元が管理 */
}
```

<strong>3. sync.Pool を使う</strong>

頻繁に作成・破棄するオブジェクトは、プールで再利用

```go
var bufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 1024)
    },
}

func process() {
    buf := bufPool.Get().([]byte)
    defer bufPool.Put(buf)
    /* buf を使用 */
}
```

### 注意点

エスケープを避けることは、常に正しいわけではありません

- 可読性が低下する場合がある
- 早すぎる最適化になる可能性
- プロファイリングで問題を特定してから最適化すべき

まずは正しく動くコードを書き、必要に応じて最適化を検討してください

---

## 用語集

| 用語             | 英語               | 説明                                     |
| ---------------- | ------------------ | ---------------------------------------- |
| エスケープ       | Escape             | 変数がスコープ外で参照される可能性       |
| エスケープ解析   | Escape Analysis    | エスケープするかどうかを判定する解析     |
| スタック割り当て | Stack Allocation   | スタックにメモリを確保すること           |
| ヒープ割り当て   | Heap Allocation    | ヒープにメモリを確保すること             |
| インライン化     | Inlining           | 関数呼び出しを展開する最適化             |
| スカラー置換     | Scalar Replacement | オブジェクトをフィールドに分解する最適化 |
| ロック省略       | Lock Elision       | 不要なロックを削除する最適化             |
| クロージャ       | Closure            | 外部変数をキャプチャした関数             |

---

## 参考資料

このページの内容は、以下のソースに基づいています

<strong>Go</strong>

- [Go FAQ: Stack or Heap](https://go.dev/doc/faq#stack_or_heap)
  - Go のスタック/ヒープ選択の FAQ
- [Go Compiler Directives](https://pkg.go.dev/cmd/compile)
  - コンパイラオプションの説明

<strong>JVM</strong>

- [Escape Analysis - OpenJDK Wiki](https://wiki.openjdk.org/display/HotSpot/EscapeAnalysis)
  - JVM のエスケープ解析

<strong>本編との関連</strong>

- [04-stack-heap](../04-stack-heap.md)
  - スタックとヒープの基本
- [05-gc](../05-gc.md)
  - ガベージコレクションの仕組み
