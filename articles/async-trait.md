---
title: "Rust 1.75.0におけるtrait内のasync fn"
emoji: "🐙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

2023年の12月28日にRust 1.75.0がリリースされました

https://blog.rust-lang.org/2023/12/28/Rust-1.75.0.html

今回の更新の目玉の一つが`async fn`をtrait内で使えるようになったことです。詳しい解説が別のブログポストで出ています。

https://blog.rust-lang.org/2023/12/21/async-fn-rpit-in-traits.html

今年の頭に[安定化への道についてブログ](https://blog.rust-lang.org/inside-rust/2023/05/03/stabilizing-async-fn-in-trait.html)が発表されついにここまで来たか、という感じですね。
とはいえ、現段階では様々な制限があるようです。
この記事では現段階でできること・できないことを詳しく見ていこうと思います。

## 今回できるようになったこと
`async fn`がtrait内で使えるようになった、という表現しましたが、正確にはtrait内での関数の戻り値として`-> iml Trait`が使えるようになった、というのが正体です。`async fn`は`-> impl Future`を返す関数のシンタックスシュガーであるため結果的に`async fn`も使えるようになりました。
return-position `impl Trait`` in trait (RPITIT)と呼ばれいているようです。

```rust
trait MyTrait {
    // async fnが使えるようになった
    // fn func1(&self) -> impl Future<Output = bool>; と書いても良い
    async fn func1(&self) -> bool;
    // impl Traitも返せる
    fn func2(&self) -> impl Iterator<Item = &String>;
}

struct MyStruct {
    names: Vec<String>,
}


impl MyTrait for MyStruct {
    async fn func1(&self) -> bool {
        do_async_op().await
    }
    
    fn func2(&self) -> impl Iterator<Item = &String> {
        self.names.iter()
    }
}
```

## できないこと
公式ブログによるとこの`async fn`をpublic traitのメソッドで使うことは非推薦としています。
実際に先ほどのコードの`MyTrait`をpubにすると以下のような警告が出ます

```
  |
3 |     async fn func1(&self) -> bool;
  |     ^^^^^
  |
  = note: you can suppress this lint if you plan to use the trait only in your own code, or do not care about auto traits like `Send` on the `Future`
  = note: `#[warn(async_fn_in_trait)]` on by default
help: you can alternatively desugar to a normal `fn` that returns `impl Future` and add any desired bounds such as `Send`, but these cannot be relaxed without a breaking API change
  |
3 -     async fn func1(&self) -> bool;
3 +     fn func1(&self) -> impl std::future::Future<Output = bool> + Send;
  |
```

`Send` traitは他のスレッドに値を渡すことができることを表すtraitです。この`func1`から返されるFutureが`Send`を実装していないと他のスレッドに移すことができず、work-stealingと言われるタスクがスレッド間を移動するような環境で動作できません。例えば`tokio::task::spawn`は引数に`Future + Send + 'static`を要求しています。

このような問題の対処法として、[trait-variant](https://crates.io/crates/trait-variant)というクレートを使うことを提案しています。
このクレートを用いることで`Send`の制約をつけたtraitとそうでないものを簡単につくることができます。
ドキュメントの例をそのまま引用すると
```rust
#[trait_variant::make(IntFactory: Send)]
trait LocalIntFactory {
    async fn make(&self) -> i32;
    fn stream(&self) -> impl Iterator<Item = i32>;
    fn call(&self) -> u32;
}

```
と書くことで、以下のようなtraitが生成されます。

```rust
trait IntFactory: Send {
   fn make(&self) -> impl Future<Output = i32> + Send;
   fn stream(&self) -> impl Iterator<Item = i32> + Send;
   fn call(&self) -> u32;
}
```

もう一つの制限として、RPITITを使ってしまったtraitはobject safeでなくなり、dynamic dispatchに対応できなくなるとも書かれています。
object safetyについては[公式のドキュメント](https://doc.rust-lang.org/beta/reference/items/traits.html#object-safety)に詳しく書かれていますが、ざっくり言うと実行時にtrait objectをつくることができるかどうかということです。
object safetyを満たしていないtraitは`dyn Trait`としては使えなくなります。
今後、`trait-variant`クレートのアップデートでdynamic dispatchができるようにするユーティリティが提供される予定があるようです。

## まとめ
今回の更新で`async fn`がtrait内で使えるようになりましたが、注意すべき点がいくつかあります。

- `async fn`を公開traitで使う場合、`Send`制約がつかないので`trait-variant`クレートを用いるか`Send`制約を明示的につける
- dynamic dispatchは使えなくなる

現状での制約が煩わしい場合は素直に`async_trait`クレートを使い続けるなど従来の方法を使うのがいいかもしれません。
今回の更新は安定化への道のMVP1に相当するものだと思われるので、今後もアップデートが続き使い勝手がさらに良くなりそうです。今後の更新にも期待しましょう！
