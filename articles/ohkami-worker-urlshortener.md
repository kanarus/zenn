---
title: "Rust の自作 web framework を Cloudflare Workers で動かして URL 短縮サービス を作ってみた"
emoji: "🐺"
type: "tech"
topics: ["rust", "cloudflareworkers", "ohkami"]
published: true
---

:::message

2025-02-24 追記

今や筆者自身も Ohkami もこの記事のときから相当進化しているので、情報が古くなっています。最新情報は

https://github.com/ohkami-rs/ohkami

をご覧ください ( この記事は当時の日記的な意味合いで残しておきます )

:::

## 背景など

HTTP の勉強も兼ねて [ohkami](https://github.com/kana-rus/ohkami) という Rust の web framework を作っていて、以前 yusukebe さんの

https://zenn.dev/yusukebe/articles/8e4e3831070adc

を読んだときにこれは現時点の出来を測るのにちょうどいい題材ではと思って頭にストックしてあったのですが、最近

https://blog.ojisan.io/cf-axum-muriyari/

https://developers.cloudflare.com/workers/languages/rust/

を偶然読んで、これは ohkami 君普通に Workers で動きそうだなと思い立ち~~大学の授業をサボりまくって~~ Workers 対応を行い、dog fooding の一環として URL Shortener を作ってみました。

https://github.com/kana-rus/ohkami-worker-urlshortener

https://ohkami-urlshortener.kanarus.workers.dev/

( これ自身のドメインが長すぎて実用性皆無なのは許してください )


## 開発の流れ

### セットアップ

以下、

- Cloudflare のアカウント
- `npm`
- Rust toolchain
- `wasm32-unknown-unknown` target

があることが前提です。加えて、`wasm-opt` がインストールされていると release build 時に勝手に見つけて使ってくれます。

https://github.com/kana-rus/ohkami-templates

に Workers 用のテンプレートを用意してあるので

```sh
npm create cloudflare ./path/to/project-dir -- --template https://github.com/kana-rus/ohkami-templates/worker

cd ./path/to/project-dir
```

で開発を始められます。( GitHub にリポジトリを作る場合は `wrangler.toml` を `.gitignore` に追加しておきましょう ) 。あとは

```sh
npm run dev
```

でローカルサーバーが立ち上がります。


### Hello, world!

初期状態で `src/lib.rs` は

```rust:src/main.rs
use ohkami::prelude::*;

#[ohkami::worker]
async fn my_worker() -> Ohkami {
    #[cfg(feature = "DEBUG")]
    console_error_panic_hook::set_once();

    Ohkami::new((
        "/".GET(|| async {"Hello, world!"}),
    ))
}
```

となっているはずです。`npm run dev` して `http://localhost:8787` にアクセスすると `Hello, world!` が返ってきます。( `dev` では `DEBUG` feature が有効になるので、Rust の panic を console.error として表示してくれます )

ここからは、ohkami の紹介を兼ねて [yusukebe さんの記事](https://zenn.dev/yusukebe/articles/8e4e3831070adc#%E3%82%B3%E3%83%BC%E3%83%89%E3%82%92%E6%9B%B8%E3%81%8F) をある程度なぞる形で開発の流れを書いてみます。


### HTML のレイアウトを整える

まずこのサービスにおける HTML の rendering についてですが、ohkami には Hono の JSX のような便利なものはないので、適当に crate を持ってきます。ここでは [yarte](https://crates.io/crates/yarte) を使います。

```diff:Cargo.toml
[dependencies]
console_error_panic_hook = { version = "0.1.7", optional = true }

ohkami = { version = "0.17", features = ["rt_worker"] }
worker = { version = "0.1.0" }

+ yarte = { version = "0.15.7" }
```

yarte は `templates/` 以下にテンプレートファイルを置いて

```rust
use yarte::Template;

#[derive(Template)]
#[template(path = "card.html")]
struct Card {
    title: String
}
```

みたいに使うことが推奨されていますが、今回は規模も小さいので `#[template(src = "...")]` で直書きしたほうが見通しがいいと ( 個人的には ) 思います。とはいえ単純に直書きするとそれはそれで微妙なところがあるので、

```rust
macro_rules! page {
    ($name:ident = ($({$( $field:ident: $t:ty ),*})? $(;$semi:tt)?) => $template:literal) => {
        #[derive(Template)]
        #[template(src = $template)]
        pub struct $name $({
            $( pub $field: $t ),*
        })? $($semi)?
    };
}
```

というマクロを用意して JSX 風に書くことにします。

```diff:lib.rs
+ mod pages;
```

```rust:src/pages.rs
use yarte::Template;


macro_rules! page {
    ($name:ident = ($({$( $field:ident: $t:ty ),*})? $(;$semi:tt)?) => $template:literal) => {
        #[derive(Template)]
        #[template(src = $template)]
        pub struct $name $({
            $( pub $field: $t ),*
        })? $($semi)?
    };
}

page!(Layout = ({ content: String }) => r#"<!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <link rel="stylesheet" href="https://fonts.xz.style/serve/inter.css" />
        <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@exampledev/new.css@1.1.2/new.min.css" />
        <title>URL Shortener</title>
    </head>
    <body>
        <header>
            <h1>
                <a href="/">URL Shortener</a>
            </h1>
        </header>
        <div>{{{ content }}}</div>
    </body>
    </html>
"#);
```

( `{{ }}` で通常の埋め込み、`{{{ }}}` でエスケープされない埋め込みになります )

そしてレイアウトの適用は、ohkami のミドルウェアシステムである fangs で実現します。
`LayoutFang` という struct に「 レスポンスボディが HTML だったら `LayoutPage` に包む 」という挙動を実装し、`Ohkami::with` で渡します。

```diff:lib.rs
+ mod fangs;

+ use fangs::LayoutFang;

use ohkami::prelude::*;


#[ohkami::worker]
async fn my_worker() -> Ohkami {
    #[cfg(feature = "DEBUG")]
    console_error_panic_hook::set_once();

-   Ohkami::new((
+   Ohkami::with(LayoutFang, (
        "/".GET(|| async {"Hello, world!"}),
    ))
}
```

```rust:fangs.rs
use ohkami::prelude::*;
use yarte::Template;
use crate::pages::Layout;


#[derive(Clone)]
pub struct LayoutFang;
impl FangAction for LayoutFang {
    async fn back<'a>(&'a self, res: &'a mut Response) {
        if res.headers.ContentType().is_some_and(|ct| ct.starts_with("text/html")) {
            let content = res.drop_content()
                .map(|bytes| String::from_utf8(bytes.into_owned()).unwrap())
                .unwrap_or_else(String::new);
            *res = match (Layout { content }.call()) {// ← Template::call
                Ok(html) => Response::OK().with_html(html),
                Err(err) => //
            };
        }
    }
}
```

ここでレンダリングエラーをハンドリングしたいので、エラー型を用意します。

```diff:lib.rs
+ mod errors;

+ use errors::AppError;
```

```rust:errors.rs
use ohkami::{Response, IntoResponse};


pub enum AppError {
    RenderingHTML(yarte::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        match self {
            Self::RenderingHTML(err) => {
                worker::console_error!("Failed to render HTML: {err}");
                Response::InternalServerError()
            }
        }
    }
}
```

```diff:fangs.rs
+ use crate::AppError;

〜

- Err(err) => 
+ Err(err) => AppError::RenderingHTML(err).into_response(),

〜
```


### トップページを作る

```diff:pages.rs
+ page!(IndexPage = (;;) => r#"
+     <div>
+         <h2>Create shorten URL!</h2>
+         <form action="/create" method="post">
+             <input
+                 autofocus
+                 type="text"
+                 name="url"
+                 autocomplete="off"
+                 style="width: 80%;"
+             />
+             &nbsp;
+             <button type="submit">Create</button>
+         </form>
+     </div>
+ "#);
```

```rust:lib.rs
mod errors;
mod fangs;
mod pages;

use errors::AppError;
use fangs::LayoutFang;

use ohkami::prelude::*;


#[ohkami::worker]
async fn my_worker() -> Ohkami {
    #[cfg(feature = "DEBUG")]
    console_error_panic_hook::set_once();

    Ohkami::with(LayoutFang, (
        "/".GET(index),
    ))
}

async fn index() -> Result<String, AppError> {
    use yarte::Template;

    match (pages::IndexPage).call() {
        Ok(html) => Response::OK().with_html(html),
        Err(err) => //
    }
}
```

ここまで来ると各 page が `IntoResponse` を実装しているべきなのは明らかなので、リファクタリングしておきましょう。

```diff:pages.rs
+ use crate::AppError;
use yarte::Template;
+ use ohkami::{IntoResponse, Response};


macro_rules! page {
    ($name:ident = ($({$( $field:ident: $t:ty ),*})? $(;$semi:tt)?) => $template:literal) => {
        #[derive(Template)]
        #[template(src = $template)]
        pub struct $name $({
            $( pub $field: $t ),*
        })? $($semi)?

+       impl IntoResponse for $name {
+           fn into_response(self) -> Response {
+               match self.call() {
+                   Ok(html) => Response::OK().with_html(html),
+                   Err(err) => AppError::RenderingHTML(err).into_response(),
+               }
+           }
+       }
    };
}
```

```diff:fangs.rs
            let content = res.drop_content()
                .map(|bytes| String::from_utf8(bytes.into_owned()).unwrap())
                .unwrap_or_else(String::new);
-           *res = match (Layout { content }.call()) { /* Template::call */
-               Ok(html) => Response::OK().with_html(html),
-               Err(err) => 
-           };
+           *res = Layout { content }.into_response();
```

```rust:lib.rs
async fn index() -> pages::IndexPage {
    pages::IndexPage
}
```

これで `http://localhost:8787` にアクセスすると、`LayoutFang` が効いて完全な HTML が返ってくることが確認できると思います。


### `/create` を作る

```diff:pages.rs
+ page!(CreatedPage = ({ shorten_url: String }) => r#"
+     <div>
+         <h2>Created!</h2>
+         <a href="{{ shorten_url }}">
+             {{ shorten_url }}
+         </a>
+     </div>
+ "#);
```

```rust:lib.rs
〜

+ use ohkami::typed::Payload;
+ use ohkami::builtin::payload::URLEncoded;
+ use std::borrow::Cow;

〜

#[ohkami::worker]
async fn my_worker() -> Ohkami {
    #[cfg(feature = "DEBUG")]
    console_error_panic_hook::set_once();

    Ohkami::with(LayoutFang, (
        "/"
            .GET(index),
        "/create"
            .POST(create),
    ))
}

〜

#[Payload(URLEncoded)]
#[derive(ohkami::serde::Deserialize)]
struct CreateShortenURLForm<'req> {
    url: Cow<'req, str>,
}

async fn create(
    env:  &worker::Env,
    form: CreateShortenURLForm<'_>,
) -> Result<pages::CreatedPage, AppError> {
    // worker::Url を借りてきて URL のバリデーション
    if let Err(_) = worker::Url::parse(&form.url) {
        return Err(AppError::Validation(
            String::from("invalid URL")
        ))
    }

    todo!()
}
```

```diff:errors.rs
pub enum AppError {
    RenderingHTML(yarte::Error),
+   Validation(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        match self {
            Self::RenderingHTML(err) => {
                worker::console_error!("Failed to render HTML: {err}");
                Response::InternalServerError()
            }
+           Self::Validation(msg) => {
+               worker::console_error!("Validation failed: {msg}");
+               Response::BadRequest()
+           }
        }
    }
}
```

ハンドラーは基本的に `FromRequest<'_>` を実装している型の値を引数にとります。`&'_ worker::Env` には ohkami 内部で `FromRequest<'_>` を実装してあり、`CreateShortenURLForm` は `Payload` と `Deserialize` を実装しているため `FromRequest` が実装されます。


:::details typed::Payload について

payload としての振る舞いを構造体自体に持たせるためのシステムです。

「 構造体自体に 」というのが何を念頭に置いた表現かというと、例えば axum では payload は

```rust
#[derive(Deserialize)]
struct CreateUser {
    email: String,
    password: String,
}

async fn create_user(extract::Json(payload): extract::Json<CreateUser>) {
    // payload is a `CreateUser`
}
```
( https://docs.rs/axum/0.7.5/axum/struct.Json.html から引用 )

みたいな感じで扱うのが通例ですが、これだと `CreateUser` は `Json` 以外の extractor で包めば `application/json` 以外の payload にも普通に流用できます。が、フレームワークとしてそれはどうなんでしょうか？

- 複数の形式のリクエストボディを受け付ける
- クライアントがクエリパラメータ等でレスポンスボディの形式を指定できる

というような場合を除き、

「 ある構造体が payload として扱われるときの形式は、その構造体自身が知っている 」

のが健全ではないでしょうか？

この視点からすると、`Json` という形式を `CreateUser` の外からはめ込むのではなく

```rust
#[derive(Deserialize)]
struct CreateUser {
    email: String,
    password: String,
}

impl Payload for CreateUser {
    type Format = Json;
}
```

みたいに payload としての形式を associated type として持たせ、payload としての振る舞い (

- リクエストボディからのデシリアライズ処理
- レスポンスボディとしてのシリアライズ処理

) はその assiciated type が知っている、という形にすると、まさにちょうど欲しい構造を型で表せています。これを１行でやってくれるのが `#[Payload( 〜 )]` です。

<br>

( axum の上に同じものを作ることもできますが、ohkami はこれを builtin で推奨しているということが大事だと思っています )

:::


なので上記の `create` の中で引数の `env` を使って KV にアクセスできます。

( KV の準備については yusukebe さんの記事に譲ります )

ところが、試してみるとわかるのですが、`worker::Env` からアクセスできる `worker::kv::KvStore` も関連するエラー型の `worker::kv::KvError` も `Send` でなくそのままでは扱いづらいので、ラッパーを作った方がよさそうです。`KvStore` をラップした `KV` 型を `models` という module に 定義します。ついでにこのタイミングで `CreateShortenURLForm`, `IndexPage`, `CreatedPage` を `models` から export する形にしておきます。

:::message
**追記 / 当時の自分へのヒント**

- ハンドラーに `#[worker::send]` つければ `SendWrapper` とかいらない
- `AppError` に `From<worker::Error>` と `From<KvError>` を impl しておけば各所で `.map_err(AppError::〜)` を書きまくらなくてよくなる

以上を合わせると `KV` などというラッパーも必要なく、とてもすっきりする ( https://github.com/kana-rus/ohkami-worker-urlshortener )
:::

```diff:lib.rs
+ mod models;
```

```rust:models.rs
use ohkami::{Response, IntoResponse, Request, FromRequest};
use ohkami::{typed::Payload, builtin::payload::URLEncoded};
use worker::send::{SendFuture, SendWrapper};
use worker::kv::{KvStore, ToRawKvValue};
use std::{borrow::Cow, future::Future};
use crate::{pages, AppError};


pub use pages::IndexPage;

pub use pages::CreatedPage;

#[Payload(URLEncoded/D)] // Payload + Deserialize の shorthand
#[derive(Debug)]
pub struct CreateShortenURLForm<'req> {
    pub url: Cow<'req, str>,
}

// KvStore が Send でないので SendWrapper で包む
pub struct KV(SendWrapper<KvStore>);
impl<'req> FromRequest<'req> for KV {
    type Error = AppError;
    fn from_request(req: &'req Request) -> Option<Result<Self, Self::Error>> {
        Some(req.env().kv("KV").map_err(AppError::Worker)
            .map(|kv| Self(SendWrapper(kv)))
        )
    }
}
impl KV {
    // このサービスではテキストの value しか扱わないので .text() 決め打ち
    // 
    // .text().await の部分で出る KvError が Send でないので全体を SendFuture で包んで返す
    pub fn get<'kv>(&'kv self,
        key: &'kv str,
    ) -> impl Future<Output = Result<Option<String>, AppError>> + Send + 'kv {
        SendFuture::new(async move {
            self.0.get(key)
                .text().await
                .map_err(AppError::kv)
        })
    }

    // .execute().await の部分で出る KvError が Send でないので全体を SendFuture で包んで返す
    pub fn put<'kv>(&'kv self,
        key:   &'kv str,
        value: impl ToRawKvValue + 'kv,
    ) -> impl Future<Output = Result<(), AppError>> + Send + 'kv {
        SendFuture::new(async move {
            self.0.put(key, value).unwrap()
                .execute().await.map_err(AppError::kv)
        })
    }
}
```

```diff:errors.rs
+ use worker::send::SendWrapper;


// AppError は Send であってほしいが
// KvError が Send でないので SendWrapper で包む
pub enum AppError {
    RenderingHTML(yarte::Error),
    Validation(String),
+   KV(SendWrapper<worker::kv::KvError>),
}

// 毎回 AppError::KV(SendWrapper( 〜 )) とするのは面倒なので
// ショートカットを用意
+ impl AppError {
+     pub fn kv(kv_error: worker::kv::KvError) -> Self {
+         Self::KV(SendWrapper(kv_error))
+     }
+ }

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        match self {
            Self::RenderingHTML(err) => {
                worker::console_error!("Failed to render HTML: {err}");
                Response::InternalServerError()
            }
            Self::Validation(msg) => {
                worker::console_error!("Validation failed: {msg}");
                Response::BadRequest()
            }
+           Self::KV(err) => {
+               worker::console_error!("Error from KV: {err}");
+               Response::InternalServerError()
+           }
        }
    }
}
```

これで、`lib.rs` に `use models::{IndexPage, CreatedPage, CreateShortenURLForm, KV};` を追加してこんな感じにできます：

```rust:lib.rs
〜

const ORIGIN: &str = if cfg!(feature = "DEBUG") {
    "http://localhost:8787"
} else {
    "https://＜worker name＞.＜workers subdomain＞"

    // 冒頭のリポジトリでは
    // "https://ohkami-urlshortener.kanarus.workers.dev"
};

〜

async fn index() -> IndexPage {
    IndexPage
}

async fn create(
    kv:   KV,
    form: CreateShortenURLForm<'_>,
) -> Result<CreatedPage, AppError> {
    if let Err(_) = worker::Url::parse(&form.url) {
        return Err(AppError::Validation(
            String::from("invalid URL")
        ))
    }

    let key = loop {
        let key = std::sync::Arc::new(
            /* uuid の左から６文字 */
        );
        if kv.get(&*key).await?.is_none() {
            break key
        }
    };

    kv.put(&*key.clone(), form.url).await?;

    Ok(CreatedPage {
        shorten_url: format!("{ORIGIN}/{key}")
    })
}
```

さて `/* uuid の左から６文字 */` の部分ですが、

https://x.com/kana_rus/status/1782403152534528339

https://x.com/kana_rus/status/1782446441144959388

ということで、`wasm_bindgen` で JavaScript に救いを求めましょう。

```diff:Cargo.toml
[dependencies]
console_error_panic_hook = { version = "0.1.7", optional = true }

ohkami = { version = "0.17", features = ["rt_worker"] }
worker = { version = "0.1.0" }

yarte          = { version = "0.15.7" }
+ wasm-bindgen = { version = "0.2.92" }
```

```diff:lib.rs
+ mod js;
```

```rust:js.rs
use wasm_bindgen::prelude::wasm_bindgen;


#[wasm_bindgen(js_namespace = crypto)]
extern "C" {
    pub fn randomUUID() -> String;
}
```

```rust:lib.rs
async fn create(
    kv:   KV,
    form: CreateShortenURLForm<'_>,
) -> Result<CreatedPage, AppError> {
    if let Err(_) = worker::Url::parse(&form.url) {
        return Err(AppError::Validation(
            String::from("invalid URL")
        ))
    }

    let key = loop {
        let key = std::sync::Arc::new({
            let mut uuid = js::randomUUID();

            // trancate が好きなので
            unsafe { uuid.as_mut_vec().trancate(6) }
            // 
            // while uuid.len() > 6 {uuid.pop();}
            // とかが普通 (？)

            uuid
        });
        if kv.get(&*key).await?.is_none() {
            break key
        }
    };

    kv.put(&*key.clone(), form.url).await?;

    Ok(CreatedPage {
        shorten_url: format!("{ORIGIN}/{key}")
    })
}
```

:::message
**追記 / 当時の自分へのヒント**
`String::trancate` あるよ ( しかも unsafe じゃないよ )
:::

### リダイレクトさせる

```diff:lib.rs
〜

+ use ohkami::typed::status;


#[ohkami::worker]
async fn my_worker() -> Ohkami {
    #[cfg(feature = "DEBUG")]
    console_error_panic_hook::set_once();

    Ohkami::with(LayoutFang, (
        "/"
            .GET(index),
        "/create"
            .POST(create),
+       "/:shorten_url"
+           .GET(redirect_from_shorten_url),
    ))
}

〜

+ async fn redirect_from_shorten_url(shorten_url: &str,
+     kv: KV,
+ ) -> Result<status::Found, AppError> {
+     match kv.get(shorten_url).await? {
+         Some(url) => Ok(status::Found::at(url)),
+         None      => Ok(status::Found::at("/")),
+     }
+ }
```

:::message
**追記 / 当時の自分へのヒント**
`:shorten_url` じゃなくて `:key` とかの方が自然 ( uuid の先頭６文字が渡ってくるわけなので "url" じゃないよね )
:::

ハンドラーの最初の引数が `FromParam` を実装している型もしくはそのタプル型である場合に、ohkami はそれをパスパラメータと解釈し、ルーティング時にマッチしたパラメータを渡します。


:::details typed::status について

主に正常系レスポンスが１種類のハンドラーのレスポンスを型レベルでうまく表現するためのシステムです。

axum でよく

```rust
async fn with_status(uri: Uri) -> (StatusCode, String) {
    (StatusCode::NOT_FOUND, format!("Not Found: {}", uri.path()))
}
```
( https://docs.rs/axum/0.7.5/axum/response/index.html から引用 )

みたいなハンドラーが書かれますが、

```rust
async fn with_status(uri: Uri) -> NotFound<String> {
    NotFound(format!("Not Found: {}", uri.path()))
}
```

と比べてどっちが好きですか？ 僕は圧倒的に後者です。シグネチャを見ただけで何が返ってくるかよく分かって良いですよね。

<br>

( axum の上に同じものを作ることもできますが、ohkami はこれを builtin で推奨しているということが大事だと思っています )

:::


### エラー処理をする

`create` の `if let Err(_) = worker::Url::parse(&form.url) { 〜 }` のところでエラーページを返したいので、まず page を作ります。

```diff:pages.rs
+ page!(ErrorPage = (;;) => r#"
+     <div>
+         <h2>Error!</h2>
+         <a href="/">Back to top</a>
+     </div>
+ "#);
```

そして `CreatedPage` と `ErrorPage` を出し分けるための enum を用意しましょう。

```diff:models.rs
- pub use pages::CreatedPage;

+ pub enum CreatedOrErrorPage {
+     Created { shorten_url: String },
+     Error,
+ }
+ impl IntoResponse for CreatedOrErrorPage {
+     fn into_response(self) -> Response {
+         match self {
+             Self::Created { shorten_url } => pages::CreatedPage { shorten_url }.into_response(),
+             Self::Error => pages::ErrorPage.into_response(),
+         }
+     }
+ }
```

あとは `lib.rs` で `use` して

```rust
async fn create(
    kv:   KV,
    form: CreateShortenURLForm<'_>,
) -> Result<CreatedOrErrorPage, AppError> {
    if let Err(_) = worker::Url::parse(&form.url) {
        return Ok(CreatedOrErrorPage::Error)
    }

    let key = loop {
        let key = std::sync::Arc::new({
            let mut uuid = js::randomUUID();
            unsafe { uuid.as_mut_vec().truncate(6) }
            uuid
        });
        if kv.get(&*key).await?.is_none() {
            break key
        }
    };

    kv.put(&key.clone(), form.url).await?;
    
    Ok(CreatedOrErrorPage::Created {
        shorten_url: format!("{ORIGIN}/{key}"),
    })
}
```

とすれば URL でない入力に対してエラーページを返せます。

:::message
それはセンス悪くて、単に `AppError::Valudation::into_response` で `ErrorPage` を返すようにすればいいだけ。ハンドラーの返り値はあくまで

```rust
Result<CreatedPage/* 正常系 */, AppError/* 異常系 */>
```

となっているべき
:::


### CSRFプロテクターを入れる

現在 ohkami には builtin の CSRF fang はないので、ひとまず

```diff:fangs.rs
+ #[derive(Clone)]
+ pub struct CSRFang;
+ impl FangAction for CSRFang {
+     async fn fore<'a>(&'a self, req: &'a mut Request) -> Result<(), Response> {
+         let origin = req.headers.Origin()
+             .ok_or_else(|| Response::BadRequest())?;
+         (origin == crate::ORIGIN)
+             .then_some(())
+             .ok_or_else(|| {
+                 worker::console_warn!("Unexpected request from {origin}");
+                 Response::Forbidden()
+             })
+     }
+ }
```

のようにしておきます。`lib.rs` で `use` して

```rust
#[ohkami::worker]
async fn my_worker() -> Ohkami {
    #[cfg(feature = "DEBUG")]
    console_error_panic_hook::set_once();

    Ohkami::with(LayoutFang, (
        "/"
            .GET(index),
        "/:shorten_url"
            .GET(redirect_from_shorten_url),
        "/create".By(Ohkami::with(CSRFang,
            "/".POST(create),
        ))
    ))
}
```

で `/create` 以下へのリクエストに `CSRFang` が発動するようになります。


## まとめ

読んでいただきありがとうございます。

おそらくこの記事を読んだ方のほとんどが ohkami を初めて見たと思うのですが、どう感じたでしょうか？ 他のフレームワークに比べて書いていて楽しそうと思っていただけたら幸いです。

元々は actix-web や axum のルーティングを初めて見て「 うーん ~~...ダサくね？~~ 」と思って色々と勉強しながら作り始めたフレームワークで、幾度となく根本的な書き直しを経て少しずつまともになり、今や少なくとも Cloudflare Workers で普通に動くところまで来ました。
まだまだ大きな課題が色々とありますが、今後も成長していく予定なので、気に入った方はぜひスターを ...！🐺

https://github.com/kana-rus/ohkami

この記事の実装は誰でも無限に KV を叩けるなど元の記事で[宿題](https://zenn.dev/yusukebe/articles/8e4e3831070adc#%E3%81%BE%E3%81%A8%E3%82%81)とされている点もそのままなので、気が向いた方は

https://github.com/kana-rus/ohkami-worker-urlshortener

を clone してそのあたりに手を入れてみるのも面白いかもしれません。
