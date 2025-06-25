# データモデル

> kerutaシステムの主要データモデル（Kotlin実装例）

## 目次
- [Taskモデル](#taskモデル)
- [Agentモデル](#agentモデル)
- [Repositoryモデル](#repositoryモデル)
- [関連ドキュメント](#関連ドキュメント)

---

## Taskモデル
```kotlin
import java.time.LocalDateTime

data class Task(
    val id: String? = null,
    val title: String = "a",
    val description: String? = null,
    val priority: Int = 0,
    val status: TaskStatus = TaskStatus.PENDING,
    val documents: List<Document> = emptyList(),
    val image: String? = null,
    val namespace: String = "default",
    val jobName: String? = null,
    val podName: String? = null, // 互換用
    val additionalEnv: Map<String, String> = emptyMap(),
    val kubernetesManifest: String? = null,
    val logs: String? = null,
    val agentId: String? = null,
    val repositoryId: String? = null, // init container用
    val parentId: String? = null, // 親子タスク用
    val createdAt: LocalDateTime = LocalDateTime.now(),
    val updatedAt: LocalDateTime = LocalDateTime.now()
)

enum class TaskStatus {
    PENDING,
    IN_PROGRESS,
    COMPLETED,
    CANCELLED,
    FAILED
}
```

## Agentモデル
```kotlin
import java.time.LocalDateTime

data class Agent(
    val id: String? = null,
    val name: String,
    val languages: List<String> = listOf(),
    val status: AgentStatus = AgentStatus.AVAILABLE,
    val currentTaskId: String? = null,
    val installCommand: String = "",
    val executeCommand: String = "",
    val createdAt: LocalDateTime = LocalDateTime.now(),
    val updatedAt: LocalDateTime = LocalDateTime.now()
)

enum class AgentStatus {
    AVAILABLE,
    BUSY,
    OFFLINE
}
```

## Repositoryモデル
```kotlin
import java.time.LocalDateTime

data class Repository(
    val id: String? = null,
    val name: String,
    val url: String,
    val description: String = "",
    val isValid: Boolean = false,
    val setupScript: String = "",
    val usePvc: Boolean = false,
    val pvcStorageSize: String = "1Gi",
    val pvcAccessMode: String = "ReadWriteOnce",
    val createdAt: LocalDateTime = LocalDateTime.now(),
    val updatedAt: LocalDateTime = LocalDateTime.now()
)
```

## 関連ドキュメント
- [システム概要・アーキテクチャ](./systemOverview.md)
- [API仕様詳細](./apiSpec.md)

[目次に戻る](./taskQueueSystemDesign.md)

- [システム概要・アーキテクチャ](./systemOverview.md)
- [API仕様](./apiSpec.md)
- [Kotlin実装例](./kotlinExamples.md)
- [Kubernetes構成・デプロイ手順](./kubernetesAndDeploy.md)
- [エラー発生時の自動修正タスク追加仕様](./autoFixTask.md)
- [監視・ヘルスチェック](./monitoring.md)
- [まとめ・関連リンク](./summaryAndLinks.md) 