---
title: "自作 Web Framework が AWS Lambda で動くようになった (Experimental Support)"
emoji: "🐺"
type: "tech"
topics: ["Rust", "Ohkami", "AWS", "Lambda"]
published: true
---

Ohkami v0.22 を出しました。

https://github.com/ohkami-rs/ohkami

native な async runtime ( `tokio`, `async-std`, `smol`, `glommio`, `nio` ) と Cloudflare Workers ( `worker` ) に加えて、今回 AWS Lambda を試験的にサポートしたので紹介します。

## できること

* Function URLs での動作
* API Gateway での動作
* 呼び出しモード `BUFFERED` でのレスポンス
* 呼び出しモード `RESPONSE_STREAM` でのストリーミング

Lambda の仕様上、native runtime や Cloudflare Workers とは違い、ストリーミングするエンドポイントとそうでないエンドポイントと併置することはできません。

( 厳密にいうと、`RESPONSE_STREAM` な Lambda ではストリーミングしないエンドポイントは

```txt
{"statusCode":200,"headers":{"Content-Type":"text/plain; charset=UTF-8","Content-Length":"18","Date":"Sat, 08 Feb 2025 06:26:09 GMT"},"body":"Hello, AWS Lambda!","isBase64Encoded":false}
```

みたいなチャンク１つだけを持つ `Content-Type: application/octet-stream` なストリーミングレスポンスを返すので、これをクライアント側でパースする形でよければ使えます )

## できないこと

* API Gateway の WebSocket API による WebSocket ハンドリング

これはちょっと特殊すぎるのでインターフェースに悩み、今のところ何も提供しないことにしました ( そもそも Lambda で WebSocket やる需要ってどれくらいあるんですかね？ ) 。

## つかいかた

README に書いてある通り、[`lambda_runtime`](https://crates.io/crates/lambda_runtime) というクレートに乗っかります。

https://github.com/awslabs/aws-lambda-rust-runtime

そして [`cargo lambda`](https://www.cargo-lambda.info/guide/installation.html) という CLI を使うのがおすすめで、用意してある [プロジェクトテンプレート](https://github.com/ohkami-rs/ohkami-templates/tree/main/template) もそれを前提にしています。

https://www.cargo-lambda.info

<br>

```sh
cargo lambda new ＜project dir＞ --template https://github.com/ohkami-rs/ohkami-templates
```

でプロジェクトを作成できます。main.rs は

```rust
use ohkami::prelude::*;

#[tokio::main]
async fn main() -> Result<(), lambda_runtime::Error> {
    let o = Ohkami::new((
        "/".GET(|| async {"Hello, AWS Lambda!"}),
    ));

    lambda_runtime::run(o).await
}
```

となっていて、普通の `Ohkami` インスタンスを `lambda_runtime::run` に渡すだけです。

これを

```sh
cargo lambda build ＜flags＞
cargo lambda deploy ＜flags＞
```

するとデプロイできます。詳細はプロジェクトテンプレートの README を見てください。

## サンプル

https://github.com/kanarus/ohkami-lambda-hello-sample

２月８日現在 `https://ddhryperz4xvp6hji4bhxcinhu0qesbi.lambda-url.ap-northeast-1.on.aws` で動かしてます。

https://github.com/kanarus/ohkami-lambda-streaming-sample

２月８日現在 `https://75p6fhc2rwn3yefkmvhiyerely0xhhqx.lambda-url.ap-northeast-1.on.aws` で動かしてます ( `curl` で叩く場合 `--no-buffer` をつけるとよいです ) 。
