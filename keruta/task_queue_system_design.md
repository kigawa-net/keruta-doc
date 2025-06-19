# コーディングエージェントタスクキューシステム設計書
## 軽量版 - Kotlin + Kubernetes

## 1. システム概要

### 1.1 目的
小規模チーム向けの軽量なコーディングエージェントタスクキューシステムです。Kotlinの簡潔性とKubernetesの基本機能を活用し、必要最小限の構成で開発タスクの自動化を実現します。

### 1.2 主要機能
- シンプルなタスク管理・実行
- 基本的な優先度制御
- 軽量なエージェント管理
- 基本的な監視・ログ

## 2. システムアーキテクチャ

### 2.1 全体構成（シンプル版）
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

### 2.2 コンポーネント

#### 2.2.1 API Service (Kotlin)
- Spring Boot による軽量REST API
- タスクの受付・状態管理
- シンプルな認証機能

#### 2.2.2 Worker Service (Kotlin)
- タスク処理ワーカー
- 基本的なエージェント実行機能
- MongoDBからのタスク取得・状態更新

## 3. データモデル（簡略版）

### 3.1 タスクモデル
```kotlin
@Entity
data class Task(
    @Id
    val id: String = UUID.randomUUID().toString(),
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

### 3.2 エージェントモデル
```kotlin
@Entity
data class Agent(
    @Id
    val id: String = UUID.randomUUID().toString(),
    val name: String,
    val languages: List<String>,
    val status: AgentStatus = AgentStatus.AVAILABLE,
    val currentTaskId: String? = null
)

enum class AgentStatus { AVAILABLE, BUSY, OFFLINE }
```

## 4. API仕様（基本版）

### 4.1 タスク管理API

#### タスク作成
```http
POST /api/tasks
Content-Type: application/json

{
  "title": "ログイン機能の実装",
  "description": "基本的な認証機能を実装してください",
  "priority": "HIGH",
  "language": "kotlin"
}
```

#### タスク一覧取得
```http
GET /api/tasks?status=QUEUED&limit=10

Response:
{
  "tasks": [
    {
      "id": "task-123",
      "title": "ログイン機能の実装",
      "status": "PROCESSING",
      "priority": "HIGH",
      "createdAt": "2025-06-18T10:00:00Z"
    }
  ]
}
```

#### タスク詳細取得
```http
GET /api/tasks/{taskId}

Response:
{
  "id": "task-123",
  "title": "ログイン機能の実装",
  "description": "基本的な認証機能を実装してください",
  "status": "COMPLETED",
  "result": {
    "code": "生成されたコード",
    "files": ["LoginController.kt", "User.kt"]
  }
}
```

## 5. Kotlin実装例

### 5.1 タスクサービス
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

### 5.2 ワーカーサービス
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

### 5.3 コーディングエージェントサービス
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

## 6. Kubernetes構成（シンプル版）

### 6.1 デプロイメント
```yaml
# API Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: task-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: task-api
  template:
    metadata:
      labels:
        app: task-api
    spec:
      containers:
      - name: api
        image: task-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "200m"

---
# Worker Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: task-worker
spec:
  replicas: 1
  selector:
    matchLabels:
      app: task-worker
  template:
    metadata:
      labels:
        app: task-worker
    spec:
      containers:
      - name: worker
        image: task-worker:latest
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        resources:
          requests:
            memory: "512Mi"
            cpu: "200m"
          limits:
            memory: "1Gi"
            cpu: "500m"

---
# Service
apiVersion: v1
kind: Service
metadata:
  name: task-api-service
spec:
  selector:
    app: task-api
  ports:
  - port: 80
    targetPort: 8080
```

### 6.2 データベース
```yaml
# MongoDB
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:latest
        env:
        - name: MONGO_INITDB_DATABASE
          value: "taskqueue"
        - name: MONGO_INITDB_ROOT_USERNAME
          value: "admin"
        - name: MONGO_INITDB_ROOT_PASSWORD
          value: "password"
        volumeMounts:
        - name: mongodb-storage
          mountPath: /data/db
      volumes:
      - name: mongodb-storage
        persistentVolumeClaim:
          claimName: mongodb-pvc

---
# Redis
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
```

## 7. 設定ファイル

### 7.1 application.yml
```yaml
spring:
  data:
    mongodb:
      host: mongodb
      port: 27017
      database: taskqueue
      username: admin
      password: password
      authentication-database: admin

server:
  port: 8080

logging:
  level:
    com.example.taskqueue: INFO
```

### 7.2 build.gradle.kts
```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-mongodb")
    implementation ("org.jetbrains.kotlin:kotlin-reflect")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core")
    implementation("org.mongodb:mongodb-driver-sync")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.testcontainers:mongodb")
}
```

## 8. 基本的な監視

### 8.1 ヘルスチェック
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

## 9. デプロイ手順

### 9.1 ビルド
```bash
# アプリケーションビルド
./gradlew build

# Dockerイメージ作成
docker build -t task-api:latest .
docker build -t task-worker:latest .
```

### 9.2 デプロイ
```bash
# Kubernetesデプロイ
kubectl apply -f kubernetes/
kubectl get pods
kubectl logs -f deployment/task-api
```

## 10. まとめ

この軽量版システムは以下の特徴を持ちます：

**シンプルな構成**
- 最小限のコンポーネント
- 基本的なKubernetes機能のみ使用
- 小規模チーム向けの機能セット

**技術スタック**
- Kotlin + Spring Boot
- MongoDB
- Kubernetes（基本機能のみ）
- Docker

**適用規模**
- 開発者数：3-10名
- 同時タスク数：10-50個
- 1日のタスク処理数：100-500個

必要に応じて機能拡張や性能向上を段階的に実装できる設計となっています。

## 11. エラー発生時の自動修正タスク追加仕様

### 11.1 概要
タスク実行時にエラーが発生した場合、システムは自動的に「修正タスク」を生成し、タスクキューに追加します。これにより、エラー発生時の修正対応を自動化し、タスクの継続的な改善を促進します。

### 11.2 仕様詳細
- タスクワーカーがタスク実行中に例外やエラーを検知した場合、元タスクの内容とエラー内容を元に新たな修正タスクを作成します。
- 修正タスクの`title`は「【自動修正】元タスクタイトル」とし、`description`には元タスクの説明とエラー内容を追記します。
- 修正タスクの`priority`や`language`は元タスクを引き継ぎます。
- 修正タスクは通常のタスクと同様にキューに追加され、次回以降のワーカー処理で実行されます。
- **無限ループ防止のため、同一タスクから自動生成される修正タスクの回数には上限（例：3回）を設けます。上限に達した場合は修正タスクを追加しません。修正タスクには元タスクIDと自動生成回数をメタ情報として保持します。**

### 11.3 フロー図
```
[タスク実行] → [成功] → [COMPLETED]
                ↓
              [失敗]
                ↓
      [FAILEDに更新] → [修正タスク自動生成（上限未満なら）] → [キューに追加]
                ↓
      [上限到達時は修正タスク追加せず終了]
```

### 11.4 実装例（Kotlin擬似コード）
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

### 11.5 備考
- 修正タスクには元タスクIDやエラー発生時刻などのメタ情報を付与することで、追跡性を高めることができます。
