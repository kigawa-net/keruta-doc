# keruta-executor

## 概要

keruta-executorは、keruta-apiからペンディングタスクを検知してcoder環境を起動する環境管理サービスです。
Spring Bootベースのアプリケーションとして実装されており、タスクの実行は行わず、coder環境の起動のみを担当します。

## 機能

### 主要機能

- **タスク監視**: keruta-apiから定期的にペンディング状態のタスクを監視
- **coder環境起動**: ペンディングタスク検知時にcoder環境を起動
- **環境管理**: coder環境の起動状態を管理
- **設定可能な監視**: application.propertiesによる柔軟な設定

### 責任範囲

- ペンディングタスクの検知
- coder環境の起動
- 環境状態の管理
- ※タスクの実際の実行は行わない（coderが担当）

## アーキテクチャ

### システム構成

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   keruta-api    │◄───│ keruta-executor │───▶│     coder       │
│  (タスク管理)    │    │ (環境管理のみ)   │    │  (タスク実行)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### コンポーネント構成

- **TaskMonitor**: ペンディングタスクの監視
- **TaskApiService**: keruta-apiとの通信
- **CoderLauncher**: coder環境の起動制御

## 技術仕様

### 実行環境

- **言語**: Kotlin (99.3%)
- **フレームワーク**: Spring Boot
- **ビルドツール**: Gradle
- **コンテナ**: Docker対応

### 依存関係

- Spring Boot Framework
- coder (環境起動対象)
- keruta-api (REST API連携)
- Kotlin Coroutines

### 開発ツール

- **コード品質**: ktlint
- **テスト**: JUnit
- **ビルド**: Gradle Wrapper

## 設定

### application.properties

```properties
# keruta-api との連携設定
keruta.api.base-url=http://localhost:8080
keruta.executor.processing-delay=5000

# coder環境設定
coder.launch.timeout=60000
coder.launch.workspace-directory=/tmp/keruta-workspaces
```

### 主要設定項目

- **keruta.api.base-url**: keruta-apiのベースURL
- **keruta.executor.processing-delay**: タスク監視間隔（ミリ秒）
- **coder.launch.timeout**: coder環境起動タイムアウト
- **coder.launch.workspace-directory**: ワークスペースディレクトリ

## ビルドと実行

### Gradleでのビルド

```bash
./gradlew build
./gradlew bootRun
```

### Dockerでの実行

```bash
docker-compose up -d
```

### テストの実行

```bash
./gradlew test
```

### コード品質チェック

```bash
./gradlew ktlintCheck
./gradlew ktlintFormat
```

## 動作フロー

### 環境管理フロー

1. **タスク監視**: keruta-apiからペンディングタスクを定期的に監視
2. **タスク検知**: ペンディング状態のタスクを発見
3. **coder起動**: CoderLauncherでcoder環境を起動
4. **環境待機**: coder環境がタスクを処理するまで待機

### 監視サイクル

```
┌─────────────────┐
│  Task Monitor   │
└─────────┬───────┘
          │
          ▼
┌─────────────────┐
│ Pending Check   │
└─────────┬───────┘
          │
          ▼
┌─────────────────┐
│  Coder Launch   │
└─────────┬───────┘
          │
          ▼
┌─────────────────┐
│   Wait & Repeat │
└─────────────────┘
```

## 関連リポジトリ

- **keruta-api**: メインAPIサーバー（タスク管理）
- **keruta-admin**: 管理画面
- **keruta-doc**: ドキュメンテーション（本リポジトリ）

## 開発

### 開発環境のセットアップ

1. keruta-apiサーバーの起動
2. coderの準備・設定
3. application.propertiesの設定
4. keruta-executorの起動

### デバッグ

- Spring Bootのデバッグモードでの実行
- coder環境起動ログの監視
- ログレベルの調整によるデバッグ出力

## 注意事項

- keruta-executorはタスクの実行は行わない
- coder環境の起動と管理のみを担当
- 実際のタスク実行はcoder側で行われる
- ペンディングタスクの監視は継続的に実行される