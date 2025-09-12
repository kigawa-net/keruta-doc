# keruta - システム詳細ドキュメント

> **概要**: kerutaシステム全体の構成、セットアップ、API概要、技術スタック、プロジェクト構造をまとめたドキュメントです。

## 目次
- [システム概要](#システム概要)
- [セットアップ](#セットアップ)
- [技術スタック](#技術スタック)
- [プロジェクト構造](#プロジェクト構造)
- [API概要](#api概要)
- [関連ドキュメント](#関連ドキュメント)

## システム概要
kerutaは、タスクのキュー管理・自動化・Kubernetes連携を特徴とする軽量なタスク管理システムです。Web UI・REST API・管理パネル・Kubernetes連携・Gitリポジトリ管理などを備えています。

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
```env
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
- 管理パネル: http://localhost:8080/admin
- Swagger UI: http://localhost:8080/swagger-ui.html

## 技術スタック
- Kotlin / Spring Boot / MongoDB / Gradle（マルチモジュール構成）

## プロジェクト構造
```text
keruta/
├── buildSrc/                    # ビルド構成の共通化
├── core/                        # コアモジュール（ドメイン・ユースケース）
├── infra/                       # インフラ層（永続化・セキュリティ等）
├── api/                         # API・管理パネル
└── ...
```

## API概要
- タスク管理・ドキュメント管理・Gitリポジトリ管理・Kubernetes連携・認証/認可 など
- 詳細なAPI仕様は [apiSpec.md](./apiSpec.md) を参照

## データモデル
- 主要なデータモデル（Task, Agent, Repository等）は [dataModel.md](./dataModel.md) を参照

## 関連ドキュメント
- [システム概要・アーキテクチャ](./systemOverview.md)
- [API仕様詳細](./apiSpec.md)
- [データモデル定義](./dataModel.md)
- [Keycloak認証連携](../auth/keycloakIntegration.md)
- [Kubernetes連携](../kubernetes/kubernetesIntegration.md)
- [管理パネル](../admin/adminPanel.md)

