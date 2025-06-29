# ドキュメント管理API

> kerutaシステムのドキュメント管理に関するAPIエンドポイントとリクエスト/レスポンス例

## 目次
- [ドキュメント一覧取得](#ドキュメント一覧取得)
- [ドキュメント作成](#ドキュメント作成)
- [ドキュメント詳細取得](#ドキュメント詳細取得)
- [ドキュメント更新](#ドキュメント更新)
- [ドキュメント削除](#ドキュメント削除)
- [ドキュメント検索](#ドキュメント検索)

---

## ドキュメント一覧取得

### エンドポイント
```http
GET /api/v1/documents
```

### クエリパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| tag | string | 任意 | タグでフィルタリング |
| search | string | 任意 | タイトル・内容で検索 |

### レスポンス例
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

## ドキュメント作成

### エンドポイント
```http
POST /api/v1/documents
Content-Type: application/json
```

### リクエストボディ
```json
{
  "title": "システム概要",
  "content": "kerutaシステムは、Kubernetes上でタスクを実行するためのプラットフォームです。",
  "tags": ["概要"]
}
```

### レスポンス例
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

## ドキュメント詳細取得

### エンドポイント
```http
GET /api/v1/documents/{id}
```

### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | ドキュメントID |

### レスポンス例
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

## ドキュメント更新

### エンドポイント
```http
PUT /api/v1/documents/{id}
Content-Type: application/json
```

### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | ドキュメントID |

### リクエストボディ
```json
{
  "title": "更新されたシステム概要",
  "content": "kerutaシステムは、Kubernetes上でタスクを実行するためのプラットフォームです。最新の機能が追加されました。",
  "tags": ["概要", "更新"]
}
```

### レスポンス例
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

## ドキュメント削除

### エンドポイント
```http
DELETE /api/v1/documents/{id}
```

### パスパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| id | string | 必須 | ドキュメントID |

### レスポンス
```http
204 No Content
```

---

## ドキュメント検索

### エンドポイント
```http
GET /api/v1/documents/search
```

### クエリパラメータ
| パラメータ | 型 | 必須 | 説明 |
|-----------|----|------|------|
| q | string | 必須 | 検索クエリ |
| type | string | 任意 | 検索タイプ（title, content, all） |

### レスポンス例
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
  "error": "Document not found",
  "message": "Document with id 'doc-001' does not exist"
}
```

### 500 Internal Server Error
```json
{
  "error": "Internal server error",
  "message": "An unexpected error occurred"
}
``` 