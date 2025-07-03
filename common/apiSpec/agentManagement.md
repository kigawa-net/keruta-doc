# エージェント管理API仕様

kerutaシステムのエージェント管理に関するAPIエンドポイントとリクエスト/レスポンス例を記載します。

## 目次
- エージェント一覧取得
- エージェント作成
- エージェント詳細取得
- エージェント更新
- エージェント削除

---

### エージェント一覧取得

#### エンドポイント
```http
GET /api/v1/agents
```

#### クエリパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| status | string | 任意 | エージェントのステータスでフィルタリング |
| language | string | 任意 | 対応言語でフィルタリング |

#### レスポンス例
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

### エージェント作成

#### エンドポイント
```http
POST /api/v1/agents
Content-Type: application/json
```

#### リクエストボディ
```json
{
  "name": "KotlinAgent",
  "languages": ["kotlin"],
  "installCommand": "apt-get update && apt-get install -y kotlin",
  "executeCommand": "kotlinc -script main.kts"
}
```

#### レスポンス例
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

### エージェント詳細取得

#### エンドポイント
```http
GET /api/v1/agents/{id}
```

#### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | エージェントID |

#### レスポンス例
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

### エージェント更新

#### エンドポイント
```http
PUT /api/v1/agents/{id}
Content-Type: application/json
```

#### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | エージェントID |

#### リクエストボディ
```json
{
  "name": "UpdatedKotlinAgent",
  "languages": ["kotlin", "java"],
  "installCommand": "apt-get update && apt-get install -y kotlin openjdk-11",
  "executeCommand": "kotlinc -script main.kts"
}
```

#### レスポンス例
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

### エージェント削除

#### エンドポイント
```http
DELETE /api/v1/agents/{id}
```

#### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | エージェントID |

#### レスポンス
```http
204 No Content
``` 