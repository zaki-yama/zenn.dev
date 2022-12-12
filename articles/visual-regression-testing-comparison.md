---
title: "2023年にVisual Regression Testingを始めるならどんな選択肢があるか"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["フロントエンド", "vrt"]
publication_name: "loglass"
published: false
---

![](https://storage.googleapis.com/zenn-user-upload/b74f3f1447ad-20221208.png)

:::message
本記事は「[株式会社ログラス Product チーム Advent Calendar 2022](https://qiita.com/advent-calendar/2022/loglass)」12 日目の記事です。
:::

# はじめに

フロントエンドのテスト手法の 1 つに Visual Regression Testing（以下、VRT）があります。
これは、アプリケーションの画面を画像として保存し、画像の差分比較をすることで意図せぬ変更が生じていないかテストする方法です。
ここ数年で広く普及し、用語としても一般的になったように思います。

私も以前、とある OSS に reg-suit & Storycap を使った VRT を導入したことがあるのですが、その後もいくつか VRT のためのライブラリが登場したもののキャッチアップできていませんでした。
そこで今回は知識のアップデートを目的として、ここ最近登場した（と思われる）VRT のライブラリをいくつかご紹介します。

なお、今回紹介するツールはすべてこちらのリポジトリで試しています。
具体的な設定ファイルや動作結果を確認できるようになっていますので、ご興味があればぜひ。

https://github.com/zaki-yama-labs/visual-regression-testing-comparison

# 1. reg-suit + Storycap

https://github.com/reg-viz/reg-suit

個人的には現在もっとも普及しているのがこれかなと思います。
各社のテックブログでの事例紹介記事が多数見つかります。

- [Visual Regression Testing で安心できるフロントエンド環境を作る - Sansan Tech Blog](https://buildersbox.corp-sansan.com/entry/2021/03/18/110000)
- [Visual Regression Testing はじめました – 具体的な運用 Tips – PSYENCE:MEDIA](https://blog.recruit.co.jp/rmp/front-end/visual-regression-testing/)
- [ABEMA でスナップショットテストをやめて Visual Regression Testing に移行する話 | CyberAgent Developers Blog](https://developers.cyberagent.co.jp/blog/archives/29784/)

[reg-suit](https://github.com/reg-viz/reg-suit) は画像の差分検出を行う CLI で、これ自体に画像を保存する機能はありません。
そこで合わせて使われることが多いのが [Storycap](https://github.com/reg-viz/storycap) です。Storycap は Storybook の内容を画像として保存してくれる Storybook Addon です。

:::message
reg-suit に画像を保存する機能はないので、言い換えると Storycap 以外のツールと組み合わせれば Storybook に依存しない形での VRT が可能です。
:::

reg-suit を使うと、差分比較結果をこのようなレポート画面で確認できます。

![](https://storage.googleapis.com/zenn-user-upload/1d86c92137ea-20221208.gif)

詳しいセットアップ手順についてはすでに多くの解説記事があるため割愛しますが、主な特徴としては以下が挙げられます。

- 画像の保存先は S3 や Google Cloud Storage などのストレージサービスを用意する必要がある
  - これらのストレージサービスとの連携を簡単にセットアップするためのプラグインも提供されている
- GitHub Actions などの CI と連携し、PR 上で比較結果のサマリを確認できる

  ![](https://storage.googleapis.com/zenn-user-upload/59f8fab921f4-20221208.png)

- その他、Slack や Chatwork などのチャットサービスに通知するためのプラグインも提供されている

すでに AWS や GCP を別の用途で導入している場合はスムーズに導入ができると思います。
設定も `reg-suit init` コマンドで対話的に入力していくだけでほぼ完了します。

```
$ npx reg-suit init
[reg-suit] info version: 0.12.1
? Plugin(s) to install (bold: recommended) (Press <space> to select, <a> to toggle all, <i> to invert selection, and <enter> to proceed)
 ◉  reg-keygen-git-hash-plugin : Detect the snapshot key to be compare with using Git hash.
 ◉  reg-notify-github-plugin : Notify reg-suit result to GitHub repository
 ◉  reg-publish-s3-plugin : Fetch and publish snapshot images to AWS S3.
❯◯  reg-notify-chatwork-plugin : Notify reg-suit result to Chatwork channel.
 ◉  reg-notify-github-with-api-plugin : Notify reg-suit result to GHE repository using API
 ◯  reg-notify-gitlab-plugin : Notify reg-suit result to GitLab repository
 ◉  reg-notify-slack-plugin : Notify reg-suit result to Slack channel.
(Move up and down to reveal more choices)
```

# 2. Chromatic

https://www.chromatic.com/

[Chromatic](https://www.chromatic.com/) は Storybook のホスティングや VRT をマネジドサービスとして提供してくれる SaaS です。
Storybook のメンテナーである Dominic Nguyen（[@domyen](https://twitter.com/domyen)）氏が開発しています。

VRT としてできることは先ほどの reg-suit + Storycap とほぼ同等です。
特徴としてはマネジドサービスだけあってセットアップが非常に簡単に行える点です。
提供されている CLI を実行するとキャプチャの取得からデプロイ、差分比較までやってくれますし、画像の保存先を自前で用意する必要もありません。

![](https://storage.googleapis.com/zenn-user-upload/5353a0bcbe10-20221207.png =400x)
_セットアップは管理画面でリポジトリを選択し、その後のガイドに従うだけで完了する_

![](https://storage.googleapis.com/zenn-user-upload/ad2b13c4af23-20221208.gif)
_差分を見ながら Accept をポチポチ押してチェックを完了させる体験は良い_

気になる料金については [Pricing](https://www.chromatic.com/pricing) に記載されています。Free プランでも月 5000 スナップショットまでは使えるようです。

# 3. Playwright

[Playwright](https://github.com/microsoft/playwright) は Microsoft が開発しているブラウザ自動操作ライブラリです。
E2E テストの文脈で言及されることも多いですが、簡単に VRT が行える API も提供されています。

https://playwright.dev/docs/test-snapshots

以下はサンプルコードです（リンク先の公式ドキュメントから引用）。

```typescript
// example.spec.ts
import { test, expect } from "@playwright/test";

test("example test", async ({ page }) => {
  await page.goto("https://playwright.dev");
  await expect(page).toHaveScreenshot();
});
```

上記のように特定の URL にアクセスした後、 [`toHaveScreenshot()`](https://playwright.dev/docs/test-assertions#page-assertions-to-have-screenshot-2) を実行します。
`toHaveScreenshot()` は初回は画面のスナップショットを画像として出力し、2 回目以降は保存されている画像との差分比較を行います。

差分比較結果は先に紹介した reg-suit や Chromatic と似ています。

![](https://storage.googleapis.com/zenn-user-upload/3132a4b754e3-20221208.gif)

主な特徴としては次の通りです。

- 画面をキャプチャするためのメソッドが提供されているので導入は楽
- Storybook に依存しない。ただし対象は画面（URL）単位
- 画像はリポジトリにコミットして管理する

私も本記事を書くにあたって初めて試してみたのですが、テストが簡単に書けるのに加え `init` コマンドの流れで GitHub Actions の設定ファイルを自動的に作成してくれるなどの体験は良かったです。
一方、CI で実行する場合、差分が検出されてもすぐに目視で確認する術がなく、artifacts にアップロードしたレポートを手元にダウンロードして確認する感じになるため、先に紹介した 2 つのツールと比べると体験としては劣るのかな、という感想を持ちました。

# 4. Playwright のコンポーネント単位のテスト（experimental）

1 つ前で紹介した Playwright ですが、v1.22 から experimental な機能としてコンポーネント単位のテスト機能が提供されています。

https://playwright.dev/docs/test-components

記事執筆時点での最新バージョンは v1.28.1 ですが、ドキュメント上はまだ experimental となっていました。
以下はサンプルコードです（リンク先の公式ドキュメントから引用）。

```typescript
import { test, expect } from "@playwright/experimental-ct-react";
import App from "./App";

test.use({ viewport: { width: 500, height: 500 } });

test("should work", async ({ mount }) => {
  const component = await mount(<App />);
  await expect(component).toContainText("Learn React");
});
```

このテストにおいても `toHaveScreenshot()` が使えるため、コンポーネント単位での VRT が実現できます。
また設定ファイルや実行コマンドは従来のテストとは独立しているため、同じリポジトリ内に両者を共存させることができます。
（参考：[Q) Can I use `@playwright/test` and `@playwright/experimental-ct-{react,svelte,vue,solid}`?](https://playwright.dev/docs/test-components#q-can-i-use-playwrighttest-and-playwrightexperimental-ct-reactsveltevuesolid)）

まだ experimental ではありますが、Storybook に依存しない形でコンポーネント単位での VRT を手軽に導入できる手段が増えるのは良いですね。

# 5. lost-pixel

https://docs.lost-pixel.com/user-docs

最後に紹介する [lost-pixel](https://github.com/lost-pixel/lost-pixel) は今回試した中でもっとも新しいライブラリです。
私もどういうきっかけでこのライブラリを知ったか忘れてしまいましたが、メンテナーのツイートによると 2022 年 8 月 29 日に正式リリースされたばかりのようです。

https://twitter.com/DivDev_/status/1564224097390940163?s=20&t=xlG7p6vwwLmE7mxsGrxUAA

特徴としては

- Storybook, Ladle, Next.js をサポート。コンポーネント単位と画面単位の両方で VRT ができる
  - [Ladle](https://github.com/tajo/ladle) は Vite ベースの Storybook 代替ライブラリ
- CI は GitHub Actions 前提。[公式の GitHub Action](https://github.com/marketplace/actions/lost-pixel) が提供されている
- 差分比較画面のような機能は提供しない。差分があったら自動で PR を作る GitHub Actions を紹介するのでこれ使ってねというストロングスタイル
  - 参考：[Automatic baseline update PR - User Docs](https://docs.lost-pixel.com/user-docs/recipes/automatic-baseline-update-pr)
  - 試してみましたが、ワークフローは手動実行なのでちょっと面倒じゃないかなあ...

# おわりに

ここ 1,2 年ぐらいで見聞きした VRT ライブラリをいくつか紹介しました。
あくまで個人的なおすすめですが、セットアップの手間や差分確認のしやすさ、世に出ている情報の多さなどを考慮すると
今プロダクションに導入するなら引き続き reg-suit + Storycap かなと思いました。
ただ Storybook を導入していることが前提にはなってしまうので、そういう意味では Playwright のコンポーネント単位のテストは非常に可能性を感じます。

余談ですが、調べる過程で VRT というテーマ単体で Awesome リポジトリがあることを知りました。こちらについては全然見れませんでした。。。

https://github.com/mojoaxel/awesome-regression-testing

# 参考リンク

- reg-suit & Storycap
  - https://github.com/reg-viz/reg-suit
  - https://github.com/reg-viz/storycap
  - [Visual Regression Testing で安心できるフロントエンド環境を作る - Sansan Tech Blog](https://buildersbox.corp-sansan.com/entry/2021/03/18/110000)
  - [Visual Regression Testing はじめました – 具体的な運用 Tips – PSYENCE:MEDIA](https://blog.recruit.co.jp/rmp/front-end/visual-regression-testing/)
  - [ABEMA でスナップショットテストをやめて Visual Regression Testing に移行する話 | CyberAgent Developers Blog](https://developers.cyberagent.co.jp/blog/archives/29784/)
- Chromatic
  - https://www.chromatic.com/
  - [Pairs ブラウザ版でページ丸ごと Visual Regression Test したらすごいコスパ良かった | by Hiroshi Ohsuga | Eureka Engineering | Medium](https://medium.com/eureka-engineering/pairs%E3%83%96%E3%83%A9%E3%82%A6%E3%82%B6%E7%89%88%E3%81%A7%E3%83%9A%E3%83%BC%E3%82%B8%E4%B8%B8%E3%81%94%E3%81%A8visual-regression-test%E3%81%97%E3%81%9F%E3%82%89%E3%82%84%E3%81%A3%E3%81%B1%E3%82%8A%E3%82%B3%E3%82%B9%E3%83%91%E8%89%AF%E3%81%8B%E3%81%A3%E3%81%9F-726ad2352ace)
- Playwright
  - [Visual comparisons | Playwright](https://playwright.dev/docs/next/test-snapshots)
  - [Automating visual UI tests with Playwright and GitHub Actions · mmazzarolo.com](https://mmazzarolo.com/blog/2022-09-09-visual-regression-testing-with-playwright-and-github-actions/)
  - [Playwright & Vite ではじめる脱レガシー向け軽量 Visual Regression Test - Cybozu Inside Out | サイボウズエンジニアのブログ](https://blog.cybozu.io/entry/2022/03/18/100000)
  - [Playwright でフロントエンドの E2E テストを自動化してみた話](https://zenn.dev/mikana0918/articles/b6eb66377fb25a)
- lost-pixel
  - https://github.com/lost-pixel/lost-pixel
