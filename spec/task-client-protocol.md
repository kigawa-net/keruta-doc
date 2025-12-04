# keruta task client protocol (KTCP) 仕様

## 概要

keruta task client protocol (KTCP) は、タスクサーバーとタスク実行プロバイダー（provider）間の通信を制御するためのプロトコルです。タスクの実行管理、状態同期、ログストリーミングを実現します。

## アーキテクチャ

```
┌─────────────────┐          KTCP          ┌─────────────────┐
│   Task Server   │◄─────────────────────►│    Provider     │
│  (keruta-api)   │    WebSocket only     │ (keruta-agent)  │
└─────────────────┘                       └─────────────────┘
```

## 基底プロトコル

KTCP は WebSocket を唯一の通信プロトコルとして使用します。

- **到達保証**: TCP/IP による保証
- **認証**: JWT トークンベース
- **メッセージ形式**: JSON-RPC 2.0 ベース
- **接続**: 双方向全二重通信

## 通信プロトコル詳細

### WebSocket 接続

#### 接続確立
```
ws://keruta-api:8080/ws/ktcp
wss://keruta-api:8080/ws/ktcp (推奨)
```

#### 認証メッセージ
```json
{
  "type": "authenticate",
  "token": "<jwt-token>",
  "clientType": "provider",
  "clientVersion": "1.0.0",
  "capabilities": ["kubernetes", "docker", "local"]
}
```

#### ハートビート
```json
{
  "type": "heartbeat",
  "timestamp": "2024-01-01T10:00:00Z",
  "status": "healthy"
}
```

### メッセージタイプ

#### 1. タスク実行要求
```json
{
  "type": "task_execute",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "name": "Build Project",
    "script": "#!/bin/bash\necho 'Hello World'",
    "parameters": {"env": "production"},
    "timeout": 3600,
    "workingDirectory": "/work",
    "environment": {"NODE_VERSION": "18"}
  },
  "timestamp": "2024-01-01T10:00:00Z"
}
```

#### 2. タスク状態更新
```json
{
  "type": "task_status_update",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "status": "PROCESSING",
    "progress": 50,
    "message": "ビルド進行中",
    "startedAt": "2024-01-01T10:05:00Z"
  },
  "timestamp": "2024-01-01T10:05:00Z"
}
```

#### 3. ログストリーミング
```json
{
  "type": "task_log",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "level": "INFO",
    "message": "ビルドを開始します",
    "timestamp": "2024-01-01T10:05:30Z",
    "metadata": {
      "phase": "build",
      "step": "npm install"
    }
  },
  "timestamp": "2024-01-01T10:05:30Z"
}
```

#### 4. タスク完了通知
```json
{
  "type": "task_completed",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "status": "COMPLETED",
    "exitCode": 0,
    "completedAt": "2024-01-01T10:10:00Z",
    "resourceUsage": {
      "cpuTime": 45.2,
      "memoryPeak": "256MB",
      "executionTime": 300
    }
  },
  "timestamp": "2024-01-01T10:10:00Z"
}
```

#### 5. エラー通知
```json
{
  "type": "task_error",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "status": "FAILED",
    "errorCode": "BUILD_FAILED",
    "errorMessage": "npm install に失敗しました",
    "exitCode": 1,
    "retryable": true,
    "failedAt": "2024-01-01T10:08:00Z"
  },
  "timestamp": "2024-01-01T10:08:00Z"
}
```

#### 6. タスクキャンセル要求
```json
{
  "type": "task_cancel",
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "data": {
    "reason": "ユーザーによるキャンセル",
    "force": false
  },
  "timestamp": "2024-01-01T10:07:00Z"
}
```

## タスク実行ライフサイクル

### 1. 接続確立フェーズ
1. **WebSocket 接続**: プロバイダーがサーバーに接続
2. **認証**: JWT トークンによる認証
3. **登録**: プロバイダー能力の通知
4. **ハートビート開始**: 定期的な生存確認

### 2. タスク実行フェーズ
1. **タスク受信**: サーバーから実行要求を受信
2. **環境準備**: 作業ディレクトリ、依存関係の準備
3. **実行開始**: スクリプト実行の開始
4. **状態同期**: リアルタイムでの状態更新
5. **ログストリーミング**: 実行ログの送信

### 3. 完了フェーズ
1. **実行完了**: スクリプト終了
2. **結果通知**: 完了/失敗状態の送信
3. **リソース解放**: クリーンアップ
4. **次のタスク待機**: アイドル状態へ

## エラーハンドリング

### エラーメッセージ形式
```json
{
  "type": "error",
  "code": "CONNECTION_LOST",
  "message": "WebSocket 接続が切断されました",
  "timestamp": "2024-01-01T10:00:00Z",
  "retryable": true
}
```

### エラーコード
- **CONNECTION_LOST**: 接続切断
- **AUTH_FAILED**: 認証失敗
- **INVALID_MESSAGE**: 不正メッセージ
- **TASK_NOT_FOUND**: タスク不存在
- **EXECUTION_FAILED**: 実行失敗
- **RESOURCE_EXHAUSTED**: リソース不足

### リトライポリシー
- **指数バックオフ**: 1s, 2s, 4s, 8s...
- **最大リトライ数**: 5回
- **対象エラー**: 接続エラー、一時的な失敗

## セキュリティ

### 通信の暗号化
- **必須**: WSS (WebSocket Secure) の使用
- **証明書**: 有効な SSL/TLS 証明書

### 認証と認可
- **JWT トークン**: Keycloak 発行のトークン
- **スコープ検証**: プロバイダー権限の確認
- **トークン更新**: 有効期限内の自動更新

### メッセージ検証
- **スキーマ検証**: JSON スキーマによる構造検証
- **入力サニタイズ**: スクリプト、コマンドの安全チェック
- **サイズ制限**: メッセージサイズの上限設定

## パフォーマンス最適化

### 接続管理
- **接続プーリング**: 複数プロバイダーの同時接続
- **再接続ロジック**: 自動再接続とエクスポネンシャルバックオフ
- **接続タイムアウト**: アイドル接続の自動切断

### メッセージ最適化
- **バイナリメッセージ**: 大きなログデータの効率的転送
- **圧縮**: メッセージペイロードの圧縮
- **バッチ処理**: 複数ログの一括送信

### リソース管理
- **同時実行制限**: プロバイダーごとの最大タスク数
- **キューイング**: 過負荷時のタスクキューイング
- **負荷分散**: 複数プロバイダー間でのタスク分散

## バージョン互換性

### プロトコルバージョン
- **KTCP/1.0**: 基本 WebSocket 通信
- **KTCP/1.1**: バイナリメッセージ、圧縮サポート
- **KTCP/1.2**: 高度なエラーハンドリング、バッチ処理

### バージョン識別
```json
{
  "type": "handshake",
  "protocolVersion": "KTCP/1.2",
  "clientVersion": "keruta-agent/1.2.0"
}
```

## 監視とメトリクス

### 接続メトリクス
- アクティブ接続数
- 接続継続時間
- 再接続回数

### メッセージメトリクス
- メッセージスループット
- エラー率
- レスポンスタイム

### タスクメトリクス
- 実行中のタスク数
- 完了/失敗タスク数
- 平均実行時間

この KTCP 仕様により、タスクサーバーとプロバイダー間の信頼性が高く効率的な通信を実現します。

