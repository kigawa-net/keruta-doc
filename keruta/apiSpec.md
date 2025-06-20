# API仕様

## タスク管理API

### タスク作成
```http
POST /api/tasks
Content-Type: application/json

{
  "title": "ログイン機能の実装",
  "description": "基本的な認証機能を実装してください",
  "priority": "HIGH",
  "language": "kotlin"
}
```

### タスク一覧取得
```http
GET /api/tasks?status=QUEUED&limit=10

Response:
{
  "tasks": [
    {
      "id": "task-123",
      "title": "ログイン機能の実装",
      "status": "PROCESSING",
      "priority": "HIGH",
      "createdAt": "2025-06-18T10:00:00Z"
    }
  ]
}
```

### タスク詳細取得
```http
GET /api/tasks/{taskId}

Response:
{
  "id": "task-123",
  "title": "ログイン機能の実装",
  "description": "基本的な認証機能を実装してください",
  "status": "COMPLETED",
  "result": {
    "code": "生成されたコード",
    "files": ["LoginController.kt", "User.kt"]
  }
}
``` 