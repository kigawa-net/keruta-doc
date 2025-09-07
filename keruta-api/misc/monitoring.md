# 監視・ヘルスチェック

[目次に戻る](./taskQueueSystemDesign.md)

- [システム概要・アーキテクチャ](./systemOverview.md)
- [データモデル](./dataModel.md)
- [API仕様](./apiSpec.md)
- [Kotlin実装例](./kotlinExamples.md)
- [Kubernetes構成・デプロイ手順](./kubernetesAndDeploy.md)
- [エラー発生時の自動修正タスク追加仕様](./autoFixTask.md)
- [まとめ・関連リンク](./summaryAndLinks.md)

---

## ヘルスチェックAPI
```kotlin
@RestController
class HealthController {

    @GetMapping("/health")
    fun health(): Map<String, String> {
        return mapOf(
            "status" to "UP",
            "timestamp" to LocalDateTime.now().toString()
        )
    }

    @GetMapping("/metrics")
    fun metrics(): Map<String, Any> {
        return mapOf(
            "active_tasks" to getActiveTaskCount(),
            "queue_length" to getQueueLength()
        )
    }
}
``` 