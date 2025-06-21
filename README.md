# keruta-doc

本リポジトリは、kerutaプロジェクトの設計・運用・技術情報をまとめたドキュメント集です。

## サブプロジェクト
- `keruta`: keruta APIサーバーの設計・運用ドキュメント（`keruta/` ディレクトリに配置）
- `keruta-github`: kerutaをGitHubから操作するためのツール/サービス（`keruta-github/` ディレクトリに配置）
- `keruta-agent`: kerutaによって実行されるjobで利用できるkerutaコマンドを実装するサブプロジェクト（`keruta-agent/` ディレクトリに配置）

## kerutaドキュメント一覧

- [adminPanel.md](keruta/adminPanel.md)
  - 管理パネルの機能・画面・操作方法についてまとめたドキュメント。
- [keycloakIntegration.md](keruta/keycloakIntegration.md)
  - Keycloakによる認証・認可の設定手順とセキュリティ考慮事項。
- [kubernetesIntegration.md](keruta/kubernetesIntegration.md)
  - タスク情報をKubernetesのPod環境変数として渡す仕組みと設定方法。
- [projectDetails.md](keruta/projectDetails.md)
  - keruta APIサーバーの概要、セットアップ、機能、API仕様などの詳細。
- [taskQueueSystemDesign.md](keruta/taskQueueSystemDesign.md)
  - コーディングエージェントタスクキューシステムの設計書（Kotlin+Kubernetesベース）。

## keruta-agentドキュメント一覧

- [README.md](keruta-agent/README.md)
  - keruta-agentの概要、機能仕様、コマンド仕様、使用方法などの詳細。
- [commandReference.md](keruta-agent/commandReference.md)
  - keruta-agentで利用可能なすべてのコマンドとその使用方法をまとめたリファレンス。
- [apiSpec.md](keruta-agent/apiSpec.md)
  - keruta-agentがkeruta APIサーバーと通信する際のAPI仕様。
- [implementation.md](keruta-agent/implementation.md)
  - keruta-agentの実装例とサンプルコード。

---

各ドキュメントの詳細はリンク先を参照してください。
