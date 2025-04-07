---
title: "MCPサーバー自作入門"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mcp", "cursor"]
published: false
---

## はじめに

すでに日本語でも紹介記事が多数ありますが、私もMCP（Model Context Protocol）サーバーの開発を試してみたので備忘録として。

MCPサーバー開発用のSDKは Python/TypeScript/Java/Kotlin版が提供されていますが、本記事ではTypeScript SDKを使用します。
また実装したMCPサーバーを利用するMCPホストにはCursorを使用します。

基本的に公式ドキュメントを参考にしています。

https://modelcontextprotocol.io/

本記事で触れること

- TypeScript SDKを用いたMCPサーバーの実装方法
- 実装したMCPサーバーの配布(Publish)方法
- 実装したMCPサーバーをCursorに組み込んで使用する方法
- デバッグ方法：Inspector の使い方、ロギング

本記事では触れないこと

- MCPのアーキテクチャ
- MCPの通信の仕組み

## MCPサーバーの実装

MCPサーバーを実装するには、以下のような手順で進めていきます。

### 1. 開発環境のセットアップ

まず、MCPサーバーを開発するための環境をセットアップします。
MCPサーバーは様々な言語で実装できますが、ここでは[TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)を使用した例を示します。

まずは、プロジェクトを作成し必要なライブラリをインストールします。

```bash
mkdir mcp-server-example
cd mcp-server-example

npm init -y
npm install @modelcontextprotocol/sdk zod
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

`tsconfig.json` を作成します。

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

MCPサーバーの基本的な実装例を示します。
ここでは公式ドキュメントの [Quickstart > For Server Developers](https://modelcontextprotocol.io/quickstart/server) を簡略化したものを実装します。

```typescript
#!/usr/bin/env node
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const NWS_API_BASE = "https://api.weather.gov";
const USER_AGENT = "weather-app/1.0";

// Create server instance
const server = new McpServer({
  name: "weather",
  version: "1.0.0",
});

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

interface AlertFeature {
  properties: {
    event?: string;
    areaDesc?: string;
    severity?: string;
    status?: string;
    headline?: string;
  };
}

// Format alert data
function formatAlert(feature: AlertFeature): string {
  const props = feature.properties;
  return [
    `Event: ${props.event || "Unknown"}`,
    `Area: ${props.areaDesc || "Unknown"}`,
    `Severity: ${props.severity || "Unknown"}`,
    `Status: ${props.status || "Unknown"}`,
    `Headline: ${props.headline || "No headline"}`,
    "---",
  ].join("\n");
}

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

server.tool(
  "get-forecast",
  "Get weather forecast for a location",
  {
    latitude: z.number().min(-90).max(90).describe("Latitude of the location"),
    longitude: z
      .number()
      .min(-180)
      .max(180)
      .describe("Longitude of the location"),
  },
  async ({ latitude, longitude }) => {
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
  },
);

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("Weather MCP Server running on stdio");
}

main().catch((error) => {
  console.error("Fatal error in main():", error);
  process.exit(1);
});
```

## 3. サーバーの機能

MCPサーバーは以下の3つの主要な機能を提供できます：

- Resources

Resourcesは、クライアントが読み取れるファイルライクなデータです。
APIレスポンスやファイルの内容などが該当します。

### Tools

Toolsは、LLMが呼び出すことができる関数です。
ローカル環境のデータやサービスにアクセスするための機能を提供します。

### Prompts

Promptsは、ユーザーが特定のタスクを遂行するのを手助けするテンプレートです。
LLMの振る舞いをカスタマイズするために使用されます。

# MCPサーバーの配布

MCPサーバーを配布するには、以下のような手順で進めていきます。

## 1. パッケージング

MCPサーバーを配布する前に、適切なパッケージングを行う必要があります。
TypeScriptで実装した場合、以下のような手順でパッケージングを行います：

```bash
# TypeScriptの設定ファイルを作成
npx tsc --init

# ビルド
npm run build

# パッケージング
npm pack
```

## 2. 配布方法

MCPサーバーは、以下のような方法で配布することができます：

### npm

npmを使用して配布する場合、以下の手順で行います：

1. npmにアカウントを作成する
2. パッケージを公開する
```bash
npm publish
```

### GitHub Releases

GitHub Releasesを使用して配布する場合、以下の手順で行います：

1. GitHubにリポジトリを作成する
2. リリースを作成する
3. ビルド済みのファイルをアップロードする

### その他の方法

その他にも、以下のような配布方法があります：

- Dockerイメージとして配布
- バイナリファイルとして配布
- インストーラーとして配布

## 3. インストール方法

ユーザーがMCPサーバーをインストールするには、以下のような方法があります：

### npmを使用する場合

```bash
npm install -g mcp-server-example
```

### GitHub Releasesからダウンロードする場合

1. リリースページから最新のバイナリをダウンロード
2. 実行権限を付与
```bash
chmod +x mcp-server-example
```

### その他の方法

- Dockerイメージをプルして実行
- インストーラーを実行
- バイナリを直接実行

# デバッグ

MCPサーバーをデバッグするには、以下のような方法があります。

## 1. Inspector

Inspectorは、MCPサーバーの動作を監視するためのツールです。
以下のような情報を確認することができます：

- リクエストとレスポンスの内容
- エラーメッセージ
- パフォーマンスメトリクス

Inspectorを使用するには、以下の手順で行います：

1. Inspectorをインストールする
2. MCPサーバーを起動する際に、Inspectorを有効にする
3. InspectorのUIで情報を確認する

## 2. Logging

Loggingは、MCPサーバーの動作を記録するための機能です。
以下のような情報を記録することができます：

- デバッグ情報
- エラー情報
- 警告情報

Loggingを使用するには、以下の手順で行います：

1. ログレベルを設定する
2. ログを出力する
3. ログファイルを確認する

## 3. その他のデバッグ方法

その他にも、以下のようなデバッグ方法があります：

- デバッガーを使用する
- テストを書く
- モニタリングツールを使用する

## Cursor へのインストール方法

Cursorでは、プロジェクトごとまたはグローバルにMCPサーバーを設定することができます。
プロジェクトごとの設定は `<project_root>/.cursor/mcp.json`、グローバルな設定は `~/.cursor/mcp.json` に記述します。

# Cursorで動かす

CursorでMCPサーバーを動かすには、以下のような手順で進めていきます。

## 1. Cursorのインストール

まず、Cursorをインストールします。
Cursorは、AIを活用したコードエディタで、MCPサーバーを利用してローカル環境のデータやサービスにアクセスすることができます。

## 2. MCPサーバーの設定

CursorでMCPサーバーを利用するには、以下の手順で設定を行います：

1. Cursorを開く
2. 設定を開く
3. MCPサーバーを有効にする
4. MCPサーバーの設定を行う

## 3. MCPサーバーの利用

MCPサーバーを利用するには、以下のような手順で行います：

1. プロジェクトを開く
2. MCPサーバーを起動する
3. CursorでMCPサーバーを利用する

## 4. 設定ファイル

Cursorでは、プロジェクトごとまたはグローバルにMCPサーバーを設定することができます。
プロジェクトごとの設定は `<project_root>/.cursor.mcp.json`、グローバルな設定は `~/.cursor/mcp.json` に記述します。

## 5. その他の機能

Cursorでは、MCPサーバーを利用して以下のような機能を提供しています：

- ローカルファイルの読み取り
- データベースへのアクセス
- APIの呼び出し
- その他のローカルサービスへのアクセス

## 参考リンク

- https://modelcontextprotocol.io/
- [Cursor – Model Context Protocol](https://docs.cursor.com/context/model-context-protocol)
