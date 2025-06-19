# Kubernetes Integration - タスク環境変数によるPod作成

## 概要

kerutaシステムは、最新のタスク情報を環境変数として設定したKubernetesポッドを作成する機能を提供します。この機能により、タスクキューから取得したタスクの情報をKubernetesポッドに渡し、タスクに関連する処理を自動化することができます。

## 機能詳細

### 環境変数マッピング

タスクの各フィールドは以下のように環境変数にマッピングされます：

| タスクフィールド    | 環境変数名                   | 説明                                                    |
|-------------|-------------------------|-------------------------------------------------------|
| id          | KERUTA_TASK_ID          | タスクの一意識別子                                             |
| title       | KERUTA_TASK_TITLE       | タスクのタイトル                                              |
| description | KERUTA_TASK_DESCRIPTION | タスクの詳細説明                                              |
| priority    | KERUTA_TASK_PRIORITY    | タスクの優先度（数値）                                           |
| status      | KERUTA_TASK_STATUS      | タスクのステータス（PENDING, IN_PROGRESS, COMPLETED, CANCELLED） |
| createdAt   | KERUTA_TASK_CREATED_AT  | タスク作成日時（ISO-8601形式）                                   |
| updatedAt   | KERUTA_TASK_UPDATED_AT  | タスク最終更新日時（ISO-8601形式）                                 |

### 設定

Kubernetes統合の設定はデータベースに保存されます。初回起動時には、以下のデフォルト設定が使用されます：

1. `application.properties`ファイルのデフォルト設定：

```properties
# Kubernetes設定のデフォルト値
keruta.kubernetes.enabled=true
keruta.kubernetes.config-path=/path/to/kube/config
keruta.kubernetes.in-cluster=false
keruta.kubernetes.default-namespace=default
```

これらの設定は管理パネルの「Kubernetes設定」画面から変更できます。変更された設定はデータベースに保存され、アプリケーションの再起動後も保持されます。

| 設定項目                | 説明                                  |
|---------------------|-------------------------------------|
| 有効/無効              | Kubernetes統合機能の有効/無効（true/false）    |
| 設定ファイルパス           | kubeconfig ファイルのパス（クラスター外で実行する場合）   |
| クラスター内実行           | クラスター内で実行する場合はtrue、外部から接続する場合はfalse |
| デフォルトネームスペース       | デフォルトのネームスペース                       |
| デフォルトイメージ          | タスク実行用のデフォルトDockerイメージ               |
| プロセッサーネームスペース      | ジョブプロセッサー用のネームスペース                   |

### セキュリティ考慮事項

1. **RBAC権限**: kerutaサービスアカウントには、ポッド作成のための適切なRBAC権限が必要です。
2. **機密情報**: 機密情報を含むタスクを処理する場合は、Kubernetes Secretsの使用を検討してください。
3. **ネットワークポリシー**: 作成されたポッドに適切なネットワークポリシーを適用してください。

## トラブルシューティング

1. **ポッド作成失敗**: Kubernetesクラスターへの接続設定と権限を確認してください。
2. **環境変数が設定されない**: タスクが正しく取得されているか確認してください。
3. **ポッドが起動しない**: イメージ名とリソース制限を確認してください。

## 制限事項

1. 現在のバージョンでは、一度に1つのポッドのみ作成できます。
2. 長いタスク説明は環境変数の制限により切り詰められる可能性があります。
