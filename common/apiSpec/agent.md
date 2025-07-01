# keruta-agent API仕様

keruta-agentがkeruta APIサーバーと通信する際のAPI仕様をまとめます。サブプロセス実行、入力待ち状態管理、標準入出力送信機能を含みます。

## ベースURL
```
http://keruta-api:8080/api/v1
```

## 共通ヘッダー
```
Content-Type: application/json
Authorization: Bearer <JWT_TOKEN>
User-Agent: keruta-agent/1.0.0
```

## 認証
keruta-agentは、環境変数`KERUTA_API_TOKEN`から取得したJWTトークンを使用して認証を行います。

### トークンの取得例
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

1. スクリプト取得: `GET /api/tasks/{taskId}/script`
2. タスクステータス更新: `PUT /api/tasks/{taskId}/status`
3. タスク進捗更新: `PATCH /tasks/{id}/progress`
4. ログ送信: `POST /api/tasks/{taskId}/logs`
5. メトリクス送信: `POST /tasks/{id}/metrics`

（詳細なリクエスト・レスポンス例は、必要に応じて追記してください） 