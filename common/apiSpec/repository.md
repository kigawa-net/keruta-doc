# リポジトリ管理API仕様

kerutaシステムのリポジトリ管理に関するAPIエンドポイントとリクエスト/レスポンス例を記載します。

## 目次
- リポジトリ一覧取得
- リポジトリ作成
- リポジトリ詳細取得
- リポジトリ更新
- リポジトリ削除
- リポジトリ検証

---

### リポジトリ一覧取得

#### エンドポイント
```http
GET /api/v1/repositories
```

#### クエリパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| isValid | boolean | 任意 | 有効性でフィルタリング |
| usePvc | boolean | 任意 | PVC使用でフィルタリング |

#### レスポンス例
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

### リポジトリ作成

#### エンドポイント
```http
POST /api/v1/repositories
Content-Type: application/json
```

#### リクエストボディ
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

#### レスポンス例
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

### リポジトリ詳細取得

#### エンドポイント
```http
GET /api/v1/repositories/{id}
```

#### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | リポジトリID |

#### レスポンス例
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

### リポジトリ更新

#### エンドポイント
```http
PUT /api/v1/repositories/{id}
Content-Type: application/json
```

#### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | リポジトリID |

#### リクエストボディ
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

#### レスポンス例
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

### リポジトリ削除

#### エンドポイント
```http
DELETE /api/v1/repositories/{id}
```

#### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | リポジトリID |

#### レスポンス
```http
204 No Content
```

---

### リポジトリ検証

#### エンドポイント
```http
POST /api/v1/repositories/{id}/validate
```

#### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | リポジトリID |

#### レスポンス例
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