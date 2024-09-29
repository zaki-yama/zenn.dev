---
title: "Cloudflare WorkersのCron TriggersでHabitifyの習慣記録を定期的にSlackに流す"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudflare", "habitify", "slack"]
publication_name: "loglass"
published: false
---

:::message
この記事は[毎週必ず記事がでるテックブログ "Loglass Tech Blog Sprint"](https://zenn.dev/loglass/articles/7298a3cd4c5fc6) の **59 週目**の記事です！
2 年間連続達成まで **残り 47 週** となりました！
:::

CRE（Customer Reliability Engineer）の山﨑（[@zaki\_\_\_yama](https://twitter.com/zaki___yama)）です。
今回は業務と全く関係ない内容です。

## はじめに

最近、日々の習慣化したい行動を[Habitify](https://www.habitify.me/ja)というアプリに記録するようにしました。
きっかけとなったのはこちらのブログ記事です。

https://kakakakakku.hatenablog.com/entry/2024/07/09/183335

https://kakakakakku.hatenablog.com/entry/2024/02/14/201844

特に「10分間読書」という取り組みが、本を読むのが苦手な自分にとって非常に良いなと感じたこと、
またメモに読んだ本を記録することでそのときにどんな本を読んでいたのか・どれくらいかかっていたのかを振り返れるのも良いと思い、真似させていただくことにしました。

参考記事ではメモの集計方法も紹介されていたのですが、せっかくであればこの作業を自動化し、月ごとにSlackに流すことで振り返るきっかけになればと思い、そのような仕組みを構築しました。

今回、構築には Cloudflare Workers を使いました。Cloudflare Workers を選んだのに大した理由はなく、**ずっと Cloudflare に入門したいと思ってできずにいたので手頃なお題が欲しかった**ことと、後述する Cron Triggers という機能を使えば Worker の処理を任意のタイミングで定期実行できるという情報をなんとなく知っていたからでした。

----------

本記事では、以下の前提条件があると仮定しています。

- **Cloudflare Workers** の基本的な使い方が分かっていること
- **Slack Webhooks** やAPIの基本操作を理解していること
- **Habitify API** にアクセスするためのトークンを取得していること

また、開発に必要なツールとして、Cloudflare CLIである **Wrangler** を使用します。セットアップ方法は公式ドキュメント（[Cloudflare Workers: Get Started](https://developers.cloudflare.com/workers/get-started/guide/)）を参照してください。

## 1. Cloudflare Workers のプロジェクト作成

Cloudflare の公式ドキュメント
[Get started - CLI](https://developers.cloudflare.com/workers/get-started/guide/)
に従い、新規プロジェクトを作成します。

```
$ npm create cloudflare@latest -- habitify-summary-notifier
$ cd habitify-summary-notifier
$ npm run start
```

http://localhost:8787 で Worker が立ち上がるので、ブラウザで開くか、curl などのコマンドでアクセスします。
以降も、コードを修正しながら上記 URL にアクセスして動作を確認していきます。


## 2. Habitify API を使い、習慣データを取得する

まず、API を叩くための API key を取得します。
アプリの設定画面から確認できます。

続いて、API を叩く処理を実装します。習慣（Habits）ごとにメモ（Notes）が記録されており、それぞれを取得するAPIは分かれています。
そのため、先に習慣の一覧を取得し、そのidを元にメモを取得していきます。

それぞれの API ドキュメントは以下の通りです。

- 習慣（Habits）：https://docs.habitify.me/core-resources/habits
- メモ（Notes）：https://docs.habitify.me/core-resources/notes



## 3. Slack にポストする

Slack への投稿については、すでに多くの記事がありますので詳細は割愛します。
Incoming webhooks を使用します。

[Sending messages using incoming webhooks | Slack](https://api.slack.com/messaging/webhooks)

また、投稿文については [Block Kit Builder](https://api.slack.com/tools/block-kit-builder) で作ったものを参考に JSON を直接組み立てていますが、もう少し凝ったフォーマットにするなら [jsx-slack](https://github.com/yhatt/jsx-slack) などのライブラリも検討しようと思います。

ステップ2 で集計した、習慣ごと・メモごとの回数を集計したオブジェクトを、Slack 

```typescript
function buildBlocks(habits: NoteCountByHabit) {
  const res = [];
  for (const [habitName, noteCounts] of Object.entries(habits)) {
    res.push({
      type: "header",
      text: { type: "plain_text", text: habitName, emoji: true },
    });
    const noteCountsString = Object.keys(noteCounts)
      .map((note) => {
        return `[${noteCounts[note].toString().padStart(2, " ")}] ${note}`;
      })
      .join("\n");
    res.push({
      type: "rich_text",
      elements: [
        {
          type: "rich_text_preformatted",
          border: 0,
          elements: [{ type: "text", text: noteCountsString }],
        },
      ],
    });
  }
  return res;
}
```

```javascript
const blocks = buildBlocks(habits);
const body = JSON.stringify({ blocks });

await fetch(SLACK_WEBHOOK_URL, {
  method: "POST",
  headers: {
    "Content-Type": "application/json"
  },
  body: JSON.stringify(message)
});
```

## 4. Cron Triggers を設定する

まず、`wrangler.toml` ファイルで、定期実行のスケジュールを設定します。例えば、毎日午前9時に実行するには以下のように記述します。

```toml
[[triggers]]
schedule = "0 9 * * *"
```

## 5. クレデンシャルは Secrets に隠す

## 6. デプロイ

## 料金・制限事項


------


### 2.1 APIリクエストの作成

Habitify APIを使用して、ユーザーの習慣データを取得します。以下のリクエスト例では、指定した期間のノート（習慣ログ）を取得しています。

```bash
curl -X GET "https://api.habitify.me/core-resources/notes?from=2024-09-01&to=2024-09-30" \
-H "Authorization: Bearer ${HABITIFY_API_KEY}"
```

レスポンス例：

```json
{
  "errors": [],
  "message": "Success",
  "data": [
    {
      "id": "note_id_1",
      "content": "ジム",
      "created_date": "2024-09-02T22:50:52",
      "note_type": 1,
      "habit_id": "habit_id_1"
    },
    {
      "id": "note_id_2",
      "content": "腹筋ローラー",
      "created_date": "2024-09-09T20:36:47",
      "note_type": 1,
      "habit_id": "habit_id_1"
    }
  ],
  "version": "v1.2",
  "status": true
}
```

取得したデータをもとに、Slackに送信するメッセージを生成します。

## 4. セキュリティとデプロイ

### 4.1 Secretsの使用

Cloudflare Workersでは、環境変数を使用する代わりにSecretsを使用することで、APIキーなどの機密情報を安全に扱うことができます。以下のコマンドで、Habitify APIのキーをSecretsに追加します。

```bash
npx wrangler secret put HABITIFY_API_KEY
```

### 4.2 デプロイ

すべての設定が完了したら、Cloudflare Workersにデプロイします。

```bash
npx wrangler publish
```

これで、Cron Triggersによって定期的にHabitifyの習慣データがSlackに送信される仕組みが完成しました。

---

この構成をベースにさらに詳細な内容を追加したり、コードの具体例を発展させたりすることで、ブログの完成度を高められます。
