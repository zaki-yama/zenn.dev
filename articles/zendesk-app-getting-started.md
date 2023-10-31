---
title: "Zendeskアプリ開発ことはじめ（前編）"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zendesk"]
publication_name: "loglass"
published: true
---

:::message
この記事は[毎週必ず記事がでるテックブログ "Loglass Tech Blog Sprint"](https://zenn.dev/loglass/articles/7298a3cd4c5fc6) の **11 週目**の記事です！
1 年間連続達成まで **残り 42 週** となりました！
:::

株式会社ログラス CRE（Customer Reliability Engineer）の [@zaki\_\_\_yama](https://twitter.com/zaki___yama) です。

CRE としての取り組みの 1 つとして、ここ半年ぐらいカスタマーサポート体制の立ち上げ、および自身もお客様からの問い合わせ対応をしながら業務プロセスの改善を進めてきました。
弊社では問い合わせ対応に Zendesk (https://www.zendesk.co.jp/) を使用しています。Zendesk には [Zendesk アプリ](https://developer.zendesk.com/documentation/apps/) と呼ばれる、Zendesk のインターフェースを拡張する仕組みが備わっています。（Chrome 拡張のようなイメージです）
これで何か業務改善につながられるのでは？という興味から、具体的に何ができるのか、どういった開発体験なのかを最近調べていたため、本記事でご紹介します。

Zendesk アプリは英語の[公式ドキュメント](https://developer.zendesk.com/documentation/apps/getting-started/overview/)は非常に充実していますが、日本語のまとまった情報はまだそれほど多くないため、私と同じようにこれから Zendesk アプリ開発を始めてみようという方にとって有益な情報になれば幸いです。

# 本記事の構成

前編・後編の 2 つの記事で構成する予定です。
前編となる本記事では、Zendesk アプリの概要と、CLI を用いた基本的な開発フローについて説明します。

# Zendesk アプリの概要

上述したように、Zendesk アプリは Zendesk の標準のインターフェースを拡張・強化する仕組みです。
参考：[トップレベルのアプリを使って Zendesk を機能拡張 （Zendesk ヘルプ）](https://support.zendesk.com/hc/ja/articles/4408829182234-%E3%83%88%E3%83%83%E3%83%97%E3%83%AC%E3%83%99%E3%83%AB%E3%81%AE%E3%82%A2%E3%83%97%E3%83%AA%E3%82%92%E4%BD%BF%E3%81%A3%E3%81%A6Zendesk%E3%82%92%E6%A9%9F%E8%83%BD%E6%8B%A1%E5%BC%B5)

たとえば以下の画像では Zendesk が公式で提供している[Salesforce 連携用のアプリ](https://www.zendesk.com/marketplace/apps/support/10/salesforce/?queryID=bb12f4add09246735b777d674bd34623) をインストールしており、Zendesk の問い合わせユーザーにひもづく Salesforce の情報をチケット横に表示しています。

![](https://storage.googleapis.com/zenn-user-upload/a2f66599023c-20231026.png)

また、今回はアプリの「開発」に主眼を置いていますが、他者が開発したアプリを簡単にインストールできる独自のマーケットプレイスも備わっています。

https://www.zendesk.com/marketplace/apps/

# 基本的な開発フロー

ここからは実際に手を動かしながら、アプリの基本的な開発フローについて学んでいきます。

## 開発アカウントの取得

本番環境で試すのもいいですが、Zendesk にはトライアルアカウントないしは開発用のスポンサーアカウントなるものが存在します。
[Getting a trial or sponsored account for development](https://developer.zendesk.com/documentation/api-basics/getting-started/getting-a-trial-or-sponsored-account-for-development/) からアカウントの申し込みができます。

※トライアルアカウントは 14 日間の期限があるようなので、私はスポンサーアカウントを取得しました。

## Zendesk Command Line Interface (ZCLI) のインストール

アプリ開発には Zendesk Command Line Interface (ZCLI)と呼ばれる専用の CLI が提供されています。

```bash
$ npm install -g @zendesk/zcli
```

でインストールできます。インストールすると `zcli` というコマンドが使えるようになります。

```bash
$ zcli help
Zendesk CLI is a single command line tool for all your zendesk needs

VERSION
  @zendesk/zcli/1.0.0-beta.38 darwin-arm64 node-v18.17.1

USAGE
  $ zcli [COMMAND]

TOPICS
  apps      manage Zendesk apps workflow
  profiles  manage cli user profiles
  themes    manage Zendesk themes workflow

COMMANDS
  autocomplete  display autocomplete installation instructions
  help          Display help for zcli.
  login         creates and/or saves an authentication token for the specified subdomain
  logout        removes an authentication token for an active profile
```

余談ですが、ZCLI の前に Zendesk Apps Tools (ZAT) という CLI も存在したようです。（筆者は未経験）
Web 上の少し古い情報だと `zat` というコマンドを使った説明が見つかるかもしれませんが、こちらの CLI は現在メンテナンスモードであり、ZCLI の使用が推奨されています。
参考: [Installing and using ZAT](https://developer.zendesk.com/documentation/apps/zendesk-app-tools-zat/installing-and-using-zat/)

## 認証

CLI をインストールしたら、CLI 経由でアカウントにログインし認証を通しておきます。

```bash
$ zcli login -i
```

を実行すると Zendesk のサブドメイン名、メールアドレス等が要求されるので入力します。
ログインが完了すると `zcli profiles:list` で認証済みアカウントの一覧が確認できます。

```bash
$ zcli profiles:list
 Accounts
 ───────────────────────
 d3v-zaki-yama <= active
```

なお、上記がプロファイルを使った方法で、別途環境変数を使った認証方法もあります。
https://developer.zendesk.com/documentation/apps/getting-started/using-zcli/#environment-variables

## アプリの雛形作成とローカル開発

ここからローカルにアプリのコードを作成していきます。
`zcli apps:new` を実行するとアプリの雛形（scaffold）が生成されます。

```bash
$ zcli apps:new
Enter a directory name to save the new app (will create the dir if it does not exist): my-app
Enter this app authors name: Shingo Yamazaki
...


$ cd my-app
$ tree
.
├── README.md
├── assets
│   ├── iframe.html
│   ├── logo-small.png
│   ├── logo.png
│   └── logo.svg
├── manifest.json
└── translations
    └── en.json

3 directories, 7 files
```

この状態で、まずはアプリをローカルで実行してみます。
それには `zcli apps:server` コマンドを使います。

```bash
$ zcli apps:server

Apps server is running on http://localhost:4567 🚀

Add ?zcli_apps=true to the end of your Zendesk URL to load these apps on your Zendesk account.
```

ローカルサーバーが起動します。

コンソールに表示されているように、この状態で Zendesk のアカウントにログインし、Support 画面のチケット URL の後ろに `?zcli_apps=true` をつけてアクセスします。
（URL は https://d3v-\*\*\*.zendesk.com/agent/tickets/1?zcli_apps=true のようになるはずです）

![](https://storage.googleapis.com/zenn-user-upload/06a9745b9a41-20231025.png)

チケット詳細画面の右側にアプリが表示されました。

## パッケージングとデプロイ

最後に、アプリを Zendesk にデプロイします。
それには `zcli apps:create` コマンドおよび `zcli apps:update` コマンドを使います。

```bash
$ zcli apps:create
Uploading app... Uploaded
Deploying app... Deployed
Successfully installed app: my-app with app_id: 981924
```

`zcli apps:create` だけで、アプリのパッケージングとデプロイまで行ってくれます。
先程アクセスした URL から `?zcli_apps=true` を除き、アプリが引き続き表示されることを確認しましょう。

デプロイしたアプリは、

```
管理センター(/admin) > アプリおよびインテグレーション > アプリ > Zendesk Support アプリ
```

で確認できます。

![](https://storage.googleapis.com/zenn-user-upload/aac770e03026-20231025.png)

また、デプロイが完了すると、ローカルには `zcli.apps.config.json` という JSON ファイルが生成されています。

```bash
$ cat zcli.apps.config.json
{"app_id":981926}
```

中には `app_id` のみ記載されており、この ID でアプリを一意に識別します。
初回のデプロイ以降、アプリを更新したい場合は `zcli apps:update` コマンドを使用します。

# マニフェストファイル（manifest.json）について

`zcli apps:new` によって生成されたディレクトリの中身を見ると、 `manifest.json` というファイルがあるのがわかります。
アプリに関する設定情報はこのマニフェストファイルに記載します。

すべてのプロパティについては触れませんが、そのうちの 1 つに　`location` というプロパティがあります。

```json
"location": {
  "support": {
    "ticket_sidebar": {
      "url": "assets/iframe.html",
      "flexible": true
    }
  }
},
```

`location` のすぐ下に `support` が指定されていますが、これは問い合わせ管理用の Zendesk Support 以外に、Chat や Sell といった機能でもアプリを利用できるため、どの機能で使用するかを設定しています。
またその下には `ticket_sidebar` というプロパティがありますが、これにより画面のどこにアプリを表示するか？を設定しています。

`ticket_sidebar` 以外に取りうる値や、画面のどこに表示できるのか？については
[Introduction | Zendesk Developer Docs](https://developer.zendesk.com/api-reference/apps/apps-support-api/introduction/)
の画像がわかりやすいです。

![](https://storage.googleapis.com/zenn-user-upload/75daca2cc54a-20231026.png)

（上記公式ドキュメントより引用）

最後に

```json
"url": "assets/iframe.html"
```

がありますが、ここでアプリとして表示する HTML ファイルの場所を指定しています。

manifest.json のその他のプロパティについては、公式ドキュメントの
[Manifest reference](https://developer.zendesk.com/documentation/apps/app-developer-guide/manifest/)
を参照してください。

# まとめ

本記事では Zendesk アプリの概要と、基本的な開発フローについて紹介しました。
アプリの開発フローは基本的にはすべて専用の CLI で完結します。よく使うコマンドをまとめておきます。

| コマンド                                 | 役割                 |
| :--------------------------------------- | :------------------- |
| `zcli apps:new`                          | アプリの雛形作成     |
| `zcli apps:server`                       | ローカルサーバー起動 |
| `zcli apps:create` or `zcli apps:update` | アプリのデプロイ     |

# リファレンス

- [Zendesk app quick start | Zendesk Developer Docs](https://developer.zendesk.com/documentation/apps/getting-started/zendesk-app-quick-start/)
- [Using ZCLI | Zendesk Developer Docs](https://developer.zendesk.com/documentation/apps/getting-started/using-zcli/)
- [Manifest reference | Zendesk Developer Docs](https://developer.zendesk.com/documentation/apps/app-developer-guide/manifest/)
- [zendesk/zcli: A command-line tool for Zendesk](https://github.com/zendesk/zcli)
