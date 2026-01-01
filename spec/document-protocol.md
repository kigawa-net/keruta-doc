# keruta document protocol (KDOP) 仕様

## 概要

keruta document protocol (KDOP) は、kerutaシステムにおけるドキュメントの作成、編集、管理を制御するためのREST APIです。HTTP/HTTPSベースの標準的なRESTful APIとして設計され、ドキュメントのCRUD操作とバージョン管理をサポートします。

KDOPはkerutaシステムのドキュメント管理機能をAPIとして提供し、他のコンポーネントとの統合を容易にします。

## アーキテクチャ

```
┌─────────────────┐          HTTP/HTTPS         ┌─────────────────┐
│   Client        │◄──────────────────────────►│   KDOP Server   │
│  (keruta-admin) │    REST API calls          │  (keruta-api)   │
└─────────────────┘                             └─────────────────┘
```

## 基底プロトコル

KDOPはRESTful APIとしてHTTP/HTTPSを通信プロトコルとして使用します。

- **プロトコル**: HTTP/1.1 以上、HTTPS推奨
- **認証**: HTTPヘッダによるBearerトークン認証 (OIDC)
- **データ形式**: JSON (Content-Type: application/json)
- **HTTPメソッド**: GET, POST, PUT, DELETE, PATCH

### APIエンドポイント

すべてのKDOPエンドポイントは `/v1/` をベースパスとします。

#### 主要エンドポイント

| エンドポイント | メソッド | 説明 |
|---------------|---------|------|
| `/v1/documents` | GET | ドキュメント一覧取得 |
| `/v1/documents` | POST | ドキュメント作成 |
| `/v1/documents/{id}` | GET | ドキュメント詳細取得 |
| `/v1/documents/{id}` | PUT | ドキュメント更新 |
| `/v1/documents/{id}` | DELETE | ドキュメント削除 |
| `/v1/documents/{id}/versions` | GET | バージョン履歴取得 |
| `/v1/documents/{id}/versions/{version}` | GET | 特定バージョン取得 |

### リクエスト/レスポンス形式

#### 共通レスポンスヘッダー
```
Content-Type: application/json
X-API-Version: 1.0
X-Request-ID: <uuid>
```

#### 共通レスポンスボディ
```json
{
  "success": true,
  "data": { /* エンドポイント固有のデータ */ },
  "timestamp": "2024-01-01T10:00:00Z",
  "version": "1.0"
}
```

#### エラーレスポンス
```json
{
  "success": false,
  "error": {
    "code": "DOCUMENT_NOT_FOUND",
    "message": "指定されたドキュメントが見つかりません",
    "details": { /* 追加情報 */ }
  },
  "timestamp": "2024-01-01T10:00:00Z"
}
```

## ドキュメント処理ライフサイクル

### 1. ドキュメント作成
1. **POST /v1/documents**: 新規ドキュメント作成リクエスト
2. **サーバー側検証**: 入力データの検証
3. **ストレージ保存**: ドキュメントの永続化
4. **レスポンス返却**: 作成されたドキュメント情報

### 2. ドキュメント編集
1. **GET /v1/documents/{id}**: 現在のドキュメント取得
2. **PUT /v1/documents/{id}**: 更新リクエスト
3. **バージョン管理**: 変更履歴の保存
4. **競合検知**: 同時編集時の競合解決

### 3. ドキュメント管理
1. **GET /v1/documents**: 一覧取得とフィルタリング
2. **DELETE /v1/documents/{id}**: 論理削除
3. **バージョン操作**: 履歴閲覧とロールバック

## エラーハンドリング

### HTTPステータスコード

| ステータスコード | 説明 |
|----------------|------|
| 200 OK | 成功 |
| 201 Created | リソース作成成功 |
| 204 No Content | 削除成功 |
| 400 Bad Request | リクエスト不正 |
| 401 Unauthorized | 認証失敗 |
| 403 Forbidden | 権限不足 |
| 404 Not Found | リソース不存在 |
| 409 Conflict | バージョン競合 |
| 422 Unprocessable Entity | バリデーションエラー |
| 500 Internal Server Error | サーバーエラー |

### エラーコード

- **DOCUMENT_NOT_FOUND**: 404 - ドキュメント不存在
- **DOCUMENT_LOCKED**: 409 - ドキュメントロック中
- **INVALID_OPERATION**: 400 - 不正操作
- **VERSION_CONFLICT**: 409 - バージョン競合
- **PERMISSION_DENIED**: 403 - 権限不足
- **VALIDATION_FAILED**: 422 - 入力検証失敗
- **STORAGE_ERROR**: 500 - ストレージエラー

## セキュリティ

### 通信の暗号化
- **必須**: HTTPS の使用
- **証明書**: 有効な SSL/TLS 証明書
- **HSTS**: HTTP Strict Transport Security ヘッダー

### 認証と認可
- **Keycloak OIDC Access Token**: BearerトークンとしてAuthorizationヘッダーで送信
- **トークン検証**: Keycloakの公開鍵による署名検証
- **スコープ検証**: ドキュメント操作権限の確認
- **RBAC**: ロールベースアクセス制御
- **APIキー**: サービス間認証用

### 入力検証
- **スキーマ検証**: JSON スキーマによる構造検証
- **サニタイズ**: XSS対策の入力サニタイズ
- **サイズ制限**: リクエストボディの上限設定

## パフォーマンス最適化

### キャッシュ
- **ETag**: リソースバージョンのキャッシュ制御
- **Last-Modified**: 更新日時ベースのキャッシュ
- **Cache-Control**: キャッシュポリシーの指定

### ページネーション
- **オフセットベース**: `?offset=0&limit=20`
- **カーソルベース**: `?cursor=<id>&limit=20`

### フィルタリングとソート
- **クエリパラメータ**: `?tag=document&sort=createdAt:desc`

## バージョン互換性

### APIバージョン
- **v1**: 基本ドキュメント操作
- **v1.1**: バージョン管理機能
- **v1.2**: 高度な検索とフィルタリング

### バージョン指定
- **ヘッダー**: `Accept: application/vnd.keruta.v1+json`
- **URL**: `/v1/documents`

このKDOP仕様により、kerutaシステムでの標準的なドキュメント管理APIを提供します。