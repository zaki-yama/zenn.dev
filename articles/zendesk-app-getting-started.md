---
title: "Zendeskアプリ開発ことはじめ"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zendesk"]
publication_name: "loglass"
published: false
---

:::message
この記事は[毎週必ず記事がでるテックブログ "Loglass Tech Blog Sprint"](https://zenn.dev/loglass/articles/7298a3cd4c5fc6) の **X 週目**の記事です！
1 年間連続達成まで **残り XX 週** となりました！
:::

こんにちは。[株式会社ログラス](https://www.loglass.jp/) で CRE（Customer Reliability Engineer）をやっている山﨑（[@zaki\_\_\_yama](https://twitter.com/zaki___yama)）です。

CRE としての取り組みの 1 つとして、ここ半年ぐらいはカスタマーサポート体制の立ち上げ、および自身もお客様からの問い合わせ対応をしながら業務プロセスの改善を進めてきました。
弊社では問い合わせ対応に Zendesk (https://www.zendesk.co.jp/) を使用しています。Zendesk には [Zendesk Apps](https://developer.zendesk.com/documentation/apps/) と呼ばれる、Zendesk の標準機能を拡張する仕組みが備わっています。（Chrome 拡張のようなイメージです）
将来的に業務改善につながられるのでは？と思い、具体的にどういった開発体験になるのか、何ができるのかを調査したため、その結果をまとめます。

# 基本編

基本編では、基本となるアプリ開発の一連の手順を理解します。

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

コンソールに表示されているように、この状態で Zendesk のアカウントにログインし、Support のチケット URL の後ろに `?zcli_apps=true` をつけてアクセスします。
（URL は https://d3v-\*\*\*.zendesk.com/agent/tickets/1?zcli_apps=true のようになるはずです）

![](https://storage.googleapis.com/zenn-user-upload/06a9745b9a41-20231025.png)

チケット詳細画面の右側にアプリが表示されました。

## パッケージングとデプロイ

`zcli apps:server` コマンドによってローカル開発が可能であることがわかりました。
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

# 応用編

## Zendesk Apps framework (ZAF) を使った Zendesk リソースの操作

## Zendesk Garden を使った UI 開発

## 外部 API へのリクエスト

## React を使う

# リファレンス

- [zendesk/zcli: A command-line tool for Zendesk](https://github.com/zendesk/zcli) （GitHub リポジトリ）
