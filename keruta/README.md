# keruta ドキュメント目次

本ディレクトリには、kerutaシステムに関する技術・運用ドキュメントが含まれています。カテゴリごとに概要とリンクをまとめます。

## 1. システム概要・全体像
- [system/projectDetails.md](./system/projectDetails.md)  
  システム全体の詳細・API仕様・セットアップ方法
- [system/systemOverview.md](./system/systemOverview.md)  
  システムの全体像
- [system/apiSpec.md](./system/apiSpec.md)  
  API仕様詳細
- [system/dataModel.md](./system/dataModel.md)  
  データモデル定義

## 2. セットアップ・インストール
- [setup/setupScript.md](./setup/setupScript.md)  
  初期セットアップ・自動実行スクリプト
- [setup/installScriptSpecification.md](./setup/installScriptSpecification.md)  
  インストールスクリプト仕様

## 3. 認証・認可
- [auth/keycloakIntegration.md](./auth/keycloakIntegration.md)  
  Keycloakによる認証・認可設定

## 4. 管理パネル
- [admin/adminPanel.md](./admin/adminPanel.md)  
  管理パネルの機能・画面・操作方法
- [admin/adminPanelScriptGenerator.md](./admin/adminPanelScriptGenerator.md)  
  インストールスクリプト作成機能

## 5. タスク・ジョブ管理
- [task/taskQueueSystemDesign.md](./task/taskQueueSystemDesign.md)  
  タスクキューシステム設計・API
- [task/autoFixTask.md](./task/autoFixTask.md)  
  自動修正タスク

## 6. Kubernetes連携
- [kubernetes/kubernetesIntegration.md](./kubernetes/kubernetesIntegration.md)  
  タスク情報のKubernetes連携
- [kubernetes/kubernetesJobSpec.md](./kubernetes/kubernetesJobSpec.md)  
  Job/Pod仕様・リソース制限
- [kubernetes/kubernetesInitContainer.md](./kubernetes/kubernetesInitContainer.md)  
  init containerによるリポジトリクローン
- [kubernetes/gitCloneInitContainer.md](./kubernetes/gitCloneInitContainer.md)  
  リポジトリクローンinit container
- [kubernetes/kubernetesPVC.md](./kubernetes/kubernetesPVC.md)  
  PVCによる永続化・共有
- [kubernetes/kubernetesLogCollection.md](./kubernetes/kubernetesLogCollection.md)  
  Podログの自動収集・参照
- [kubernetes/kubernetesAndDeploy.md](./kubernetes/kubernetesAndDeploy.md)  
  Kubernetesへのデプロイ

## 7. Gitリポジトリ管理
- [git/repositoryManagement.md](./git/repositoryManagement.md)  
  Gitリポジトリ管理方法
- [git/gitExcludeSpec.md](./git/gitExcludeSpec.md)  
  `.sourceignorer`による除外設定

## 8. 監視・運用
- [misc/monitoring.md](./misc/monitoring.md)  
  システム監視・運用

## 9. サンプル・共通ドキュメント
- [misc/kotlinExamples.md](./misc/kotlinExamples.md)  
  Kotlinサンプルコード
- [misc/commonDocuments/README.md](./misc/commonDocuments/README.md)  
  共通ドキュメント

---

各ドキュメントの詳細はリンク先をご参照ください。
