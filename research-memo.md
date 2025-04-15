## 参考リンク

- [MCPで広がるLLM　〜Clineでの動作原理〜](https://zenn.dev/codeciao/articles/cline-mcp-server-overview)
- [MCPでLLMに行動させる - Terraformを例とした tfmcp の紹介 - じゃあ、おうちで学べる](https://syu-m-5151.hatenablog.com/entry/2025/03/09/020057)
- [Model Context Protocol の現在地](https://zenn.dev/layerx/articles/9bdefe4d435882)
- [TypeScript で MCP サーバーを実装し、Claude Desktop から利用する](https://azukiazusa.dev/blog/typescript-mcp-server/)
- [MCPサーバーが切り拓く！自社サービス運用の新次元 - エムスリーテックブログ](https://www.m3tech.blog/entry/future-with-mcp-servers)
- [プロンプトだけでCloudflare Workersにブログを作る](https://zenn.dev/yusukebe/articles/dbcf9c451b0b0e)
    - MCPの自作例としてとてもわかりやすい


## TODO

- ✅ デバッグ方法（Inspector）
    - [デバッグ](https://www.notion.so/1b9e4f3f6a5780d5b1a0c596f3f3b711?pvs=21)
- ✅ CursorへのMCPサーバーインストール
    - [CursorへのMCPサーバーインストール](https://www.notion.so/Cursor-MCP-1bbe4f3f6a5780899e40cda1c6d8b6b6?pvs=21)
- MCPサーバーの公開・配布
    - ✅ パッケージとして公開、インストールさせる方法
        - →PythonならPyPi、などパッケージとして公開してるぽい
        - 例：https://github.com/MarkusPfundstein/mcp-obsidian
        - どういうフォーマット？
        - →サーバーを起動する、でいいのでは
- SSEとは？
- Resources、Prompts、Tools

## 用語

- コアコンセプト
    - Resources
    - Prompts
    - Tools
- Sampling
- Roots
- 

### 2025-03-17 公式ドキュメント

https://modelcontextprotocol.io/introduction

アーキテクチャ

- **MCP Hosts**: Programs like Claude Desktop, IDEs, or AI tools that want to access data through MCP
- **MCP Clients**: Protocol clients that maintain 1:1 connections with servers
- **MCP Servers**: Lightweight programs that each expose specific capabilities through the standardized Model Context Protocol
- **Local Data Sources**: Your computer’s files, databases, and services that MCP servers can securely access
- **Remote Services**: External systems available over the internet (e.g., through APIs) that MCP servers can connect to

[For Server Developers - Model Context Protocol](https://modelcontextprotocol.io/quickstart/server)

- `get-alerts` と `get-forecast` という2つのtool
- コアMCPコンセプト
    - Resouces：クライアントが読める、ファイルライクなデータ（APIレスポンスやファイルの中身）
    - Tools：LLMに呼ばれる関数
    - Prompts：ユーザーが特定のタスクを遂行するのを手助けするテンプレート
- このチュートリアルではtoolsにフォーカス
- 

### Claude Desktopトラブルシュート

[【Could not connect to MCP server】ClaudeのMCPサーバ接続エラーを解決する](https://zenn.dev/sun_asterisk/articles/58647adfba4609)

これだった。Nodeフルパスで指定

```
2025-03-17T15:31:00.412Z [info] [weather] Initializing server...
2025-03-17T15:31:00.432Z [error] [weather] spawn node ENOENT
2025-03-17T15:31:00.432Z [error] [weather] spawn node ENOENT
2025-03-17T15:31:00.439Z [info] [weather] Server transport closed
2025-03-17T15:31:00.439Z [info] [weather] Client transport closed
2025-03-17T15:31:00.439Z [info] [weather] Server transport closed unexpectedly, this is likely due to the process exiting early. If you are developing this MCP server you can add output to stderr (i.e. `console.error('...')` in JavaScript, `print('...', file=sys.stderr)` in python) and it will appear in this log.
2025-03-17T15:31:00.439Z [error] [weather] Server disconnected. For troubleshooting guidance, please visit our [debugging documentation](https://modelcontextprotocol.io/docs/tools/debugging)
2025-03-17T15:31:01.364Z [info] [weather] Initializing server...
2025-03-17T15:31:01.371Z [error] [weather] spawn node ENOENT
2025-03-17T15:31:01.372Z [error] [weather] spawn node ENOENT
```

別のエラー

![SS 2025-03-18 13.40.12.png](attachment:5cdf956a-f736-4454-b5e7-8ab146a0ca70:SS_2025-03-18_13.40.12.png)

```
2025-03-18T04:40:35.747Z [info] [weather] Message from client: {"method":"resources/list","params":{},"jsonrpc":"2.0","id":77}
2025-03-18T04:40:35.750Z [info] [weather] Message from server: {"jsonrpc":"2.0","id":77,"error":{"code":-32601,"message":"Method not found"}}
2025-03-18T04:40:35.751Z [info] [weather] Message from client: {"method":"prompts/list","params":{},"jsonrpc":"2.0","id":78}
2025-03-18T04:40:35.752Z [info] [weather] Message from server: {"jsonrpc":"2.0","id":78,"error":{"code":-32601,"message":"Method not found"}}

```

## デバッグ

### [Debugging - Model Context Protocol](https://modelcontextprotocol.io/docs/tools/debugging)

1. MCP Inspector
2. Claude Desktop Developer Tools
3. Server Logging

### Inspector

https://modelcontextprotocol.io/docs/tools/inspector

```
npx -y @modelcontextprotocol/inspector node build/index.js
```

npmとかパッケージ化されたものに対しては

```
npx -y @modelcontextprotocol/inspector npx server-postgres postgres://127.0.0.1/testdb
```

### Logging

https://modelcontextprotocol.io/docs/tools/debugging#implementing-logging

```json
server.sendLoggingMessage({
  level: "info",
  data: "Server started successfully",
});
```

これは capabilities に logging を指定する必要があり、かつそれはMcpServerクラスでは不可能

https://github.com/modelcontextprotocol/typescript-sdk/issues/175

```json
const server = new Server(
	{
		name: "example-server",
		version: "1.0.0",
	},
	{
		capabilities: {
			prompts: {},
			logging: {},
		},
	},
);
```

## CursorへのMCPサーバーインストール

[Cursor – Model Context Protocol](https://docs.cursor.com/context/model-context-protocol)

- プロジェクトごとなら `<project_root>/.cursor.mcp.json`
- グローバルにいれるなら `~/.cursor/mcp.json`

JSONにはこのように定義する

npmパッケージとして提供している場合

```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "mcp-server"],
      "env": {
        "API_KEY": "value"
      }
    }
  }
}
```

ローカルのファイル

```json
{
  "mcpServers": {
    "weather": {
      "command": "node",
      "args": [
        "/Users/yamazaki/repos/github.com/zaki-yama-labs/mcp-server-quickstart/build/index.js"
      ]
    }
  }

```

## MCPサーバーの配布(Publish)
