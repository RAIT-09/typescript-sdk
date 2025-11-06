# 2. クイックスタート

このセクションでは、ACP TypeScript SDKを使用してエージェントとクライアントを実装する基本的な方法を学びます。実際に動作するコード例を通じて、ACPプロトコルの実装方法を理解できます。

---

## 2.1 エージェントの実装

エージェント（Agent）は、AIを使用してコードを自律的に修正するプログラムです。ここでは、最小限の機能を持つエージェントを実装します。

### 基本構造

エージェントを実装するには、`Agent`インターフェースを実装したクラスを作成し、`AgentSideConnection`で接続を確立します。

```typescript
import * as acp from "@agentclientprotocol/sdk";

class ExampleAgent implements acp.Agent {
  private connection: acp.AgentSideConnection;
  private sessions: Map<string, AgentSession>;

  constructor(connection: acp.AgentSideConnection) {
    this.connection = connection;
    this.sessions = new Map();
  }

  // Agent インターフェースのメソッドを実装
}
```

### 必須メソッドの実装

#### 1. initialize - 接続の初期化

`initialize`メソッドは、クライアントとの接続確立時に呼ばれます。プロトコルバージョンとエージェントの機能を返します。

```typescript
async initialize(
  _params: acp.InitializeRequest,
): Promise<acp.InitializeResponse> {
  return {
    protocolVersion: acp.PROTOCOL_VERSION,
    agentCapabilities: {
      loadSession: false,  // セッション読み込みは未対応
    },
  };
}
```

**重要なポイント**:
- `PROTOCOL_VERSION`定数を使用して現在のプロトコルバージョンを返す
- `agentCapabilities`でサポートする機能を宣言する
- `loadSession: false`は既存セッションの復元機能を持たないことを示す

#### 2. newSession - セッションの作成

`newSession`メソッドは、新しい会話セッションを作成します。

```typescript
interface AgentSession {
  pendingPrompt: AbortController | null;
}

async newSession(
  _params: acp.NewSessionRequest,
): Promise<acp.NewSessionResponse> {
  // ユニークなセッションIDを生成
  const sessionId = Math.random().toString(36).substring(2);

  // セッション状態を保存
  this.sessions.set(sessionId, {
    pendingPrompt: null,
  });

  return {
    sessionId,
  };
}
```

**重要なポイント**:
- ユニークなセッションIDを生成して返す
- セッションごとの状態（会話履歴、設定など）を保存する
- `AbortController`でキャンセル可能な状態を管理する

#### 3. authenticate - 認証処理

`authenticate`メソッドは、認証が必要な場合に呼ばれます。

```typescript
async authenticate(
  _params: acp.AuthenticateRequest,
): Promise<acp.AuthenticateResponse | void> {
  // 認証不要の場合は空のレスポンスを返す
  return {};
}
```

**重要なポイント**:
- 認証が不要な場合は空のオブジェクト`{}`を返す
- 認証が必要な場合は、トークンやAPIキーを検証する

#### 4. prompt - プロンプトの処理

`prompt`メソッドは、ユーザーからのメッセージを処理し、AI応答を生成します。これがエージェントの中核機能です。

```typescript
async prompt(params: acp.PromptRequest): Promise<acp.PromptResponse> {
  const session = this.sessions.get(params.sessionId);

  if (!session) {
    throw new Error(`Session ${params.sessionId} not found`);
  }

  // 既存のプロンプトをキャンセル
  session.pendingPrompt?.abort();
  session.pendingPrompt = new AbortController();

  try {
    // AI応答を生成（詳細は後述）
    await this.simulateTurn(params.sessionId, session.pendingPrompt.signal);
  } catch (err) {
    if (session.pendingPrompt.signal.aborted) {
      return { stopReason: "cancelled" };
    }
    throw err;
  }

  session.pendingPrompt = null;

  return {
    stopReason: "end_turn",
  };
}
```

**重要なポイント**:
- セッションIDでセッション状態を取得
- `AbortController`でキャンセル可能にする
- キャンセルされた場合は`stopReason: "cancelled"`を返す
- 正常終了時は`stopReason: "end_turn"`を返す

#### 5. cancel - キャンセル処理

`cancel`メソッドは、進行中のプロンプト処理をキャンセルします。

```typescript
async cancel(params: acp.CancelNotification): Promise<void> {
  this.sessions.get(params.sessionId)?.pendingPrompt?.abort();
}
```

**重要なポイント**:
- 通知型メソッド（応答を返さない）
- `AbortController.abort()`を呼び出してキャンセルシグナルを送る

### オプショナルメソッドの実装

#### setSessionMode - モード変更

```typescript
async setSessionMode(
  _params: acp.SetSessionModeRequest,
): Promise<acp.SetSessionModeResponse> {
  // モード変更ロジックを実装
  return {};
}
```

### エージェントからクライアントへの通信

エージェントは、`connection`オブジェクトを通じてクライアントにメッセージを送信できます。

#### sessionUpdate - 出力の送信

AI応答やツール実行結果をリアルタイムでクライアントに送信します。

```typescript
// テキストメッセージの送信
await this.connection.sessionUpdate({
  sessionId,
  update: {
    sessionUpdate: "agent_message_chunk",
    content: {
      type: "text",
      text: "こんにちは！どうお手伝いしましょうか？",
    },
  },
});
```

#### ツール呼び出しの通知

```typescript
// ツール呼び出しの開始を通知
await this.connection.sessionUpdate({
  sessionId,
  update: {
    sessionUpdate: "tool_call",
    toolCallId: "call_1",
    title: "ファイルを読み込んでいます",
    kind: "read",
    status: "pending",
    locations: [{ path: "/project/README.md" }],
    rawInput: { path: "/project/README.md" },
  },
});

// ツール実行の完了を通知
await this.connection.sessionUpdate({
  sessionId,
  update: {
    sessionUpdate: "tool_call_update",
    toolCallId: "call_1",
    status: "completed",
    content: [
      {
        type: "content",
        content: {
          type: "text",
          text: "# My Project\n\nプロジェクトの内容...",
        },
      },
    ],
    rawOutput: { content: "ファイルの内容..." },
  },
});
```

#### requestPermission - パーミッションの要求

機密性の高い操作を実行する前に、ユーザーの承認を求めます。

```typescript
const permissionResponse = await this.connection.requestPermission({
  sessionId,
  toolCall: {
    toolCallId: "call_2",
    title: "重要な設定ファイルを変更しています",
    kind: "edit",
    status: "pending",
    locations: [{ path: "/project/config.json" }],
    rawInput: {
      path: "/project/config.json",
      content: '{"database": {"host": "new-host"}}',
    },
  },
  options: [
    {
      kind: "allow_once",
      name: "この変更を許可",
      optionId: "allow",
    },
    {
      kind: "reject_once",
      name: "この変更をスキップ",
      optionId: "reject",
    },
  ],
});

// ユーザーの選択を処理
if (permissionResponse.outcome.outcome === "selected") {
  switch (permissionResponse.outcome.optionId) {
    case "allow":
      // 操作を実行
      break;
    case "reject":
      // 操作をスキップ
      break;
  }
} else if (permissionResponse.outcome.outcome === "cancelled") {
  // プロンプトがキャンセルされた
}
```

### 接続の確立

最後に、標準入出力を使用してエージェントを起動します。

```typescript
import { Readable, Writable } from "node:stream";

// stdoutに書き込み、stdinから読み込み
const input = Writable.toWeb(process.stdout);
const output = Readable.toWeb(process.stdin) as ReadableStream<Uint8Array>;

// JSON-RPCストリームを作成
const stream = acp.ndJsonStream(input, output);

// 接続を確立
new acp.AgentSideConnection((conn) => new ExampleAgent(conn), stream);
```

**重要なポイント**:
- `ndJsonStream`は改行区切りJSON（nd-JSON）フォーマットを処理
- stdoutに書き込み、stdinから読み込むことで、サブプロセスとして動作
- エージェントはクライアントからのメッセージを待機し続ける

---

## 2.2 クライアントの実装

クライアント（Client）は、エージェントをホストして制御するコードエディタやIDEです。ここでは、エージェントをサブプロセスとして起動し、対話する最小限のクライアントを実装します。

### 基本構造

クライアントを実装するには、`Client`インターフェースを実装し、`ClientSideConnection`で接続を確立します。

```typescript
import * as acp from "@agentclientprotocol/sdk";

class ExampleClient implements acp.Client {
  // Client インターフェースのメソッドを実装
}
```

### 必須メソッドの実装

#### 1. requestPermission - パーミッションリクエストの処理

エージェントからのパーミッション要求を受け取り、ユーザーに確認します。

```typescript
async requestPermission(
  params: acp.RequestPermissionRequest,
): Promise<acp.RequestPermissionResponse> {
  console.log(`🔐 パーミッション要求: ${params.toolCall.title}`);

  console.log(`\n選択肢:`);
  params.options.forEach((option, index) => {
    console.log(`   ${index + 1}. ${option.name} (${option.kind})`);
  });

  // ユーザー入力を受け取る（簡易実装）
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
  });

  const answer = await rl.question("\n選択してください: ");
  const optionIndex = parseInt(answer.trim()) - 1;

  if (optionIndex >= 0 && optionIndex < params.options.length) {
    return {
      outcome: {
        outcome: "selected",
        optionId: params.options[optionIndex].optionId,
      },
    };
  } else {
    // 無効な入力の場合は拒否
    return {
      outcome: {
        outcome: "selected",
        optionId: "reject",
      },
    };
  }
}
```

**重要なポイント**:
- `params.toolCall`にツール情報が含まれる
- `params.options`から選択肢を表示
- `outcome: "selected"`とともに選択された`optionId`を返す
- キャンセル時は`outcome: "cancelled"`を返す

#### 2. sessionUpdate - セッション更新の処理

エージェントからの出力更新を受信して表示します。

```typescript
async sessionUpdate(params: acp.SessionNotification): Promise<void> {
  const update = params.update;

  switch (update.sessionUpdate) {
    case "agent_message_chunk":
      // テキストメッセージを表示
      if (update.content.type === "text") {
        process.stdout.write(update.content.text);
      }
      break;

    case "tool_call":
      // ツール呼び出しの開始
      console.log(`\n🔧 ${update.title} (${update.status})`);
      break;

    case "tool_call_update":
      // ツール実行の更新
      console.log(`🔧 ${update.toolCallId}: ${update.status}`);
      break;

    case "plan":
    case "agent_thought_chunk":
    case "user_message_chunk":
      // その他の更新タイプ
      console.log(`[${update.sessionUpdate}]`);
      break;

    default:
      break;
  }
}
```

**重要なポイント**:
- 通知型メソッド（応答を返さない）
- `update.sessionUpdate`で更新タイプを判別
- `agent_message_chunk`: AIメッセージのストリーミング
- `tool_call`: ツール呼び出しの開始
- `tool_call_update`: ツール実行状況の更新

### オプショナルメソッドの実装

#### readTextFile - ファイル読み取り

```typescript
async readTextFile(
  params: acp.ReadTextFileRequest,
): Promise<acp.ReadTextFileResponse> {
  const content = await fs.readFile(params.path, "utf-8");
  return { content };
}
```

#### writeTextFile - ファイル書き込み

```typescript
async writeTextFile(
  params: acp.WriteTextFileRequest,
): Promise<acp.WriteTextFileResponse> {
  await fs.writeFile(params.path, params.content, "utf-8");
  return {};
}
```

### エージェントの起動と接続

クライアントはエージェントをサブプロセスとして起動し、stdio経由で通信します。

```typescript
import { spawn } from "node:child_process";
import { Readable, Writable } from "node:stream";

async function main() {
  // エージェントをサブプロセスとして起動
  const agentProcess = spawn("npx", ["tsx", "path/to/agent.ts"], {
    stdio: ["pipe", "pipe", "inherit"],  // stdin, stdout, stderr
  });

  // ストリームを作成
  const input = Writable.toWeb(agentProcess.stdin!);
  const output = Readable.toWeb(
    agentProcess.stdout!,
  ) as ReadableStream<Uint8Array>;

  // クライアント接続を作成
  const client = new ExampleClient();
  const stream = acp.ndJsonStream(input, output);
  const connection = new acp.ClientSideConnection((_agent) => client, stream);

  // 以降、connectionを使用してエージェントと通信
}
```

**重要なポイント**:
- `spawn`でエージェントプロセスを起動
- `stdio: ["pipe", "pipe", "inherit"]`で標準入出力をパイプ接続
- `ndJsonStream`でJSON-RPCメッセージをやり取り

### エージェントとの対話フロー

#### 1. 初期化

```typescript
const initResult = await connection.initialize({
  protocolVersion: acp.PROTOCOL_VERSION,
  clientCapabilities: {
    fs: {
      readTextFile: true,
      writeTextFile: true,
    },
    terminal: false,
  },
});

console.log(`✅ 接続しました (protocol v${initResult.protocolVersion})`);
```

#### 2. セッション作成

```typescript
const sessionResult = await connection.newSession({
  cwd: process.cwd(),           // 作業ディレクトリ
  mcpServers: [],               // MCPサーバー設定
});

console.log(`📝 セッション作成: ${sessionResult.sessionId}`);
```

#### 3. プロンプト送信

```typescript
const promptResult = await connection.prompt({
  sessionId: sessionResult.sessionId,
  prompt: [
    {
      type: "text",
      text: "こんにちは、エージェント！",
    },
  ],
});

console.log(`✅ 完了: ${promptResult.stopReason}`);
```

**Stop Reasons（終了理由）**:
- `"end_turn"`: 正常に完了
- `"cancelled"`: ユーザーがキャンセル
- `"max_turns"`: 最大ターン数に到達
- `"error"`: エラーが発生

#### 4. クリーンアップ

```typescript
try {
  // エージェントと対話
} finally {
  // プロセスを終了
  agentProcess.kill();
  process.exit(0);
}
```

---

## 2.3 最小限の実装例

ここでは、エージェントとクライアントの完全な最小実装を示します。これらは実際に動作するコードです。

### 最小限のエージェント実装

**ファイル: `minimal-agent.ts`**

```typescript
#!/usr/bin/env node
import * as acp from "@agentclientprotocol/sdk";
import { Readable, Writable } from "node:stream";

class MinimalAgent implements acp.Agent {
  private connection: acp.AgentSideConnection;

  constructor(connection: acp.AgentSideConnection) {
    this.connection = connection;
  }

  async initialize(): Promise<acp.InitializeResponse> {
    return {
      protocolVersion: acp.PROTOCOL_VERSION,
      agentCapabilities: { loadSession: false },
    };
  }

  async newSession(): Promise<acp.NewSessionResponse> {
    return {
      sessionId: Math.random().toString(36).substring(2),
    };
  }

  async authenticate(): Promise<void> {
    return;
  }

  async prompt(params: acp.PromptRequest): Promise<acp.PromptResponse> {
    // シンプルな応答を送信
    await this.connection.sessionUpdate({
      sessionId: params.sessionId,
      update: {
        sessionUpdate: "agent_message_chunk",
        content: {
          type: "text",
          text: "こんにちは！これは最小限のエージェントです。",
        },
      },
    });

    return { stopReason: "end_turn" };
  }

  async cancel(): Promise<void> {
    // キャンセル処理（何もしない）
  }
}

// 接続を確立
const input = Writable.toWeb(process.stdout);
const output = Readable.toWeb(process.stdin) as ReadableStream<Uint8Array>;
const stream = acp.ndJsonStream(input, output);
new acp.AgentSideConnection((conn) => new MinimalAgent(conn), stream);
```

### 最小限のクライアント実装

**ファイル: `minimal-client.ts`**

```typescript
#!/usr/bin/env node
import * as acp from "@agentclientprotocol/sdk";
import { spawn } from "node:child_process";
import { Readable, Writable } from "node:stream";

class MinimalClient implements acp.Client {
  async requestPermission(
    params: acp.RequestPermissionRequest,
  ): Promise<acp.RequestPermissionResponse> {
    // 常に承認
    return {
      outcome: {
        outcome: "selected",
        optionId: params.options[0].optionId,
      },
    };
  }

  async sessionUpdate(params: acp.SessionNotification): Promise<void> {
    const update = params.update;
    if (update.sessionUpdate === "agent_message_chunk") {
      if (update.content.type === "text") {
        process.stdout.write(update.content.text);
      }
    }
  }
}

async function main() {
  // エージェントを起動
  const agentProcess = spawn("npx", ["tsx", "minimal-agent.ts"], {
    stdio: ["pipe", "pipe", "inherit"],
  });

  const input = Writable.toWeb(agentProcess.stdin!);
  const output = Readable.toWeb(
    agentProcess.stdout!,
  ) as ReadableStream<Uint8Array>;

  const stream = acp.ndJsonStream(input, output);
  const connection = new acp.ClientSideConnection(
    () => new MinimalClient(),
    stream,
  );

  try {
    // 初期化
    await connection.initialize({
      protocolVersion: acp.PROTOCOL_VERSION,
      clientCapabilities: {},
    });

    // セッション作成
    const session = await connection.newSession({
      cwd: process.cwd(),
      mcpServers: [],
    });

    // プロンプト送信
    await connection.prompt({
      sessionId: session.sessionId,
      prompt: [{ type: "text", text: "Hello!" }],
    });
  } finally {
    agentProcess.kill();
    process.exit(0);
  }
}

main().catch(console.error);
```

### 実行方法

#### エージェントを単体で実行

```bash
npx tsx minimal-agent.ts
```

このモードでは、標準入力からJSON-RPCメッセージを受け付けます。手動でメッセージを送信することもできます：

```bash
echo '{"jsonrpc":"2.0","id":0,"method":"initialize","params":{"protocolVersion":1}}' | npx tsx minimal-agent.ts
```

#### クライアントを実行（エージェントを自動起動）

```bash
npx tsx minimal-client.ts
```

クライアントがエージェントをサブプロセスとして起動し、対話します。

### Zedエディタでエージェントをテストする

ACPエージェントは、Zedエディタから直接実行できます。

**Zedの設定ファイルに追加**:

```json
{
  "agent_servers": {
    "My Agent": {
      "command": "npx",
      "args": ["tsx", "/path/to/minimal-agent.ts"]
    }
  }
}
```

これで、ZedのAgent Panelから"My Agent"を起動して対話できます。

---

## 実装のポイントまとめ

### エージェント実装

1. **必須メソッド**: `initialize`, `newSession`, `authenticate`, `prompt`, `cancel`を実装
2. **セッション管理**: セッションIDごとに状態を保存
3. **キャンセル対応**: `AbortController`を使用してキャンセル可能にする
4. **リアルタイム更新**: `sessionUpdate`でクライアントに進捗を送信
5. **パーミッション**: 機密操作の前に`requestPermission`で承認を得る

### クライアント実装

1. **必須メソッド**: `requestPermission`, `sessionUpdate`を実装
2. **エージェント起動**: サブプロセスとして起動し、stdioで通信
3. **機能宣言**: `initialize`時に`clientCapabilities`で提供機能を宣言
4. **更新処理**: `sessionUpdate`でエージェントの出力をUIに反映
5. **クリーンアップ**: 終了時にプロセスをkill

### 共通の注意点

- **JSON-RPC 2.0**: すべての通信はJSON-RPC形式
- **nd-JSON**: 改行区切りJSONで複数メッセージを送受信
- **型安全性**: TypeScript型とZodスキーマで検証
- **エラーハンドリング**: `RequestError`を使用して適切なエラーを返す

---

## 次のステップ

第2章では、エージェントとクライアントの基本的な実装方法を学びました。次の章では、接続クラスの詳細なAPIリファレンスを提供します。

→ [3. 接続クラス](./03-connection-classes.md)
