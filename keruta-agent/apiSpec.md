# keruta-agent API仕様書

> **概要**: keruta-agentがkeruta APIサーバーと通信する際のAPI仕様を定義したドキュメントです。

## 目次
- [概要](#概要)
- [認証](#認証)
- [エンドポイント一覧](#エンドポイント一覧)
- [リクエスト・レスポンス形式](#リクエストレスポンス形式)
- [エラーハンドリング](#エラーハンドリング)
- [レート制限](#レート制限)
- [サンプルコード](#サンプルコード)

## 概要
keruta-agentは、keruta APIサーバーとRESTful APIを通じて通信します。すべてのAPI呼び出しはHTTPSで行われ、JWTトークンによる認証が必要です。

### ベースURL
```
http://keruta-api:8080/api/v1
```

### 共通ヘッダー
```
Content-Type: application/json
Authorization: Bearer <JWT_TOKEN>
User-Agent: keruta-agent/1.0.0
```

## 認証
keruta-agentは、環境変数`KERUTA_API_TOKEN`から取得したJWTトークンを使用して認証を行います。

### トークンの取得
```bash
# ログインしてトークンを取得
curl -X POST http://keruta-api:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "agent",
    "password": "password"
  }'
```

### レスポンス例
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "refresh_token_here",
  "expiresIn": 3600
}
```

## エンドポイント一覧

### 1. タスクステータス更新

#### PATCH /tasks/{id}/status
タスクのステータスを更新します。

**パラメータ:**
- `id` (path): タスクID

**リクエストボディ:**
```json
{
  "status": "PROCESSING|COMPLETED|FAILED",
  "message": "ステータス更新の理由",
  "progress": 75,
  "startedAt": "2024-01-01T10:00:00Z",
  "completedAt": "2024-01-01T11:00:00Z"
}
```

**レスポンス:**
```json
{
  "id": "task-id",
  "status": "COMPLETED",
  "message": "タスクが正常に完了しました",
  "progress": 100,
  "updatedAt": "2024-01-01T11:00:00Z"
}
```

### 2. タスク進捗更新

#### PATCH /tasks/{id}/progress
タスクの進捗率を更新します。

**パラメータ:**
- `id` (path): タスクID

**リクエストボディ:**
```json
{
  "progress": 50,
  "message": "データ処理中...",
  "estimatedTimeRemaining": 300
}
```

**レスポンス:**
```json
{
  "id": "task-id",
  "progress": 50,
  "message": "データ処理中...",
  "updatedAt": "2024-01-01T10:30:00Z"
}
```

### 3. ログ送信

#### POST /tasks/{id}/logs
タスクの実行ログを送信します。

**パラメータ:**
- `id` (path): タスクID

**リクエストボディ:**
```json
{
  "level": "INFO",
  "message": "データベースクエリを実行中...",
  "timestamp": "2024-01-01T10:00:00Z",
  "metadata": {
    "source": "keruta-agent",
    "version": "1.0.0"
  }
}
```

**レスポンス:**
```json
{
  "id": "log-id",
  "taskId": "task-id",
  "level": "INFO",
  "message": "データベースクエリを実行中...",
  "timestamp": "2024-01-01T10:00:00Z"
}
```

### 4. メトリクス送信

#### POST /tasks/{id}/metrics
タスクの実行メトリクスを送信します。

**パラメータ:**
- `id` (path): タスクID

**リクエストボディ:**
```json
{
  "executionTime": 3600,
  "memoryUsage": 512,
  "cpuUsage": 25.5,
  "diskUsage": 1024,
  "networkUsage": 2048,
  "timestamp": "2024-01-01T11:00:00Z"
}
```

**レスポンス:**
```json
{
  "id": "metric-id",
  "taskId": "task-id",
  "executionTime": 3600,
  "memoryUsage": 512,
  "cpuUsage": 25.5,
  "timestamp": "2024-01-01T11:00:00Z"
}
```

### 5. エラー報告

#### POST /tasks/{id}/errors
タスク実行中のエラーを報告します。

**パラメータ:**
- `id` (path): タスクID

**リクエストボディ:**
```json
{
  "errorCode": "DB_CONNECTION_ERROR",
  "message": "データベース接続に失敗しました",
  "details": "Connection timeout after 30 seconds",
  "stackTrace": "java.sql.SQLException: Connection timeout...",
  "timestamp": "2024-01-01T10:30:00Z",
  "autoFixRequested": true
}
```

**レスポンス:**
```json
{
  "id": "error-id",
  "taskId": "task-id",
  "errorCode": "DB_CONNECTION_ERROR",
  "message": "データベース接続に失敗しました",
  "autoFixTaskId": "auto-fix-task-id",
  "timestamp": "2024-01-01T10:30:00Z"
}
```

### 6. 自動修正タスク作成

#### POST /tasks/{id}/auto-fix
エラー発生時に自動修正タスクを作成します。

**パラメータ:**
- `id` (path): 元のタスクID

**リクエストボディ:**
```json
{
  "errorCode": "DB_CONNECTION_ERROR",
  "originalError": "データベース接続に失敗しました",
  "suggestedFix": "データベース接続設定の確認と修正",
  "priority": "HIGH"
}
```

**レスポンス:**
```json
{
  "id": "auto-fix-task-id",
  "originalTaskId": "original-task-id",
  "title": "自動修正: データベース接続エラー",
  "description": "データベース接続設定の確認と修正",
  "priority": "HIGH",
  "status": "QUEUED",
  "createdAt": "2024-01-01T10:30:00Z"
}
```

### 7. ヘルスチェック

#### GET /health
keruta APIサーバーのヘルスチェックを行います。

**レスポンス:**
```json
{
  "status": "UP",
  "timestamp": "2024-01-01T10:00:00Z",
  "version": "1.0.0",
  "components": {
    "database": "UP",
    "cache": "UP"
  }
}
```

## リクエスト・レスポンス形式

### 共通レスポンス形式
すべてのAPIレスポンスは以下の形式に従います：

```json
{
  "success": true,
  "data": {
    // レスポンスデータ
  },
  "message": "操作が成功しました",
  "timestamp": "2024-01-01T10:00:00Z"
}
```

### エラーレスポンス形式
```json
{
  "success": false,
  "error": {
    "code": "TASK_NOT_FOUND",
    "message": "指定されたタスクが見つかりません",
    "details": "Task with id 'invalid-id' does not exist"
  },
  "timestamp": "2024-01-01T10:00:00Z"
}
```

## エラーハンドリング

### HTTPステータスコード
- `200 OK`: 成功
- `201 Created`: リソース作成成功
- `400 Bad Request`: リクエストが不正
- `401 Unauthorized`: 認証失敗
- `403 Forbidden`: 権限不足
- `404 Not Found`: リソースが見つからない
- `409 Conflict`: リソース競合
- `422 Unprocessable Entity`: バリデーションエラー
- `500 Internal Server Error`: サーバーエラー

### エラーコード一覧
| エラーコード | 説明 | HTTPステータス |
|-------------|------|---------------|
| `TASK_NOT_FOUND` | タスクが見つかりません | 404 |
| `TASK_ALREADY_COMPLETED` | タスクは既に完了しています | 409 |
| `INVALID_STATUS_TRANSITION` | 無効なステータス遷移です | 422 |
| `UNAUTHORIZED` | 認証が必要です | 401 |
| `FORBIDDEN` | アクセス権限がありません | 403 |
| `RATE_LIMIT_EXCEEDED` | レート制限を超えました | 429 |

## レート制限
keruta-agentのAPI呼び出しには以下のレート制限が適用されます：

- **リクエスト制限**: 100リクエスト/分
- **ログ送信**: 1000ログ/分

レート制限を超えた場合、`429 Too Many Requests`ステータスが返されます。

## サンプルコード

### Go言語での実装例
```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
    "os"
    "time"
)

type TaskStatus struct {
    Status      string    `json:"status"`
    Message     string    `json:"message"`
    Progress    int       `json:"progress"`
    StartedAt   time.Time `json:"startedAt"`
    CompletedAt time.Time `json:"completedAt,omitempty"`
}

type APIResponse struct {
    Success   bool        `json:"success"`
    Data      interface{} `json:"data"`
    Message   string      `json:"message"`
    Timestamp time.Time   `json:"timestamp"`
}

func updateTaskStatus(taskID, status, message string, progress int) error {
    apiURL := os.Getenv("KERUTA_API_URL")
    token := os.Getenv("KERUTA_API_TOKEN")
    
    statusData := TaskStatus{
        Status:    status,
        Message:   message,
        Progress:  progress,
        StartedAt: time.Now(),
    }
    
    if status == "COMPLETED" || status == "FAILED" {
        statusData.CompletedAt = time.Now()
    }
    
    jsonData, err := json.Marshal(statusData)
    if err != nil {
        return err
    }
    
    req, err := http.NewRequest("PATCH", 
        fmt.Sprintf("%s/api/v1/tasks/%s/status", apiURL, taskID),
        bytes.NewBuffer(jsonData))
    if err != nil {
        return err
    }
    
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer "+token)
    
    client := &http.Client{Timeout: 30 * time.Second}
    resp, err := client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("API call failed with status: %d", resp.StatusCode)
    }
    
    return nil
}

func main() {
    taskID := os.Getenv("KERUTA_TASK_ID")
    
    // タスク開始
    err := updateTaskStatus(taskID, "PROCESSING", "タスクを開始します", 0)
    if err != nil {
        fmt.Printf("Error starting task: %v\n", err)
        os.Exit(1)
    }
    
    // 処理実行
    // ... 実際の処理 ...
    
    // タスク完了
    err = updateTaskStatus(taskID, "COMPLETED", "タスクが完了しました", 100)
    if err != nil {
        fmt.Printf("Error completing task: %v\n", err)
        os.Exit(1)
    }
}
```

### curlコマンドでの使用例
```bash
# タスク開始
curl -X PATCH "http://keruta-api:8080/api/v1/tasks/$KERUTA_TASK_ID/status" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $KERUTA_API_TOKEN" \
  -d '{
    "status": "PROCESSING",
    "message": "タスクを開始します",
    "progress": 0
  }'

# 進捗更新
curl -X PATCH "http://keruta-api:8080/api/v1/tasks/$KERUTA_TASK_ID/progress" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $KERUTA_API_TOKEN" \
  -d '{
    "progress": 50,
    "message": "データ処理中..."
  }'

# タスク完了
curl -X PATCH "http://keruta-api:8080/api/v1/tasks/$KERUTA_TASK_ID/status" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $KERUTA_API_TOKEN" \
  -d '{
    "status": "COMPLETED",
    "message": "タスクが完了しました",
    "progress": 100
  }'
``` 