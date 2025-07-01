# 共通API仕様

このドキュメントでは、kerutaプロジェクトにおける各サブシステムで共通して利用されるAPI仕様および、kerutaシステム全体・keruta-agent個別のAPI仕様についてまとめます。

## 目次
- API設計方針
- 認証・認可
- エラーハンドリング
- 共通レスポンス形式
- バージョニング
- その他共通事項
- kerutaシステムAPIエンドポイント
- keruta-agent API仕様
- 関連ドキュメント
- keruta API詳細

---

## API設計方針
- RESTful設計を基本とする。
- リソース指向のエンドポイント命名。

## 認証・認可
- JWTベースの認証を推奨。
- 必要に応じてOAuth2やAPIキーも利用可能。
- すべてのAPIリクエストには認証が必要です。

```http
Authorization: Bearer <access_token>
```

## エラーハンドリング
- エラー時はHTTPステータスコードとともに、以下のJSON形式で返却する。

```json
{
  "error": {
    "code": "エラーコード",
    "message": "エラーメッセージ"
  }
}
```

## 共通レスポンス形式
- 正常時もJSON形式を基本とする。
- ページネーションやメタ情報は`meta`フィールドで返却。

```json
{
  "data": {
    // レスポンスデータ
  },
  "meta": {
    "timestamp": "2024-06-18T10:00:00Z",
    "version": "1.0.0"
  }
}
```

- エラー時:
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "エラーメッセージ",
    "details": {}
  },
  "meta": {
    "timestamp": "2024-06-18T10:00:00Z",
    "version": "1.0.0"
  }
}
```

## バージョニング
- URLパスにバージョン番号を含める（例: `/api/v1/resource`）。

## その他共通事項
- タイムゾーンはJST（日本標準時）を基準とする。
- 日時はISO 8601形式（例: `2024-06-01T12:00:00+09:00`）で返却。

---

## kerutaシステムAPIエンドポイント

### 目次
- [タスク管理API](keruta/system/api/taskApi.md)
- [エージェント管理API](keruta/system/api/agentApi.md)
- [リポジトリ管理API](keruta/system/api/repositoryApi.md)
- [ドキュメント管理API](keruta/system/api/documentApi.md)
- [関連ドキュメント](#関連ドキュメント)

### 基本URL
```
https://api.keruta.example.com
```

### HTTPステータスコード
| コード | 説明 |
|--------|------|
| 200 | OK - リクエスト成功 |
| 201 | Created - リソース作成成功 |
| 204 | No Content - 削除成功 |
| 400 | Bad Request - リクエスト不正 |
| 401 | Unauthorized - 認証失敗 |
| 403 | Forbidden - 権限不足 |
| 404 | Not Found - リソース不在 |
| 409 | Conflict - 競合状態 |
| 422 | Unprocessable Entity - 処理不可 |
| 500 | Internal Server Error - サーバーエラー |

### ページネーション
リスト取得APIでは、ページネーションがサポートされています。

#### クエリパラメータ
| パラメータ | 型 | デフォルト | 説明 |
|-----------|----|-----------|------|
| page | integer | 1 | ページ番号 |
| per_page | integer | 20 | 1ページあたりの件数 |
| sort | string | created_at | ソート項目 |
| order | string | desc | ソート順序（asc/desc） |

#### レスポンス例
```json
{
  "data": [
    // データ配列
  ],
  "meta": {
    "pagination": {
      "page": 1,
      "per_page": 20,
      "total": 100,
      "total_pages": 5
    },
    "timestamp": "2024-06-18T10:00:00Z",
    "version": "1.0.0"
  }
}
```

---

## keruta-agent API仕様

> **概要**: keruta-agentがkeruta APIサーバーと通信する際のAPI仕様を定義します。サブプロセス実行、入力待ち状態管理、標準入出力送信機能を含みます。

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

### 認証
keruta-agentは、環境変数`KERUTA_API_TOKEN`から取得したJWTトークンを使用して認証を行います。

#### トークンの取得
```bash
curl -X POST http://keruta-api:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "agent",
    "password": "password"
  }'
```

#### レスポンス例
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "refresh_token_here",
  "expiresIn": 3600
}
```

### エンドポイント一覧

#### 1. スクリプト取得
- **GET** `/api/tasks/{taskId}/script`
- 指定されたタスクIDのスクリプトを取得します。

#### 2. タスクステータス更新
- **PUT** `/api/tasks/{taskId}/status`
- タスクのステータスを更新します。

#### 3. タスク進捗更新
- **PATCH** `/tasks/{id}/progress`
- タスクの進捗率を更新します。

#### 4. ログ送信
- **POST** `/api/tasks/{taskId}/logs`
- タスクの実行ログを送信します。

#### 5. メトリクス送信
- **POST** `/tasks/{id}/metrics`
- タスクの実行メトリクスを送信します。

（詳細なリクエスト・レスポンス例は、元のkeruta-agent/apiSpec.mdを参照し、必要に応じて追記してください）

---

## 関連ドキュメント
- [システム詳細・セットアップ](keruta/system/projectDetails.md)
- [データモデル定義](keruta/system/dataModel.md)
- [タスク管理API詳細](keruta/system/api/taskApi.md)
- [エージェント管理API詳細](keruta/system/api/agentApi.md)
- [リポジトリ管理API詳細](keruta/system/api/repositoryApi.md)
- [ドキュメント管理API詳細](keruta/system/api/documentApi.md)

---

## keruta API詳細

### タスク管理API

> kerutaシステムのタスク管理に関するAPIエンドポイントとリクエスト/レスポンス例

#### 目次
- タスク一覧取得
- タスク作成
- タスク詳細取得
- タスク削除
- タスクスクリプト取得
- タスクスクリプト更新

---

#### タスク一覧取得

##### エンドポイント
```http
GET /api/v1/tasks
```

##### クエリパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| status | string | 任意 | タスクのステータスでフィルタリング |
| priority | integer | 任意 | 優先度でフィルタリング |
| agentId | string | 任意 | エージェントIDでフィルタリング |
| repositoryId | string | 任意 | リポジトリIDでフィルタリング |

##### レスポンス例
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

#### タスク作成

##### エンドポイント
```http
POST /api/v1/tasks
Content-Type: application/json
```

##### リクエストボディ
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

##### レスポンス例
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

#### タスク詳細取得

##### エンドポイント
```http
GET /api/v1/tasks/{id}
```

##### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | タスクID |

##### レスポンス例
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

#### タスク削除

##### エンドポイント
```http
DELETE /api/v1/tasks/{id}
```

##### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | タスクID |

##### レスポンス
```http
204 No Content
```

---

#### タスクスクリプト取得

##### エンドポイント
```http
GET /api/v1/tasks/{id}/script
```

##### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | タスクID |

##### レスポンス例
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

#### タスクスクリプト更新

##### エンドポイント
```http
PUT /api/v1/tasks/{id}/script
Content-Type: application/json
```

##### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | タスクID |

##### リクエストボディ
```json
{
  "installScript": "#!/bin/bash\necho 'Updated install script...'",
  "executeScript": "#!/bin/bash\necho 'Updated execute script...'",
  "cleanupScript": "#!/bin/bash\necho 'Updated cleanup script...'"
}
```

##### レスポンス例
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

#### エラーレスポンス

##### 400 Bad Request
```json
{
  "error": "Invalid request parameters",
  "message": "Title is required"
}
```

##### 404 Not Found
```json
{
  "error": "Task not found",
  "message": "Task with id 'task-123' does not exist"
}
```

##### 500 Internal Server Error
```json
{
  "error": "Internal server error",
  "message": "An unexpected error occurred"
}
```

---

### エージェント管理API

> kerutaシステムのエージェント管理に関するAPIエンドポイントとリクエスト/レスポンス例

#### 目次
- エージェント一覧取得
- エージェント作成
- エージェント詳細取得
- エージェント更新
- エージェント削除

---

#### エージェント一覧取得

##### エンドポイント
```http
GET /api/v1/agents
```

##### クエリパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| status | string | 任意 | エージェントのステータスでフィルタリング |
| language | string | 任意 | 対応言語でフィルタリング |

##### レスポンス例
```json
[
  {
    "id": "agent-001",
    "name": "KotlinAgent",
    "languages": ["kotlin"],
    "status": "AVAILABLE",
    "currentTaskId": null,
    "installCommand": "apt-get update && apt-get install -y kotlin",
    "executeCommand": "kotlinc -script main.kts",
    "createdAt": "2024-06-18T10:00:00Z",
    "updatedAt": "2024-06-18T10:00:00Z"
  }
]
```

---

#### エージェント作成

##### エンドポイント
```http
POST /api/v1/agents
Content-Type: application/json
```

##### リクエストボディ
```json
{
  "name": "KotlinAgent",
  "languages": ["kotlin"],
  "installCommand": "apt-get update && apt-get install -y kotlin",
  "executeCommand": "kotlinc -script main.kts"
}
```

##### レスポンス例
```json
{
  "id": "agent-001",
  "name": "KotlinAgent",
  "languages": ["kotlin"],
  "status": "AVAILABLE",
  "currentTaskId": null,
  "installCommand": "apt-get update && apt-get install -y kotlin",
  "executeCommand": "kotlinc -script main.kts",
  "createdAt": "2024-06-18T10:00:00Z",
  "updatedAt": "2024-06-18T10:00:00Z"
}
```

---

#### エージェント詳細取得

##### エンドポイント
```http
GET /api/v1/agents/{id}
```

##### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | エージェントID |

##### レスポンス例
```json
{
  "id": "agent-001",
  "name": "KotlinAgent",
  "languages": ["kotlin"],
  "status": "AVAILABLE",
  "currentTaskId": null,
  "installCommand": "apt-get update && apt-get install -y kotlin",
  "executeCommand": "kotlinc -script main.kts",
  "createdAt": "2024-06-18T10:00:00Z",
  "updatedAt": "2024-06-18T10:00:00Z"
}
```

---

#### エージェント更新

##### エンドポイント
```http
PUT /api/v1/agents/{id}
Content-Type: application/json
```

##### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | エージェントID |

##### リクエストボディ
```json
{
  "name": "UpdatedKotlinAgent",
  "languages": ["kotlin", "java"],
  "installCommand": "apt-get update && apt-get install -y kotlin openjdk-11",
  "executeCommand": "kotlinc -script main.kts"
}
```

##### レスポンス例
```json
{
  "id": "agent-001",
  "name": "UpdatedKotlinAgent",
  "languages": ["kotlin", "java"],
  "status": "AVAILABLE",
  "currentTaskId": null,
  "installCommand": "apt-get update && apt-get install -y kotlin openjdk-11",
  "executeCommand": "kotlinc -script main.kts",
  "createdAt": "2024-06-18T10:00:00Z",
  "updatedAt": "2024-06-18T10:00:00Z"
}
```

---

#### エージェント削除

##### エンドポイント
```http
DELETE /api/v1/agents/{id}
```

##### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | エージェントID |

##### レスポンス
```http
204 No Content
```

---

#### エラーレスポンス

##### 400 Bad Request
```json
{
  "error": "Invalid request parameters",
  "message": "Name is required"
}
```

##### 404 Not Found
```json
{
  "error": "Agent not found",
  "message": "Agent with id 'agent-001' does not exist"
}
```

##### 409 Conflict
```json
{
  "error": "Agent in use",
  "message": "Agent is currently executing a task"
}
```

##### 500 Internal Server Error
```json
{
  "error": "Internal server error",
  "message": "An unexpected error occurred"
}
```

---

### リポジトリ管理API

> kerutaシステムのリポジトリ管理に関するAPIエンドポイントとリクエスト/レスポンス例

#### 目次
- リポジトリ一覧取得
- リポジトリ作成
- リポジトリ詳細取得
- リポジトリ更新
- リポジトリ削除
- リポジトリ検証

---

#### リポジトリ一覧取得

##### エンドポイント
```http
GET /api/v1/repositories
```

##### クエリパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| isValid | boolean | 任意 | 有効性でフィルタリング |
| usePvc | boolean | 任意 | PVC使用でフィルタリング |

##### レスポンス例
```json
[
  {
    "id": "repo-001",
    "name": "keruta-app",
    "url": "https://github.com/user/keruta.git",
    "description": "keruta本体リポジトリ",
    "isValid": true,
    "setupScript": "#!/bin/bash\necho 'Setting up repository...'",
    "usePvc": false,
    "pvcStorageSize": "1Gi",
    "pvcAccessMode": "ReadWriteOnce",
    "createdAt": "2024-06-18T10:00:00Z",
    "updatedAt": "2024-06-18T10:00:00Z"
  }
]
```

---

#### リポジトリ作成

##### エンドポイント
```http
POST /api/v1/repositories
Content-Type: application/json
```

##### リクエストボディ
```json
{
  "name": "keruta-app",
  "url": "https://github.com/user/keruta.git",
  "description": "keruta本体リポジトリ",
  "setupScript": "#!/bin/bash\necho 'Setting up repository...'",
  "usePvc": false,
  "pvcStorageSize": "1Gi",
  "pvcAccessMode": "ReadWriteOnce"
}
```

##### レスポンス例
```json
{
  "id": "repo-001",
  "name": "keruta-app",
  "url": "https://github.com/user/keruta.git",
  "description": "keruta本体リポジトリ",
  "isValid": true,
  "setupScript": "#!/bin/bash\necho 'Setting up repository...'",
  "usePvc": false,
  "pvcStorageSize": "1Gi",
  "pvcAccessMode": "ReadWriteOnce",
  "createdAt": "2024-06-18T10:00:00Z",
  "updatedAt": "2024-06-18T10:00:00Z"
}
```

---

#### リポジトリ詳細取得

##### エンドポイント
```http
GET /api/v1/repositories/{id}
```

##### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | リポジトリID |

##### レスポンス例
```json
{
  "id": "repo-001",
  "name": "keruta-app",
  "url": "https://github.com/user/keruta.git",
  "description": "keruta本体リポジトリ",
  "isValid": true,
  "setupScript": "#!/bin/bash\necho 'Setting up repository...'",
  "usePvc": false,
  "pvcStorageSize": "1Gi",
  "pvcAccessMode": "ReadWriteOnce",
  "createdAt": "2024-06-18T10:00:00Z",
  "updatedAt": "2024-06-18T10:00:00Z"
}
```

---

#### リポジトリ更新

##### エンドポイント
```http
PUT /api/v1/repositories/{id}
Content-Type: application/json
```

##### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | リポジトリID |

##### リクエストボディ
```json
{
  "name": "updated-keruta-app",
  "url": "https://github.com/user/keruta.git",
  "description": "Updated keruta repository",
  "setupScript": "#!/bin/bash\necho 'Updated setup script...'",
  "usePvc": true,
  "pvcStorageSize": "5Gi",
  "pvcAccessMode": "ReadWriteMany"
}
```

##### レスポンス例
```json
{
  "id": "repo-001",
  "name": "updated-keruta-app",
  "url": "https://github.com/user/keruta.git",
  "description": "Updated keruta repository",
  "isValid": true,
  "setupScript": "#!/bin/bash\necho 'Updated setup script...'",
  "usePvc": true,
  "pvcStorageSize": "5Gi",
  "pvcAccessMode": "ReadWriteMany",
  "createdAt": "2024-06-18T10:00:00Z",
  "updatedAt": "2024-06-18T10:00:00Z"
}
```

---

#### リポジトリ削除

##### エンドポイント
```http
DELETE /api/v1/repositories/{id}
```

##### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | リポジトリID |

##### レスポンス
```http
204 No Content
```

---

#### リポジトリ検証

##### エンドポイント
```http
POST /api/v1/repositories/{id}/validate
```

##### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | リポジトリID |

##### レスポンス例
```json
{
  "id": "repo-001",
  "isValid": true,
  "validationResult": {
    "accessible": true,
    "cloneable": true,
    "branch": "main",
    "lastCommit": "abc123def456",
    "lastCommitDate": "2024-06-18T10:00:00Z"
  },
  "validatedAt": "2024-06-18T10:00:00Z"
}
```

---

#### エラーレスポンス

##### 400 Bad Request
```json
{
  "error": "Invalid request parameters",
  "message": "Repository URL is required"
}
```

##### 404 Not Found
```json
{
  "error": "Repository not found",
  "message": "Repository with id 'repo-001' does not exist"
}
```

##### 409 Conflict
```json
{
  "error": "Repository in use",
  "message": "Repository is currently being used by active tasks"
}
```

##### 422 Unprocessable Entity
```json
{
  "error": "Invalid repository",
  "message": "Repository URL is not accessible or invalid"
}
```

##### 500 Internal Server Error
```json
{
  "error": "Internal server error",
  "message": "An unexpected error occurred"
}
```

---

### ドキュメント管理API

> kerutaシステムのドキュメント管理に関するAPIエンドポイントとリクエスト/レスポンス例

#### 目次
- ドキュメント一覧取得
- ドキュメント作成
- ドキュメント詳細取得
- ドキュメント更新
- ドキュメント削除
- ドキュメント検索

---

#### ドキュメント一覧取得

##### エンドポイント
```http
GET /api/v1/documents
```

##### クエリパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| tag | string | 任意 | タグでフィルタリング |
| search | string | 任意 | タイトル・内容で検索 |

##### レスポンス例
```json
[
  {
    "id": "doc-001",
    "title": "システム概要",
    "content": "kerutaシステムは、Kubernetes上でタスクを実行するためのプラットフォームです。",
    "tags": ["概要"],
    "createdAt": "2024-06-18T10:00:00Z",
    "updatedAt": "2024-06-18T10:00:00Z"
  }
]
```

---

#### ドキュメント作成

##### エンドポイント
```http
POST /api/v1/documents
Content-Type: application/json
```

##### リクエストボディ
```json
{
  "title": "システム概要",
  "content": "kerutaシステムは、Kubernetes上でタスクを実行するためのプラットフォームです。",
  "tags": ["概要"]
}
```

##### レスポンス例
```json
{
  "id": "doc-001",
  "title": "システム概要",
  "content": "kerutaシステムは、Kubernetes上でタスクを実行するためのプラットフォームです。",
  "tags": ["概要"],
  "createdAt": "2024-06-18T10:00:00Z",
  "updatedAt": "2024-06-18T10:00:00Z"
}
```

---

#### ドキュメント詳細取得

##### エンドポイント
```http
GET /api/v1/documents/{id}
```

##### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | ドキュメントID |

##### レスポンス例
```json
{
  "id": "doc-001",
  "title": "システム概要",
  "content": "kerutaシステムは、Kubernetes上でタスクを実行するためのプラットフォームです。",
  "tags": ["概要"],
  "createdAt": "2024-06-18T10:00:00Z",
  "updatedAt": "2024-06-18T10:00:00Z"
}
```

---

#### ドキュメント更新

##### エンドポイント
```http
PUT /api/v1/documents/{id}
Content-Type: application/json
```

##### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | ドキュメントID |

##### リクエストボディ
```json
{
  "title": "更新されたシステム概要",
  "content": "kerutaシステムは、Kubernetes上でタスクを実行するためのプラットフォームです。最新の機能が追加されました。",
  "tags": ["概要", "更新"]
}
```

##### レスポンス例
```json
{
  "id": "doc-001",
  "title": "更新されたシステム概要",
  "content": "kerutaシステムは、Kubernetes上でタスクを実行するためのプラットフォームです。最新の機能が追加されました。",
  "tags": ["概要", "更新"],
  "createdAt": "2024-06-18T10:00:00Z",
  "updatedAt": "2024-06-18T10:00:00Z"
}
```

---

#### ドキュメント削除

##### エンドポイント
```http
DELETE /api/v1/documents/{id}
```

##### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | ドキュメントID |

##### レスポンス
```http
204 No Content
```

---

#### ドキュメント検索

##### エンドポイント
```http
GET /api/v1/documents/search
```

##### クエリパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| q | string | 必須 | 検索クエリ |
| type | string | 任意 | 検索タイプ（title, content, all） |

##### レスポンス例
```json
{
  "query": "システム",
  "results": [
    {
      "id": "doc-001",
      "title": "システム概要",
      "content": "kerutaシステムは、Kubernetes上でタスクを実行するためのプラットフォームです。",
      "tags": ["概要"],
      "score": 0.95,
      "highlights": [
        {
          "field": "title",
          "snippet": "システム概要"
        },
        {
          "field": "content",
          "snippet": "kerutaシステムは、Kubernetes上でタスクを実行するためのプラットフォームです。"
        }
      ]
    }
  ],
  "total": 1,
  "page": 1,
  "perPage": 10
}
```

---

#### エラーレスポンス

##### 400 Bad Request
```json
{
  "error": "Invalid request parameters",
  "message": "Title is required"
}
```

##### 404 Not Found
```json
{
  "error": "Document not found",
  "message": "Document with id 'doc-001' does not exist"
}
```

##### 500 Internal Server Error
```json
{
  "error": "Internal server error",
  "message": "An unexpected error occurred"
}
``` 