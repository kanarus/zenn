---
title: "[Rust] Ohkami の OpenAPI ドキュメント生成アプローチ"
emoji: "🐺"
type: "tech"
topics: ["Rust", "Ohkami", "OpenAPI"]
published: true
---

https://github.com/ohkami-rs/ohkami

Ohkami が v0.21 から提供している OpenAPI ドキュメント生成機能がマクロに頼らない独自のアプローチをとっているので紹介します。


## 具体例

例として以下のコードを考えます。 README の openapi の例から OpenAPI 要素を抜いたもので、適当ユーザー API サーバーです。

```rust:main.rs
use ohkami::prelude::*;
use ohkami::typed::status;

#[derive(Deserialize)]
struct CreateUser<'req> {
    name: &'req str,
}

#[derive(Serialize)]
struct User {
    id: usize,
    name: String,
}

async fn create_user(
    JSON(CreateUser { name }): JSON<CreateUser<'_>>
) -> status::Created<JSON<User>> {
    status::Created(JSON(User {
        id: 42,
        name: name.to_string()
    }))
}

async fn list_users() -> JSON<Vec<User>> {
    JSON(vec![])
}

#[tokio::main]
async fn main() {
    let o = Ohkami::new((
        "/users"
            .GET(list_users)
            .POST(create_user),
    ));

    o.howl("localhost:5000").await;
}
```

`openapi` feature を無効にするとこれは普通にビルドが通り、適当ユーザー API サーバーとして動きます。

対して、`openapi` feature を有効にすると上記のコードではコンパイルエラーが出るようになります。具体的には、`User` と `CreateUser` が `ohkami::openapi::Schema` を実装していないということで怒られるはずです。

このように、Ohkami の `openapi` feature では型情報をうまく使うことでマクロに頼ることなく `Ohkami` インスタンスがエンドポイントのメタデータを把握できるようになっていて、最終的に

```rust:main.rs
use ohkami::openapi;

...

let o = Ohkami::new((
    "/users"
        .GET(list_users)
        .POST(create_user),
));

o.generate(openapi::OpenAPI {
    title: "Users Server",
    version: "0.1.0",
    servers: &[openapi::Server::at("localhost:5000")],
});
```

のように `Ohkami::generate` を呼ぶだけでメタデータを統合してファイルに吐けます。

その `Schema` ですが、自分で impl してもいいですし、この場合 derive で対応できます：

```diff:main.rs
- #[derive(Deserialize)]
+ #[derive(Deserialize, openapi::Schema)]
  struct CreateUser<'req> {
      name: &'req str,
  }

- #[derive(Serialize)]
+ #[derive(Serialize, openapi::Schema)]
  struct User {
      id: usize,
      name: String,
  }
```

これだけで、`Ohkami::generate` は以下のようなファイルを生成します：

```json:openapi.json
{
  "openapi": "3.1.0",
  "info": {
    "title": "Users Server",
    "version": "0.1.0"
  },
  "servers": [
    {
      "url": "localhost:5000"
    }
  ],
  "paths": {
    "/users": {
      "get": {
        "operationId": "list_users",
        "responses": {
          "200": {
            "description": "OK",
            "content": {
              "application/json": {
                "schema": {
                  "type": "array",
                  "items": {
                    "type": "object",
                    "properties": {
                      "id": {
                        "type": "integer"
                      },
                      "name": {
                        "type": "string"
                      }
                    },
                    "required": [
                      "id",
                      "name"
                    ]
                  }
                }
              }
            }
          }
        }
      },
      "post": {
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "properties": {
                  "id": {
                    "type": "integer"
                  },
                  "name": {
                    "type": "string"
                  }
                },
                "required": [
                  "id",
                  "name"
                ]
              }
            }
          }
        },
        "responses": {
          "201": {
            "description": "Created",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "id": {
                      "type": "integer"
                    },
                    "name": {
                      "type": "string"
                    }
                  },
                  "required": [
                    "id",
                    "name"
                  ]
                }
              }
            }
          }
        }
      }
    }
  }
}

```

さらに、使い回したい `User` は、derive の場合

```diff:main.rs
  #[derive(Serialize)]
  #[derive(Serialize, openapi::Schema)]
+ #[openapi(component)]
  struct User {
      id: usize,
      name: String,
  }
```

とすることで `component` として認識され、生成ファイルは以下のようになります：

```diff:openapi.json
{
  "openapi": "3.1.0",
  "info": {
    "title": "Users Server",
    "version": "0.1.0"
  },
  "servers": [
    {
      "url": "localhost:5000"
    }
  ],
  "paths": {
    "/users": {
      "get": {
        "operationId": "list_users",
        "responses": {
          "200": {
            "description": "OK",
            "content": {
              "application/json": {
                "schema": {
                  "type": "array",
                  "items": {
-                   "type": "object",
-                   "properties": {
-                     "id": {
-                       "type": "integer"
-                     },
-                     "name": {
-                       "type": "string"
-                     }
-                   },
-                   "required": [
-                     "id",
-                     "name"
-                   ]
+                   "$ref": "#/components/schemas/User"
                  }
                }
              }
            }
          }
        }
      },
      "post": {
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "properties": {
                  "id": {
                    "type": "integer"
                  },
                  "name": {
                    "type": "string"
                  }
                },
                "required": [
                  "id",
                  "name"
                ]
              }
            }
          }
        },
        "responses": {
          "201": {
            "description": "Created",
            "content": {
              "application/json": {
                "schema": {
-                 "type": "object",
-                 "properties": {
-                   "id": {
-                     "type": "integer"
-                   },
-                   "name": {
-                     "type": "string"
-                   }
-                 },
-                 "required": [
-                   "id",
-                   "name"
-                 ]
+                 "$ref": "#/components/schemas/User"
                }
              }
            }
          }
        }
      }
    }
  },
+ "components": {
+   "schemas": {
+     "User": {
+       "type": "object",
+       "properties": {
+         "id": {
+           "type": "integer"
+         },
+         "name": {
+           "type": "string"
+         }
+       },
+       "required": [
+         "id",
+         "name"
+       ]
+     }
+   }
+ }
}

```

それから、`#[openapi::operation]` で `operationId` を設定したり `summary`, `description`, 各レスポンスの `description` をカスタマイズできます。

ここだけマクロに頼っていますが、これらはオプショナルというか、クライアントの型を生成するにあたって必須でないプラスアルファ的な要素なので別にいいかなと思っています。

```rust:main.rs
#[openapi::operation({
    summary: "...",
    200: "List of all users",
})]
/// This doc comment is used for the
/// `description` field of OpenAPI document
async fn list_users() -> JSON<Vec<User>> {
    JSON(vec![])
}
```

```json:openapi.json
{
  ...

  "paths": {
    "/users": {
      "get": {
        "operationId": "list_users",
        "summary": "...",
        "description": "This doc comment is used for the\n`description` field of OpenAPI document",
        "responses": {
          "200": {
            "description": "List of all users",
            "content": {
              "application/json": {
                "schema": {
                  "type": "array",
                  "items": {
                    "$ref": "#/components/schemas/User"

  ...
```

一応、同名の unit struct に `IntoHandler` を impl して `map_openapi_operation` というメソッドでメタデータを登録しているだけなので、手動でもそこまで面倒ではないです。

:::details 手で書くと
```rust
use ohkami::handler::IntoHandler;

struct ListUsers;
impl ListUsers {
    async fn handle() -> JSON<Vec<User>> {
        JSON(vec![])
    }
}
impl IntoHandler<ListUsers> for ListUsers {
    fn into_handler(self) -> ohkami::handler::Handler {
        (Self::handle).into_handler().map_openapi_operation(|mut op| {
            op = op
                .operationId("list_users")
                .description("This doc comment is used for the\n`description` field of OpenAPI document")
                .summary("...");
            op.override_response_description(200, "List of all users");
            op
        })
    }
}
```

( v0.23 以降は

```rust
.map_openapi_operation(|op| op
    .operationId("list_users")
    .description("This doc comment is used for the\n`description` field of OpenAPI document")
    .summary("...")
    .response_description(200, "List of all users")
)
```

とできるようになりそうです )
:::


## どうなっているのか

このドキュメント生成の裏側を簡単に解説します。

### 1. `Schema`

まず `Schema` trait ですが、上記の `#[derive(Schema)]` は以下のように展開されます：

- `CreateUser`

    ```rust
    impl<'req> ::ohkami::openapi::Schema for CreateUser<'req> {
        fn schema() -> impl Into<::ohkami::openapi::schema::SchemaRef> {
            {
                let mut schema = ::ohkami::openapi::object();
                schema = schema
                    .property(
                        "name",
                        ::ohkami::openapi::schema::Schema::<
                            ::ohkami::openapi::schema::Type::any,
                        >::from(
                            <&'req str as ::ohkami::openapi::Schema>::schema()
                                .into()
                                .into_inline()
                                .unwrap(),
                        ),
                    );
                schema
            }
        }
    }
    ```
  
    手で書くと
  
    ```rust
    impl openapi::Schema for CreateUser<'_> {
        fn schema() -> impl Into<openapi::schema::SchemaRef> {
            openapi::object()
                .property("name", openapi::string())
        }
    }
    ```

- `User`

    ```rust
    impl ::ohkami::openapi::Schema for User {
        fn schema() -> impl Into<::ohkami::openapi::schema::SchemaRef> {
            ::ohkami::openapi::component(
                "User",
                {
                    let mut schema = ::ohkami::openapi::object();
                    schema = schema
                        .property(
                            "id",
                            ::ohkami::openapi::schema::Schema::<
                                ::ohkami::openapi::schema::Type::any,
                            >::from(
                                <usize as ::ohkami::openapi::Schema>::schema()
                                    .into()
                                    .into_inline()
                                    .unwrap(),
                            ),
                        );
                    schema = schema
                        .property(
                            "name",
                            ::ohkami::openapi::schema::Schema::<
                                ::ohkami::openapi::schema::Type::any,
                            >::from(
                                <String as ::ohkami::openapi::Schema>::schema()
                                    .into()
                                    .into_inline()
                                    .unwrap(),
                            ),
                        );
                    schema
                },
            )
        }
    }
    ```

    手で書くと

    ```rust
    impl openapi::Schema for User {
        fn schema() -> impl Into<openapi::schema::SchemaRef> {
            openapi::component(
                "User",
                openapi::object()
                    .property("id", openapi::integer())
                    .property("name", openapi::string())
            )
        }
    }
    ```

このように DSL が整備されているので、手で impl するのも簡単です。

`Schema` trait は `openapi::schema::SchemaRef` というものを対象の構造体に紐付ける役割を担います。

### 2. `FromParam`, `FromRequest`, `IntoResponse` の `openapi_*` hooks

Ohkami のハンドラは

```rust
async fn({FromParam tuple}?, {FromRequest item}*) -> {IntoResponse item}
```

という形をしているわけですが、`openapi` feature 有効時には、これらの trait にそれぞれ

```rust
fn openapi_param() -> openapi::Parameter
```
```rust
fn openapi_inbound() -> openapi::Inbound
```
```rust
fn openapi_responses() -> openapi::Responses
```

というメソッドが生えます。これらを

https://github.com/ohkami-rs/ohkami/blob/6e243ac823e21f286aca2660f9d38f7bde381c5a/ohkami/src/fang/handler/into_handler.rs#L328-L335

のように `IntoHandler` 解決時に `openapi::Operation` に適用していくことで、型情報を活かして operation を作ります。

ここで、この `openapi::{Parameter, Inbound, Responses}` という人達が impl 先の構造体に紐づく `openapi::schema::SchemaRef` によって各要素のスキーマを把握してくれています。

また、例えば

https://github.com/ohkami-rs/ohkami/blob/6e243ac823e21f286aca2660f9d38f7bde381c5a/ohkami/src/response/into_response.rs#L114-L128

のように適切にスキーマ情報を伝搬させ、極力ユーザーが自分のアプリ上の型・スキーマのことだけ考えればいいようになっています。

### 3. `routes` of router

Ohkami の内部の `router::base::Router` というものは

https://github.com/ohkami-rs/ohkami/blob/6e243ac823e21f286aca2660f9d38f7bde381c5a/ohkami/src/router/base.rs#L8-L18

となっており、ハンドラを追加するたびに `routes` にパスを入れて記憶するようになっています。

:::details for me
- これは無理せず `String` でいい
- `Method` も一緒に入れておくと余計な探索がいらなくなる

https://github.com/ohkami-rs/ohkami/issues/357
:::

この `routes` は、`finalize` という処理で `router::final::Router` というものを作る際に一緒に返されます。

https://github.com/ohkami-rs/ohkami/blob/6e243ac823e21f286aca2660f9d38f7bde381c5a/ohkami/src/router/base.rs#L198-L206

### 4. `generate`

そして最後に `generate` ですが、これのメインは

https://github.com/ohkami-rs/ohkami/blob/6e243ac823e21f286aca2660f9d38f7bde381c5a/ohkami/src/router/final.rs#L54-L59

という処理です。`Ohkami::generate` は、この結果の `openapi::document::Document` という物体をシリアライズして、所定のファイルに書き出しているだけです。

`gen_openapi_doc` は、おおよそ次のような処理になっています：

```rust
let mut doc = Document::new(/* ... */);

for route in routes {
    let (openapi_path, openapi_path_param_names) = {
        // "/api/users/:id"
        // ↓
        // ("/api/users/{id}", ["id"])
    };

    let mut operations = Operations::new();
    for (openapi_method, router) in [
        ("get",    &self.GET),
        ("put",    &self.PUT),
        ("post",   &self.POST),
        ("patch",  &self.PATCH),
        ("delete", &self.DELETE),
    ] {
        // `router` の `route` にある Node に
        // operation が登録されていれば
        // 前処理をしてから
        // `operations` に追加する
    }

    doc = doc.path(openapi_path, operations);
}

doc
```


## Cloudflare Workers

Ohkami は `rt_worker` で Cloudflare Workers に対応していますが、その場合 Ohkami は WASM になって Miniflare や Cloudflare Workers にロードされるので、OpenAPI document をデータとして生成するところまでいっても、ローカルのファイルシステムに触れないため、そこで止まってしまいます。

https://x.com/kanarus_x/status/1880507359203852442

そこで、[scripts/workers_openapi.js](https://github.com/ohkami-rs/ohkami/blob/6e243ac823e21f286aca2660f9d38f7bde381c5a/scripts/workers_openapi.js) という CLI ツールを用意しました。

これを、例えば

https://github.com/ohkami-rs/ohkami-templates/tree/main/worker-openapi

の

```json:package.json
{
    ...
    "scripts": {
    	"deploy": "export OHKAMI_WORKER_DEV='' && wrangler deploy",
    	"dev": "export OHKAMI_WORKER_DEV=1 && wrangler dev",
    	"openapi": "node -e \"$(curl -s https://raw.githubusercontent.com/ohkami-rs/ohkami/refs/heads/main/scripts/workers_openapi.js)\" -- --features openapi"
    },
    ...
}
```

ように npm script に仕込んでおいて、おもむろに

```sh
npm run openapi
```

とすれば OpenAPI document が吐かれるという、なかなか面白い体験になっています。


## おわりに

Rust の OpenAPI document generation といえば [utoipa](https://github.com/juhaku/utoipa) ですが、やたらマクロに頼ることになるのが気に食わず、Ohkami では Ohkami 専用に、内部実装まで食い込んだ highly integrated な機構を作りました。個人的にはユーザー体験が良く気に入っています。
