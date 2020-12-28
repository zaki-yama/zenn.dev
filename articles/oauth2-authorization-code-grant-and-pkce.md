---
title: "OAuth 2.0 認可コードフロー+PKCE をシーケンス図で理解する"
emoji: "💪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["oauth2", "pkce"]
published: true
---

# はじめに

OAuth 2.0 のフローをシーケンス図で説明したWeb上の記事や書籍を何度か見かけたことがありますが、

- フローの概要に加え、クライアントや認可サーバー側でどういったパラメータを元に何を検証しているのかも一連のフローとして理解したかった
- [RFC 7636](https://tools.ietf.org/html/rfc7636) Proof Key for Code Exchange (PKCE) も含めた流れを整理したかった

というモチベーションがあり、自分でシーケンス図を書きながら流れを整理してみた、という趣旨です。

# 記事の前提や注意事項

- OAuth 2.0 の各種フローのうち、認可コードフローのみ取り上げています
- 認可コードフローとはなにか、PKCE とはなにかという説明は割愛しています
  - 概要について、個人的にはこちらの動画が非常にわかりやすかったです: [OAuth & OIDC 入門編 by #authlete - YouTube](https://www.youtube.com/watch?v=PKPj_MmLq5E)
    - 認可コードフローは 38:00、PKCE は 1:15:00 あたり
- 文中でたびたび RFC 6749 を参照していますが、リンク先および引用文は OpenID Foundation Japan による翻訳版(https://openid-foundation-japan.github.io/rfc6749.ja.html)になっています
- リクエスト・レスポンス例では、クライアントおよび認可サーバーのエンドポイントは以下のURLの想定で書いています

```
- クライアント: web アプリ https://client.example.com
  - リダイレクト URI: https://client.example.com/cb
- 認可サーバー:
  - 認可エンドポイント: https://server.example.com/authorize
  - トークンエンドポイント: https://server.example.com/token
```

# 1) 認可コードフロー

はじめに、PKCE を含まない通常の認可コードフローについて見ていきます。
シーケンス図はこちらです。

[![](https://storage.googleapis.com/zenn-user-upload/zgdvnxray0tvf15knewqg0ztbfmp)](https://github.com/zaki-yama/zenn.dev/blob/master/articles/images/oauth2-authorization-code-grant-and-pkce/authorization-code-flow.png)

以下、この図の説明です。
フロー開始から認可コードを取得するまでの流れを見ていきます。
## 0. クライアント登録

フローを開始する前に、クライアントを認可サーバーに登録する必要があります。
クライアント登録時に保存する情報として、 [RFC 6749 「2. クライアント登録」](https://openid-foundation-japan.github.io/rfc6749.ja.html#anchor13) には以下のように記載されています。

> クライアントを登録する場合, クライアント開発者は以下を満たすものとする (SHALL).
>
> - [Section 2.1](https://openid-foundation-japan.github.io/rfc6749.ja.html#client-types) で説明されているようなクライアントタイプを指定し,
> - [Section 3.1.2](https://openid-foundation-japan.github.io/rfc6749.ja.html#redirect-uri) で説明されているようなリダイレクト URI を提供し,
> - 認可サーバーが要求するその他の情報 (例えばアプリケーション名, Web サイト, 説明, ロゴイメージ, 利用規則など) を提供する.

## 1. フロー開始 〜 2. `state` の生成

フローを開始する際、クライアントは `state` パラメータというランダムな文字列を生成します。
[RFC 6749 「4.1.1. 認可リクエスト」](https://openid-foundation-japan.github.io/rfc6749.ja.html#code-authz-req) に以下の記載があります。

> state
> 推奨 (RECOMMENDED). リクエストとコールバックの間で状態を維持するために使用するランダムな値. 認可サーバーはリダイレクトによってクライアントに処理を戻す際にこの値を付与する.

認可レスポンスを受けとった際、レスポンスに含まれる値と突き合わせるため、生成した文字列はクライアント内部で保持しておきます。
## 4. 認可リクエスト

クライアントはリダイレクトを利用して、リソースオーナーを認可エンドポイントに導きます。
リクエストは以下のような形です。

```
GET /authorize
  ?response_type=code
  &client_id=<client_id>
  &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
  &scope=read
  &state=xyz
Host: server.example.com
```

送信するパラメータ、および各パラメータが必須か任意かについては、[RFC 6749 「4.1.1. 認可リクエスト」](https://openid-foundation-japan.github.io/rfc6749.ja.html#code-authz-req) によれば以下の通りです。

- `response_type`: 必須(REQUIRED)。値は必ず `code` にしなければならない(MUST)
- `client_id`: 必須(REQUIRED)
- `redirect_uri`: 任意(OPTIONAL)
- `scope`: 任意(OPTIONAL)
- `state`: 推奨(RECOMMENDED)

## 5. パラメータの検証

認可エンドポイントへのリクエストを受け取った認可サーバー側では、はじめに送られてきたパラメータが正しいか検証します。
具体的には、 `state` を除く各種パラメータについて

- `response_type`: 値が `code` か
- `client_id`: 登録済みのクライアントの中で、ID が一致するものがあるか
- `redirect_uri`: 登録済みの内容と一致するか
- `scope`: (登録されていれば) 登録済みの内容と一致するか

であることを検証します。

:::message
`scope` の扱いについては理解が不十分なところがあるのですが、[RFC 6749 「3.3. アクセストークンのスコープ」](https://openid-foundation-japan.github.io/rfc6749.ja.html#scope) によれば

> 認可サーバーは, 認可サーバーのポリシーまたはリソースオーナーの指示に基づいて, クライアントに要求されたスコープの一部もしくはすべてを無視してもよい (MAY). 発行されたアクセストークンのスコープがクライアントから要求されたものと異なる場合, 認可サーバーは実際に許可されたスコープをクライアントに通知するため, scope レスポンスパラメーターを含めなくてはならない (MUST).

> 認可リクエスト時にクライアントがスコープパラメーターを省略した場合, 認可サーバーは事前に定義されたデフォルト値が指定されたものとするか, 無効なスコープを意味しているものとしてリクエスト失敗とするかのどちらかを処理しなければならない (MUST). 認可サーバーはスコープ要件と (もし定義されていれば) デフォルト値をドキュメント化するべきである (SHOULD).

とあるので、

- `scope` パラメータが省略されたときにエラーとするか
- `scope` パラメータが登録済みの内容と一致しなかったり、認可サーバーの仕様で許可された値以外が渡されたときにどうするか

といった場合の処理は各認可サーバーの実装依存になるかと思います。
:::

## 6. 認証画面表示　〜 9. 認可

認可サーバーはリソースオーナーに対し、クライアントへ各種リソースへの認可を行うかどうか確認します。
一般的には、リソースオーナーが未ログインであればログイン画面を表示してユーザー名・パスワードによる認証を行った後、「◯◯（クライアント）へ以下のリソースへのアクセスを許可しますか？」という画面を表示することで許可/拒否を尋ねます。

なお、[RFC 6749 「3.1. 認可エンドポイント」](https://openid-foundation-japan.github.io/rfc6749.ja.html#anchor17) には

> 認可サーバーが用いるリソースオーナーの認証方法 (ユーザー名とパスワードによるログイン, セッションクッキー) については, 本仕様の定めるところではない.

と記載されています。

## 10. 認可コード発行

リソースオーナーの許可が得られた後、認可サーバーは認可コードを発行します。
発行した認可コードはこの後のアクセストークンリクエストの際にリクエストパラメータと突き合わせるため、 `client_id` とひもづけて保存しておきます。

参考: [RFC 6749 「4.1.2. 認可レスポンス」](https://openid-foundation-japan.github.io/rfc6749.ja.html#code-authz-resp)

> 認可コードはクライアント識別子とリダイレクト URI に紐づく.

また、認可コードの有効期限については、同じ項に

> 漏洩のリスクを軽減するため, 認可コードは発行されてから短期間で無効にしなければならない (MUST). 認可コードの有効期限は最大でも 10 分を推奨する (RECOMMENDED).

という記載があります。

<!-- TODO: `scope` もひもづけて保存しておくことと、その理由を書く。アクセストークンリクエスト時に取り出せるようにするため。 -->

## 11. 認可レスポンス

認可コードを発行した後、認可サーバーはリクエスト時に送られてきた `redirect_uri` にリソースオーナーをリダイレクトさせます。
このとき、リダイレクト URI には 2 つのクエリパラメータが付与されています。

- `code`: 直前に発行した認可コード
- (optional) `state`: リクエスト時にクライアントから送られてきた場合のみ。受け取った値をそのまま返す

以下は認可レスポンスの例です。

```
HTTP/1.1 302 Found
Location: https://client.example.com/cb?code=<認可コード>&state=xyz
```

## 12. `state` の検証

Step 10 で発行した `state` の値と、認可レスポンスで返された `state` の値が一致することを検証します。
一致しなかった場合はエラーとします。

## 13. アクセストークンリクエスト

先ほど取得した認可コードを付与して、トークンエンドポイントに POST リクエストを送信します。
リクエストは以下のような形式です。

```
POST /token HTTP/1.1
  Host: server.example.com
  Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
  Content-Type: application/x-www-form-urlencoded

  grant_type=authorization_code
  &code=<認可コード>
  &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
```

送信するパラメータ、および各パラメータが必須か任意かについては、[RFC6749 「4.1.3. アクセストークンリクエスト」](https://openid-foundation-japan.github.io/rfc6749.ja.html#token-req) によれば以下の通りです。

- `grant_type`: 必須(REQUIRED)。値は必ず `authorization_code` にしなければならない(MUST)
- `code`: 必須(REQUIRED)。認可コード
- `redirect_uri`: 認可リクエスト時に送信した場合は必須(REQUIRED)。認可リクエスト時と同じ値でなければならない(MUST)
- `client_id`: クライアント認証を行わない場合は必須(REQUIRED)

また、コンフィデンシャルクライアントの場合、`client_id`と`client_secret`を Basic 認証を使って送信します。
具体的には、`client_id`, `client_secret` それぞれを URL エンコードし、それらを `:` で結合した文字列を Base64 エンコードしたものを `Authorization` ヘッダーに設定します。

なお [RFC 6749 「2.3.1 クライアントパスワード」](https://openid-foundation-japan.github.io/rfc6749.ja.html#client-password) によれば、Basic 認証スキームの代わりに 2 つのパラメータを直接リクエストボディーに含める方法もありますが、非推奨(NOT RECOMMENDED)とされています。

パブリッククライアントの場合、`client_id` のみをリクエストボディーに設定して送信することになります。

## 14. & 15. パラメータの検証

認可サーバー側でアクセストークンリクエストに含まれる各種パラメータを検証します。
検証する内容については以下のとおりです。

- `grant_type`: 値が `authorization_code` か
- `Authorization` ヘッダーをデコードして `client_id` と `client_secret` を取り出し、登録済みのクライアント情報に一致するものが存在することを確認する
- `code` と一致する保存済みの認可コードが存在するか確認する
- さらに、認可コードにひもづいて保存されている内容から以下を確認する
  - 有効期限が切れてないこと
  - `client_id` と `redirect_uri` が保存済みの内容と一致すること

参考: [RFC 6749 「4.1.3. アクセストークンリクエスト」](https://openid-foundation-japan.github.io/rfc6749.ja.html#token-req)

> - クライアントがコンフィデンシャルクライアントの場合は, 認可コードが確かに認証されたクライアントに対して発行されたことを確認する. クライアントがパブリッククライアントの場合は, 認可コードが確かに指定された client_id に紐づくクライアントに対して発行されていることを確認する.
> - 認可コードが正当であることを検証する.
> - [Section 4.1.1](https://openid-foundation-japan.github.io/rfc6749.ja.html#code-authz-req) で述べた認可リクエスト時に redirect_uri パラメータが含まれていた場合, ここでも redirect_uri が存在し認可リクエスト時と同じ値であることを確認する.


## 16. 認可コードの削除

Step 15 のパラメータの検証で、保存済みの認可コードが存在することが確認できたら、同じ認可コードを再利用できないよう削除します。

[RFC 6749 「[10.5. 認可コード](https://openid-foundation-japan.github.io/rfc6749.ja.html#anchor35)」に以下の記載があります。

> 認可コードの有効期間は短く, かつ一度限りしか利用されてはならない (MUST). もし認可サーバーが単一の認可コードをアクセストークンへ交換しようとする複数の試行を検出したならば, 認可サーバーはその認可コードに基づき既に付与されたすべてのアクセストークンを無効化することを試みるべきである (SHOULD).

<!-- TODO: 認可コードを削除するタイミングについて -->
## 17. アクセストークン発行 (& 18, リフレッシュトークン発行)

送られてきたパラメータの正当であることを検証した後、認可サーバーはアクセストークンを発行します。
また、任意でリフレッシュトークンを発行します。

アクセストークン、リフレッシュトークンとひもづけて登録するその他の情報としては、以下があります。

- `client_id`
- 有効期限
- `scope`: 認可コードにひもづけて保存していた値
- `user_id`: リソースオーナーを識別するための識別子

`scope` を保存する理由は、この後クライアントが取得したアクセストークンを使ってリソースにアクセスする際、アクセストークンに与えられたスコープがリソースにアクセスするだけの条件を満たしているかをチェックする必要があるためです。

また、リソースオーナーの識別子については RFC 6749 に記載があるわけではありませんが、「管理画面などから自分が許可した OAuth クライアントの一覧を表示し、必要であれば手動で revoke する」ことができるようにしてあるのが一般的ですので、「誰が発行したアクセストークンか」という情報もひもづけて保存すると考えます。


## 19. アクセストークンレスポンス

発行したアクセストークンをクライアントに送信します。
レスポンスに含めるパラメーターについては [RFC 6749 「5.1. 成功レスポンス」](https://openid-foundation-japan.github.io/rfc6749.ja.html#token-response) に記載があります。

- `access_token`: 必須(REQUIRED)
- `token_type`: 必須(REQUIRED)。 トークンのタイプ。`Bearer` の場合がほとんど。
- `expires_in`: 推奨(RECOMMENDED)。アクセストークンの有効期限を表す秒数 (例： `3600` ならば1時間後に期限切れ)
- `refresh_token`: 任意(OPTIONAL)
- `scope`: アクセストークンのスコープ。クライアントから全く同一のスコープが要求された場合は任意 (OPTIONAL)。その他は必須 (REQUIRED)

以下はレスポンスの例です。

```
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "access_token":"2YotnFZFEjr1zCsicMWpAA",
  "token_type":"Bearer",
  "expires_in":3600,
  "scope": "read",
  "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA"
}
```

# 2) 認可コードフロー + PKCE

ここまで見てきた認可コードフローにPKCEも含めたシーケンス図がこちらです。

[![](https://storage.googleapis.com/zenn-user-upload/g70494a7cajzdcqufi5amuty9uwo)](https://github.com/zaki-yama/zenn.dev/blob/master/articles/images/oauth2-authorization-code-grant-and-pkce/authorization-code-flow-pkce.png)

赤字部分が PKCE によって追加された処理です。それ以外はここまで説明した内容と変わらないため、以下ではこの赤字部分のみ説明します。

## 3. `code_verifier` の生成
フロー開始時、クライアントは `code_verifier` と呼ばれるランダムな文字列を生成します。

## 4. `code_verifier` から `code_challenge` の算出
生成した `code_verifier` から、決められたメソッドでハッシュ値を計算し、 `code_challenge` とします。
`code_challenge` 算出の際のメソッドは `code_challenge_method` パラメーターと呼ばれ、 `plain` または `S256` のいずれかの値を取ります。
それぞれの値に対する `code_challenge` の算出方法は以下のとおりです。

- `plain`: `code_challenge` は `code_verifier` の値そのもの
- `S256`: `code_verifier` の SHA-256 ハッシュ値を計算し、Base64URL エンコードした値

ただし、[RFC 7636 「4.2.  Client Creates the Code Challenge」](https://tools.ietf.org/html/rfc7636#section-4.2) にも

> If the client is capable of using "S256", it MUST use "S256", as
> "S256" is Mandatory To Implement (MTI) on the server.  Clients are
> permitted to use "plain" only if they cannot support "S256" for some
> technical reason and know via out-of-band configuration that the
> server supports "plain".

という記載があり、実際には特別な事情がない限り `code_challenge_method` は `S256` 一択のようです。

## 6. 認可リクエスト
認可リクエスト時、通常の認可コードフローのパラメータに加え、`code_challenge` と `code_challenge_method` パラメーターも認可サーバーに送信します。

```
GET /authorize
  ?response_type=code
  &client_id=<client_id>
  &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
  &scope=read
  &state=xyz
  &code_challenge=<計算したハッシュ値>
  &code_challenge_method=S256
Host: server.example.com
```

## 13. `code_challenge`, `code_challenge_method` を認可コードにひもづけて保存
認可コードを発行して保存する際、リクエストパラメータに送られてきた `code_challenge` と `code_challenge_method` も認可コードにひもづけて保存します。
これは、この後のアクセストークンリクエスト時に検証のために使用します。

## 16. アクセストークンリクエスト
認可コードを受け取ったクライアントは、通常の認可コードフローと同じようにアクセストークンリクエストを送信します。
その際、今度は `code_verifier` パラメータを追加して送信します。

```
POST /token HTTP/1.1
  Host: server.example.com
  Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW
  Content-Type: application/x-www-form-urlencoded

  grant_type=authorization_code
  &code=<認可コード>
  &redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb
  &code_verifier=<Step 3 で生成したランダムな文字列>
```

## 20. `code_verifier` の検証
アクセストークンリクエストを受け取った認可サーバーは、以下の手順に従って `code_verifier` が正しいか検証します。

1. 保存済みの認可コードにひもづく `code_challenge` と `code_challenge_method` を取り出す
2. リクエストパラメータで受け取った `code_verifier` と、保存されていた `code_challenge_method` に従ってハッシュ値を計算する
3. 計算して得られたハッシュ値と、保存されていた `code_challenge` が一致することを確認する

これにより、認可リクエストを送ってきたクライアントとアクセストークンリクエストを送ってきたクライアントが同一であることを担保できます。

# 参考リンク

- [OAuth & OIDC 入門編 by #authlete - YouTube](https://www.youtube.com/watch?v=PKPj_MmLq5E)
  - 冒頭でも紹介した YouTube 動画です。PKCE だけでなく OAuth 2.0 の基本的なフローについて非常にわかりやすく解説されています
- [OAuth 2.0 の勉強のために認可サーバーを自作する - Qiita](https://qiita.com/TakahikoKawasaki/items/e508a14ed960347cff11)
  - 認可コードやアクセストークンがどういうデータとともに保存されているのか、を理解する上で参考になりました
- 📕 [雰囲気でOAuth2.0を使っているエンジニアがOAuth2.0を整理して、手を動かしながら学べる本](https://booth.pm/ja/items/1296585)
  - 基本的な知識を身につけるのにとても役立ちました
- 📕 [OAuth 徹底入門 セキュアな認可システムを適用するための原則と実践](https://www.shoeisha.co.jp/book/detail/9784798159294)
  - Node.js を使ったクライアント、認可サーバー、リソースサーバーのサンプルプログラムが充実しており、実装するとしたらこういうロジックになるという具体的なイメージがしやすかったです
