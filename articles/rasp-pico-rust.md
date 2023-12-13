---
title: "RustでRaspberry Pi Picoのベアメタルプログラミング"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "Raspberry-Pi-Pico", "embedded"]
published: true
---

これは[Rust Advent Calendar 2023](https://qiita.com/advent-calendar/2023/rust)の10日目の記事です。

## Raspberry Pi Picoとは

https://www.raspberrypi.com/products/raspberry-pi-pico/

Raspberry Pi PicoはRaspberry Pi財団が開発しているマイコンボードで、よく知られているRaspberry Piとは異なり、Cortex-M0+という非常に小さいアーキテクチャのCPUを搭載しています。
具体的には通常のCortex-AシリーズにあったようなCPUモードがなく、MMUのようなページング機構もないため、Raspberry Piとは異なり、Linuxを動かすことはできません。

しかしながら、その分非常に省電力で動作し価格も安価です（秋月電子通商でこのブログ執筆時点で770円、無線機能がついたPico Wで1200円）。
ちょっとしたガジェットや自作キーボードのコントローラーに使う例などを見かけます。

## RustでPicoを動かす
最もお手軽にPicoを動かしたいなら公式の出しているC/C++のSDKを使うのが楽です

https://github.com/raspberrypi/pico-sdk

Rustでもお手軽に動かすライブラリとして、[rp-pico](https://crates.io/crates/rp-pico)というものがあります。
これはRust Embeddedチームが開発した[cortex-m-rt](https://crates.io/crates/cortex-m-rt)をベースにして、ランタイムの提供と各種ペリフェラルのラッパーを提供しています。

しかし今回はランタイムを自作したいと思ったので、これらの既存のライブラリをなるべく使わずにPicoを動かしてみました。

## ブートローダー
Picoのブートプロセスはやや複雑で、Second Stage bootloaderというものを用いて自分のプログラムを起動する必要があります。
Second stage bootloaderは公式のSDKでも提供されていて、さらにこのブートローダーをRust向けにラップした[rp2040_boot2](https://github.com/rp-rs/rp2040-boot2)というものもあります。

今回作りたいのはランタイムであって、ブートプロセスについてはこちらのブートローダーを使うことにします。

ブートローダーまで完璧に自作したい方はRP2040のデータシートの2.8章を参照すると起動シーケンスについての解説があるのでそちらを参照してください。

## ランタイム
ブートさえできれば、ランタイムの自作は[The embedonomicon](https://docs.rust-embedded.org/embedonomicon/)や拙著の[Rustで始める自作組込みOS入門](https://nextpublishing.jp/book/14912.html)どおりにやればできます。

ただし、今回はPicoのブートローダーを組み込む必要があるのでまずリンカスクリプトにブートローダー用の領域を定義する必要があります。

```ld
MEMORY
{
  /* ブートローダー用の領域 */
  BOOT_LOADER : ORIGIN = 0x10000000, LENGTH = 0x100
  FLASH : ORIGIN = 0x10000100, LENGTH = 2048K - 0x100
  RAM : ORIGIN = 0x20000000, LENGTH = 256K
}

EXTERN(Reset);
ENTRY(Reset);

SECTIONS {

  /* ### Boot loader */
  .boot_loader ORIGIN(BOOT_LOADER) :
  {
    KEEP(*(.boot_loader*));
  } > BOOT_LOADER

  .vector_table ORIGIN(FLASH):
  {
  /* 以下省略 */
```

また、ライブラリからブートローダーをリンクさせるために以下をmain.rsに追加します。

```rust
#[link_section = ".boot_loader"]
#[used]
pub static BOOT_LOADER: [u8; 256] = rp2040_boot2::BOOT_LOADER_W25Q080;
```

## GPIO
プログラムを書き込んで動作確認するためにLEDを点滅させるプログラムを書きます。
LEDはGPIOを操作することで制御できます。25番ピンがボード内臓のLEDに接続されていますが、Pico Wの場合はこのLEDが無線モジュールに接続されているため制御がやや難しいです。
今回自分はPico Wを使うため、別の外付けのGPIOピンにLEDを接続して動作確認しました。

GPIOを使うには以下の操作が必要です

1. Subsystem ResetによりIO_BANK0とPADS_BANK0をリセットしてリセットが完了するまで待つ
2. IO_BANK0の対応するピンのCTRLレジスタでfunction selectを設定する
3. PADS_BANK0の対応するピンのODビッド（output disable）をクリアする
4. SIOのGPIO_OEレジスタで対応するピンを出力を有効にする

PADS_BANK0のODビットはリセット時にクリアはされているはずなのですが、念の為におこなっています
この設定が終わったあとはSIOのGPIO_OUTレジスタを操作することで出力を制御できます。

## 書き込み
[`elf2uf2-rs`](https://crates.io/crates/elf2uf2-rs)というライブラリを使うと便利です。uf2というフォーマットに変換してRaspberry Pi Picoで実行できます。
`.cargo/config`にrunnerを設定しておくと`cargo run`でuf2ファイルを書き込むことができます。

```toml
[target.'cfg(all(target_arch = "arm", target_os = "none"))']
runner = "elf2uf2-rs -d"
rustflags = [
    "-C", "link-arg=-Tlink.ld",
]


[build]
target = "thumbv6m-none-eabi"
```

また、`openocd`経由で書き込むことも出来ます。公式のフォークしたopenocdにはPico用の設定ファイルが含まれています。

https://github.com/raspberrypi/openocd/tree/rp2040-v0.12.0

こちらを以下のようにビルドします

```sh
./bootstrap
./configure --enable-ftdi --enable-sysfsgpio --enable-bcm2835gpio --enable-picoprobe
make
sudo make install
```

こうすると`/usr/local/bin/openocd`以下にインストールされるので以下のコマンドで接続できます。
```sh
/usr/local/bin/openocd -f interface/cmsis-dap.cfg -f target/rp2040.cfg -s tcl 
```

Picoをデバッガにつなぐ方法はもう一台のPicoをデバッガとして使う方法が有名でよく使われているようです。

https://www.raspberrypi.com/documentation/microcontrollers/raspberry-pi-pico.html#debugging-using-another-raspberry-pi-pico


## コード
以下が今回作成したコードです。Systickを使って1秒ごとにLEDを点滅させています。

2023-12-13 追記: 初期化コードが間違っていたので修正しました

https://github.com/garasubo/my-pico-test/tree/blog-v1.1


## おわりに
本当はprobe-rsとかも使う方法を模索したかったんだけどよくわからなかったにょ

## 参考

https://mickey-happygolucky.hatenablog.com/entry/2021/02/26/122623

https://github.com/dwelch67/raspberrypi-pico
