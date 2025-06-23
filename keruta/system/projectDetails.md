# keruta - システム詳細ドキュメント

> **概要**: kerutaシステム全体の構成、API仕様、セットアップ方法、技術スタックなどの詳細をまとめたドキュメントです。

## 目次
- [概要](#概要)
- [セットアップ](#セットアップ)
- [機能一覧](#機能一覧)
- [技術スタック](#技術スタック)
- [プロジェクト構造](#プロジェクト構造)
- [APIエンドポイント一覧](#apiエンドポイント一覧)
- [関連リンク](#関連リンク)

## 概要
このAPIサーバーは、タスクのキューを管理するRESTfulインターフェースを提供します。作成されたタスクは必ずキューに登録され、優先順位に基づいて処理されます。

## セットアップ
### 必要条件
- Java 21
- Docker / Docker Compose（MongoDB用）

### MongoDBセットアップ
```bash
docker-compose up -d
```
- 初期認証: admin / password

### 環境変数例
```
SPRING_DATA_MONGODB_HOST=localhost
SPRING_DATA_MONGODB_PORT=27017
SPRING_DATA_MONGODB_DATABASE=keruta
SPRING_DATA_MONGODB_USERNAME=admin
SPRING_DATA_MONGODB_PASSWORD=password
SPRING_DATA_MONGODB_AUTHENTICATION_DATABASE=admin
```

### ビルド・起動
```bash
./gradlew build
./gradlew bootRun
```
- アプリ起動: http://localhost:8080

### 管理パネル・API仕様書
- 管理パネル: http://localhost:8080/admin
- Swagger UI: http://localhost:8080/swagger-ui.html

## 機能一覧
- タスクのCRUD・自動キュー登録・優先順位付け・ステータス管理
- ドキュメント保存・GitリポジトリURL保存
- JWT認証によるアクセス制御
- Swagger/OpenAPIによるAPI仕様書自動生成
- 管理パネル（タスク・ドキュメント・リポジトリ管理）
- タスク情報を環境変数としてKubernetes Pod作成

## 技術スタック
- Kotlin / Spring Boot / MongoDB / Gradle（マルチモジュール）

## プロジェクト構造
```
keruta/
├── buildSrc/                    # ビルド構成の共通化
│   └── src/main/kotlin/        # ビルドロジック
├── core/                       # コアモジュール
│   ├── domain/                 # ドメインモデル
│   └── usecase/                # ユースケース実装
├── infra/                      # インフラストラクチャレイヤー
│   ├── persistence/            # MongoDB永続化実装
│   └── security/               # JWT認証実装
└── api/                        # Gitリポジトリ管理API
```

## APIエンドポイント一覧
### 認証・認可
```
POST   /api/v1/auth/login         # ログイン（JWTトークン返却）
POST   /api/v1/auth/refresh       # JWTトークンリフレッシュ
```
### タスク管理
```
GET    /api/v1/tasks              # タスク一覧
POST   /api/v1/tasks              # 新規タスク作成
GET    /api/v1/tasks/{id}         # タスク取得
PUT    /api/v1/tasks/{id}         # タスク更新
DELETE /api/v1/tasks/{id}         # タスク削除
GET    /api/v1/tasks/queue/next   # キューから次のタスク取得
PATCH  /api/v1/tasks/{id}/status  # ステータス更新
PATCH  /api/v1/tasks/{id}/priority # 優先度更新
POST   /api/v1/tasks/kubernetes/create # タスク情報でKubernetes Pod作成
```
### ドキュメント管理
```
GET    /api/v1/documents          # ドキュメント一覧
POST   /api/v1/documents          # 新規作成
GET    /api/v1/documents/{id}     # 取得
PUT    /api/v1/documents/{id}     # 更新
DELETE /api/v1/documents/{id}     # 削除
GET    /api/v1/documents/search   # 検索
```
### Gitリポジトリ管理
```
GET    /api/v1/repositories       # 一覧
POST   /api/v1/repositories       # 新規登録
GET    /api/v1/repositories/{id}  # 取得
PUT    /api/v1/repositories/{id}  # 更新
DELETE /api/v1/repositories/{id}  # 削除
GET    /api/v1/repositories/{id}/validate # URL有効性確認
```

## 関連リンク
- [adminPanel.md](./adminPanel.md)
- [auth/keycloakIntegration.md](./auth/keycloakIntegration.md)
- [kubernetes/kubernetesIntegration.md](./kubernetes/kubernetesIntegration.md)

