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

## リポジトリ管理API

### リポジトリ情報更新
リポジトリ名やインストールスクリプトなどを更新します。

```http
PATCH /api/repositories/{repoId}
Content-Type: application/json

{
  "name": "keruta-app",
  "installScript": "#!/bin/bash\\n\\npip install -r requirements.txt"
}
```

### インストールスクリプト実行
登録されたインストールスクリプトを実行します。

```http
POST /api/repositories/{repoId}/install

Response:
{
  "status": "QUEUED",
  "executionId": "exec-abc-123"
}
``` 