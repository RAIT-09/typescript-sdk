# 1. はじめに

## 1.1 概要

### ACP（Agent Client Protocol）とは

**Agent Client Protocol (ACP)** は、コードエディタ（インタラクティブなソースコード閲覧・編集プログラム）と、コーディングエージェント（生成AIを使用してコードを自律的に修正するプログラム）間の通信を標準化するプロトコルです。

ACPは以下の目的で設計されています：

- **標準化された通信**: 異なるエディタとエージェントが相互運用可能な共通プロトコル
- **双方向のインタラクション**: エディタ（クライアント）とエージェント間で双方向にメソッドを呼び出し可能
- **型安全な実装**: TypeScript型定義による開発時の安全性確保
- **拡張性**: カスタムメソッドによる独自機能の追加が可能

### このSDKの役割

**@agentclientprotocol/sdk** は、ACPの公式TypeScript実装です。このSDKを使用することで、以下を開発できます：

1. **ACPエージェント**: AIを使用してコードを自律的に修正するエージェント
2. **ACPクライアント**: エージェントをホストして制御するコードエディタやIDE

### 主要な特徴

- **JSON-RPC 2.0ベース**: 業界標準のRPCプロトコルを使用
- **厳格な型定義**: すべてのプロトコルメッセージに対するTypeScript型
- **ランタイムバリデーション**: Zodスキーマによる実行時の型検証
- **セッション管理**: 複数の独立した会話コンテキストをサポート
- **パーミッションシステム**: ツール実行前のユーザー承認メカニズム
- **ファイルシステムアクセス**: エディタ環境内のファイル読み書き
- **ターミナルサポート**: シェルコマンドの実行と出力キャプチャ
- **接続ライフサイクル管理**: AbortSignalとPromiseによるクリーンな終了処理

### 公式リソース

- **プロトコル仕様**: https://agentclientprotocol.com
- **GitHubリポジトリ**: https://github.com/agentclientprotocol/typescript-sdk
- **NPMパッケージ**: https://www.npmjs.com/package/@agentclientprotocol/sdk
- **TypeDoc APIリファレンス**: https://agentclientprotocol.github.io/typescript-sdk

---

## 1.2 インストール

### 基本インストール

npmを使用してパッケージをインストールします：

```bash
npm install @agentclientprotocol/sdk
```

### パッケージ情報

- **パッケージ名**: `@agentclientprotocol/sdk`
- **現在のバージョン**: 0.5.1
- **ライセンス**: Apache 2.0
- **作者**: Zed Industries

### 依存関係

このSDKには以下のランタイム依存関係があります：

- **zod** (^3.0.0): TypeScript優先のスキーマバリデーションライブラリ
  - すべての受信プロトコルメッセージの検証に使用
  - 型安全なパラメータ解析とランタイムバリデーション

### モジュールシステム

このパッケージはES Modulesとして配布されています：

```json
{
  "type": "module",
  "main": "dist/acp.js",
  "types": "dist/acp.d.ts"
}
```

**インポート例**:

```typescript
// ES Modules
import * as acp from "@agentclientprotocol/sdk";

// 個別のエクスポート
import { AgentSideConnection, ClientSideConnection, RequestError } from "@agentclientprotocol/sdk";
```

### TypeScript設定

このSDKはTypeScript 5.9以上で開発されており、ES2022をターゲットとしています。プロジェクトで使用する場合、以下の設定を推奨します：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true
  }
}
```

---

## 1.3 基本コンセプト

### Agent（エージェント）

**エージェント**は、生成AIを使用してコードを自律的に修正するプログラムです。

**役割**:
- クライアントからのリクエストを処理
- 言語モデル（LLM）を使用してユーザープロンプトに応答
- ツール（ファイル読み書き、コマンド実行など）を呼び出してタスクを実行
- セッション状態と会話履歴を管理

**実装するインターフェース**: `Agent`

**必須メソッド**:
- `initialize`: プロトコルバージョンと機能のネゴシエーション
- `newSession`: 新しい会話セッションの作成
- `authenticate`: 認証処理
- `prompt`: ユーザープロンプトの処理
- `cancel`: 進行中の操作のキャンセル

### Client（クライアント）

**クライアント**は、エージェントをホストして制御するコードエディタやIDEです。

**役割**:
- エージェントとの接続を確立
- ユーザーインターフェースを提供
- エージェントからのパーミッションリクエストを処理
- ファイルシステムアクセスを提供
- ターミナル環境を提供
- エージェントからの更新通知を受信して表示

**実装するインターフェース**: `Client`

**必須メソッド**:
- `requestPermission`: ツール実行の承認リクエストを処理
- `sessionUpdate`: エージェントからの出力更新を受信

**オプショナルメソッド**:
- `readTextFile`: ファイル読み取り
- `writeTextFile`: ファイル書き込み
- `createTerminal`: ターミナル作成
- `terminalOutput`: ターミナル出力取得
- その他のターミナル操作メソッド

### Session（セッション）

**セッション**は、エージェントとユーザー間の独立した会話コンテキストです。

**特徴**:
- 一意のセッションIDで識別
- 独自の会話履歴と状態を保持
- 複数のセッションを同時に実行可能
- MCPサーバーとの接続を管理

**セッションのライフサイクル**:
1. **作成**: `newSession`メソッドで新規作成、または`loadSession`で既存セッションを復元
2. **対話**: `prompt`メソッドでユーザーメッセージを送信し、エージェントが応答
3. **モード変更**: `setSessionMode`で操作モードを切り替え（例: "ask", "code", "architect"）
4. **終了**: セッションは明示的に終了されるか、接続が切断されるまで継続

### JSON-RPC 2.0ベースの通信

ACPは**JSON-RPC 2.0**プロトコルに基づいています。

**メッセージタイプ**:

1. **Request（リクエスト）**: 応答を期待するメソッド呼び出し
   ```json
   {
     "jsonrpc": "2.0",
     "id": 1,
     "method": "initialize",
     "params": {"protocolVersion": 1}
   }
   ```

2. **Response（レスポンス）**: リクエストに対する応答
   ```json
   {
     "jsonrpc": "2.0",
     "id": 1,
     "result": {"protocolVersion": 1, "agentCapabilities": {...}}
   }
   ```

3. **Error Response（エラーレスポンス）**: エラーが発生した場合
   ```json
   {
     "jsonrpc": "2.0",
     "id": 1,
     "error": {"code": -32601, "message": "Method not found"}
   }
   ```

4. **Notification（通知）**: 応答を期待しない一方向メッセージ
   ```json
   {
     "jsonrpc": "2.0",
     "method": "session/update",
     "params": {"sessionId": "abc123", "update": {...}}
   }
   ```

### Tool（ツール）とパーミッション

**ツール**は、エージェントが実行できる操作です。

**利用可能なツール**:
- **ファイルシステム操作**:
  - `fs/read_text_file`: テキストファイルの読み取り
  - `fs/write_text_file`: テキストファイルの書き込み
- **ターミナル操作**:
  - `terminal/create`: コマンド実行用のターミナル作成
  - `terminal/output`: ターミナル出力の取得
  - `terminal/wait_for_exit`: コマンド完了まで待機
  - `terminal/kill`: コマンドの強制終了
  - `terminal/release`: ターミナルリソースの解放

**パーミッションシステム**:

エージェントがツールを実行する前に、クライアントに対して`session/request_permission`メソッドで承認を要求できます。

**パーミッション結果**:
- `approved`: ユーザーが承認
- `denied`: ユーザーが拒否
- `cancelled`: プロンプトターンがキャンセルされた

### 双方向通信

ACPは双方向プロトコルです：

**Agent → Client**:
- `session/request_permission`: パーミッション要求
- `session/update`: 出力更新通知
- `fs/read_text_file`: ファイル読み取り
- `fs/write_text_file`: ファイル書き込み
- `terminal/*`: ターミナル操作

**Client → Agent**:
- `initialize`: 初期化
- `session/new`: セッション作成
- `session/load`: セッション読み込み
- `session/prompt`: プロンプト送信
- `session/cancel`: キャンセル通知
- `session/set_mode`: モード変更
- `authenticate`: 認証

### プロトコルメソッド定数

SDKは、すべてのプロトコルメソッド名を定数として提供しています：

```typescript
import { AGENT_METHODS, CLIENT_METHODS } from "@agentclientprotocol/sdk";

// エージェント向けメソッド
AGENT_METHODS.initialize          // "initialize"
AGENT_METHODS.session_new         // "session/new"
AGENT_METHODS.session_prompt      // "session/prompt"
AGENT_METHODS.session_cancel      // "session/cancel"
AGENT_METHODS.authenticate        // "authenticate"
AGENT_METHODS.session_load        // "session/load"
AGENT_METHODS.session_set_mode    // "session/set_mode"
AGENT_METHODS.session_set_model   // "session/set_model" (UNSTABLE)

// クライアント向けメソッド
CLIENT_METHODS.session_request_permission  // "session/request_permission"
CLIENT_METHODS.session_update             // "session/update"
CLIENT_METHODS.fs_read_text_file          // "fs/read_text_file"
CLIENT_METHODS.fs_write_text_file         // "fs/write_text_file"
CLIENT_METHODS.terminal_create            // "terminal/create"
CLIENT_METHODS.terminal_output            // "terminal/output"
CLIENT_METHODS.terminal_wait_for_exit     // "terminal/wait_for_exit"
CLIENT_METHODS.terminal_kill              // "terminal/kill"
CLIENT_METHODS.terminal_release           // "terminal/release"
```

---

## 1.4 プロトコルバージョン

### 現在のプロトコルバージョン

ACP TypeScript SDKは、**プロトコルバージョン1**を実装しています。

```typescript
import { PROTOCOL_VERSION } from "@agentclientprotocol/sdk";

console.log(PROTOCOL_VERSION); // 1
```

### バージョンネゴシエーション

プロトコルバージョンは、接続確立時の`initialize`メソッドでネゴシエーションされます。

**クライアントからの初期化リクエスト**:
```typescript
const initResponse = await connection.initialize({
  protocolVersion: acp.PROTOCOL_VERSION,
  clientCapabilities: {
    // クライアントの機能を宣言
  }
});
```

**エージェントからの応答**:
```typescript
async initialize(params: acp.InitializeRequest): Promise<acp.InitializeResponse> {
  return {
    protocolVersion: acp.PROTOCOL_VERSION,
    agentCapabilities: {
      loadSession: false, // セッション読み込みをサポートしない
    }
  };
}
```

### バージョン互換性

- クライアントとエージェントは、互いにサポートするプロトコルバージョンを交換します
- 両者が共通のバージョンをサポートしている必要があります
- 互換性のないバージョンの場合、接続は失敗します

### Capabilities（機能宣言）

プロトコルバージョンに加えて、クライアントとエージェントは`initialize`時に利用可能な機能を宣言します。

**エージェントの機能**:
```typescript
interface AgentCapabilities {
  loadSession?: boolean;  // 既存セッションの読み込みをサポート
}
```

**クライアントの機能**:
```typescript
interface ClientCapabilities {
  fs?: {
    readTextFile?: boolean;   // ファイル読み取りをサポート
    writeTextFile?: boolean;  // ファイル書き込みをサポート
  };
  terminal?: boolean;         // ターミナル操作をサポート
}
```

機能宣言により、実装側は動的に利用可能な機能を判断できます。宣言されていない機能を使用しようとすると、`Method not found`エラーが返されます。

### SDKバージョンとプロトコルバージョンの関係

- **SDKバージョン**: TypeScriptライブラリ自体のバージョン（現在: 0.5.1）
- **プロトコルバージョン**: ACPプロトコル仕様のバージョン（現在: 1）
- **スキーマバージョン**: ビルド時にダウンロードされるJSON Schema（現在: v0.6.3）

SDKバージョンが更新されても、プロトコルバージョンが同じであれば互換性が保たれます。

---

## 次のステップ

第1章では、ACPとこのTypeScript SDKの基本概念を学びました。次の章では、実際にエージェントやクライアントを実装するためのクイックスタートガイドを提供します。

→ [2. クイックスタート](./02-quickstart.md)
