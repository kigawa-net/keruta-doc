# システム概要・アーキテクチャ

[目次に戻る](./taskQueueSystemDesign.md)

- [データモデル](./dataModel.md)
- [API仕様](./apiSpec.md)
- [Kotlin実装例](./kotlinExamples.md)
- [Kubernetes構成・デプロイ手順](./kubernetesAndDeploy.md)
- [エラー発生時の自動修正タスク追加仕様](./autoFixTask.md)
- [監視・ヘルスチェック](./monitoring.md)
- [まとめ・関連リンク](./summaryAndLinks.md)

---

## システム概要
小規模チーム向けの軽量なコーディングエージェントタスクキューシステム。KotlinとKubernetesを活用し、タスク管理・自動化を実現します。

## アーキテクチャ
- Web UI, API Service(Spring Boot), MongoDB, Agent/Worker Pods
- シンプルなREST APIとワーカーによる自動処理

### 全体構成（シンプル版）
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

### コンポーネント

#### API Service (Kotlin)
- Spring Boot による軽量REST API
- タスクの受付・状態管理
- シンプルな認証機能

#### Worker Service (Kotlin)
- タスク処理ワーカー
- 基本的なエージェント実行機能
- MongoDBからのタスク取得・状態更新 