<div align="right">
<img src="https://img.shields.io/badge/AI-ASSISTED_STUDY-3b82f6?style=for-the-badge&labelColor=1e293b&logo=bookstack&logoColor=white" alt="AI Assisted Study" />
</div>

# 07-ffi：言語間の橋渡し

## はじめに

ここまで、ランタイムの様々な側面を学んできました

- [01-runtime](./01-runtime.md)：ランタイムとは何か
- [02-compilation](./02-compilation.md)：プログラムが実行可能になるまで
- [03-vm](./03-vm.md)：仮想マシンとバイトコード
- [04-stack-heap](./04-stack-heap.md)：スタックとヒープ
- [05-gc](./05-gc.md)：ガベージコレクション
- [06-concurrency](./06-concurrency.md)：並行処理とランタイム

最後のトピックでは、異なる言語のランタイム間で「どうやって連携するか」を学びます

Python から C のライブラリを呼べるのはなぜでしょうか？

Go のプログラムから SQLite（C で書かれている）を使えるのはなぜでしょうか？

この「言語間の橋渡し」を<strong>FFI（Foreign Function Interface）</strong>と呼びます

### 日常の例え

国際会議を想像してください

各国の代表が、それぞれの言語で話しています

日本語、英語、フランス語、中国語…

しかし、全員が全ての言語を話せるわけではありません

そこで、<strong>共通語</strong>を決めます

多くの国際会議では、英語が共通語として使われます

各国の代表は、発言を英語に翻訳し、他の代表も英語を理解できるようにします

プログラミングの世界では、<strong>C 言語</strong>がこの「英語」の役割を果たしています

| 概念           | 日常の例え             |
| -------------- | ---------------------- |
| 各言語         | 各国の言語             |
| C 言語         | 国際共通語（英語）     |
| FFI            | 翻訳のルール           |
| ABI            | 発言のフォーマット規則 |
| システムコール | 会議場の設備を使う申請 |

### このページで学ぶこと

このページでは、以下の概念を学びます

- <strong>なぜ言語間連携が必要か</strong>
  - 既存ライブラリの活用、パフォーマンス
- <strong>なぜ C が共通語なのか</strong>
  - 歴史、ABI の安定性
- <strong>各言語の FFI</strong>
  - Python、Go、Rust、Java の方法
- <strong>FFI の注意点</strong>
  - メモリ管理、GC との相互作用

---

## 目次

1. [なぜ言語間連携が必要か](#なぜ言語間連携が必要か)
2. [なぜ C が共通語なのか](#なぜ-c-が共通語なのか)
3. [ABI とは何か](#abi-とは何か)
4. [各言語の FFI](#各言語の-ffi)
5. [FFI の注意点](#ffi-の注意点)
6. [システムコールとの関係](#システムコールとの関係)
7. [まとめ：このリポジトリで学んだこと](#まとめこのリポジトリで学んだこと)
8. [用語集](#用語集)
9. [参考資料](#参考資料)

---

## なぜ言語間連携が必要か

すべてを一つの言語で書ければ、FFI は不要です

しかし、現実には言語間連携が必要な場面が多くあります

### 既存ライブラリの活用

多くの重要なライブラリは C で書かれています

| ライブラリ | 用途         |
| ---------- | ------------ |
| OpenSSL    | 暗号化       |
| SQLite     | データベース |
| libpng     | PNG 画像処理 |
| zlib       | 圧縮         |

これらを使うために、毎回別の言語で書き直すのは非効率です

FFI を使えば、既存の C ライブラリをそのまま活用できます

### パフォーマンス

一部の処理は、C で書いた方が高速です

Python は書きやすいですが、CPU 集約的な処理は遅くなりがちです

NumPy が高速なのは、内部で C（や Fortran）を使っているからです

### システム API へのアクセス

OS のシステムコールは、基本的に C の API として提供されています

高水準言語から OS の機能を使うには、C との連携が必要です

---

## なぜ C が共通語なのか

プログラミング言語の「共通語」として、C が選ばれる理由があります

### 歴史的経緯

C 言語は UNIX の実装言語として開発され、広く普及しました

その後、ほぼすべての OS が C で書かれるようになりました

Windows も、Linux も、macOS も、C（または C に近い言語）で書かれています

OS の API が C で提供される以上、他の言語も C と連携する必要があります

### ABI の安定性

C 言語の<strong>ABI（Application Binary Interface）</strong>は、非常に安定しています

新しいバージョンの C コンパイラで作られたライブラリでも、古いプログラムから呼び出せます

C++ は ABI が不安定なため、FFI には向いていません

C++ では<strong>名前マングリング</strong>により、関数名に型情報が付加されます

例えば `add(int, int)` という関数は `_Z3addii` のような名前に変換されます

この変換ルールはコンパイラごとに異なるため、異なるコンパイラ間での連携が困難です

### シンプルな呼び出し規約

C の関数呼び出しは、非常にシンプルです

引数をスタックやレジスタに積み、関数を呼び、戻り値を受け取る

このシンプルさが、他の言語からの呼び出しを容易にしています

---

## ABI とは何か

### ABI の定義

<strong>ABI（Application Binary Interface）</strong>は、バイナリレベルでの「約束事」です

API がソースコードレベルの約束事（関数名や引数の型）なのに対し、ABI はコンパイル後のバイナリでの約束事（レジスタの使い方、メモリ配置）です

| 約束事           | 説明                             |
| ---------------- | -------------------------------- |
| 呼び出し規約     | 引数の渡し方、戻り値の受け取り方 |
| データレイアウト | 構造体のメモリ配置               |
| シンボル名       | 関数名の表現方法                 |

### 呼び出し規約

関数を呼び出すとき、引数はどこに置くのでしょうか？

<strong>System V AMD64 ABI</strong>（Linux、macOS で使用）では：

| 引数の順番 | 格納先       |
| ---------- | ------------ |
| 1 番目     | RDI レジスタ |
| 2 番目     | RSI レジスタ |
| 3 番目     | RDX レジスタ |
| 4 番目     | RCX レジスタ |
| 5 番目     | R8 レジスタ  |
| 6 番目     | R9 レジスタ  |
| 7 番目以降 | スタック     |

戻り値は RAX レジスタに格納されます

この「約束」があるから、異なるコンパイラで作られたコード同士が連携できるのです

### データレイアウト

構造体のメモリ配置も ABI で決まっています

```c
struct Example {
    char a;    /* 1 バイト */
    int b;     /* 4 バイト */
};
```

この構造体のサイズは 5 バイトではなく、8 バイトになります

`int` は 4 バイト境界に配置される必要があるため、パディングが入ります

```
メモリ: [a][padding][padding][padding][b][b][b][b]
オフセット: 0   1       2       3      4   5   6   7
```

---

## 各言語の FFI

主要な言語が、どのように C と連携するかを見てみましょう

### Python：ctypes

Python の<strong>ctypes</strong>は、C ライブラリを呼び出すための標準モジュールです

```python
from ctypes import cdll, c_int

# C ライブラリを読み込む
libc = cdll.LoadLibrary("libc.so.6")

# C の関数を呼び出す
result = libc.abs(-42)
print(result)  # 42
```

`ctypes` は、Python オブジェクトを C の型に変換し、C 関数を呼び出します

### Go：cgo

Go の<strong>cgo</strong>は、Go から C コードを呼び出す仕組みです

```go
package main

/*
#include <stdlib.h>
*/
import "C"
import "fmt"

func main() {
    result := C.abs(-42)
    /*
     * 42
     */
    fmt.Println(result)
}
```

`import "C"` の直前のコメントに C コードを書きます

cgo は、Go と C のデータ型を相互に変換するコードを自動生成します

### Rust：extern "C"

Rust は<strong>extern "C"</strong>で C 関数を宣言します

```rust
extern "C" {
    fn abs(x: i32) -> i32;
}

fn main() {
    unsafe {
        let result = abs(-42);
        /*
         * 42
         */
        println!("{}", result);
    }
}
```

C 関数の呼び出しは `unsafe` ブロック内で行う必要があります

Rust のコンパイラは、C コードの安全性を保証できないためです

### Java：JNI

Java は<strong>JNI（Java Native Interface）</strong>で C と連携します

JNI は他の FFI より複雑です

```java
public class Example {
    static {
        /*
         * libexample.so を読み込む
         */
        System.loadLibrary("example");
    }

    /*
     * C で実装
     */
    public native int abs(int x);
}
```

C 側では、JNI のヘッダを使って関数を実装します

### 比較表

| 言語   | FFI 方法   | 特徴                      |
| ------ | ---------- | ------------------------- |
| Python | ctypes     | 動的読み込み、簡単        |
| Go     | cgo        | コメントに C コードを記述 |
| Rust   | extern "C" | unsafe ブロックが必要     |
| Java   | JNI        | 複雑、ヘッダ生成が必要    |

---

## FFI の注意点

FFI には、いくつかの注意点があります

### メモリ管理の責任

C で確保したメモリは、C で解放する必要があります

```python
from ctypes import cdll, c_void_p

libc = cdll.LoadLibrary("libc.so.6")
ptr = libc.malloc(1000)  # C でメモリ確保
# ... 使用 ...
libc.free(ptr)  # C で解放（Python の GC は解放しない）
```

Python の GC は、`malloc` で確保したメモリを知りません

明示的に `free` を呼ばないと、メモリリークになります

### GC との相互作用

GC がある言語から C を呼ぶとき、オブジェクトの寿命に注意が必要です

```python
s = "hello"
ptr = ctypes.c_char_p(s.encode())
some_c_function(ptr)
# この時点で s が GC されると、ptr は無効になる可能性
```

C 関数にポインタを渡している間、元のオブジェクトが GC されないようにする必要があります

### 型の不一致

C と高水準言語では、型の表現が異なります

| 問題       | 説明                              |
| ---------- | --------------------------------- |
| 文字列     | C は NULL 終端、Python は長さ付き |
| 整数サイズ | C の int は環境依存               |
| 浮動小数点 | 精度の違い                        |

型の変換を間違えると、クラッシュやデータ破損が起きます

### オーバーヘッド

FFI 呼び出しには、オーバーヘッドがあります

- 型変換のコスト
- GIL の解放・取得（Python）：C 関数実行中は他のスレッドが動けるよう GIL を解放し、戻るときに再取得する
- スタック切り替え（cgo）：Go と C ではスタックサイズや管理方法が異なるため、呼び出し時にスタックを切り替える

小さな関数を何度も呼ぶと、オーバーヘッドが無視できなくなります

---

## システムコールとの関係

FFI とシステムコールは、どう関係しているのでしょうか？

### 最終的な行き先

高水準言語でファイルを読むとき、最終的にはシステムコールが呼ばれます

```
Python: open("file.txt")
    ↓
CPython: Python の組み込み関数
    ↓
libc: open() 関数
    ↓
システムコール: open(2)
    ↓
カーネル: ファイルを開く処理
```

言語のランタイムは、libc（C 標準ライブラリ）を経由してシステムコールを呼びます

### libc の役割

libc は、システムコールをラップして使いやすくしています

| libc 関数        | システムコール |
| ---------------- | -------------- |
| `printf`         | `write`        |
| `malloc`         | `mmap`、`brk`  |
| `fopen`          | `open`         |
| `pthread_create` | `clone`        |

多くの言語のランタイムは、直接システムコールを呼ばず、libc を使います

### Go の特殊性

Go は Linux では例外的に、libc を使わずに直接システムコールを呼びます

```go
/*
 * Go は Linux では libc を経由しない
 */
syscall.Open("/tmp/file", os.O_RDONLY, 0)
```

これにより、Go のバイナリは libc に依存せず、静的リンクが容易になります

ただし、cgo を使うと libc への依存が発生します

---

## まとめ：このリポジトリで学んだこと

このリポジトリでは、「言語がどうやって動いているか」を学びました

### 学んだトピック

| トピック       | 内容                             |
| -------------- | -------------------------------- |
| 01-runtime     | ランタイムとは何か、OS との関係  |
| 02-compilation | AOT、JIT、インタプリタの違い     |
| 03-vm          | 仮想マシンとバイトコード         |
| 04-stack-heap  | スタックとヒープの使い分け       |
| 05-gc          | ガベージコレクションの仕組み     |
| 06-concurrency | 並行処理とランタイムスケジューラ |
| 07-ffi         | 言語間の橋渡し（FFI）            |

### 全体像

```
┌─────────────────────────────────────────┐
│           ユーザーのコード                │
├─────────────────────────────────────────┤
│              ランタイム                   │
│  ・コンパイル/インタプリタ (02)           │
│  ・仮想マシン (03)                       │
│  ・メモリ管理：スタック/ヒープ (04)       │
│  ・ガベージコレクション (05)             │
│  ・並行処理スケジューラ (06)             │
│  ・FFI (07)                              │
├─────────────────────────────────────────┤
│           libc / システムコール           │
├─────────────────────────────────────────┤
│              OS（カーネル）               │
├─────────────────────────────────────────┤
│             ハードウェア                  │
└─────────────────────────────────────────┘
```

ランタイムがプロセス内で動くことを理解していれば、コンテナの仕組みも理解しやすくなります

<strong>言語固有の深掘り</strong>

- Go のスケジューラ（GMP モデル）
- JVM のアーキテクチャ
- Python の async/await

各言語のランタイムを深く学ぶ基礎ができました

<strong>パフォーマンスチューニング</strong>

GC のチューニング、メモリ配置の最適化、FFI のオーバーヘッド削減

ランタイムの仕組みを知っていれば、これらの最適化が可能になります

---

## 用語集

| 用語             | 英語                         | 説明                                                 |
| ---------------- | ---------------------------- | ---------------------------------------------------- |
| FFI              | Foreign Function Interface   | 他言語の関数を呼び出す仕組み                         |
| ABI              | Application Binary Interface | バイナリレベルの呼び出し規約                         |
| 呼び出し規約     | Calling Convention           | 引数の渡し方、戻り値の受け取り方                     |
| ctypes           | ctypes                       | Python の FFI モジュール                             |
| GIL              | Global Interpreter Lock      | Python が一度に1スレッドだけ実行するための排他ロック |
| cgo              | cgo                          | Go の C 言語連携機能                                 |
| JNI              | Java Native Interface        | Java の C 言語連携機能                               |
| extern "C"       | extern "C"                   | C リンケージを指定する宣言                           |
| unsafe           | unsafe                       | Rust で安全性チェックを無効化するブロック            |
| libc             | libc                         | C 標準ライブラリ                                     |
| 名前マングリング | Name Mangling                | C++ でシンボル名を変換すること                       |
| パディング       | Padding                      | アライメントのための隙間                             |
| アライメント     | Alignment                    | データの配置境界                                     |

---

## 参考資料

このページの内容は、以下のソースに基づいています

<strong>ABI</strong>

- [System V Application Binary Interface](https://refspecs.linuxfoundation.org/elf/x86_64-abi-0.99.pdf)
  - AMD64 の ABI 仕様

<strong>Python</strong>

- [ctypes - Python Documentation](https://docs.python.org/3/library/ctypes.html)
  - ctypes の公式ドキュメント

<strong>Go</strong>

- [cgo - Go Documentation](https://pkg.go.dev/cmd/cgo)
  - cgo の公式ドキュメント

<strong>Rust</strong>

- [FFI - The Rust Book](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#using-extern-functions-to-call-external-code)
  - Rust の FFI 解説

<strong>Java</strong>

- [JNI Specification](https://docs.oracle.com/en/java/javase/21/docs/specs/jni/index.html)
  - JNI の公式仕様
