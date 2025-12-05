# keruta coder provider 仕様

## 概要

keruta-coder-provider は、keruta-task-server からペンディングタスクを検知して coder 環境を起動する環境管理サービスです。keruta-task-server とは [KTCP (keruta task client protocol)](./task-client-protocol.md) を使用して通信し、タスクの実行は行わず、coder 環境の起動と管理のみを担当します。

## 責任範囲

### 実施する機能
- ペンディングタスクの検知
- coder 環境の起動
- 環境状態の管理

### 実施しない機能
- タスクの実際の実行（coder が担当）
- タスク結果の処理
- ログの収集・保存

## アーキテクチャ

```
┌─────────────────┐          KTCP          ┌─────────────────┐    ┌─────────────────┐
│ keruta-task-    │◄─────────────────────►│keruta-coder-    │───▶│     coder       │
│   server        │    WebSocket only     │   provider      │    │  (タスク実行)    │
│ (タスク管理)     │                       │ (環境管理のみ)   │    │                 │
└─────────────────┘                       └─────────────────┘    └─────────────────┘
```

## 外部インターフェース

### keruta-task-server との連携

#### KTCP 通信

keruta-task-server とは [KTCP (keruta task client protocol)](./task-client-protocol.md) を使用して通信します。

**接続先**: `ws://keruta-task-server:8080/ws/ktcp` または `wss://keruta-task-server:8080/ws/ktcp` (推奨)

**使用する KTCP メッセージタイプ**:
- `authenticate` - 認証
- `heartbeat` - 生存確認
- `task_list` - タスク一覧取得要求
- `task_list_response` - タスク一覧取得応答
- `task_status_update` - タスク状態更新
- `error` - エラー通知

詳細は [task-client-protocol.md](./task-client-protocol.md) を参照してください。

### coder との連携

coder CLI を使用して、以下の操作を実行します。

#### ワークスペース起動

**コマンド**: `coder start <workspace-name>`

**必要な環境変数**:
- `CODER_URL`: coder サーバー URL
- `CODER_SESSION_TOKEN`: 認証トークン

#### ワークスペース状態確認

**コマンド**: `coder list --output json`

## 設定仕様

### 環境変数

| 変数名 | 説明 | デフォルト値 | 必須 |
|--------|------|-------------|------|
| `KERUTA_TASK_SERVER_WS_URL` | keruta-task-server の WebSocket URL | `ws://localhost:8080/ws/ktcp` | ○ |
| `KERUTA_TASK_SERVER_JWT_TOKEN` | KTCP 認証用 JWT トークン | - | ○ |
| `KERUTA_CODER_PROVIDER_PROCESSING_DELAY` | タスク監視間隔（ミリ秒） | `5000` | × |
| `CODER_LAUNCH_TIMEOUT` | coder 環境起動タイムアウト（ミリ秒） | `60000` | × |
| `CODER_URL` | coder サーバー URL | - | ○ |
| `CODER_SESSION_TOKEN` | coder セッショントークン | - | ○ |

## 動作フロー

### 起動フロー

```
┌──────────────────────────────┐
│ 1. WebSocket 接続確立         │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ 2. KTCP 認証                  │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ 3. ハートビート開始            │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ 4. タスク監視ループ開始        │
└──────────────────────────────┘
```

### 環境管理フロー

```
┌──────────────────────────────┐
│ 1. KTCP task_list 送信        │
│   (status=PENDING)           │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ 2. task_list_response 受信    │
└──────────┬───────────────────┘
           │
           ▼
      ┌────────┐
      │タスクあり│
      └────┬───┘
           │Yes
           ▼
┌──────────────────────────────┐
│ 3. coder 環境起動             │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ 4. KTCP task_status_update    │
│    送信 (status=PROCESSING)   │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ 5. 処理遅延                   │
└──────────┬───────────────────┘
           │
           ▼
  (ループ継続)
```

### エラー処理フロー

```
┌──────────────────────────────┐
│ エラー発生                    │
└──────────┬───────────────────┘
           │
           ▼
      ┌────────┐
      │リトライ可│
      └────┬───┘
           │Yes
           ▼
┌──────────────────────────────┐
│ リトライカウント確認           │
└──────────┬───────────────────┘
           │
           ▼
      ┌────────┐
      │上限未満 │
      └────┬───┘
           │Yes
           ▼
┌──────────────────────────────┐
│ 指数バックオフ待機             │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│ 操作再試行                    │
└──────────────────────────────┘
           │No
           ▼
┌──────────────────────────────┐
│ KTCP error メッセージ送信      │
└──────────────────────────────┘
```

## 監視とヘルスチェック

### ヘルスチェックエンドポイント

**エンドポイント**: `GET /actuator/health`

**レスポンス例**:
```json
{
  "status": "UP",
  "components": {
    "ktcpConnection": {
      "status": "UP"
    },
    "coderService": {
      "status": "UP"
    }
  }
}
```

### メトリクス

**エンドポイント**: `GET /actuator/metrics`

収集されるメトリクス:
- `keruta.provider.tasks.monitored` - 監視されたタスク数
- `keruta.provider.workspaces.launched` - 起動されたワークスペース数
- `keruta.provider.errors.count` - エラー発生数
- `keruta.provider.ktcp.messages.sent` - KTCP 送信メッセージ数
- `keruta.provider.ktcp.messages.received` - KTCP 受信メッセージ数

## エラーハンドリング

### エラーコード

| コード | 説明 | リトライ可能 |
|--------|------|-------------|
| `CONNECTION_LOST` | WebSocket 接続切断 | ○ |
| `AUTH_FAILED` | KTCP 認証失敗 | × |
| `CODER_LAUNCH_FAILED` | coder 環境起動失敗 | ○ |
| `CODER_TIMEOUT` | coder 起動タイムアウト | ○ |
| `INVALID_MESSAGE` | 不正な KTCP メッセージ | × |

### リトライポリシー

- **リトライ対象**: 一時的なエラー（接続エラー、タイムアウトなど）
- **リトライ方式**: 指数バックオフ
- **待機時間**: 1s, 2s, 4s, 8s, 16s
- **最大リトライ回数**: 5回
- **最大待機時間**: 32秒

## セキュリティ

### 認証

- **KTCP**: JWT トークンによる認証
- **coder**: セッショントークンによる認証

### 通信の暗号化

- **KTCP**: WSS (WebSocket Secure) 必須
- **coder**: HTTPS 必須

### 認可

- タスク取得・更新の権限検証
- coder ワークスペース操作の権限検証

## バージョン互換性

### プロトコルバージョン

- **KTCP**: KTCP/1.0 以上対応
- **coder CLI**: v2.x 対応

### 下位互換性

本仕様は以下の変更に対して下位互換性を保証:
- 設定項目の追加
- 新しいエラーコードの追加
- メトリクスの追加