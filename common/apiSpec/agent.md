# keruta-agent API仕様

> keruta-agentがkeruta APIサーバーと通信する際のAPI仕様を定義します。サブプロセス実行、入力待ち状態管理、標準入出力送信機能を含みます。

## ベースURL
```
http://keruta-api:8080
```

## 共通ヘッダー
```
Content-Type: application/json
Authorization: Bearer <JWT_TOKEN>
User-Agent: keruta-agent/1.0.0
```

## 認証
keruta-agentは、環境変数`KERUTA_API_TOKEN`から取得したJWTトークンを使用して認証を行います。

### トークンの取得
```bash
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
- **GET** `/api/tasks/{taskId}/script`
- 指定されたタスクIDのスクリプトを取得します。

### 2. タスクステータス更新
- **PUT** `/api/tasks/{taskId}/status`
- タスクのステータスを更新します。

### 3. タスク進捗更新
- **PATCH** `/tasks/{id}/progress`
- タスクの進捗率を更新します。

### 4. ログ送信
- **POST** `/api/tasks/{taskId}/logs`
- タスクの実行ログを送信します。

### 5. メトリクス送信
- **POST** `/tasks/{id}/metrics`
- タスクの実行メトリクスを送信します。

（詳細なリクエスト・レスポンス例は、元のkeruta-agent/apiSpec.mdを参照し、必要に応じて追記してください） 
