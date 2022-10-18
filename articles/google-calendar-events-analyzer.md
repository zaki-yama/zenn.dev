---
title: "Googleカレンダーの予定を色別に集計し、グラフをSlackに投稿するBotを作った"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gas", "slack"]
published: false
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

## Google カレンダーの予定をカテゴリーごとに集計する

この Bot のメイン機能とも呼べる部分です。
まず、Google カレンダーの予定を取得する処理については特にハードルはありませんでした。GAS なので Google の他サービスとの連携は簡単に行えます。
今回で言えば、 `CalendarApp.getEventsForDay(targetDate)` を呼ぶだけで `targetDate` の予定を取得できます。
あとは辞退した予定や週日予定などを除外してやれば OK です。

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
「あ、これスプレッドシートのピボットテーブルにやらせれば実装しなくて済むのでは？」
ということに気づき、スクリプトでは取得した予定にカテゴリ情報をくっつけてシートに保存するだけに留めました。

![](https://storage.googleapis.com/zenn-user-upload/7efc78ef158b-20221019.png)

![](https://storage.googleapis.com/zenn-user-upload/dac10b4608b5-20221019.png)

またその場合、他の人に使ってもらうことを考えるとピボットテーブルの作成などは自動化できると初期セットアップの手間としては望ましいです。
が、そこについても「テンプレートとなるスプレッドシートを用意しておいて、コピーしてもらえばいいのでは？」という発想になり、特にスクリプトでの実装は不要としました。

## グラフを Slack に投稿する

## グラフ範囲を GAS で更新する

## 入力用のフォームダイアログ

## モジュール分割
