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

@Document(collection = "tasks")
data class Task(
    @Id
    val id: String = "",
    val sessionId: String, // セッションID
    val parentTaskId: String? = null, // 親タスクID
    val name: String, // タスク名
    val description: String = "", // タスク説明
    val script: String = "", // 実行スクリプト
    val status: TaskStatus = TaskStatus.PENDING,
    val message: String = "", // ステータスメッセージ
    val progress: Int = 0, // 進捗（0-100）
    val errorCode: String = "", // エラーコード
    val parameters: Map<String, Any> = emptyMap(), // パラメータ
    val createdAt: LocalDateTime = LocalDateTime.now(),
    val updatedAt: LocalDateTime = LocalDateTime.now(),
)

enum class TaskStatus {
    PENDING,
    IN_PROGRESS,
    COMPLETED,
    FAILED,
    WAITING_FOR_INPUT, // 入力待ち状態
    RETRYING, // リトライ中
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