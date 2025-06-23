# エラー発生時の自動修正タスク追加仕様

[目次に戻る](./taskQueueSystemDesign.md)

- [システム概要・アーキテクチャ](./systemOverview.md)
- [データモデル](./dataModel.md)
- [API仕様](./apiSpec.md)
- [Kotlin実装例](./kotlinExamples.md)
- [Kubernetes構成・デプロイ手順](./kubernetesAndDeploy.md)
- [監視・ヘルスチェック](./monitoring.md)
- [まとめ・関連リンク](./summaryAndLinks.md)

---

## 概要
タスク実行時にエラーが発生した場合、システムは自動的に「修正タスク」を生成し、タスクキューに追加します。これにより、エラー発生時の修正対応を自動化し、タスクの継続的な改善を促進します。

## 仕様詳細
- タスクワーカーがタスク実行中に例外やエラーを検知した場合、元タスクの内容とエラー内容を元に新たな修正タスクを作成します。
- 修正タスクの`title`は「【自動修正】元タスクタイトル」とし、`description`には元タスクの説明とエラー内容を追記します。
- 修正タスクの`priority`や`language`は元タスクを引き継ぎます。
- 修正タスクは通常のタスクと同様にキューに追加され、次回以降のワーカー処理で実行されます。
- **無限ループ防止のため、同一タスクから自動生成される修正タスクの回数には上限（例：3回）を設けます。上限に達した場合は修正タスクを追加しません。修正タスクには元タスクIDと自動生成回数をメタ情報として保持します。**

## フロー図
```
[タスク実行] → [成功] → [COMPLETED]
                ↓
              [失敗]
                ↓
      [FAILEDに更新] → [修正タスク自動生成（上限未満なら）] → [キューに追加]
                ↓
      [上限到達時は修正タスク追加せず終了]
```

## 実装例（Kotlin擬似コード）
```kotlin
catch (e: Exception) {
    taskService.updateTaskStatus(taskId, TaskStatus.FAILED)
    logger.error("Task processing failed: $taskId", e)

    val originalTask = taskService.getTaskById(taskId)
    val fixCount = originalTask.fixCount ?: 0
    val maxFixCount = 3
    if (fixCount < maxFixCount) {
        val fixTaskRequest = CreateTaskRequest(
            title = "【自動修正】${originalTask.title}",
            description = "${originalTask.description}\n---\nエラー内容: ${e.message}",
            priority = originalTask.priority,
            language = originalTask.language,
            parentTaskId = originalTask.id,
            fixCount = fixCount + 1
        )
        taskService.createTask(fixTaskRequest)
    } else {
        logger.warn("修正タスク自動生成の上限に到達: $taskId")
    }
}
```

## 備考
- 修正タスクには元タスクIDやエラー発生時刻などのメタ情報を付与することで、追跡性を高めることができます。 