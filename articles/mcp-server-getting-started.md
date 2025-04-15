---
title: "MCPサーバー自作入門"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mcp", "cursor"]
published: false
---

## はじめに

すでに日本語でも紹介記事が多数ありますが、私も MCP（Model Context Protocol）サーバーの開発を試してみたので備忘録として。
MCP の仕組みはともかくまずは作り方が知りたい！という方向けです。

MCP サーバー開発用の SDK は Python, Java, TypeScript など複数の言語をサポートしていますが、本記事では[**TypeScript SDK**](https://github.com/modelcontextprotocol/typescript-sdk)**を使用**します。
また開発した MCP サーバーを利用する MCP クライアントには[**Cursor**](https://www.cursor.com/)**を使用**します。

基本的に公式ドキュメントを参考にしています。

https://modelcontextprotocol.io/

**🙆‍♂️本記事で触れること**

- TypeScript SDK を用いた MCP サーバーの実装方法
- 実装した MCP サーバーを Cursor で使用する方法
- 実装した MCP サーバーの配布(Publish)方法
- デバッグ方法：Inspector の使い方、ロギング

**🙅‍♂️本記事では触れないこと**

- MCP のアーキテクチャ
- MCP の通信の仕組み
- Server-Sent Events (SSE) での実装方法

## MCPサーバーの実装手順

参考：[Quickstart > For Server Developers](https://modelcontextprotocol.io/quickstart/server)

はじめに、MCP サーバーを実装し Cursor で利用するまでの流れを説明します。
次の手順で進めていきます。

### 1. 開発環境のセットアップ

まず、MCP サーバーを開発するために必要なライブラリをインストールします。

```bash
mkdir mcp-server-quickstart
cd mcp-server-quickstart

npm init -y
npm install @modelcontextprotocol/sdk
npm install -D @types/node typescript
```

`package.json` を修正し、 `type: "module"` を指定します。
また `build` スクリプトなどを定義します。

```json
{
  "type": "module",
  "bin": {
    "my-mcp-server": "./build/index.js"
  },
  "scripts": {
    "build": "tsc && chmod 755 build/index.js"
  },
  "files": [
    "build"
  ],
}
```

続いて、`tsconfig.json` を作成します。

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "./build",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules"]
}
```

### 2. 基本的なサーバーの実装

MCP サーバーの基本的な実装例を示します。
題材として、公式ドキュメントの [Quickstart > For Server Developers](https://modelcontextprotocol.io/quickstart/server) と同じく、[National Weather News (NWS)](https://www.weather.gov/documentation/services-web-api) の API を使って米国の気象情報を取得する MCP サーバーを作ります。

はじめにMCPサーバーの実装にあたり重要なポイントだけ抜き出したコード片を示し、最後に動作するコードの全文を記載します。

前提として MCP サーバーは **Resources, Tools, Prompts という3つの機能を提供できます**が、今回は Tools に絞っています。
これら 3 つの機能については後述します。
Tools は、MCP クライアントから呼び出せるアクション（群）ぐらいの理解で大丈夫です。

#### 2.1 `Server` インスタンスの生成

まず、MCP サーバーのインスタンスを作成します。

```typescript:src/index.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";

const server = new Server(
  {
    name: "weather",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
    },
  },
);
```

Tools を提供する場合、 `Server` インスタンスを作成する際の第二引数で `capabilities: { tools: {} }` を指定します。

#### 2.2 利用可能なツールの一覧を定義

続いて、この MCP サーバーが提供するツールの一覧を定義します。

```typescript
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "get_forecast",
        description: "Get weather forecast for a location",
        inputSchema: {
          type: "object",
          properties: {
            latitude: {
              type: "number",
              description: "Latitude of the location",
            },
            longitude: {
              type: "number",
              description: "Longitude of the location",
            },
          },
        },
      },
    ],
  };
});
```

今回はツールは1つだけですが、複数のツールも提供できるため配列で返します。
配列の1要素が1つのツール定義にあたり、中身は以下のプロパティを持つオブジェクトです。

- `name`: ツール名
- `description`: ツールの説明
- `inputSchema`: このツールを呼び出す際の引数となるパラメータ定義

#### 2.3 ツール実行時の処理を定義

```typescript
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name !== "get_forecast") {
    throw new Error("Unknown prompt");
  }

  const { latitude, longitude } = request.params.arguments as { latitude: number, longitude: number };

  // Get grid point data
  const pointsUrl = `${NWS_API_BASE}/points/${latitude.toFixed(4)},${longitude.toFixed(4)}`;
  const pointsData = await makeNWSRequest<PointsResponse>(pointsUrl);
  ...
  // Get forecast data
  ...
  // Format forecast periods
  ...
  return {
    content: [
      {
        type: "text",
        text: forecastText,
      },
    ],
  };
});
```

MCP クライアントが特定のツールを実行した際、第二引数の関数が呼ばれます。
`request.params.name` にリクエストされたツール名、 `request.params.arguments` に引数が渡されるので、ツール名に応じた処理を実装します。
ツール内の具体的な処理はMCPサーバーの実装方法とは関係ないため、省略します。

#### 2.4 サーバーを起動する

最後に以下の2行を記述します。

```typescript
const transport = new stdioservertransport();
await server.connect(transport);
```

これでMCPサーバーの実装は完了です。

:::details コード全文

```typescript:src/index.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  ListToolsRequestSchema,
  CallToolRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const NWS_API_BASE = "https://api.weather.gov";
const USER_AGENT = "weather-app/1.0";

// [1] サーバーインスタンスの初期化
const server = new Server(
  {
    name: "weather",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
    },
  },
);

interface ForecastPeriod {
  name?: string;
  temperature?: number;
  temperatureUnit?: string;
  windSpeed?: string;
  windDirection?: string;
  shortForecast?: string;
}

interface PointsResponse {
  properties: {
    forecast?: string;
  };
}

interface ForecastResponse {
  properties: {
    periods: ForecastPeriod[];
  };
}

// Helper function for making NWS API requests
async function makeNWSRequest<T>(url: string): Promise<T | null> {
  const headers = {
    "User-Agent": USER_AGENT,
    Accept: "application/geo+json",
  };

  try {
    const response = await fetch(url, { headers });
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    return (await response.json()) as T;
  } catch (error) {
    console.error("Error making NEW request:", error);
    return null;
  }
}

// [2] 利用可能なToolの一覧を返す
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "get_forecast",
        description: "Get weather forecast for a location",
        inputSchema: {
          type: "object",
          properties: {
            latitude: {
              type: "number",
              description: "Latitude of the location",
            },
            longitude: {
              type: "number",
              description: "Longitude of the location",
            },
          },
        },
      },
    ],
  };
});

// [3] Toolの利用
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name !== "get_forecast") {
    throw new Error("Unknown prompt");
  }

  const { latitude, longitude } = request.params.arguments as { latitude: number, longitude: number };

  // Get grid point data
  const pointsUrl = `${NWS_API_BASE}/points/${latitude.toFixed(4)},${longitude.toFixed(4)}`;
  const pointsData = await makeNWSRequest<PointsResponse>(pointsUrl);

  if (!pointsData) {
    return {
      content: [
        {
          type: "text",
          text: `Failed to retrieve grid point data for coordinates: ${latitude}, ${longitude}. This location may not be supported by the NWS API (only US locations are supported).`,
        },
      ],
    };
  }

  const forecastUrl = pointsData.properties?.forecast;
  if (!forecastUrl) {
    return {
      content: [
        {
          type: "text",
          text: "Failed to get forecast URL from grid point data",
        },
      ],
    };
  }

  // Get forecast data
  const forecastData = await makeNWSRequest<ForecastResponse>(forecastUrl);
  if (!forecastData) {
    return {
      content: [
        {
          type: "text",
          text: "Failed to retrieve forecast data",
        },
      ],
    };
  }

  const periods = forecastData.properties?.periods || [];
  if (periods.length === 0) {
    return {
      content: [
        {
          type: "text",
          text: "No forecast periods available",
        },
      ],
    };
  }

  // Format forecast periods
  const formattedForecast = periods.map((period: ForecastPeriod) =>
    [
      `${period.name || "Unknown"}:`,
      `Temperature: ${period.temperature || "Unknown"}°${period.temperatureUnit || "F"}`,
      `Wind: ${period.windSpeed || "Unknown"} ${period.windDirection || ""}`,
      `${period.shortForecast || "No forecast available"}`,
      "---",
    ].join("\n"),
  );

  const forecastText = `Forecast for ${latitude}, ${longitude}:\n\n${formattedForecast.join("\n")}`;

  return {
    content: [
      {
        type: "text",
        text: forecastText,
      },
    ],
  };
});

const transport = new StdioServerTransport();
await server.connect(transport);
```

:::


### 3. Cursor の設定

開発した MCP サーバーを Cursor で利用します。
Cursor では、**プロジェクトごと**または**グローバル**に MCP サーバーを設定できます。
プロジェクトごとの設定は `<project_root>/.cursor/mcp.json`、グローバルな設定は `~/.cursor/mcp.json` に記述します。

`mcp.json` の中身はこのような構成です。

```json
{
  "mcpServers": {
    "weather-forecast": {
      "command": "node",
      "args": [
        "/Users/yamazaki/repos/github.com/zaki-yama-labs/mcp-server-quickstart/build/index.js"
      ]
    }
  }
}
```

`weather-forecast` は MCP サーバー名を識別するための任意の文字列なので、好きに設定して構いません。
その中の `command` および `args` で MCP サーバーを起動するためのコマンドを指定します。

また、MCP サーバーが `API_KEY` などの環境変数を使用する場合、 `env` というパラメータで渡すことができます。

```json
{
  "mcpServers": {
    "weather-forecast": {
      "command": "node",
      "args": [
        "/Users/yamazaki/repos/github.com/zaki-yama-labs/mcp-server-quickstart/build/index.js"
      ],
      "env": {
        "API_KEY": "XXX"
      }
    }
  }
}
```

### 4. 動かしてみる

まさしく設定できると、Cursor Settings > MCP に MCP サーバーの情報が表示されます。

![](https://storage.googleapis.com/zenn-user-upload/7881328b0bb8-20250415.png)

最初は無効化されているので、Disabled 部分をクリックして有効化します。
エラーなく起動できた場合、緑色のインジケーターが表示されます。

![](https://storage.googleapis.com/zenn-user-upload/d6751c73adab-20250415.png)

さて、実際に動かしてみましょう。新しいチャットウィンドウで「ロサンゼルスの天気を教えて」と尋ねてみます。

![](https://storage.googleapis.com/zenn-user-upload/2c2d274b2b2f-20250415.png)

無事回答が返されました。
また、 `get_forecast` というツールが実行されたこと、その際のパラメータと戻り値が確認できました。

## MCP サーバーが提供する3つの機能

MCP サーバーは次の 3 つの主要な機能を提供します。

- [Resources](https://modelcontextprotocol.io/docs/concepts/resources)
  - MCP サーバーがクライアントに提供する読み取り専用のデータです。これには、ファイルの内容、データベースのレコード、API のレスポンスなどが含まれます
- [Tools](https://modelcontextprotocol.io/docs/concepts/tools)
  - MCP サーバーがクライアントに提供する実行可能な機能です。これにより、AI モデルが外部システムと連携したり、計算を行ったり、実世界での操作を実行することが可能になります
- [Prompts](https://modelcontextprotocol.io/docs/concepts/prompts)
  - 再利用可能なプロンプトテンプレートやワークフローです

また、これらのうち**どの機能をサポートしているかはクライアントによって異なります。**
公式ドキュメントのこちらのページに表が掲載されています。

https://modelcontextprotocol.io/clients

例として、代表的なクライアントである Claude Desktop, Cline, Cursor を比較すると、

- Claude Desktop はいずれも対応
- Cline は Tools, Resources に対応
- Cursor は Tools のみ

であることがわかります。

![](https://storage.googleapis.com/zenn-user-upload/80ceb2adb773-20250414.png)
*Claude Desktop, Cline, Cursorの対応状況<br>（表は https://modelcontextprotocol.io/clients#feature-support-matrix より引用）*

また、Cursor のドキュメントにも次の記載があります。

> MCP servers offer two main capabilities: tools and resources. Tools are availabe in Cursor today, and allow Cursor to execute the tools offered by an MCP server, and use the output in it’s further steps. However, resources are not yet supported in Cursor. We are hoping to add resource support in future releases.

（[Model Context Protocol > Limitations](https://docs.cursor.com/context/model-context-protocol#limitations)）



## MCPサーバーの配布(Publish)

本記事を執筆した 2025-04-14 時点では、npm のような**MCPサーバー専用のレジストリは存在しません。**
（ロードマップには記載があります：[https://modelcontextprotocol.io/development/roadmap#registry](https://modelcontextprotocol.io/development/roadmap#registry)）

そのため、MCP サーバーを配布する代表的な方法としては、次の選択肢があります。

1. npm や pip など、その言語でポピュラーなパッケージレジストリに publish する
2. Docker イメージとして配布する（例：[modelcontextprotocol/servers リポジトリの postgres サーバー](https://github.com/modelcontextprotocol/servers/tree/2025.4.6/src/postgres#docker)
3. ソースコードを公開し、利用者自身の端末でビルドしてもらう

ここでは 1 の例として、先ほど作成した MCP サーバーを npm レジストリに publish し、Cursor で使用する方法を紹介します。

まず、先ほどの `src/index.ts` の 1 行目に shebang を追加します。

```typescript
#!/usr/bin/env node
```

ビルド、publish などは通常の npm パッケージ開発の手順に従います。

最後に、Cursor の `mcp.json` をこのような書き方に変更します。
（`@zaki-yama-labs/mcp-server-quickstart` というパッケージ名だったとします）

```json:.cursor/mcp.json
{
  "mcpServers": {
    "weather-forecast": {
      "command": "npx",
      "args": [
        "-y",
        "@zaki-yama-labs/mcp-server-quickstart"
      ]
    }
  }
}
```

## デバッグ（Inspector、Logging）

開発中の MCP サーバーのデバッグ方法として、

1. MCP Inspector
2. Logging

の 2 つを紹介します。

なお、公式ドキュメントの [Debugging](https://modelcontextprotocol.io/docs/tools/debugging) にはもう 1 つ、Claude Desktop の Developer Tools を使う方法が紹介されていますが、本記事では割愛します。

### 1. MCP Inspector

[MCP Inspector](https://github.com/modelcontextprotocol/inspector) は、MCP サーバーの動作を GUI でテストするためのツールです。
このツールは npm パッケージとして提供されているため、起動方法は `npx @modelcontextprotocol/inspector` の後ろに MCP サーバー起動用のコマンドを並べるだけです。

```bash
# ローカルにあるMCPサーバーをテストする場合
$ npx @modelcontextprotocol/inspector node path/to/server/index.js args...

# npm パッケージのMCPサーバーをテストする場合
$ npx -y @modelcontextprotocol/inspector npx <package-name> <args>
```

起動すると、ローカルで inspector が起動するので、表示される URL にアクセスします。

```bash
# mcp-server-quickstart ディレクトリで
$ npx @modelcontextprotocol/inspector node build/index.js
Starting MCP inspector...
⚙️ Proxy server listening on port 6277
🔍 MCP Inspector is up and running at http://127.0.0.1:6274 🚀
```

この画面で、実装したツールの一覧を確認したり、実際にパラメータを渡してツールの動作をテストできます。

![](https://storage.googleapis.com/zenn-user-upload/df477ca1e64e-20250415.png)

### 2. Logging

参考：[Debugging > Implementing logging](https://modelcontextprotocol.io/docs/tools/debugging#implementing-logging)

サーバーにログを仕込むには、 `server.sendLoggingMessage()` というメソッドを使用します。

```typescript
server.sendLoggingMessage({
  level: "info",
  data: "Server started successfully",
});
```

説明不要かと思いますが、ログレベルとメッセージを渡します。
なお、注意点として**このメソッドは `server.connect(transport)` 後に実行する必要があります。**

## その他実装のポイント

### McpServer か Server か

公式ドキュメントの [Quickstart > For Server Developers](https://modelcontextprotocol.io/quickstart/server) と本記事の実装を見比べると、コードの書き方が異なることがわかります。
公式ドキュメントでは `McpServer` クラスを使用しているのに対し、本記事では `Server` クラスを使用しています。

SDK の [README](https://github.com/modelcontextprotocol/typescript-sdk?tab=readme-ov-file#low-level-server) を読むと、`Server` クラスの方がより低レベルなAPIで、細かいコントロールが可能な分記述は冗長になるようです。

今回のような簡単な MCP サーバーであれば `McpServer` クラスの利用で問題なさそうですが、次の理由から本記事では `Server` クラスの方を紹介しました。

- [modelcontextprotocols/servers](https://github.com/modelcontextprotocol/servers) リポジトリのコードを読むと、 `McpServer` で実装されているものはなかった
- `McpServer` クラスは logging に対応していない
  - 参考：[Property 'sendLoggingMessage' does not exist on type 'McpServer' · Issue #175 · modelcontextprotocol/typescript-sdk](https://github.com/modelcontextprotocol/typescript-sdk/issues/175)
- [Resources](https://modelcontextprotocol.io/docs/concepts/resources), [Prompts](https://modelcontextprotocol.io/docs/concepts/prompts), [Tools](https://modelcontextprotocol.io/docs/concepts/tools) のページのサンプルコードがいずれも `Server` クラスによる実装だった

### ツール名に関する注意

今回サンプルとして実装したツールには `get_forecast` という名前をつけました。
Cursor での利用を想定した場合、**ハイフンは使わず、アンダースコアを使う**必要があります。

理由として、なぜかはわかりませんがCursor は内部的にケバブケース（`get-forecast`）をスネークケース（`get_forecast`）に勝手に変換しているようです。
（そのうち直りそうですが）

参考：

https://forum.cursor.com/t/im-getting-mcp-error-with-the-latest-update/72733/50

### ツールの引数に zod を使う

本記事での実装では、簡単のため `inputSchema` にはツールに渡すパラメータ定義をベタ書きしました。

```typescript
inputSchema: {
  type: "object",
  properties: {
    latitude: {
      type: "number",
      description: "Latitude of the location",
    },
    longitude: {
      type: "number",
      description: "Longitude of the location",
    },
  },
},
```

別の方法としては、パラメータ定義に [zod](https://github.com/colinhacks/zod) を使い、[zod-to-json-schema](https://github.com/StefanTerdell/zod-to-json-schema) で JSON Schema に変換したものを渡すこともできます。

例：modelcontextprotocol/servers の GitHub サーバーの実装
（[http://github.com/modelcontextprotocol/servers/blob/2025.4.6/src/github](http://github.com/modelcontextprotocol/servers/blob/2025.4.6/src/github)）

パラメータ定義

```typescript:operations/issues.ts
export const GetIssueSchema = z.object({
  owner: z.string(),
  repo: z.string(),
  issue_number: z.number(),
});
```

利用可能なツール一覧の定義

```typescript:index.ts
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      ...
      {
        name: "get_issue",
        description: "Get details of a specific issue in a GitHub repository.",
        inputSchema: zodToJsonSchema(issues.GetIssueSchema)
      },
      ...
```

ツール実行

```typescript:index.ts
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  try {
    ...
    switch (request.params.name) {
      ...
      case "get_issue": {
        const args = issues.GetIssueSchema.parse(request.params.arguments);
        const issue = await issues.getIssue(args.owner, args.repo, args.issue_number);
        return {
          content: [{ type: "text", text: JSON.stringify(issue, null, 2) }],
        };
      }
  ...
});
```

### [未検証] FastMCP というフレームワークもある

本記事では SDK のみを用いた素朴な実装で説明しましたが、[FastMCP](https://github.com/punkpeye/fastmcp) というMCPサーバー開発用のフレームワークもあるようです。

## おまけ：MCP サーバーの開発を LLM に任せる

ここまで手作業での開発方法を紹介してきましたが、公式ドキュメントではサーバーの開発に LLM を活用する方法が紹介されています。

https://modelcontextprotocol.io/tutorials/building-mcp-with-llms

具体的には、次の手順に従い必要なコンテキストを LLM に渡すのがポイントのようです。

1. https://modelcontextprotocol.io/llms-full.txt 、および SDK の README や関連ドキュメントの内容をコピーし、LLM に渡す
2. 次のポイントをおさえながら、実装したいサーバーの仕様を LLM に指示する
   - 提供したいリソース、ツール、プロンプトは何か
   - 連携する外部システムは何か

```:プロンプト例（ドキュメントから引用）
Build an MCP server that:
- Connects to my company's PostgreSQL database
- Exposes table schemas as resources
- Provides tools for running read-only SQL queries
- Includes prompts for common data analysis tasks
```

Cursor の場合、上記ドキュメントを Docs として追加しておけば、 `@Doc` で簡単に参照できて便利そうですね。

## TODO: テスト

今回、公式ドキュメントを読んだ限りでは、MCP サーバーの動作をユニットテストで担保する方法については言及されていませんでした。
他の方の記事を読むと `InMemoryTransport` を使ったテストの書き方がありそうですが...調べてわかったらまた追記します。

- [Deno で RooCode 用にローカルMCPサーバーをさっと作る](https://zenn.dev/mizchi/articles/deno-mcp-server)
- [TypeScript で MCP サーバーを実装し、Claude Desktop から利用する](https://azukiazusa.dev/blog/typescript-mcp-server/)

## サンプルコード

今回動作確認に使ったサンプルコードはこちらのリポジトリにあります。

https://github.com/zaki-yama-labs/mcp-server-quickstart

## 参考リンク

今回、主に参考にしたのは MCP および Cursor の公式ドキュメント

https://modelcontextprotocol.io/

https://docs.cursor.com/context/model-context-protocol

ならびに、Anthropic が管理している MCP サーバーの公式実装が置かれたこちらのリポジトリです。

https://github.com/modelcontextprotocol/servers
