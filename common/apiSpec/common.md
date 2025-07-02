# API共通仕様

このドキュメントでは、kerutaプロジェクトにおける各サブシステムで共通して利用されるAPI設計方針や認証・認可、エラーハンドリング、レスポンス形式、バージョニング等の共通仕様についてまとめます。

## API設計方針
- RESTful設計を基本とする。
- リソース指向のエンドポイント命名。

## 認証・認可
- JWTベースの認証を推奨。
- 必要に応じてOAuth2やAPIキーも利用可能。
- すべてのAPIリクエストには認証が必要です。

### 認証方式
- JWT（JSON Web Token）ベースの認証を標準とする。
- 認証トークンはHTTPリクエストヘッダー `Authorization: Bearer <access_token>` で送信する。
- トークンの有効期限切れ時は401 Unauthorizedを返す。
- 必要に応じてOAuth2やAPIキー認証もサポート可能。

### トークン取得・リフレッシュ
- 認証API（例: `/api/v1/auth/login`）でユーザー名・パスワードを送信し、JWTトークンとリフレッシュトークンを取得。
- リフレッシュトークンによるトークン再発行API（例: `/api/v1/auth/refresh`）も提供。

### サンプルフロー
1. ログインAPIでJWTトークン取得
2. 取得したトークンを全APIリクエストのヘッダーに付与
3. トークン期限切れ時はリフレッシュAPIで再取得

### サンプルリクエスト・レスポンス
```http
POST /api/v1/auth/login
Content-Type: application/json

{
  "username": "user",
  "password": "password"
}
```
```json
{
  "token": "JWT_TOKEN_HERE",
  "refreshToken": "REFRESH_TOKEN_HERE",
  "expiresIn": 3600
}
```

### 認証エラー例
- 401 Unauthorized: トークン不正・期限切れ
- 403 Forbidden: 権限不足

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json

{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "認証トークンが無効または期限切れです"
  }
}
```

```http
HTTP/1.1 403 Forbidden
Content-Type: application/json

{
  "error": {
    "code": "FORBIDDEN",
    "message": "この操作を実行する権限がありません"
  }
}
```

## エラーハンドリング
- エラー時はHTTPステータスコードとともに、以下のJSON形式で返却する。
- `details`フィールドには、エラーの詳細な原因や追加情報（バリデーションエラーの項目、内部例外の種類、外部APIの応答内容など）を含めることができる。

### サンプル
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力値に誤りがあります",
    "details": {
      "fields": {
        "email": "メールアドレスの形式が不正です",
        "password": "8文字以上で入力してください"
      }
    }
  },
  "meta": {
    "timestamp": "2024-06-18T10:00:00Z",
    "version": "1.0.0"
  }
}
```

### 推奨事項
- `details`はオブジェクト型で、状況に応じて柔軟に構造を設計する。
- バリデーションエラーの場合は、エラーとなったフィールド名と理由を含める。
- 外部API連携エラーの場合は、外部APIのレスポンスやエラーコードを含める。
- 予期しない例外の場合は、内部例外名やスタックトレースの一部を含めることも可能（セキュリティに配慮）。

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

## CORS (Cross-Origin Resource Sharing)
- すべてのAPIエンドポイントはCORSに対応しています。
- 以下のCORSヘッダーが設定されています：

### 許可されるオリジン
- 開発環境: `http://localhost:3000`, `http://localhost:8000`
- 本番環境: `https://*.keruta.example.com`

### 許可されるメソッド
- `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`

### 許可されるヘッダー
- `Content-Type`, `Authorization`, `X-Requested-With`, `Accept`, `Origin`, `X-CSRF-Token`

### 認証情報
- `Access-Control-Allow-Credentials: true`

### プリフライトリクエストのキャッシュ時間
- `Access-Control-Max-Age: 86400` (24時間)

### サンプルレスポンスヘッダー
```http
Access-Control-Allow-Origin: https://admin.keruta.example.com
Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS
Access-Control-Allow-Headers: Content-Type, Authorization, X-Requested-With, Accept, Origin, X-CSRF-Token
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400
```

## その他共通事項
- タイムゾーンはJST（日本標準時）を基準とする。
- 日時はISO 8601形式（例: `2024-06-01T12:00:00+09:00`）で返却。 
