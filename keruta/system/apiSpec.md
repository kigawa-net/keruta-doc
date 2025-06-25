# API仕様

> kerutaシステムの主要APIエンドポイントとリクエスト/レスポンス例

## 目次
- [タスク管理API](#タスク管理api)
- [エージェント管理API](#エージェント管理api)
- [リポジトリ管理API](#リポジトリ管理api)
- [ドキュメント管理API](#ドキュメント管理api)
- [関連ドキュメント](#関連ドキュメント)

---

## タスク管理API

### タスク一覧取得
```http
GET /api/v1/tasks

Response:
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

### タスク作成
```http
POST /api/v1/tasks
Content-Type: application/json

{
  "title": "ログイン機能の実装",
  "description": "基本的な認証機能を実装してください",
  "priority": 1
}

Response: 201 Created
{
  "id": "task-123",
  ...
}
```

### タスク詳細取得
```http
GET /api/v1/tasks/{id}

Response:
{
  "id": "task-123",
  "title": "ログイン機能の実装",
  ...
}
```

### タスク削除
```http
DELETE /api/v1/tasks/{id}

Response: 204 No Content
```

---

## エージェント管理API

### エージェント一覧取得
```http
GET /api/v1/agents

Response:
[
  {
    "id": "agent-001",
    "name": "KotlinAgent",
    "languages": ["kotlin"],
    "status": "AVAILABLE",
    "currentTaskId": null,
    "installCommand": "",
    "executeCommand": "",
    "createdAt": "2024-06-18T10:00:00Z",
    "updatedAt": "2024-06-18T10:00:00Z"
  }
]
```

### エージェント作成
```http
POST /api/v1/agents
Content-Type: application/json

{
  "name": "KotlinAgent",
  "languages": ["kotlin"]
}

Response: 201 Created
{
  "id": "agent-001",
  ...
}
```

---

## リポジトリ管理API

### リポジトリ一覧取得
```http
GET /api/v1/repositories

Response:
[
  {
    "id": "repo-001",
    "name": "keruta-app",
    "url": "https://github.com/user/keruta.git",
    "description": "keruta本体リポジトリ",
    "isValid": true,
    "setupScript": "",
    "usePvc": false,
    "pvcStorageSize": "1Gi",
    "pvcAccessMode": "ReadWriteOnce",
    "createdAt": "2024-06-18T10:00:00Z",
    "updatedAt": "2024-06-18T10:00:00Z"
  }
]
```

### リポジトリ作成
```http
POST /api/v1/repositories
Content-Type: application/json

{
  "name": "keruta-app",
  "url": "https://github.com/user/keruta.git"
}

Response: 201 Created
{
  "id": "repo-001",
  ...
}
```

---

## ドキュメント管理API

### ドキュメント一覧取得
```http
GET /api/v1/documents

Response:
[
  {
    "id": "doc-001",
    "title": "システム概要",
    "content": "...",
    "tags": ["概要"],
    "createdAt": "2024-06-18T10:00:00Z",
    "updatedAt": "2024-06-18T10:00:00Z"
  }
]
```

### ドキュメント作成
```http
POST /api/v1/documents
Content-Type: application/json

{
  "title": "システム概要",
  "content": "...",
  "tags": ["概要"]
}

Response: 201 Created
{
  "id": "doc-001",
  ...
}
```

---

## 関連ドキュメント
- [システム詳細・セットアップ](./projectDetails.md)
- [データモデル定義](./dataModel.md) 