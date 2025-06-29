# タスク管理API

> kerutaシステムのタスク管理に関するAPIエンドポイントとリクエスト/レスポンス例

## 目次
- [タスク一覧取得](#タスク一覧取得)
- [タスク作成](#タスク作成)
- [タスク詳細取得](#タスク詳細取得)
- [タスク削除](#タスク削除)
- [タスクスクリプト取得](#タスクスクリプト取得)
- [タスクスクリプト更新](#タスクスクリプト更新)

---

## タスク一覧取得

### エンドポイント
```http
GET /api/v1/tasks
```

### クエリパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| status | string | 任意 | タスクのステータスでフィルタリング |
| priority | integer | 任意 | 優先度でフィルタリング |
| agentId | string | 任意 | エージェントIDでフィルタリング |
| repositoryId | string | 任意 | リポジトリIDでフィルタリング |

### レスポンス例
```json
[
  {
    "id": "task-123",
    "title": "ログイン機能の実装",
    "description": "基本的な認証機能を実装してください",
    "priority": 1,
    "status": "PENDING",
    "documents": [],
    "image": null,
    "namespace": "default",
    "jobName": null,
    "podName": null,
    "additionalEnv": {},
    "kubernetesManifest": null,
    "logs": null,
    "agentId": null,
    "repositoryId": null,
    "parentId": null,
    "createdAt": "2024-06-18T10:00:00Z",
    "updatedAt": "2024-06-18T10:00:00Z"
  }
]
```

---

## タスク作成

### エンドポイント
```http
POST /api/v1/tasks
Content-Type: application/json
```

### リクエストボディ
```json
{
  "title": "ログイン機能の実装",
  "description": "基本的な認証機能を実装してください",
  "priority": 1,
  "repositoryId": "repo-001",
  "agentId": "agent-001",
  "additionalEnv": {
    "DEBUG": "true"
  }
}
```

### レスポンス例
```json
{
  "id": "task-123",
  "title": "ログイン機能の実装",
  "description": "基本的な認証機能を実装してください",
  "priority": 1,
  "status": "PENDING",
  "documents": [],
  "image": null,
  "namespace": "default",
  "jobName": null,
  "podName": null,
  "additionalEnv": {
    "DEBUG": "true"
  },
  "kubernetesManifest": null,
  "logs": null,
  "agentId": "agent-001",
  "repositoryId": "repo-001",
  "parentId": null,
  "createdAt": "2024-06-18T10:00:00Z",
  "updatedAt": "2024-06-18T10:00:00Z"
}
```

---

## タスク詳細取得

### エンドポイント
```http
GET /api/v1/tasks/{id}
```

### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | タスクID |

### レスポンス例
```json
{
  "id": "task-123",
  "title": "ログイン機能の実装",
  "description": "基本的な認証機能を実装してください",
  "priority": 1,
  "status": "PENDING",
  "documents": [],
  "image": null,
  "namespace": "default",
  "jobName": null,
  "podName": null,
  "additionalEnv": {},
  "kubernetesManifest": null,
  "logs": null,
  "agentId": null,
  "repositoryId": null,
  "parentId": null,
  "createdAt": "2024-06-18T10:00:00Z",
  "updatedAt": "2024-06-18T10:00:00Z"
}
```

---

## タスク削除

### エンドポイント
```http
DELETE /api/v1/tasks/{id}
```

### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | タスクID |

### レスポンス
```http
204 No Content
```

---

## タスクスクリプト取得

### エンドポイント
```http
GET /api/v1/tasks/{id}/script
```

### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | タスクID |

### レスポンス例
```json
{
  "taskId": "task-123",
  "script": {
    "installScript": "#!/bin/bash\necho 'Installing dependencies...'\n# 依存関係のインストール処理",
    "executeScript": "#!/bin/bash\necho 'Executing task...'\n# タスク実行処理",
    "cleanupScript": "#!/bin/bash\necho 'Cleaning up...'\n# クリーンアップ処理"
  },
  "environment": {
    "REPOSITORY_URL": "https://github.com/user/project.git",
    "TASK_ID": "task-123",
    "WORKSPACE_PATH": "/workspace"
  },
  "createdAt": "2024-06-18T10:00:00Z",
  "updatedAt": "2024-06-18T10:00:00Z"
}
```

---

## タスクスクリプト更新

### エンドポイント
```http
PUT /api/v1/tasks/{id}/script
Content-Type: application/json
```

### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | タスクID |

### リクエストボディ
```json
{
  "installScript": "#!/bin/bash\necho 'Updated install script...'",
  "executeScript": "#!/bin/bash\necho 'Updated execute script...'",
  "cleanupScript": "#!/bin/bash\necho 'Updated cleanup script...'"
}
```

### レスポンス例
```json
{
  "taskId": "task-123",
  "script": {
    "installScript": "#!/bin/bash\necho 'Updated install script...'",
    "executeScript": "#!/bin/bash\necho 'Updated execute script...'",
    "cleanupScript": "#!/bin/bash\necho 'Updated cleanup script...'"
  },
  "environment": {
    "REPOSITORY_URL": "https://github.com/user/project.git",
    "TASK_ID": "task-123",
    "WORKSPACE_PATH": "/workspace"
  },
  "updatedAt": "2024-06-18T10:00:00Z"
}
```

---

## エラーレスポンス

### 400 Bad Request
```json
{
  "error": "Invalid request parameters",
  "message": "Title is required"
}
```

### 404 Not Found
```json
{
  "error": "Task not found",
  "message": "Task with id 'task-123' does not exist"
}
```

### 500 Internal Server Error
```json
{
  "error": "Internal server error",
  "message": "An unexpected error occurred"
}
``` 