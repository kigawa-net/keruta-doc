# keruta ドキュメント目次

本ディレクトリには、kerutaシステムに関する各種技術・運用ドキュメントが含まれています。以下にカテゴリごとに概要とリンクをまとめます。

## 1. システム全体・管理パネル
- [projectDetails.md](./projectDetails.md)  
  kerutaシステム全体の詳細・API仕様・セットアップ方法などを記載。
- [adminPanel.md](./adminPanel.md)  
  管理パネルの機能・画面・操作方法・権限管理について解説。
- [adminPanelScriptGenerator.md](./adminPanelScriptGenerator.md)  
  管理パネルからインストールスクリプトを視覚的に作成・編集できる機能の詳細。

## 2. 認証・認可
- [auth/keycloakIntegration.md](./auth/keycloakIntegration.md)  
  Keycloakによる認証・認可の設定方法、セキュリティのポイント、トラブルシューティング。

## 3. Kubernetes関連
- [kubernetes/kubernetesIntegration.md](./kubernetes/kubernetesIntegration.md)  
  タスク情報を環境変数としてKubernetes Jobに渡す仕組みや設定方法。
- [kubernetes/kubernetesJobSpec.md](./kubernetes/kubernetesJobSpec.md)  
  Job/Podの基本仕様、起動コマンド、リソース制限、セキュリティ設定など。
- [kubernetes/kubernetesInitContainer.md](./kubernetes/kubernetesInitContainer.md)  
  init containerによるリポジトリクローンの仕組みとサンプルYAML。
- [kubernetes/kubernetesPVC.md](./kubernetes/kubernetesPVC.md)  
  PVCを用いたgitリポジトリの永続化・共有方法、Kotlinによるマニフェスト生成例。
- [kubernetes/kubernetesLogCollection.md](./kubernetes/kubernetesLogCollection.md)  
  Podログの自動収集・参照方法、管理パネルやAPIでの閲覧方法。

## 4. システム設計・タスクキュー
- [taskQueueSystemDesign.md](./taskQueueSystemDesign.md)  
  コーディングエージェントタスクキューシステムの設計思想、API、Kotlin実装例。

## 5. セットアップ・スクリプト
- [setupScript.md](./setupScript.md)  
  リポジトリクローン後の自動実行スクリプト機能の仕様と使用方法。

---

各ドキュメントの詳細はリンク先をご参照ください。
