---
title: "Zendeskアプリ開発ことはじめ"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zendesk"]
publication_name: "loglass"
published: true
---

:::message
この記事は[毎週必ず記事がでるテックブログ "Loglass Tech Blog Sprint"](https://zenn.dev/loglass/articles/7298a3cd4c5fc6) の **X 週目**の記事です！
1 年間連続達成まで **残り XX 週** となりました！
:::

こんにちは。[株式会社ログラス](https://www.loglass.jp/) で CRE（Customer Reliability Engineer）をやっている山﨑（[@zaki\_\_\_yama](https://twitter.com/zaki___yama)）です。

CRE としての取り組みの 1 つとして、ここ半年ぐらいはカスタマーサポート体制の立ち上げ、および自身もお客様からの問い合わせ対応をしながら業務プロセスの改善を進めてきました。
弊社では問い合わせ対応に Zendesk (https://www.zendesk.co.jp/) を使用しています。Zendesk には [Zendesk Apps](https://developer.zendesk.com/documentation/apps/) と呼ばれる、Zendesk の標準機能を拡張する仕組みが備わっています。（Chrome 拡張のようなイメージです）
**将来的に業務改善につながられるのでは？と思い、具体的にどういった開発体験になるのか、何ができるのかを調査したため、その結果をまとめます。**

# 基本編

## Zendesk 開発環境の取得

## Zendesk Command Line Interface (ZCLI) のインストール

アプリ開発には専用の CLI が提供されています。

```bash
$ npm install -g @zendesk/zcli
```

でインストールできます。インストールすると `zcli` というコマンドが使えるようになります。

```bash
$ zcli help
$ zcli help                                                                                                                                                                   (git)-[zendesk-app-getting-started]
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

## アプリのひな形作成とローカル開発

## パッケージングとデプロイ

##

# 応用編

## Zendesk Apps framework (ZAF) を使った Zendesk リソースの操作

## Zendesk Garden を使った UI 開発

## 外部 API へのリクエスト

## React を使う
