# keruta-agent API仕様書

> **概要**: keruta-agentがkeruta APIサーバーと通信する際のAPI仕様を定義したドキュメントです。サブプロセス実行、入力待ち状態管理、標準入力送信機能を含みます。

## 目次
- [概要](#概要)
- [認証](#認証)
- [エンドポイント一覧](#エンドポイント一覧)
- [リクエスト・レスポンス形式](#リクエストレスポンス形式)
- [エラーハンドリング](#エラーハンドリング)
- [レート制限](#レート制限)
- [サンプルコード](#サンプルコード)
- [データモデル](#データモデル)
- [データモデル](#データモデル)

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

### 1. スクリプト取得

#### GET /api/tasks/{taskId}/script

指定されたタスクIDのスクリプトを取得します。

**パラメータ:**
- `taskId` (path): タスクID

**レスポンス:**
```json
{
  "success": true,
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "script": {
    "content": "#!/bin/bash\necho \"Hello World\"\nread -r name\necho \"Hello, $name!\"",
    "language": "bash",
    "filename": "script.sh",
    "parameters": {
      "timeout": 300,
      "workDir": "/work"
    }
  }
}
```

**使用例:**
```bash
curl -X GET "http://keruta-api:8080/api/tasks/task123/script" \
  -H "Authorization: Bearer $KERUTA_API_TOKEN"
```

### 2. タスクステータス更新

#### PUT /api/tasks/{taskId}/status
タスクのステータスを更新します。

**パラメータ:**
- `taskId` (path): タスクID

**リクエストボディ:**
```json
{
  "status": "PROCESSING|COMPLETED|FAILED|WAITING_FOR_INPUT",
  "message": "ステータス更新メッセージ",
  "progress": 75,
  "errorCode": "ERROR_CODE",
  "autoFix": true
}
```

**レスポンス:**
```json
{
  "success": true,
  "taskId": "123e4567-e89b-12d3-a456-426614174000",
  "status": "PROCESSING",
  "updatedAt": "2024-01-01T12:00:00Z"
}
```

### 3. タスク進捗更新

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

### 4. ログ送信

#### POST /api/tasks/{taskId}/logs
タスクの実行ログを送信します。

**パラメータ:**
- `taskId` (path): タスクID

**リクエストボディ:**
```json
{
  "level": "DEBUG|INFO|WARN|ERROR",
  "message": "ログメッセージ",
  "timestamp": "2024-01-01T12:00:00Z",
  "source": "stdout|stderr|agent",
  "metadata": {
    "lineNumber": 42,
    "function": "processData"
  }
}
```

**レスポンス:**
```json
{
  "success": true,
  "logId": "log-123",
  "timestamp": "2024-01-01T12:00:00Z"
}
```

### 5. メトリクス送信

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

### 6. エラー報告

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

### 7. 自動修正タスク作成

#### POST /api/tasks/{taskId}/auto-fix
エラー発生時に自動修正タスクを作成します。

**パラメータ:**
- `taskId` (path): タスクID

**リクエストボディ:**
```json
{
  "errorCode": "DB_CONNECTION_ERROR",
  "errorMessage": "データベース接続に失敗しました",
  "suggestedFix": "データベース設定を確認してください",
  "priority": "HIGH|MEDIUM|LOW"
}
```

**レスポンス:**
```json
{
  "success": true,
  "autoFixTaskId": "auto-fix-123",
  "createdAt": "2024-01-01T12:00:00Z"
}
```

### 8. 入力待ち状態管理

#### POST /api/tasks/{taskId}/input-waiting
サブプロセスが入力待ち状態になったことを通知します。

**パラメータ:**
- `taskId` (path): タスクID

**リクエストボディ:**
```json
{
  "detectedAt": "2024-01-01T12:00:00Z",
  "processId": 12345,
  "scriptPath": "/work/script.sh",
  "lastOutput": "名前を入力してください:"
}
```

**レスポンス:**
```json
{
  "success": true,
  "waitingId": "wait-123",
  "status": "WAITING_FOR_INPUT"
}
```

#### POST /tasks/{id}/resume
入力待ち状態からタスクを再開します。

**パラメータ:**
- `id` (path): タスクID

**リクエストボディ:**
```json
{
  "input": "ユーザーが入力したテキスト"
}
```

**レスポンス:**
```json
{
  "id": "task-id",
  "status": "PROCESSING",
  "updatedAt": "2024-01-01T10:05:00Z"
}
```

### 9. ヘルスチェック

#### GET /api/health
keruta APIサーバーのヘルスチェックを行います。

**レスポンス:**
```json
{
  "status": "UP",
  "timestamp": "2024-01-01T12:00:00Z",
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
    "code": "ERROR_CODE",
    "message": "エラーメッセージ",
    "details": "詳細情報",
    "timestamp": "2024-01-01T12:00:00Z"
  }
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

### 基本的なタスク実行フロー

```bash
#!/bin/bash

TASK_ID="task123"
API_URL="http://keruta-api:8080"
API_TOKEN="your-api-token"

# 1. スクリプト取得
echo "スクリプトを取得中..."
SCRIPT_RESPONSE=$(curl -s -X GET "$API_URL/api/tasks/$TASK_ID/script" \
  -H "Authorization: Bearer $API_TOKEN")

# レスポンスからスクリプト内容を抽出
SCRIPT_CONTENT=$(echo "$SCRIPT_RESPONSE" | jq -r '.script.content')
SCRIPT_FILENAME=$(echo "$SCRIPT_RESPONSE" | jq -r '.script.filename')

# スクリプトファイルを作成
echo "$SCRIPT_CONTENT" > "/work/$SCRIPT_FILENAME"
chmod +x "/work/$SCRIPT_FILENAME"

# 2. タスク開始
echo "タスクを開始中..."
curl -X PUT "$API_URL/api/tasks/$TASK_ID/status" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "PROCESSING",
    "message": "タスク実行開始"
  }'

# 3. スクリプト実行
echo "スクリプトを実行中..."
"/work/$SCRIPT_FILENAME"

# 4. タスク完了
EXIT_CODE=$?
if [ $EXIT_CODE -eq 0 ]; then
  echo "タスク完了"
  curl -X PUT "$API_URL/api/tasks/$TASK_ID/status" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
      "status": "COMPLETED",
      "message": "タスク正常完了"
    }'
else
  echo "タスク失敗"
  curl -X PUT "$API_URL/api/tasks/$TASK_ID/status" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
      "status": "FAILED",
      "message": "タスク実行中にエラーが発生しました",
      "errorCode": "SCRIPT_EXECUTION_ERROR"
    }'
fi
```

### keruta-agentを使用した実行

```bash
# keruta-agentを使用してタスクを実行（スクリプトは自動的にAPIから取得）
keruta-agent execute --task-id task123
```

### 入力待ち状態の処理

```bash
#!/bin/bash

TASK_ID="task123"
API_URL="http://keruta-api:8080"
API_TOKEN="your-api-token"

# スクリプト取得
SCRIPT_RESPONSE=$(curl -s -X GET "$API_URL/api/tasks/$TASK_ID/script" \
  -H "Authorization: Bearer $API_TOKEN")

SCRIPT_CONTENT=$(echo "$SCRIPT_RESPONSE" | jq -r '.script.content')
echo "$SCRIPT_CONTENT" > "/work/script.sh"
chmod +x "/work/script.sh"

# タスク開始
curl -X PUT "$API_URL/api/tasks/$TASK_ID/status" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "PROCESSING", "message": "タスク実行開始"}'

# スクリプト実行（入力待ち状態を監視）
while true; do
  # スクリプトの実行状態をチェック
  WAITING_RESPONSE=$(curl -s -X GET "$API_URL/api/tasks/$TASK_ID/input-waiting" \
    -H "Authorization: Bearer $API_TOKEN")
  
  IS_WAITING=$(echo "$WAITING_RESPONSE" | jq -r '.waiting.isWaiting')
  
  if [ "$IS_WAITING" = "true" ]; then
    echo "入力待ち状態を検出"
    
    # タスク状態を入力待ちに更新
    curl -X PUT "$API_URL/api/tasks/$TASK_ID/status" \
      -H "Authorization: Bearer $API_TOKEN" \
      -H "Content-Type: application/json" \
      -d '{"status": "WAITING_FOR_INPUT", "message": "ユーザー入力を待機中"}'
    
    # 管理パネルからの入力を待機
    sleep 5
  else
    # スクリプトが完了したかチェック
    if ! pgrep -f "script.sh" > /dev/null; then
      break
    fi
    sleep 1
  fi
done

# タスク完了
curl -X PUT "$API_URL/api/tasks/$TASK_ID/status" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status": "COMPLETED", "message": "タスク正常完了"}'
```

## データモデル

### Script
```json
{
  "content": "string",
  "language": "bash|python|node|go",
  "filename": "string",
  "parameters": {
    "timeout": "number",
    "workDir": "string",
    "env": "object"
  }
}
```

### TaskStatus
```json
{
  "status": "PROCESSING|COMPLETED|FAILED|WAITING_FOR_INPUT",
  "message": "string",
  "progress": "number",
  "errorCode": "string",
  "autoFix": "boolean",
  "startedAt": "ISO 8601 date",
  "completedAt": "ISO 8601 date"
}
``` 