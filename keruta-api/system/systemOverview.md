# システム概要・アーキテクチャ

> kerutaシステムの全体像・アーキテクチャ図・主要コンポーネントの概要をまとめます。

## 目次
- [システム概要](#システム概要)
- [アーキテクチャ図](#アーキテクチャ図)
- [主要コンポーネント](#主要コンポーネント)
- [関連ドキュメント](#関連ドキュメント)

---

## システム概要
kerutaは、小規模チーム向けの軽量なコーディングエージェントタスクキューシステムです。KotlinとKubernetesを活用し、タスク管理・自動化・Gitリポジトリ連携を実現します。

## アーキテクチャ図

```
           ┌─────────────────────────────────┐
           │      Kubernetes Cluster         │
           │                                 │
┌────────┐ │ ┌─────────────┐ ┌─────────────┐ │
│Web UI  │ │ │   Ingress   │ │    API      │ │
│        │◄┼─┤             │◄┤  Service    │ │
└────────┘ │ └─────────────┘ │  (Kotlin)   │ │
           │                 └─────────────┘ │
           │                        │        │
           │                 ┌─────────────┐ │
           │                 │   MongoDB   │ │
           │                 │ (Database)  │ │
           │                 └─────────────┘ │
           │                        │        │
           │ ┌─────────────┐ ┌─────────────┐ │
           │ │   Agent     │ │   Worker    │ │
           │ │   Pods      │ │   Pods      │ │
           │ └─────────────┘ └─────────────┘ │
           └─────────────────────────────────┘
```

## 主要コンポーネント

### API Service (Kotlin)
- Spring BootによるREST API
- タスク受付・状態管理
- Keycloak統合認証・認可

### Worker/Agent Pods
- タスク処理ワーカー
- MongoDBからのタスク取得・状態更新

### Web UI / 管理パネル
- タスク・ドキュメント・リポジトリ管理
- Keycloak SSO認証対応

### MongoDB
- タスク・ドキュメント・ユーザー等のデータ永続化

## 関連ドキュメント
- [システム詳細・セットアップ](./projectDetails.md)
- [API仕様詳細](./apiSpec.md)
- [データモデル定義](./dataModel.md)

[目次に戻る](./taskQueueSystemDesign.md)

- [データモデル](./dataModel.md)
- [API仕様](./apiSpec.md)
- [Kotlin実装例](./kotlinExamples.md)
- [Kubernetes構成・デプロイ手順](./kubernetesAndDeploy.md)
- [エラー発生時の自動修正タスク追加仕様](./autoFixTask.md)
- [監視・ヘルスチェック](./monitoring.md)
- [まとめ・関連リンク](./summaryAndLinks.md) 