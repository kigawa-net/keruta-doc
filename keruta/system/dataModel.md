# データモデル

[目次に戻る](./taskQueueSystemDesign.md)

- [システム概要・アーキテクチャ](./systemOverview.md)
- [API仕様](./apiSpec.md)
- [Kotlin実装例](./kotlinExamples.md)
- [Kubernetes構成・デプロイ手順](./kubernetesAndDeploy.md)
- [エラー発生時の自動修正タスク追加仕様](./autoFixTask.md)
- [監視・ヘルスチェック](./monitoring.md)
- [まとめ・関連リンク](./summaryAndLinks.md)

---

## タスクモデル
```kotlin
@Entity
data class Task(
    @Id val id: String = UUID.randomUUID().toString(),
    val title: String,
    val description: String,
    val priority: Priority = Priority.MEDIUM,
    val language: String,
    val status: TaskStatus = TaskStatus.QUEUED,
    val createdAt: LocalDateTime = LocalDateTime.now(),
    val updatedAt: LocalDateTime = LocalDateTime.now()
)

enum class Priority { HIGH, MEDIUM, LOW }
enum class TaskStatus { QUEUED, PROCESSING, COMPLETED, FAILED }
```

## エージェントモデル
```kotlin
@Entity
data class Agent(
    @Id val id: String = UUID.randomUUID().toString(),
    val name: String,
    val languages: List<String>,
    val status: AgentStatus = AgentStatus.AVAILABLE,
    val currentTaskId: String? = null
)

enum class AgentStatus { AVAILABLE, BUSY, OFFLINE }
```

## リポジトリモデル
```kotlin
@Entity
data class Repository(
    @Id val id: String = UUID.randomUUID().toString(),
    val name: String,
    val url: String, // e.g., https://github.com/user/repo.git
    val installScript: String? = null, // インストールスクリプト
    val createdAt: LocalDateTime = LocalDateTime.now(),
    val updatedAt: LocalDateTime = LocalDateTime.now()
)
``` 