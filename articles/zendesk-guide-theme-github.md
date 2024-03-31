---
title: "Zendesk GuideのテーマをGitHubで管理する"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["zendesk"]
publication_name: "loglass"
published: false
---

:::message
この記事は[毎週必ず記事がでるテックブログ "Loglass Tech Blog Sprint"](https://zenn.dev/loglass/articles/7298a3cd4c5fc6) の **33 週目**の記事です！
1 年間連続達成まで **残り 20 週** となりました！
:::

株式会社ログラス CRE（Customer Reliability Engineer）の [@zaki\_\_\_yama](https://twitter.com/zaki___yama) です。

弊社ではユーザー向けヘルプサイトの構築に Zendesk (https://www.zendesk.co.jp/)の Guide 機能を使用しています。Guide にはテーマのカスタマイズ機能があり、たとえば全体的な色味を自社のブランドカラーと合わせたり、テンプレートをカスタマイズすることでデザインだけでなく機能を拡張することが可能です。
弊社でもこれまで Guide のテーマには何度か軽微なカスタマイズを加えてきましたが、すべて管理画面から直接行っていました。
その結果、誰が・どういった経緯で・どういうカスタマイズを加えたのか、という歴史的経緯を紐解くのが難しい状態になっていました。

幸い Guide のテーマは GitHub との連携機能を提供しており、これを機に GitHub での管理を試してみました。
環境構築の手順や注意点を紹介します。

:::message
Guide の記事そのものを GitHub で管理する機能は Zendesk が提供していないため、本記事でも触れていません。
:::

# GitHub 連携をはじめる手順

## GitHub リポジトリにテーマを準備する

まずは、管理対象となる GitHub リポジトリを作成し、テーマを準備します。
Guide のテーマ管理画面（`https://***.zendesk.com/theming/workbench`）にアクセスし、管理したいテーマのメニューから「ダウンロード」を実行します。

![](https://storage.googleapis.com/zenn-user-upload/cd7d31daede7-20240329.png)

そして、ダウンロードしたファイルを GitHub のリポジトリにコミット・プッシュしておきます。

別の方法として、Zendesk の標準テーマ（Copenhagen）は GitHub のリポジトリでも公開されています。

https://github.com/zendesk/copenhagen_theme

このリポジトリを fork して始めるという方法もあります。
ただし、CI やビルド関連など、テンプレートそのものには必要ないファイルも含まれています。

## Zendesk で GitHub 連携を設定する

準備したリポジトリを Zendesk と連携します。
Guide のテーマ管理画面から「テーマを追加」>「GitHub から追加」を選択します。

![](https://storage.googleapis.com/zenn-user-upload/ca1861366481-20240331.png)

あとはダイアログに従い、準備したリポジトリのユーザー名・リポジトリ名、およびブランチ名（任意。省略するとデフォルトのブランチ）を入力します。

![](https://storage.googleapis.com/zenn-user-upload/f2e58e759041-20240331.png =350x)

成功すると、テーマライブラリにテーマが追加されます。
これで、テーマを GitHub リポジトリ上でカスタマイズする準備が整いました。

## ローカル環境でテーマ変更をプレビューする

[Zendesk CLI（ZCLI）](https://developer.zendesk.com/documentation/apps/getting-started/using-zcli/) を使うと、ローカルでファイルを変更した結果を Zendesk Guide 上でプレビューできます。
ZCLI については以前別の記事でも紹介したので、こちらの記事のインストールおよび認証の項を参考にしてください。

https://zenn.dev/loglass/articles/zendesk-app-getting-started

認証まで済んだら、リポジトリのルートで `zcli themes:preview` というコマンドを実行します。

```bash
$ zcli themes:preview
Uploading theme... Ok
Ready https://zaki-yama.zendesk.com/hc/admin/local_preview/start 🚀
You can exit preview mode in the UI or by visiting https://zaki-yama.zendesk.com/hc/admin/local_preview/stop
```

コンソールに表示されている `Ready` の横のリンクをブラウザで開くと、Guide のページがプレビューモードで開きます。
プレビューモードで開いていることは、ページ上部のヘッダーで確認できます。

![](https://storage.googleapis.com/zenn-user-upload/212397b63c31-20240401.png)

この状態でリポジトリ内のファイルを変更すると、ブラウザが自動的に更新され、変更内容をプレビューできます。

![](https://storage.googleapis.com/zenn-user-upload/f70cd74e3ca4-20240401.gif)
*例：style.cssを編集してボタンの色を変える*


## デプロイ

ローカルでの変更が完了したら、変更内容を Guide の設定にも反映します。
まず、テーマに含まれているマニフェストファイル（`manifest.json`）内のバージョン番号を更新します。

```diff:manifest.json
 {
   "name": "My Zendesk Theme",
   "author": "Zendesk",
-  "version": "3.2.0",
+  "version": "3.2.1",
   "api_version": 3,
   "default_locale": "en-us",
   "settings": [
```

テーマを更新する際、必ず**バージョン番号を現行の数字よりも増やす必要があります**。
現行バージョンからどれだけ増やすかは自由です。
また、バージョン番号の形式は [Semantic Versioning](https://semver.org/) に従うため、`MAJOR.MINOR.PATCH`の形式である必要があります（例： `3.2.0`）。
バージョン番号を増やしたら、リポジトリにコミット・プッシュします。

最後に、テーマの管理画面より、GitHub 連携しているテーマのメニューから「GitHub から更新」を選びます。

![](https://storage.googleapis.com/zenn-user-upload/6c6b359ca1a7-20240401.png =350x)

表示されているリポジトリ・ブランチ名が正しいことを確認し、「更新」をクリックします。

![](https://storage.googleapis.com/zenn-user-upload/3d12b0c13344-20240401.png =350x)

以上で、テーマを GitHub で管理する一連の流れを紹介しました。

# GitHub 管理にするメリット・デメリット

テーマを GitHub で管理するメリットとして、変更履歴や差分を容易に確認できる点や、Pull Request を使ったレビューフローを構築できる点などが挙げられます。
また、JS や CSS ファイルに関しては、ライブラリやビルドツールを使うことができる点もメリットとしてあるでしょう。
たとえば公式の [Copenhagen テーマのリポジトリ](https://github.com/zendesk/copenhagen_theme) を見ると、JS や CSS のコンパイルに rollup を、スタイルファイルには Sass を使用しています。

反面デメリットとしては、ちょっとした修正であっても GitHub 経由で行わなければならないため、git や GitHub に習熟していないメンバーにはやや敷居が高いと感じるでしょう。このあたりは Guide を管理するメンバーのスキルセットやチームの規模感によってどちらが適しているか変わってきそうです。
また、手順の中でも触れましたが、変更を適用するために毎回必ずマニフェストのバージョンを上げない点はちょっとだけ注意が必要だなと感じました。

# Tips

:::message alert
TODO
:::

## manifest.json の中身


## 更新時の「テーマ設定を維持する」オプション

## ZCLI でテーマを更新できないの？

# リファレンス

- [GitHub インテグレーションを使用した Guide テーマの設定 – Zendesk ヘルプ](https://support.zendesk.com/hc/ja/articles/4408832476698-GitHub%E3%82%A4%E3%83%B3%E3%83%86%E3%82%B0%E3%83%AC%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%82%92%E4%BD%BF%E7%94%A8%E3%81%97%E3%81%9FGuide%E3%83%86%E3%83%BC%E3%83%9E%E3%81%AE%E8%A8%AD%E5%AE%9A)
- [Guide における GitHub の管理対象のテーマの更新方法 – Zendesk ヘルプ](https://support.zendesk.com/hc/ja/articles/4408844123162-Guide%E3%81%AB%E3%81%8A%E3%81%91%E3%82%8BGitHub%E3%81%AE%E7%AE%A1%E7%90%86%E5%AF%BE%E8%B1%A1%E3%81%AE%E3%83%86%E3%83%BC%E3%83%9E%E3%81%AE%E6%9B%B4%E6%96%B0%E6%96%B9%E6%B3%95)
- [テーマに加えた変更をローカルでプレビューする方法 – Zendesk ヘルプ](https://support.zendesk.com/hc/ja/articles/4408822095642-%E3%83%86%E3%83%BC%E3%83%9E%E3%81%AB%E5%8A%A0%E3%81%88%E3%81%9F%E5%A4%89%E6%9B%B4%E3%82%92%E3%83%AD%E3%83%BC%E3%82%AB%E3%83%AB%E3%81%A7%E3%83%97%E3%83%AC%E3%83%93%E3%83%A5%E3%83%BC%E3%81%99%E3%82%8B%E6%96%B9%E6%B3%95)
- [Github インテグレーションの Guide テーマに関する問題のトラブルシューティング – Zendesk ヘルプ](https://support.zendesk.com/hc/ja/articles/4408829072538-Github%E3%82%A4%E3%83%B3%E3%83%86%E3%82%B0%E3%83%AC%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%AEGuide%E3%83%86%E3%83%BC%E3%83%9E%E3%81%AB%E9%96%A2%E3%81%99%E3%82%8B%E5%95%8F%E9%A1%8C%E3%81%AE%E3%83%88%E3%83%A9%E3%83%96%E3%83%AB%E3%82%B7%E3%83%A5%E3%83%BC%E3%83%86%E3%82%A3%E3%83%B3%E3%82%B0)
- [ヘルプセンターテーマのカスタマイズ – Zendesk ヘルプ](https://support.zendesk.com/hc/ja/articles/4408839332250-%E3%83%98%E3%83%AB%E3%83%97%E3%82%BB%E3%83%B3%E3%82%BF%E3%83%BC%E3%83%86%E3%83%BC%E3%83%9E%E3%81%AE%E3%82%AB%E3%82%B9%E3%82%BF%E3%83%9E%E3%82%A4%E3%82%BA)
  - テンプレート内の *.hbs ファイルについての説明が記載されています
- [テーマの「設定」パネルのカスタマイズ – Zendesk ヘルプ](https://support.zendesk.com/hc/ja/articles/4408846524954-%E3%83%86%E3%83%BC%E3%83%9E%E3%81%AE-%E8%A8%AD%E5%AE%9A-%E3%83%91%E3%83%8D%E3%83%AB%E3%81%AE%E3%82%AB%E3%82%B9%E3%82%BF%E3%83%9E%E3%82%A4%E3%82%BA)
  - manifest.json について詳しい説明が記載されています
- https://github.com/zendesk/zcli/blob/master/docs/themes.md
  - ZCLI の `themes` コマンドのリファレンス
