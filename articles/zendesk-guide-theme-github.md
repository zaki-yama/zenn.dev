---
title: "Zendesk GuideのテーマをGitHubで管理する"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zendesk"]
publication_name: "loglass"
published: true
---

:::message
この記事は[毎週必ず記事がでるテックブログ "Loglass Tech Blog Sprint"](https://zenn.dev/loglass/articles/7298a3cd4c5fc6) の **33 週目**の記事です！
1 年間連続達成まで **残り 20 週** となりました！
:::

株式会社ログラス CRE（Customer Reliability Engineer）の [@zaki\_\_\_yama](https://twitter.com/zaki___yama) です。

弊社ではユーザー向けヘルプサイトの構築に Zendesk (https://www.zendesk.co.jp/)のGuide機能を使用しています。
Guide にはテーマのカスタマイズ機能があり、たとえば全体的な色味を自社のブランドカラーと合わせたり、テンプレートをカスタマイズすることでデザインだけでなく機能を拡張することが可能です。
弊社でもこれまで Guide のテーマには何度か軽微なカスタマイズを加えてきましたが、すべて管理画面から直接行っていました。
その結果、誰が・どういった経緯で・どういうカスタマイズを加えたのか、という歴史的経緯を紐解くのが難しい状態になっていました。

幸い Guide のテーマは GitHub との連携もサポートしているため、これを機に GitHub での管理を試してみました。
環境構築の手順や注意点を紹介します。

# GitHub 連携をはじめる手順

## GitHub リポジトリにテーマを準備する

まずは、管理対象となる GitHub リポジトリを作成し、テーマを準備します。
Guide のテーマ管理画面（https://xxx.zendesk.com/theming/workbench）にアクセスし、管理したいテーマのメニューから「ダウンロード」を実行します。

![](https://storage.googleapis.com/zenn-user-upload/cd7d31daede7-20240329.png)

そして、ダウンロードしたファイルを GitHub のリポジトリにコミット・プッシュしておきます。

別の方法として、Zendesk の標準テーマ（Copenhagen）は GitHub のリポジトリでも公開されています。

https://github.com/zendesk/copenhagen_theme

このリポジトリを fork して始めるという方法もあります。
ただし、CIやビルド関連など、テンプレートそのものには必要ないファイルも含まれています。

## GitHub 連携を設定する

## ローカル環境でプレビュー

## デプロイ

# GitHub 管理にするメリット

# GitHub 管理にするデメリット

# リファレンス

- [ヘルプセンターテーマのカスタマイズ – Zendesk ヘルプ](https://support.zendesk.com/hc/ja/articles/4408839332250-%E3%83%98%E3%83%AB%E3%83%97%E3%82%BB%E3%83%B3%E3%82%BF%E3%83%BC%E3%83%86%E3%83%BC%E3%83%9E%E3%81%AE%E3%82%AB%E3%82%B9%E3%82%BF%E3%83%9E%E3%82%A4%E3%82%BA)
- [GitHub インテグレーションを使用した Guide テーマの設定 – Zendesk ヘルプ](https://support.zendesk.com/hc/ja/articles/4408832476698-GitHub%E3%82%A4%E3%83%B3%E3%83%86%E3%82%B0%E3%83%AC%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%97%E3%81%9FGuide%E3%83%86%E3%83%BC%E3%83%9E%E3%81%AE%E8%A8%AD%E5%AE%9A)
- [Guide における GitHub の管理対象のテーマの更新方法 – Zendesk ヘルプ](https://support.zendesk.com/hc/ja/articles/4408844123162-Guide%E3%81%AB%E3%81%8A%E3%81%91%E3%82%8BGitHub%E3%81%AE%E7%AE%A1%E7%90%86%E5%AF%BE%E8%B1%A1%E3%81%AE%E3%83%86%E3%83%BC%E3%83%9E%E3%81%AE%E6%9B%B4%E6%96%B0%E6%96%B9%E6%B3%95)
- [テーマに加えた変更をローカルでプレビューする方法 – Zendesk ヘルプ](https://support.zendesk.com/hc/ja/articles/4408822095642-%E3%83%86%E3%83%BC%E3%83%9E%E3%81%AB%E5%8A%A0%E3%81%88%E3%81%9F%E5%A4%89%E6%9B%B4%E3%82%92%E3%83%AD%E3%83%BC%E3%82%AB%E3%83%AB%E3%81%A7%E3%83%97%E3%83%AC%E3%83%93%E3%83%A5%E3%83%BC%E3%81%99%E3%82%8B%E6%96%B9%E6%B3%95)
- [Github インテグレーションの Guide テーマに関する問題のトラブルシューティング – Zendesk ヘルプ](https://support.zendesk.com/hc/ja/articles/4408829072538-Github%E3%82%A4%E3%83%B3%E3%83%86%E3%82%B0%E3%83%AC%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%AEGuide%E3%83%86%E3%83%BC%E3%83%9E%E3%81%AB%E9%96%A2%E3%81%99%E3%82%8B%E5%95%8F%E9%A1%8C%E3%81%AE%E3%83%88%E3%83%A9%E3%83%96%E3%83%AB%E3%82%B7%E3%83%A5%E3%83%BC%E3%83%86%E3%82%A3%E3%83%B3%E3%82%B0)
- [テーマの「設定」パネルのカスタマイズ – Zendeskヘルプ](https://support.zendesk.com/hc/ja/articles/4408846524954-%E3%83%86%E3%83%BC%E3%83%9E%E3%81%AE-%E8%A8%AD%E5%AE%9A-%E3%83%91%E3%83%8D%E3%83%AB%E3%81%AE%E3%82%AB%E3%82%B9%E3%82%BF%E3%83%9E%E3%82%A4%E3%82%BA)
