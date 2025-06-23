# keruta ドキュメント目次

本ディレクトリには、kerutaシステムに関する各種技術・運用ドキュメントが含まれています。以下にカテゴリごとに概要とリンクをまとめます。

## 1. システム全体
- [system/projectDetails.md](./system/projectDetails.md)
  kerutaシステム全体の詳細・API仕様・セットアップ方法などを記載。
- [system/systemOverview.md](./system/systemOverview.md)
  システムの全体像を解説。
- [system/apiSpec.md](./system/apiSpec.md)
  API仕様詳細。
- [system/dataModel.md](./system/dataModel.md)
  データモデル定義。

## 2. 管理パネル
- [admin/adminPanel.md](./admin/adminPanel.md)
  管理パネルの機能・画面・操作方法・権限管理について解説。
- [admin/adminPanelScriptGenerator.md](./admin/adminPanelScriptGenerator.md)
  管理パネルからインストールスクリプトを視覚的に作成・編集できる機能の詳細。

## 3. 認証・認可
- [auth/keycloakIntegration.md](./auth/keycloakIntegration.md)
  Keycloakによる認証・認可の設定方法、セキュリティのポイント、トラブルシューティング。

## 4. Kubernetes関連
- [kubernetes/kubernetesIntegration.md](./kubernetes/kubernetesIntegration.md)
  タスク情報を環境変数としてKubernetes Jobに渡す仕組みや設定方法。
- [kubernetes/kubernetesJobSpec.md](./kubernetes/kubernetesJobSpec.md)
  Job/Podの基本仕様、起動コマンド、リソース制限、セキュリティ設定など。
- [kubernetes/kubernetesInitContainer.md](./kubernetes/kubernetesInitContainer.md)
  init containerによるリポジトリクローンの仕組みとサンプルYAML。
- [kubernetes/gitCloneInitContainer.md](./kubernetes/gitCloneInitContainer.md)
  init containerによるリポジトリクローンの仕組みとサンプルYAML。
- [kubernetes/kubernetesPVC.md](./kubernetes/kubernetesPVC.md)
  PVCを用いたgitリポジトリの永続化・共有方法、Kotlinによるマニフェスト生成例。
- [kubernetes/kubernetesLogCollection.md](./kubernetes/kubernetesLogCollection.md)
  Podログの自動収集・参照方法、管理パネルやAPIでの閲覧方法。
- [kubernetes/kubernetesAndDeploy.md](./kubernetes/kubernetesAndDeploy.md)
  Kubernetesへのデプロイ関連ドキュメント。

## 5. タスクシステム
- [task/taskQueueSystemDesign.md](./task/taskQueueSystemDesign.md)
  コーディングエージェントタスクキューシステムの設計思想、API、Kotlin実装例。
- [task/autoFixTask.md](./task/autoFixTask.md)
  自動修正タスクに関するドキュメント。

## 6. セットアップ
- [setup/setupScript.md](./setup/setupScript.md)
  リポジトリクローン後の自動実行スクリプト機能の仕様と使用方法。
- [setup/installScriptSpecification.md](./setup/installScriptSpecification.md)
  インストールスクリプトの仕様。

## 7. Git関連
- [git/repositoryManagement.md](./git/repositoryManagement.md)
  Gitリポジトリの管理方法。
- [git/gitExcludeSpec.md](./git/gitExcludeSpec.md)
  `.sourceignorer`によるファイル除外設定。

## 8. その他
- [misc/kotlinExamples.md](./misc/kotlinExamples.md)
  Kotlinのサンプルコード集。
- [misc/monitoring.md](./misc/monitoring.md)
  システムの監視について。
- [misc/commonDocuments/README.md](./misc/commonDocuments/README.md)
  共通ドキュメント。

---

各ドキュメントの詳細はリンク先をご参照ください。
