---
title: "Rust for Linuxを手元で試す"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

RustをLinuxカーネルに組込みプロジェクト、Rust for Linuxが進行中です。

https://github.com/Rust-for-Linux

このプロジェクトはLinuxカーネル全体をRustで置き換えるわけではなく、第二言語としてRustを採用してデバイスドライバなどのモジュールを書くことができるようにしようというものです。
RustはOSのような低レイヤーソフトウェアを実装する言語として、C言語に代わる選択肢として注目されてきたわけですが、Linuxのような広く使われているシステムに採用されるとなればかなり熱いですね。

実際にLinuxのメインラインに取り入れられるにはまだまだ課題は多いものの、Linus氏を含むLinuxの開発者からのフィードバックも比較的ポジティブでこれからが注目されています。

そんなRust for Linuxを手元でビルドして動かしてみました。
一応、基本的な手順はレポジトリ内のドキュメントにまとまっているのですが、いくつかつまずいた点も含めて紹介します。

https://github.com/Rust-for-Linux/linux/blob/rust/Documentation/rust/quick-start.rst

筆者はLinuxカーネルの開発経験がほとんどないため、誤っている点もあるかもしれませんがご了承ください。

## カーネルをビルドする
Rust for Linuxの変更はLinuxをフォークしたレポジトリ内で管理されています。

https://github.com/Rust-for-Linux/linux.git

定期的にメーリングリストにパッチとしても投稿されているので、こちらを取り込むことでも試すことができるかと思います。
この記事の執筆時点で最新のものは2022/05/23に投稿されたPatch v7です。

https://lore.kernel.org/rust-for-linux/20220523020209.11810-1-ojeda@kernel.org/T/#m3f95c64ad43390c4237df111d147367548940b32


基本的には先程出した[Quick Start](https://github.com/Rust-for-Linux/linux/blob/rust/Documentation/rust/quick-start.rst)のガイドに沿ってRustと関連ツール、またLinuxビルドに必要なものをもインストールする必要があります。
Linuxのビルドに必要なものは、使用しているOSがUbuntuの場合こちらを参考にするといいと思います。

https://wiki.ubuntu.com/Kernel/BuildYourOwnKernel

Rustは基本的にLLVMバックエンドに依存しているため、カーネルのビルドにもLLVMのツールが必要です。Ubuntu 20.04でデフォルトで入るllvmのツール群はバージョンが古く、このままではビルドできないので、llvmの公式から新しいバージョンを落としてくる必要があります。筆者はllvm-12を手元でビルドしてインストールしました。

https://releases.llvm.org/download.html

LinuxをビルドするにはKconfigという仕組みで`.config`という設定ファイルを編集して、必要なオプションを切り替えてあげる必要があります。
RustのサポートもKconfigで設定する必要があり、`CONFIG_RUST`という変数が有効になっている必要があります。
`make menuconfig`とするとTUIを使って`.config`を編集することができます。
ドキュメントによると`General setup`に`Rust support`というのがあると書かれています。が、デフォルトの状態では現れない場合があります。
これは`Rust support`が依存しているconfigが正しく設定されていない場合に起こります。
menuconfig実行中に`/`を入力することで探したい設定を検索できます。`rust`で検索すると以下のように`RUST`シンボルが何に依存しているか`Depends on`のところに書いてあります。

![設定の検索結果画面](/images/rust-kconfig-search.png)

`RUST_IS_AVAILABLE`などが有効で`GCC_PLUGIN`などは逆に無効になっている必要があります。これで満たしていないシンボルについて`/`で検索をかけて設定を変更しましょう。

`make menuconfig`の他に`scripts/config`というスクリプトで個別のシンボルの編集もできます。
`scripts/config --enable RUST`とすれば依存関係が満たされていれば`Rust support`を有効にすることができます。

configについて他に気をつける部分としては、`CONFIG_SYSTEM_TRUSTED_KEYS`という項目があり、デフォルト値のままだと自分で署名用のキーを用意しないといけません。
自分で署名用のキーを用意する、またはこの設定を無効化してしまうことで対処しましょう（[参考](https://askubuntu.com/questions/1329538/compiling-the-kernel-5-11-11)）。


最も手っ取り早い方法としては、すでに用意されているconfigファイルをコピーしてしまうことです。
Rust for LinuxはGithub Actionsを利用してCIをおこなっていて、それのための.configファイルが`.github/workflows`ディレクトリ以下に存在しています。
`.github/workflows/kernel-x86_64-debug.config`がx86_64系アーキテクチャのものでは一番利用しやすいのではないでしょうか。
これを`.config`にコピーしてあげればとりあえずビルドはできます。

`make LLVM=1`などでコンパイルができます。ただし、デフォルトだとビルドが並列化されないので`-j`オプションで並列実行しましょう。
Linuxカーネルのビルドには時間がかかるため、できるだけパフォーマンスの高いPCで実行することをおすすめします。

## ビルドしたカーネルを試す
ビルドしたカーネルを手元で試すには仮想マシン上で動かすのが手っ取り早いです。
Rust for LinuxのCIではQEMUを仮想マシンとして利用して、BusyBoxというソフトウェアと組み合わせることで簡単なテストを実行しています。
BusyBoxは標準UNIXでよく使われるコマンドたちを単一の実行ファイルに詰め込んだもので、標準のLinuxディストリビューションを用意するよりかははるかにコンパクトにLinuxを利用することができます。

BusyBoxは自分でソースコードをダウンロードしてビルドすることになります。

https://busybox.net/source.html

Linux同様、`make menuconfig`などで`.config`ファイルを編集してビルドする必要があります。
このとき、カーネル内でも動くようにライブラリを静的リンクしてあげる必要があるので、`STATIC`を有効にしてあげましょう。
Rust for LinuxのCIで使われているをコピーする手もありますが、必要最低限の機能しか有効化されていないので使えるコマンドが大きく制限されているので、こちらに関しては自分で設定を用意するほうがいいと思います。

Linuxカーネルを起動させるには多くの場合はinitramfsという、初期化処理用に使うファイルシステムをRAM上に構築するという方法を用います。
今回の起動テストでもinitramfsを用意することでQEMU上で動かします。Rust for LinuxのCI上でも同様の処理が行われています。

initramfsを簡単に構築する方法として、Linuxのカーネルツリーに存在する`usr/gen_init_cpio`というツールを利用する方法があります。
これはinitramfs上に含めるファイルを記述した設定ファイルからinitramfsを自動生成する便利なツールです。
自分がテスト用につくった設定ファイルは以下のようになります。

```desc
dir     /bin                                          0755 0 0
dir     /sys                                          0755 0 0
dir     /dev                                          0755 0 0
file    /bin/busybox  busybox/busybox                 0755 0 0
slink   /bin/sh       /bin/busybox                    0755 0 0
file    /init         qemu-init.sh                    0755 0 0
```

`busybox/busybox`というのが自分がビルドしたbuxyboxの実行バイナリへの相対パスになっています。
`/init`というのがカーネルが起動して最初に実行されるファイルです。内容としては単なるシェルスクリプトになっています。

```sh
#!/bin/sh

busybox ls
busybox sh
busybox reboot -f
```

`busybox sh`がbusybox内のシェルを呼び出すコマンドです。このシェルが終了するとrebootするようになっています。
あとは`rust-for-linux/usr/gen_init_cpio initramfs.desc > initramfs.img`などとして、initramfsを生成しましょう。

これで、QEMU実行のための準備は整いました。QEMU自体はUbuntuのaptで入るもので大丈夫です。

```sh
qemu-system-x86_64 \
    -kernel "rust-for-linux/arch/x86_64/boot/bzImage" \
    -initrd qemu-initramfs.img \
    -smp 2 \
    -nographic \
    -vga none \
    -no-reboot \
    -M pc \
    -append console=ttyS0
```

としてあげれば、busyboxのシェルが立ち上がるでしょう。`-kernel`で渡すカーネルのイメージや`-initrd`で渡すinitramfsのパスは適宜変えてください。
`-no-reboot`をつけて、最後のreboot処理が実行されるとqemuが終了するようになっています。また、カーネルパニックなどで再起動がかかった場合も終了します。

## 
