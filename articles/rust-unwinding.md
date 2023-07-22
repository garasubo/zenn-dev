---
title: "RustのC-unwind ABIで他言語での例外を扱う"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust"]
published: true
---

Rust 1.71.0がリリースされ、C-unwind ABIが安定化されました。

https://blog.rust-lang.org/2023/07/13/Rust-1.71.0.html

このC-unwind ABIというものがどういうものかを説明します。
このunwindの仕組みが自分も詳しく知らないものなので、間違っていたら指摘していただけると助かります。

## Rustにおけるunwind
Rustのエラー処理を実現する方法はいくつかあります。

1. `Option`や`Result`を使う
2. `panic`を起こしてスレッドを停止させる
3. プログラム全体をアボートする

1番目の方法が大体のケースで行われることで、2番目は`unwrap`や`expect`でめったに起こらないであろう処理を記述するときに使うことが多いと思います。どうにもならないときは3番目のプログラム全体を停止させるということになると思います。
2番目と3番目は一見すると同じように見えますが、`panic`はそのpanicを起こしたスレッドのみを停止させるので厳密には異なります。
また、`panic`は[`catch_unwind`](https://doc.rust-lang.org/std/panic/fn.catch_unwind.html)という関数を使うことで、panicしてもスレッドを停止させることなく、`Result`型に変換することもできます。他の言語で言う`try catch`みたいな感じですね。

```rust
fn do_panic() {
    panic!("This is a panic!");
}

fn main() {
    let result = std::panic::catch_unwind(|| {
        do_panic();
    });
    println!("Result: {:?}", result);
}
```

このunwind、日本語に訳すと「巻き戻し」の処理がFFI（Foregin Function Interface）を用いて他の言語の関数が混ざった場合の処理が未定義でした。
つまり、C++のような例外が発生する言語の関数をRust側から呼び出したり、逆にC++側からpanicする可能性があるRust関数の呼び出しをした場合の挙動が保障できていませんでした。
今回安定化されたC-unwind ABIはそのうちの一部の挙動の安定化をしたものです。

## `C-unwind`を使って言語をまたぐpanicをunwindする
今回安定化されたのは`C-unwind`という`extern`のあとにつけられる呼び出し規約のパターンです。
RFCに乗っている具体例をそのまま見てみましょう。
```rust
// 他言語の関数をRust側で使う場合
extern "C-unwind" {
  fn may_throw();
}

// Rustの関数を他言語で使ってもらう場合
extern "C-unwind" fn can_unwind() {
  may_throw();
}
```

今回の変更でコンパイルオプションで`panic=unwind`となっていた場合、`C-unwind`を使って呼び出した言語をまたぐpanicやC++スタイルの例外を呼び出し元でunwindできるようになりました。
つまり、Rust側からC言語側の関数を呼び出して、そのC言語の関数がRustの関数を呼び出し、その関数がpanicになったら最初の呼び出し元のRustのコードでunwindをすることが可能になった、ということです。

なぜ`C`と`C-unwind`で動作を分けているかというと、unwindingするためにはランタイムでのコストが発生してしまうため、最適化のために基本的にはunwindの準備をしたくないというのが理由のようです。

なお、今回の安定化で[RFCの表の動作](https://github.com/rust-lang/rfcs/blob/ed4c592b58dc2ef83d48fd21d556c47e8b3b492a/text/2945-c-unwind-abi.md#abi-boundaries-and-unforced-unwinding)が完全に実装されたわけではなく、`extern "C"`とした場合の挙動には変更がないようです。
筆者も手元で実験したのですが、`extern "C"`をつけた関数の中でRust側のパニックする関数を呼び出してもabortにはならずunwindできてしまいました。
しかし、最終的にはこのRFCの表を実装していくことになると思われるので、将来のバージョンではこの動作は変わるということなのでしょう。

### 実際のコード
自分が手元で試したコードは以下のようなものです。

`main.rs`でC言語用にただpanicする関数を提供しつつ、C言語側からその関数を間接的に呼び出してもらう関数をexternで読み込みます。
```rust:main.rs

#[link(name = "example")]
extern "C-unwind" {
    fn c_calling_rust();
}

#[no_mangle]
pub extern "C-unwind" fn my_rust_panic() {
    println!("Rust function");
    panic!("This is a Rust panic!");
}

fn main() {
    let result = std::panic::catch_unwind(|| unsafe {
        c_calling_rust();
        println!("Never printed");
    });
    println!("Result: {:?}", result);
    loop {
        println!("main thread running...");
        std::thread::sleep(std::time::Duration::from_secs(1));
    }
}

```

C言語側ではRust側からpanicする関数をexternで受け取りそれを呼び出す関数を定義します。
```c:lib.c
#include <stdio.h>

extern void my_rust_panic();

void c_calling_rust() {
    printf("Hello from C\n");
    my_rust_panic();
}
```

`Cargo.toml`ではコンパイラのオプションとして`panic = "unwind"`を設定して、`build.rs`用に`cc`クレートへの依存を記述します
```toml:Cargo.toml
[package]
name = "unwind-test"
version = "0.1.0"
edition = "2021"

[profile.release]
panic = "unwind"
[profile.dev]
panic = "unwind"

[build-dependencies]
cc = "1.0"
```

`build.rs`では`cc`クレートを使い、C言語を取り込めるようにします。
```rust:build.rs
fn main() {
    cc::Build::new()
        .file("src/lib.c")
        .compile("example");
    println!("cargo:rerun-if-changed=src/lib.c");
    println!("cargo:rerun-if-changed=build.rs");
}
```

このようにして`RUST_BACKTRACE=1 cargo run --bin unwind-test`と実行すれば以下のような実行結果になりました
```
Hello from C
Rust function
thread 'main' panicked at 'This is a Rust panic!', src/main.rs:9:5
stack backtrace:
   0: rust_begin_unwind
             at /rustc/8ede3aae28fe6e4d52b38157d7bfe0d3bceef225/library/std/src/panicking.rs:593:5
   1: core::panicking::panic_fmt
             at /rustc/8ede3aae28fe6e4d52b38157d7bfe0d3bceef225/library/core/src/panicking.rs:67:14
   2: my_rust_panic
   3: unwind_test::main
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
Result: Err(Any { .. })
main thread running...
main thread running...
```
panicしたときの情報がきちんと呼び出し側のRustの方に伝わっているのがわかります。
これを`panic = "abort"`としてしまうと、"main thread running..."となるループにたどり着く前にプログラムがアボートしてしまいます。

自分がリリースノートを正しく読めていれば、C++での例外もRustをまたいで呼び出し側に伝えることができるようになったようなのですが、残念ながらC++とRustを同時にコンパイルしようとするとC++のライブラリを正しくリンクする必要があり、ちょっと例がつくれなかったのでまだ検証できてないです。

と、ちょっと中途半端な検証になってしまいましたが、今回の記事はここまでにします。

2023-07-22 追記

C++ -> Rust -> C++で例外を補足する例を紹介してる記事が他の方が書いていたので紹介しておきます

https://zenn.dev/termoshtt/articles/pass-cpp-exception-through-rust
