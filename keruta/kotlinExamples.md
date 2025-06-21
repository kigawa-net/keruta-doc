# Kotlin実装例

[目次に戻る](./taskQueueSystemDesign.md)

- [システム概要・アーキテクチャ](./systemOverview.md)
- [データモデル](./dataModel.md)
- [API仕様](./apiSpec.md)
- [Kubernetes構成・デプロイ手順](./kubernetesAndDeploy.md)
- [エラー発生時の自動修正タスク追加仕様](./autoFixTask.md)
- [監視・ヘルスチェック](./monitoring.md)
- [まとめ・関連リンク](./summaryAndLinks.md)

---

## タスクサービス
```kotlin
@Service
class TaskService(
    private val taskRepository: TaskRepository
) {
    fun createTask(request: CreateTaskRequest): Task {
        val task = Task(
            title = request.title,
            description = request.description,
            priority = request.priority,
            language = request.language
        )
        return taskRepository.save(task)
    }

    fun getTasksByStatus(status: TaskStatus): List<Task> {
        return taskRepository.findByStatusOrderByCreatedAtDesc(status)
    }

    fun getNextQueuedTask(): Task? {
        return taskRepository.findFirstByStatusOrderByPriorityDescCreatedAtAsc(TaskStatus.QUEUED)
    }

    fun updateTaskStatus(taskId: String, status: TaskStatus): Task {
        val task = taskRepository.findById(taskId)
            .orElseThrow { TaskNotFoundException(taskId) }
        val updatedTask = task.copy(
            status = status,
            updatedAt = LocalDateTime.now()
        )
        return taskRepository.save(updatedTask)
    }
}
```

## ワーカーサービス
```kotlin
@Component
class TaskWorker(
    private val taskService: TaskService,
    private val codingAgentService: CodingAgentService
) {
    @Scheduled(fixedDelay = 5000) // 5秒間隔
    fun processNextTask() {
        val task = taskService.getNextQueuedTask() ?: return
        try {
            taskService.updateTaskStatus(task.id, TaskStatus.PROCESSING)
            val result = codingAgentService.executeTask(task)
            taskService.completeTask(task.id, result)
        } catch (e: Exception) {
            taskService.updateTaskStatus(task.id, TaskStatus.FAILED)
            // ... 修正タスク自動追加処理 ...
        }
    }
}
```

## コーディングエージェントサービス
```kotlin
@Service
class CodingAgentService {

    suspend fun executeTask(task: Task): TaskResult = withContext(Dispatchers.IO) {
        logger.info("Processing task: ${task.title}")

        // AIエージェントAPI呼び出し（簡略化）
        val prompt = buildPrompt(task)
        val generatedCode = callAiService(prompt)

        // 基本的な検証
        val isValid = validateCode(generatedCode, task.language)

        TaskResult(
            code = generatedCode,
            isValid = isValid,
            generatedFiles = extractFiles(generatedCode)
        )
    }

    private fun buildPrompt(task: Task): String {
        return """
            言語: ${task.language}
            タスク: ${task.title}
            説明: ${task.description}

            上記の要件に基づいてコードを生成してください。
        """.trimIndent()
    }

    private suspend fun callAiService(prompt: String): String {
        // 実際のAI APIコール
        return "// 生成されたコード\n// TODO: 実装"
    }
}
``` 