# keruta-doc

本リポジトリは、kerutaプロジェクトの設計・運用・技術情報をまとめたドキュメント集です。

> **注意**: このディレクトリは https://github.com/kigawa-net/keruta-doc リポジトリに移行されます。
> OpenAPI仕様書は自動的に生成され、GitHub Actionsによって上記リポジトリに自動的にプッシュされます。

## サブプロジェクト
- `keruta`: keruta APIサーバーの設計・運用ドキュメント（`keruta/` ディレクトリに配置）
- `keruta-github`: kerutaをGitHubから操作するためのツール/サービス（`keruta-github/` ディレクトリに配置）
- `keruta-agent`: kerutaによって実行されるKubernetes Job内でタスク実行を統合的に管理するCLIツール（`keruta-agent/` ディレクトリに配置）

## ドキュメント一覧

### keruta

- [README.md](keruta/README.md)
  - kerutaプロジェクトの概要

#### admin
- [adminPanel.md](keruta/admin/adminPanel.md)
  - 管理パネルの機能・画面・操作方法
- [adminPanelScriptGenerator.md](keruta/admin/adminPanelScriptGenerator.md)
  - 管理パネルスクリプトジェネレーター

#### auth
- [keycloakIntegration.md](keruta/auth/keycloakIntegration.md)
  - Keycloakによる認証・認可

#### git
- [gitExcludeSpec.md](keruta/git/gitExcludeSpec.md)
  - git-excludeの仕様
- [repositoryManagement.md](keruta/git/repositoryManagement.md)
  - リポジトリ管理

#### kubernetes
- [kubernetesIntegration.md](keruta/kubernetes/kubernetesIntegration.md)
  - Kubernetes連携の概要
- [gitCloneInitContainer.md](keruta/kubernetes/gitCloneInitContainer.md)
  - Git Clone Init Container
- [kubernetesAndDeploy.md](keruta/kubernetes/kubernetesAndDeploy.md)
  - Kubernetes環境へのデプロイ
- [kubernetesCleanupJob.md](keruta/kubernetes/kubernetesCleanupJob.md)
  - クリーンアップJob
- [kubernetesInitContainer.md](keruta/kubernetes/kubernetesInitContainer.md)
  - Init Container
- [kubernetesJobSpec.md](keruta/kubernetes/kubernetesJobSpec.md)
  - Jobの仕様
- [kubernetesLogCollection.md](keruta/kubernetes/kubernetesLogCollection.md)
  - ログ収集
- [kubernetesPVC.md](keruta/kubernetes/kubernetesPVC.md)
  - 永続ボリューム(PVC)

#### misc
- [kotlinExamples.md](keruta/misc/kotlinExamples.md)
  - Kotlin実装例
- [monitoring.md](keruta/misc/monitoring.md)
  - モニタリング

#### setup
- [installScriptSpecification.md](keruta/setup/installScriptSpecification.md)
  - インストールスクリプト仕様
- [setupScript.md](keruta/setup/setupScript.md)
  - セットアップスクリプト

#### system
- [summaryAndLinks.md](keruta/system/summaryAndLinks.md)
  - 概要と関連リンク
- [projectDetails.md](keruta/system/projectDetails.md)
  - プロジェクト詳細
- [systemOverview.md](keruta/system/systemOverview.md)
  - システム概要
- [dataModel.md](keruta/system/dataModel.md)
  - データモデル
- [taskApi.md](keruta/system/api/taskApi.md)
  - タスク管理API
- [agentApi.md](keruta/system/api/agentApi.md)
  - エージェント管理API
- [repositoryApi.md](keruta/system/api/repositoryApi.md)
  - リポジトリ管理API
- [documentApi.md](keruta/system/api/documentApi.md)
  - ドキュメント管理API

#### task
- [taskQueueSystemDesign.md](keruta/task/taskQueueSystemDesign.md)
  - タスクキューシステム設計
- [autoFixTask.md](keruta/task/autoFixTask.md)
  - 自動修正タスク

#### common
- [apiSpec.md](common/apiSpec.md)
  - 共通API仕様・keruta API仕様

### keruta-agent

- [README.md](keruta-agent/README.md)
  - keruta-agentの概要
- [commandReference.md](keruta-agent/commandReference.md)
  - コマンドリファレンス
- [implementation.md](keruta-agent/implementation.md)
  - 実装例

### keruta-github

- [README.md](keruta-github/README.md)
  - keruta-githubの概要

---

各ドキュメントの詳細はリンク先を参照してください。
