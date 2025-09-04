# ログストリーミング API仕様

> **概要**: ログストリーミング機能のREST APIとWebSocketエンドポイントの詳細仕様です。

## 概要

ログストリーミング機能は以下のAPIエンドポイントを提供します：

- **REST API**: ログの取得・管理
- **WebSocket**: リアルタイムログ配信
- **データモデル**: ログメッセージの構造定義

## REST API エンドポイント

### ベースURL
```
https://keruta-api.example.com/api/v1
```

### 1. ログ取得 API

#### タスクログ一覧取得
```http
GET /tasks/{taskId}/logs
```

**パラメータ**
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| `taskId` | string | ○ | タスクID |
| `source` | string | × | ログソース（stdout, stderr, agent） |
| `level` | string | × | ログレベル（DEBUG, INFO, WARN, ERROR） |
| `page` | integer | × | ページ番号（デフォルト: 0） |
| `size` | integer | × | ページサイズ（デフォルト: 100） |
| `from` | string | × | 開始日時（ISO 8601形式） |
| `to` | string | × | 終了日時（ISO 8601形式） |

**レスポンス**
```json
{
  "content": [
    {
      "id": "log-123",
      "taskId": "task-456",
      "timestamp": "2024-01-01T12:00:00Z",
      "source": "stdout",
      "level": "INFO",
      "message": "スクリプト実行開始",
      "lineNumber": 1,
      "metadata": {
        "processId": 12345,
        "scriptPath": "/work/script.sh"
      }
    }
  ],
  "totalElements": 150,
  "totalPages": 2,
  "size": 100,
  "number": 0
}
```

#### ログ統計取得
```http
GET /tasks/{taskId}/logs/stats
```

**レスポンス**
```json
{
  "taskId": "task-456",
  "totalLogs": 150,
  "logCounts": {
    "stdout": 100,
    "stderr": 45,
    "agent": 5
  },
  "levelCounts": {
    "DEBUG": 20,
    "INFO": 100,
    "WARN": 25,
    "ERROR": 5
  },
  "timeRange": {
    "start": "2024-01-01T12:00:00Z",
    "end": "2024-01-01T12:30:00Z"
  }
}
```

### 2. ログ管理 API

#### ログ削除
```http
DELETE /tasks/{taskId}/logs
```

**パラメータ**
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| `taskId` | string | ○ | タスクID |
| `before` | string | × | 指定日時以前のログを削除（ISO 8601形式） |

**レスポンス**
```json
{
  "deletedCount": 150,
  "message": "ログが正常に削除されました"
}
```

#### ログエクスポート
```http
GET /tasks/{taskId}/logs/export
```

**パラメータ**
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| `taskId` | string | ○ | タスクID |
| `format` | string | × | エクスポート形式（json, csv, txt） |
| `from` | string | × | 開始日時 |
| `to` | string | × | 終了日時 |

**レスポンス**
- Content-Type: `application/json`, `text/csv`, `text/plain`
- ファイルダウンロード

## WebSocket API

### 接続URL
```
ws://keruta-api.example.com/ws/tasks/{taskId}
```

### メッセージ形式

#### クライアント → サーバー

**ログ受信開始**
```json
{
  "type": "subscribe",
  "taskId": "task-456",
  "options": {
    "sources": ["stdout", "stderr"],
    "levels": ["INFO", "ERROR"],
    "realtime": true
  }
}
```

**ログ受信停止**
```json
{
  "type": "unsubscribe",
  "taskId": "task-456"
}
```

#### サーバー → クライアント

**ログメッセージ**
```json
{
  "type": "log",
  "taskId": "task-456",
  "timestamp": "2024-01-01T12:00:00Z",
  "source": "stdout",
  "level": "INFO",
  "message": "スクリプト実行開始",
  "lineNumber": 1,
  "metadata": {
    "processId": 12345,
    "scriptPath": "/work/script.sh"
  }
}
```

**接続確認**
```json
{
  "type": "ping",
  "timestamp": "2024-01-01T12:00:00Z"
}
```

**エラーメッセージ**
```json
{
  "type": "error",
  "code": "TASK_NOT_FOUND",
  "message": "指定されたタスクが見つかりません",
  "timestamp": "2024-01-01T12:00:00Z"
}
```

## データモデル

### LogMessage
```typescript
interface LogMessage {
  id?: string;
  type: "log" | "ping" | "error";
  taskId: string;
  timestamp: string; // ISO 8601形式
  source: "stdout" | "stderr" | "agent";
  level: "DEBUG" | "INFO" | "WARN" | "ERROR";
  message: string;
  lineNumber?: number;
  metadata?: Record<string, any>;
}
```

### LogRequest
```typescript
interface LogRequest {
  taskId: string;
  source: "stdout" | "stderr" | "agent";
  level: "DEBUG" | "INFO" | "WARN" | "ERROR";
  message: string;
  lineNumber?: number;
  metadata?: Record<string, any>;
}
```

### LogResponse
```typescript
interface LogResponse {
  id: string;
  taskId: string;
  timestamp: string;
  success: boolean;
}
```

### WebSocketSubscription
```typescript
interface WebSocketSubscription {
  type: "subscribe" | "unsubscribe";
  taskId: string;
  options?: {
    sources?: string[];
    levels?: string[];
    realtime?: boolean;
  };
}
```

## エラーコード

| コード | 説明 | HTTPステータス |
|--------|------|---------------|
| `TASK_NOT_FOUND` | タスクが見つかりません | 404 |
| `INVALID_LOG_FORMAT` | ログ形式が無効です | 400 |
| `WEBSOCKET_CONNECTION_FAILED` | WebSocket接続に失敗しました | 503 |
| `RATE_LIMIT_EXCEEDED` | レート制限を超過しました | 429 |
| `DATABASE_ERROR` | データベースエラー | 500 |

## 認証・認可

### API認証
- **Bearer Token**: JWTトークンによる認証
- **API Key**: サービス間通信用

### WebSocket認証
```json
{
  "type": "auth",
  "token": "jwt-token-here"
}
```

## レート制限

| エンドポイント | 制限 | 説明 |
|---------------|------|------|
| ログ取得 | 1000 req/min | 1分間に1000リクエスト |
| WebSocket接続 | 100 conn/min | 1分間に100接続 |
| ログ送信 | 10000 msg/min | 1分間に10000メッセージ |

## 関連ドキュメント
- [アーキテクチャ概要](./architecture.md)
- [実装仕様](./implementation-specification.md)
- [使用方法](./usage-guide.md)
- [設定・パフォーマンス](./configuration-performance.md) 