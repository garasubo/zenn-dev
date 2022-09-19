---
title: "Rust for Linuxでは独自のallocライブラリを使っている"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Rust, Linux]
published: true
---

Rustを第二言語として採用してデバイスドライバなどのモジュールをRustで書けるようにする「Rust for Linux」が近々マージされる予定だともLinus氏自身が発言しています。

https://www.zdnet.com/article/linus-torvalds-rust-may-make-it-into-the-next-linux-kernel-after-all/

そんな期待のかかるRust for Linuxですが、提案された当初は期待こそされていたものの、様々な懸念点も指摘されていました。
その1つが標準ライブラリの一部である`alloc`クレートの設計です。

https://doc.rust-lang.org/alloc/

このクレートはヒープ領域を扱う`Box`、`Vec`、`String`などRustではお馴染みの構造体を提供しています。

Rustの標準ライブラリはOSのサポートを前提とした構造体も多くあります。そのため、OSそのものを書くようなベアメタルプログラミングにおいて標準ライブラリをそのまま使うことはできません。
使えるのは`core`と呼ばれる依存関係のない全く無いライブラリがありますが、`alloc`はOSのサポートが必要なヒープ領域を扱う必要があるためこれに含まれません。
しかし、自分でアロケータを定義することで使うことができます。これについては以前プレゼンした資料があるので詳しくはそちらにどうぞ。

https://docs.google.com/presentation/d/1mT1N22j0zPIutotZSSgjsoLIn4-tp14qG02K3uazEyw/edit?usp=sharing

## allocクレートの設計の問題点

OSの内部とはいえ、allocクレートの存在なしでプログラミングをするのは難しいため、Rust for LinuxでもLinux内部のアロケータを利用することでallocクレートを使えるようにしています。
しかし、allocクレートの設計として、アロケーションに失敗してしまったときの処理に問題があります。
C言語の場合だと`malloc`が失敗したときNULLポインタが返ってくるので、これによりアロケーションに失敗したときの処理を記述することが可能です。
しかし、Rustの場合の場合、アロケーションに失敗してもそれを示す型が返ってくるわけではありません。`Box`や`Vec`のコンストラクタが`Result`ではなく`Self`を返していることからこれは明らかですね。
普通のアプリケーションだとアロケーションが失敗した場合は`panic`となり、そのままアプリケーションが異常終了してしまいます。
もう少し深堀すると、`Box`のような構造体の内部でメモリアロケーションするときに使われるのが`GlobalAlloc`というトレイトを実装したグローバルアロケータです。

https://doc.rust-lang.org/stable/std/alloc/trait.GlobalAlloc.html

この`GlobalAlloc`の`alloc`メソッドによりメモリ確保が行われるのですが、このメソッドの返り値は`*mut u8`型、つまり生ポインタを返す関数となっています。
アロケーションに失敗した場合はNULLポインタを返すことが期待されています。
NULLポインタが返された場合その呼び出し側、つまりallocライブラリ内部で、`handle_alloc_error`という関数が呼び出されることになります。

https://doc.rust-lang.org/stable/alloc/alloc/fn.handle_alloc_error.html

この関数の返り値は発散する型`!`、つまりこの関数が呼び出されてしまうと元の処理に復帰できないことになります。
`handle_alloc_error`を独自のものに置き換えることはできるのですが、型の制約上、最終的には`panic`のような異常終了のような処理に入ることになります。

このように、allocクレートは実行時にpanicする可能性を抱えていて（run-time failure panic）、このことを最初のパッチでLinus氏が問題点として指摘しています。

https://lkml.org/lkml/2021/4/14/1099

実行時にドライバーがパニックしてしまうとカーネル全体がabortとなってしまうため、このままでは受け入れられない、ということです。

## Rust for Linuxの解決策
[以前の記事](https://zenn.dev/garasubo/articles/rust-for-linux)でも少し紹介したのですが、Rust for Linuxではこの問題を独自の`alloc`ライブラリを使うことで解決しています。
Makefileの中身を見るとよくわかります。この箇所が`.rs`ファイルのコンパイルのルールを定めている箇所です。

https://github.com/Rust-for-Linux/linux/blob/9f4510ea769db8ea6d974f11a45322a1cf55e6ca/scripts/Makefile.build#L275

Rust for Linuxでは`cargo`を使わないで、`rustc`にオプションを渡すことで外部ライブラリとのリンクを行っています。
`--extern alloc`により外部クレートして独自の`alloc`ライブラリを使えるようにしています。

独自のライブラリは[`rust/alloc`](https://github.com/Rust-for-Linux/linux/tree/0473a8d0683715e8c6e4fbb8da8e68ef72b7c98c/rust/alloc)以下にあり、ドキュメントはここにあります。

https://rust-for-linux.github.io/docs/alloc


一見すると普通のallocと変わらないように見えますが、例えば`Box`だと通常の`new`メソッドが存在しません。

https://rust-for-linux.github.io/docs/alloc/boxed/struct.Box.html

その代わりのメソッドとなるのが`try_new`などの`allocator_api`のfeatureによって追加されるメソッドたちです。
これらは安定化されていないfeatureですが、本家のallocでも使うことはできます。
これらのメソッドは`Result`型を返すため、アロケーションが失敗しても発散することなく自分でエラー処理を書くことができます。

現状だと本家のallocとできるだけ同じになるようにしながら管理されていますが、本家の変更が整えばこのフォークは必要なくなるだろう、とは書かれています。

## まとめ
Rustのallocクレートはメモリアロケーションに失敗したときプログラム全体がアボートしてしまうという問題があります。
Rust for Linuxでは、アロケーション失敗時にカーネル全体がabortしないように、独自の変更を行ったallocクレートを組み込んでビルドを行っています。

本家の方のallocクレートの方も改善が続いているので、将来的にはフォークの必要がなくなるかもしれません。
