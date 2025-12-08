---
title: "低レイヤー開発者が注目すべきRustのアップデート 2025年版"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "embedded"]
published: true
---

この記事は[Rust Advent Calendar 2025](https://qiita.com/advent-calendar/2025/rust)の8日目の記事です。

Rustの言語機能は6週間ごとに新バージョンがリリースされています。
私はアップデートのたびに勝手にリリースパーティというイベントでアップデートの内容を解説しています。

https://estie.connpass.com/event/373737/

今年は1.84.0から1.91.0までリリースされています。それぞれ様々な変更が入っていてリリースノートを見ると日々改善されているのがわかります。

今回は今年あったアップデートから特に低レイヤー開発に関する内容をピックアップして紹介します。

## naked functions

1.88.0でnaked functionsの安定化がされました。Rustの普通の関数は呼び出し時と呼び出し元に戻るときにそれぞれ特殊な処理が入ります。しかし、インラインアセンブラで自前で呼び出し元に戻る場合、呼び出し元に戻る際の処理が入らず不整合が発生してしまいます。例えば、呼び出し時にレジスタをスタックに追加して戻るときにそのスタックから値を戻すという処理だった場合、スタックがずれてしまうという問題が発生します。nakedというアトリビュートを付与された関数はこれらの特殊処理が省略されるようになり、インラインアセンブラで呼び出し元に戻ってもこのような問題が起こらないようになります。

https://blog.rust-lang.org/2025/06/26/Rust-1.88.0/#naked-functions

関数に`#[unsafe(naked)]`というアトリビュートをつけます。アトリビュートにunsafeをつけているのに見慣れていない人もいるかもしれませんが、これはRust 2024 Editionで追加された機能です。

https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-attributes.html

## raw pointer関連の改善

低レイヤーで避けて通れないのが生ポインタの扱いです。生ポインタを使うとRustの特性である所有権やライフタイムの性質が効かないunsafe処理が絡むことになり扱いが難しいです。

大きな変更としては1.84.0で入ったstrict provenance APIsがあります。

https://blog.rust-lang.org/2025/01/09/Rust-1.84.0/#strict-provenance-apis

生ポインタの値がどのように生成されたかを追うことができるようにできるAPIが使えることにより、MiriやCHERIなどの静的解析ツールで恩恵が得られるようです。

1.84.0では他にもポインタの参照外しへの生ポインタの生成がsafeになったり（`addr_of!((*ptr).field)`のようなコード。これは単にアドレスをつくっているだけで、`ptr`が不正な値でもアドレス値の計算自体はできるので安全）、即座にdropされる値へのポインタをつくるとlintで警告が出る変更も追加されています。

https://github.com/rust-lang/rust/pull/129248

https://github.com/rust-lang/rust/pull/128985


1.86.0では生ポインタアクセスでnon-nullかどうかを判定するdebug assertionsが入るようになりました。これによりdebug assertionが有効なビルドの場合、ランタイムで不正なポインタアクセスをしようとするとpanicを発生させてくれるようになります。

https://blog.rust-lang.org/2025/04/03/Rust-1.86.0/#debug-assertions-that-pointers-are-non-null-when-required-for-soundness

このようにRustの生ポインタは単にアドレス値を取り扱うだけでなく、扱いやすいように様々な工夫がされているのがわかります。

## Edition 2024

今年のRust全体での大きな変更としてEdition 2024が安定化されたことが挙げられます。

Edition 2024は1.85.0から使えるようになり、詳しい変更はEdition Guideから見ることができます。

低レイヤー関連で関係がありそうなのは

- [unsafe extern](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-extern.html)
    - 外部のC言語関数を呼び出すときなどに用いられる`extern`ブロックに`unsafe`キーワードつけることができ、危険性をより明示的にできる
- [unsafe attribute](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-attributes.html)
    - `no_mangle`や`extern_name`のような書き手側で安全性を証明しなければいけないアトリビュートに`unsafe`をつけることが必要になった
- [unsafe_op_in_unsafe_fn warning](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-op-in-unsafe-fn.html)
    - `unsafe`関数内で`unsafe`関数を呼び出している場合、`unsafe`ブロックで囲っていないと警告をだすlintがデフォルトに
- [Disallow references to static mut](https://doc.rust-lang.org/edition-guide/rust-2024/static-mut-references.html)
    - `static mut`な変数への参照がコンパイルエラーになるlintがデフォルトで有効になった

といったところでしょうか。unsafeとつけるべき箇所が増え、lintがより厳しくなりました

## target features

1.86.0で`#[target_feature]`がsafe関数につけることができるようになりました。
`target_feature`が指定されているsafe関数を呼び出すとき通常はunsafeが必要ですが、target featureが指定されている関数内から同じtarget featureが指定されているsafe関数の呼び出しはunsafeではありません。

https://blog.rust-lang.org/2025/04/03/Rust-1.86.0/#allow-safe-functions-to-be-marked-with-the-target-feature-attribute

1.87.0で`std::arch`に含まれていたunsafeな関数がこのtarget featuresを利用してsafeな関数になり、1.89.0ではより多くの関数がsafeとなりました。

https://blog.rust-lang.org/2025/05/15/Rust-1.87.0/#safe-architecture-intrinsics

https://blog.rust-lang.org/2025/08/07/Rust-1.89.0/#more-x86-target-features


## asm goto

インラインアセンブラの中で、Rustで書かれたコードのブロックにジャンプができるようになりました。


https://blog.rust-lang.org/2025/05/15/Rust-1.87.0/#asm-jumps-to-rust-code

具体的にはこのように書くことができます（ドキュメントより引用）。

```Rust
unsafe {
    core::arch::asm!("jmp {}", label {
        println!("Hello from inline assembly label");
    });
}

```

## その他

その他の変更は以下のようなものがあります。カッコ内は導入されたバージョンです

- [extern "C"のfunctionでpanicした場合、unwinding中にdropが走るように](https://github.com/rust-lang/rust/pull/129582)（1.84.0）
- [externでABIを省略した場合に警告するlintがデフォルトで有効に](https://blog.rust-lang.org/2025/04/03/Rust-1.86.0/#make-missing-abi-lint-warn-by-default) (1.86)
- [doctestsがcross compileに対応](https://blog.rust-lang.org/2025/08/07/Rust-1.89.0/#cross-compiled-doctests) (1.89.0)
- [i128とu128がextern "C"の関数で利用可能に](https://blog.rust-lang.org/2025/08/07/Rust-1.89.0/#i128-and-u128-in-extern-c-functions) (1.89.0)
- [const genericsへの推論が可能に](https://blog.rust-lang.org/2025/08/07/Rust-1.89.0/#explicitly-inferred-arguments-to-const-generics) (1.89.0)

## まとめ

いかがでしたか？低レイヤー関連に絞っても1年間でかなりの変更がありました。
Rustは低レイヤー分野を安全に触れる言語として注目されていますが、これらのアップデートを活用することでより安全に書けるかもしれません。

## 宣伝

12/21にオフラインイベントをします。LTも募集中なのでぜひ！

https://estie.connpass.com/event/376594/
