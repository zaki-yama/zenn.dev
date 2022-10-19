---
title: "Googleカレンダーの予定を色別に集計し、グラフをSlackに投稿するBotを作った"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gas", "slack"]
published: true
---

# はじめに

https://twitter.com/zaki___yama/status/1581797755785072640?s=20&t=I4BRfVe7_lD15rhKo7qylQ

こちらのツイートでも軽く紹介したのですが、普段業務をしている中で大ざっぱにでも何にどれくらい時間を使えていたのか振り返りたいなという思いがありました。
そのために、しばらく Google カレンダーの予定をカテゴリごとに何色かに分けて登録するようにしていたのですが、いかんせん登録しただけで振り返りがおざなりになっていました。

そこで、登録したカレンダーのカテゴリ毎の所要時間を集計し、かつ日次や週次で Slack に投稿してくれる Bot を Google Apps Script（以下 GAS）で作りました。

![](https://storage.googleapis.com/zenn-user-upload/6404ca3f114a-20221019.png)

# この Bot でできること

大きく 2 つです。

### 1. 日次の投稿：その日のカテゴリ別所要時間と、内訳を Slack に投稿する

![](https://storage.googleapis.com/zenn-user-upload/a5498433fb98-20221019.png)

### 2. 週次の投稿：1 週間分のカテゴリ別所要時間をグラフにしたものを Slack に投稿する

![](https://storage.googleapis.com/zenn-user-upload/aa8e119bbc16-20221019.png)

# インストール方法・使い方

以下のリポジトリの README に記載しています。

https://github.com/zaki-yama/google-calendar-events-analyzer

基本的には [テンプレートとなるスプレッドシート](https://docs.google.com/spreadsheets/d/1uf5XqUqcsIfwMdeJYZg6rJ3psmPPnJhMKf9OouLl55c/edit) をコピーしてもらえれば、コードのデプロイなどは行わずに誰でも利用できます。

# 技術的なこと

この Bot を作るにあたって技術的に考えたことや難しかったことを紹介します。

## Google カレンダーの予定をカテゴリごとに集計する

この Bot のメイン機能とも呼べる部分です。
まず、Google カレンダーの予定を取得する処理については特にハードルはありませんでした。GAS なので Google の他サービスとの連携は簡単に行えます。
今回で言えば、 `CalendarApp.getEventsForDay(targetDate)` を呼ぶだけで `targetDate` の予定を取得できます。
あとは辞退した予定や終日予定などを除外してやれば OK です。

```javascript
function fetchGoogleEvents(
  targetDate: Date
): GoogleAppsScript.Calendar.CalendarEvent[] {
  const events = CalendarApp.getEventsForDay(targetDate);

  // filter events so that:
  // - include only accepted events
  // - exclude allday events
  return events.filter(
    (event) =>
      !event.isAllDayEvent() &&
      (event.getMyStatus() === CalendarApp.GuestStatus.OWNER ||
        event.getMyStatus() === CalendarApp.GuestStatus.YES)
  );
}
```

そして、取得した予定の所要時間をカテゴリごとに集計する処理についてですが、
当初はその処理をスクリプトで実装していたものの、途中から
**「あ、これスプレッドシートのピボットテーブルでやれば実装しなくて済むのでは？」**
ということに気づき、スクリプトでは取得した予定にカテゴリ情報をくっつけてシートに保存するだけに留めました。

![](https://storage.googleapis.com/zenn-user-upload/7efc78ef158b-20221019.png)
_こういう形式で予定のカテゴリ・タイトル・開始終了時刻を保存する_

![](https://storage.googleapis.com/zenn-user-upload/dac10b4608b5-20221019.png)
_ピボットテーブルではこのように集計される_

またその場合、他の人に使ってもらうことを考えるとピボットテーブルの作成などは自動化できると初期セットアップの手間としては望ましいです。
が、そこについても「テンプレートとなるスプレッドシートを用意しておいて、コピーしてもらえばいいか」という発想になり、特にスクリプトでの実装は不要としました。

## グラフを Slack に投稿する

Slack にテキストメッセージを投稿するだけであれば、 Incoming Webhooks を使えば実現できます。
[Sending messages using Incoming Webhooks | Slack](https://api.slack.com/messaging/webhooks#advanced_message_formatting)

が、これには画像を投稿する機能はありません。画像を投稿するには `files.upload` API を使います。
[files.upload method | Slack](https://api.slack.com/methods/files.upload)

Incoming Webhooks は Webhook URL があれば投稿できますが、 `files.upload` は token が必要です。結果として両方とも入力してもらう形になってしまった。。。

## グラフのデータ範囲を GAS で更新する

作っていて一番面倒だったのが、グラフのデータ範囲の更新処理です。
週次で投稿するグラフに含めるのはその週の月〜金までの 5 日分のデータだけとしていたため、翌週には次の 5 日分がデータ範囲となるよう定期的に更新する必要がありました。
かつ、テーブルのヘッダー行は見出しとして常にデータ範囲に含める必要があります。

![](https://storage.googleapis.com/zenn-user-upload/c407b81e94d4-20221020.png)

GAS でこのようなグラフ操作をするためには、 `EmbeddedChartBuilder` クラスの [`addRange(range)`](https://developers.google.com/apps-script/reference/spreadsheet/embedded-chart-builder#addrangerange) および [`clearRanges()`](https://developers.google.com/apps-script/reference/spreadsheet/embedded-chart-builder#clearranges) を使用します。
サンプルコードは以下です。

```typescript
const chart = sheet.getCharts()[0];
sheet.updateChart(
  chart
    .modify()
    .clearRanges()
    .addRange(sheet.getRange(2, 1, 1, numColumns)) // Always include header row
    .addRange(
      sheet.getRange(sheet.getLastRow() + 1, 1, CHART_RANGE_DAYS, numColumns)
    )
    .build()
);
```

`addRange(range)` は文字通りデータ範囲を「追加」するため、先に `clearRanges()` で現在のデータ範囲をクリアします。
また `addRange(range)` は複数回適用できるので、これによってヘッダー行と実際のデータを含む行を順番に追加しています。

もう 1 点。細かいところで、ピボットテーブルには「総計」列を表示するオプションがありますが、この列はグラフには含めたくありません。
このやり方は以前個人ブログの方に書いたのでよければご覧ください。
[スプレッドシートのピボットテーブルで「総計を表示」しているかどうかを GAS で判定する - dackdive's blog](https://dackdive.hateblo.jp/entry/2022/09/21/092257)

## 入力用のフォームダイアログ

Slack の Webhook URL や Bot Token などをコードに直接埋め込みたくなかったため、フォームを用意してスクリプトプロパティに保存する仕組みにしています。

![](https://storage.googleapis.com/zenn-user-upload/d19132f9aeca-20221020.png)

こちらは別途記事にしています。

https://zenn.dev/zaki_yama/articles/gas-form-dialog

## モジュール分割

GAS の開発およびデプロイには [clasp](https://github.com/google/clasp) という Google 公式 CLI を使用しました。
clasp は標準で TypeScript をサポートしているなど便利な反面、 ES Modules をサポートしていないという問題があります。

[https://github.com/google/clasp/blob/master/docs/typescript.md#modules-exports-and-imports](https://github.com/google/clasp/blob/master/docs/typescript.md#modules-exports-and-imports)

> Currently, Google Apps Script does **not** support ES modules. Hence the typical `export`/`import` pattern cannot be used and the example below will fail:

また、この問題に対しては以下 3 つのワークアラウンドが紹介されています。

1. `declare const exports:` を使う（未検証。よくわからん）
2. namespace を使う
3. Rollup などのモジュールバンドラを使う

いずれかの方法でファイルを分割してコードを整理したいなと思ったのですが、やりたいことに対して複雑すぎる点や、Web 上のスクリプトエディタを使ったデバッグがやりにくくなる点を鑑み、結局 `main.ts` に全部ごちゃっと書いたままになっています。イケてない。。。

# おわりに

というわけでまだ少しやりたいことはあるのですが、一旦形になったのでソースコード含め公開しました。

https://github.com/zaki-yama/google-calendar-events-analyzer

使ってみていただければ幸いです。
