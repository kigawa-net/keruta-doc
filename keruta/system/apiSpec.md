# API仕様

> kerutaシステムの主要APIエンドポイントとリクエスト/レスポンス例

## 目次
- [タスク管理API](./api/taskApi.md)
- [エージェント管理API](./api/agentApi.md)
- [リポジトリ管理API](./api/repositoryApi.md)
- [ドキュメント管理API](./api/documentApi.md)
- [共通仕様](#共通仕様)
- [関連ドキュメント](#関連ドキュメント)

---

## 共通仕様

### 基本URL
```
https://api.keruta.example.com
```

### 認証
すべてのAPIリクエストには認証が必要です。

```http
Authorization: Bearer <access_token>
```

### レスポンス形式
すべてのAPIレスポンスは以下の形式で返されます：

#### 成功レスポンス
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

#### エラーレスポンス
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

### HTTPステータスコード
| コード | 説明 |
|--------|------|
| 200 | OK - リクエスト成功 |
| 201 | Created - リソース作成成功 |
| 204 | No Content - 削除成功 |
| 400 | Bad Request - リクエスト不正 |
| 401 | Unauthorized - 認証失敗 |
| 403 | Forbidden - 権限不足 |
| 404 | Not Found - リソース不存在 |
| 409 | Conflict - 競合状態 |
| 422 | Unprocessable Entity - 処理不可能 |
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

## 関連ドキュメント
- [システム詳細・セットアップ](./projectDetails.md)
- [データモデル定義](./dataModel.md)
- [タスク管理API詳細](./api/taskApi.md)
- [エージェント管理API詳細](./api/agentApi.md)
- [リポジトリ管理API詳細](./api/repositoryApi.md)
- [ドキュメント管理API詳細](./api/documentApi.md) 