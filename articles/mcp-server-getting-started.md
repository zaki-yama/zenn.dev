---
title: "MCPサーバー自作入門"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mcp", "cursor"]
published: false
---

# はじめに

MCP（Model Context Protocol）は、LLM（Large Language Model）がローカル環境のデータやサービスにアクセスするための標準プロトコルです。
このプロトコルを使うことで、LLMはローカル環境のファイルやデータベース、APIなどを安全に利用できるようになります。

本記事では、MCPサーバーを自作する方法について解説します。
MCPサーバーを自作することで、LLMに独自の機能を提供したり、ローカル環境のデータを活用した対話を実現したりすることができます。

## ブログ目次

- はじめに
- MCPの構成
- MCPのホスト
    - Claude Desktop
    - Cursor
- MCPサーバーの実装
- MCPサーバーの配布
- デバッグ
    - Inspector
    - Logging
- 余談：サーバーの実装をLLMに手伝ってもらう
    - [Building MCP with LLMs - Model Context Protocol](https://modelcontextprotocol.io/tutorials/building-mcp-with-llms)
- Cursorで動かす

# MCPの構成

MCPは以下のような構成要素で成り立っています。

## MCP Hosts

MCP Hostsは、LLMを実行するプログラムです。具体的には以下のようなものが該当します：

- Claude Desktop
- Cursor
- その他のAIツール

これらのプログラムは、MCPを通じてローカル環境のデータやサービスにアクセスしたいと考えています。

## MCP Clients

MCP Clientsは、MCP HostsとMCP Serversの間の通信を担当するプロトコルクライアントです。
1対1の接続を維持し、MCP HostsからのリクエストをMCP Serversに転送します。

## MCP Servers

MCP Serversは、ローカル環境のデータやサービスにアクセスするための軽量なプログラムです。
標準化されたModel Context Protocolを通じて、特定の機能を提供します。

## Local Data Sources

Local Data Sourcesは、MCP Serversがアクセスできるローカル環境のデータやサービスです。具体的には：

- ファイル
- データベース
- その他のローカルサービス

## Remote Services

Remote Servicesは、インターネットを通じて利用可能な外部システムです。
MCP Serversは、APIなどを通じてこれらのサービスに接続することができます。

# MCPのホスト

MCPのホストとして、現在主に2つのプログラムが利用されています。

## Claude Desktop

Claude Desktopは、Anthropicが提供するデスクトップアプリケーションです。
Claude 3.5 Sonnetをローカル環境で実行し、MCPを通じてローカル環境のデータやサービスにアクセスすることができます。

Claude DesktopでMCPサーバーを利用するには、以下の手順が必要です：

1. Claude Desktopをインストールする
2. MCPサーバーをインストールする
3. Claude Desktopの設定でMCPサーバーを有効にする

## Cursor

Cursorは、AIを活用したコードエディタです。
MCPを通じて、ローカル環境のファイルやデータベース、APIなどにアクセスすることができます。

CursorでMCPサーバーを利用するには、以下の手順が必要です：

1. Cursorをインストールする
2. MCPサーバーをインストールする
3. Cursorの設定でMCPサーバーを有効にする

Cursorでは、プロジェクトごとまたはグローバルにMCPサーバーを設定することができます。
プロジェクトごとの設定は `<project_root>/.cursor.mcp.json`、グローバルな設定は `~/.cursor/mcp.json` に記述します。

# MCPサーバーの実装

MCPサーバーを実装するには、以下のような手順で進めていきます。

## 1. 開発環境のセットアップ

まず、MCPサーバーを開発するための環境をセットアップします。
MCPサーバーは様々な言語で実装できますが、ここでは[TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)を使用した例を示します。

まずは、
```bash
# プロジェクトの作成
mkdir mcp-server-example
cd mcp-server-example

# 依存関係のインストール
npm init -y
npm install @modelcontextprotocol/typescript-sdk typescript ts-node @types/node
```

## 2. 基本的なサーバーの実装

MCPサーバーの基本的な実装例を示します。

```typescript
import { Server } from "@modelcontextprotocol/typescript-sdk";

const server = new Server(
  {
    name: "example-server",
    version: "1.0.0",
  },
  {
    capabilities: {
      resources: {},
      tools: {},
      prompts: {},
    },
  }
);

// リソースの定義
server.addResource({
  name: "example",
  content: "Hello, MCP!",
});

// ツールの定義
server.addTool({
  name: "echo",
  description: "Echoes back the input",
  parameters: {
    type: "object",
    properties: {
      message: {
        type: "string",
        description: "The message to echo",
      },
    },
    required: ["message"],
  },
  handler: async (params) => {
    return { result: params.message };
  },
});

// プロンプトの定義
server.addPrompt({
  name: "example",
  content: "You are a helpful assistant.",
});

// サーバーの起動
server.start();
```

## 3. サーバーの機能

MCPサーバーは以下の3つの主要な機能を提供できます：

### Resources

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
