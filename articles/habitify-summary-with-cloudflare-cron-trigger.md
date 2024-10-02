---
title: "Cloudflare WorkersのCron TriggersでHabitifyの習慣記録を定期的にSlackに流す"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudflare", "habitify", "slack"]
publication_name: "loglass"
published: true
---

:::message
この記事は[毎週必ず記事がでるテックブログ "Loglass Tech Blog Sprint"](https://zenn.dev/loglass/articles/7298a3cd4c5fc6) の **59 週目**の記事です！
2 年間連続達成まで **残り 47 週** となりました！
:::

CRE（Customer Reliability Engineer）の山﨑（[@zaki\_\_\_yama](https://twitter.com/zaki___yama)）です。
今回は業務と全く関係ない話です。

## はじめに

何事も長続きしないのが悩みなのですが、最近、日々の習慣化したい行動を [Habitify](https://www.habitify.me/ja) というアプリで記録するようにしました。
きっかけとなったのはこちらのブログ記事です。

https://kakakakakku.hatenablog.com/entry/2024/07/09/183335

アプリ自体も便利そうですが、特に「10 分間読書」という取り組みが本を読むのが苦手な自分にとって非常に良さそうだなと感じたこと、また読んだ本をメモに記録することでそのときにどんな本を読んでいたのか・どれくらいかかっていたのかを振り返れるのも良いと思い、真似させていただくことにしました。

![](https://storage.googleapis.com/zenn-user-upload/d9b0986cf0fc-20240930.png)
*左: 私の Habitify 画面。目標回数や計測方法（回数なのか時間なのか）を習慣ごとに設定できる
右: メモ機能で、どんな本を読んだのかを記録している*

参考記事ではメモの集計方法も紹介されていたのですが、せっかくなのでこの作業を自動化し、毎月 Slack に流すことで振り返るきっかけになればと思い、そのような仕組みを構築しました。

また今回、構築には [Cloudflare Workers](https://www.cloudflare.com/ja-jp/developer-platform/workers/) を使いました。Cloudflare Workers を選んだ理由としては、単に**ずっと Cloudflare を触ってみたいと思っていたので何かきっかけが欲しかった**というのが大きいのですが、加えて後述する**Cron Triggers という機能を使えば Worker の処理を任意のタイミングで定期実行できる**という情報がなんとなく記憶にあったからでした。

## 今回作ったもの

![](https://storage.googleapis.com/zenn-user-upload/3ce6d473da70-20241001.png)

上述した Habitify の習慣メモを集計し、Slack に投稿する Bot です。
コードはこちらのリポジトリにあります。

https://github.com/zaki-yama-labs/habitify-summary-with-cloudflare-cron-trigger

ここから実際に構築した内容について詳しく説明していきます。

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

まず、API を利用するための API key を取得します。
アプリの設定画面から確認できます。

![](https://storage.googleapis.com/zenn-user-upload/dbefbed9effd-20240930.png =300x)

続いて、API を利用してデータを取得する処理を実装します。習慣（Habits）ごとにメモ（Notes）が記録されており、それぞれを取得する API は分かれています。
そのため、先に習慣の一覧を取得し、その id を元にメモを取得していきます。

それぞれの API ドキュメントは次の通りです。

- 習慣（Habits）：https://docs.habitify.me/core-resources/habits
- メモ（Notes）：https://docs.habitify.me/core-resources/notes

習慣を取得する API については、特に言及するポイントはありません。

```typescript
type Habit = {
  id: string;
  name: string;
};

const response = await fetch("https://api.habitify.me/habits", {
  headers: {
    Authorization: HABITIFY_API_KEY,
  },
});
const json = (await response.json()) as { data: Habit[] };
const habits = json.data.map((data) => ({
  id: data.id,
  name: data.name,
}));
```

今のところ id と name しか使わないため、それ以外は除いています。

次に、メモを取得する API です。
こちらは、取得対象期間を `from`, `to` というクエリパラメータで指定する必要があります。どちらも、ISO8601 形式（`YYYY-MM-ddTHH:mm:ss+HH:mm`）で指定します。
（参考：[Date Format | API Documentation](https://docs.habitify.me/date-format)）

```typescript
import { format } from "@formkit/tempo";

type NoteCount = {
  [note: string]: number;
};

export type NoteCountByHabit = {
  [habitName: string]: NoteCount;
};

const today = new Date();
// 先月1日0:00:00
const from = new Date(today.getFullYear(), today.getMonth() - 1, 1, 0, 0, 0);
// 先月の末日23:59:59
const to = new Date(today.getFullYear(), today.getMonth(), 0, 23, 59, 59);

const searchParams = new URLSearchParams({
  from: format(from, "YYYY-MM-DDTHH:mm:ssZ"),
  to: format(to, "YYYY-MM-DDTHH:mm:ssZ"),
});

const noteCountByHabit: NoteCountByHabit = {};

for (const habit of habits) {
  const response = await fetch(
    `https://api.habitify.me/notes/${habit.id}?${searchParams.toString()}`,
    {
      headers: {
        Authorization: env.HABITIFY_API_KEY,
      },
    }
  );
  const json = (await response.json()) as { data: Note[] };
  console.log(json);
  const notesCount = json.data.reduce(
    (acc: { [note: string]: number }, item) => {
      // アイテムの content をキーにして集計
      acc[item.content] = (acc[item.content] || 0) + 1;
      return acc;
    },
    {}
  );

  console.log(notesCount);
  noteCountByHabit[habit.name] = notesCount;
}
```

（なお、Date 型を `YYYY-MM-DDTHH:mm:ss+HH:mm` 形式に変換するためだけに [Tempo](https://tempo.formkit.com/#format-tokens) というライブラリを使用しています）

ここで組み立てた `noteCountByHabit` の中身は次のようになっています。

```javascript
{
  // 習慣1
  "筋トレ": {
    // メモと、その登場回数
    "腹筋ローラー": 6,
    "ジム": 3
  },
  // 習慣2
  "読書": {
    "LeanとDevOpsの科学": 1,
    "Webブラウザセキュリティ": 14,
    "イシューからはじめよ": 6,
    "The BDD Book": 1
  },
  ...
}
```

## 3. Slack にポストする

Slack への投稿については、すでに多くの記事がありますので詳細は割愛します。
Incoming webhooks を使用します。

[Sending messages using incoming webhooks | Slack](https://api.slack.com/messaging/webhooks)

また、投稿文については [Block Kit Builder](https://api.slack.com/tools/block-kit-builder) で試したものを参考に JSON を直接組み立てていますが、もう少し凝ったフォーマットにするなら [jsx-slack](https://github.com/yhatt/jsx-slack) などのライブラリを使用すると良いと思います。

ステップ 2 で集計した、習慣ごと・メモごとの回数を集計したオブジェクトを元に、Slack のメッセージブロックを組み立てる処理です。

```typescript
function buildBlocks(noteCountByHabit: NoteCountByHabit) {
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
    "Content-Type": "application/json",
  },
  body: JSON.stringify(message),
});
```

## 4. Cron Triggers を設定する

ここまでで必要な処理は一通り完成しました。次は、これらを自動的に定期実行するために、Cron Triggers を設定していきます。

https://developers.cloudflare.com/workers/configuration/cron-triggers/

まず、`wrangler.toml` ファイルで、定期実行のスケジュールを設定します。例えば、毎月 1 日の日本時間（JTC）午前 9 時に実行するには次のように記述します。

```toml
[[triggers]]
schedule = "0 0 1 * *"
```

（UTC で指定するため、2 つ目の値は `0` としています）

次に、Worker のコードを微修正します。
元々プロジェクトを作成したときは `fetch()` という関数内に処理を記述していましたが、Cron Trigger から呼び出されるのは `scheduled()` という関数になります。
参考：[Scheduled Handler](https://developers.cloudflare.com/workers/runtime-apis/handlers/scheduled/)

```typescript
async function main() {
  // 元々 fetch に書いていた処理はこちらに移動
}

export default {
  // fetch() の代わりに scheduled() を定義
  async scheduled(event, env, ctx) {
    ctx.waitUntil(main());
  },
};
```

### Cron Triggers をローカル環境で動作確認する

`scheduled()` もローカル環境で動作確認できます。
それには、Worker 起動時に `--test-scheduled` オプションをつける必要があります。

```bash
$ npx wrangler dev --test-scheduled
```

また、アクセスする URL は http://localhost:8787/\_\_scheduled というように末尾に `__scheduled` をつける必要があります。

## 5. （optional）クレデンシャルは Secrets に格納する

今回使用した Habitify の API key や Slack の webhook URL は、コード中に埋め込むよりも環境変数のような仕組みで外から渡してあげたほうが安全です。
Cloudflare Workers には、こういった機密情報を扱うための [Secrets](https://developers.cloudflare.com/workers/configuration/secrets/) という仕組みがあるため、これを利用します。

ローカル環境においては、 `.dev.vars` というファイルを作成し、`KEY="VALUE"` 形式で定義します。

```:.dev.vars
HABITIFY_API_KEY="..."
SLACK_WEBHOOK_URL="https://hooks.slack.com/..."
```

定義した Secrets は `env` から取得できます。

```typescript
export default {
  async scheduled(event, env, ctx) {
    console.log(env.HABITIFY_API_KEY); // これで .dev.vars に定義した値が取れる
    ctx.waitUntil(main(env));
  },
} satisfies ExportedHandler<Env>;
```

また、本番環境にデプロイする際は、 `wrangler secret put <key>` というコマンドで作成する必要があります。

```bash
$ npx wrangler secret put HABITIFY_API_KEY

✔ Enter a secret value: … ****************************************************************
🌀 Creating the secret for the Worker "habitify-summary-notifier"
✨ Success! Uploaded secret HABITIFY_API_KEY
```

TypeScript ユーザー向けの余談ですが、 `.dev.vars` に定義した変数も `wrangler types` で生成される型定義ファイルの対象になるようです。
そのため、変数を定義したら `wrangler types` を実行すると良いでしょう。

## 6. デプロイ

デプロイ方法は通常の Worker と同じです。

```bash
$ npm run deploy
```

デプロイ後に Cron が期待通り設定されているかどうかは、Worker の設定画面 > トリガーイベント　で確認できます。

![](https://storage.googleapis.com/zenn-user-upload/dc1e6fb1871d-20241001.png)

## 料金・制限事項

Habitify の API は Free プランでも利用できます。
また、Cloudflare の Free プランの各種上限は次の通りです。

- 登録可能な Cron Triggers の数: 5 個まで
- Worker 1 回あたりの処理時間（CPU time）: 10ms まで
- Worker の実行回数: 100,000 requests/day または 1000 requests/min

参考：[Limits | Cloudflare Workers docs](https://developers.cloudflare.com/workers/platform/limits)

そのため、今回のような月に一度実行するぐらいであれば、Free プランで実現可能です。
（CPU time だけ気をつけなければいけませんが。習慣データが増えすぎるとだめかも）

## おわりに

Habitify に日々記録している習慣メモを毎月 Slack に流す仕組みを、Cloudflare Workers の Cron Triggers を使って実現しました。
Cloudflare Workers も Cron Triggers も初めて使ってみましたが、全くハマりどころなく実装できてしまったので技術的には面白みのない中身になってしまいました 😅

ちょっとした処理を定期実行するための手段として、Cron Triggers は非常に使い勝手が良いなと思いました！
