# keruta-agent ログストリーミング仕様

> **概要**: keruta-agentからkeruta本体、データベース、クライアントへスクリプトの実行ログをリアルタイムでストリーミングする機能の仕様書です。

## 概要

keruta-agentは、Kubernetes Job内で実行されるスクリプトの標準出力・標準エラー出力をリアルタイムでキャプチャし、以下の3つの宛先にストリーミングします：

1. **keruta本体**: WebSocket経由でリアルタイム送信
2. **データベース**: 構造化ログとして永続化
3. **クライアント**: 管理パネルでのリアルタイム表示

## アーキテクチャ

```
┌─────────────────────────────────────────────────────────┐
│                Kubernetes Pod                           │
│                                                         │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              keruta-agent                          │ │
│  │                                                     │ │
│  │  ┌─────────────────┐    ┌─────────────────────────┐ │ │
│  │  │   User Script   │    │   Log Streaming        │ │ │
│  │  │   (Subprocess)  │◄───│   Engine               │ │ │
│  │  │                 │    │                         │ │ │
│  │  │ #!/bin/bash     │    │  • リアルタイムキャプチャ│ │ │
│  │  │ echo "hello"    │    │  • WebSocket送信       │ │ │
│  │  │ exit 0          │    │  • DB保存              │ │ │
│  │  └─────────────────┘    └─────────────────────────┘ │ │
│  │                                                     │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                         │
└─────────────────────────────────────────────────────────┘
                                │
                                ▼
                    ┌──────────────────────────────┐
                    │     keruta API (WebSocket)   │
                    │   (Spring Boot)              │
                    │                              │
                    │  • リアルタイムログ受信      │ │
                    │  • データベース保存          │ │
                    │  • クライアント配信          │ │
                    └──────────────────────────────┘
                                │
                                ▼
                    ┌──────────────────────────────┐
                    │         MongoDB              │
                    │     (永続化ストレージ)        │ │
                    └──────────────────────────────┘
                                │
                                ▼
                    ┌──────────────────────────────┐
                    │      管理パネル              │ │
                    │   (リアルタイム表示)          │ │
                    └──────────────────────────────┘
```

## 実装仕様

### 1. ログキャプチャ

#### サブプロセス出力のリアルタイムキャプチャ
```go
type LogStreamer struct {
    taskID    string
    apiClient *APIClient
    wsClient  *WebSocketClient
}

func (ls *LogStreamer) CaptureAndStream(cmd *exec.Cmd) error {
    // 標準出力のキャプチャ
    stdout, err := cmd.StdoutPipe()
    if err != nil {
        return err
    }
    
    // 標準エラー出力のキャプチャ
    stderr, err := cmd.StderrPipe()
    if err != nil {
        return err
    }
    
    // 並行してログをストリーミング
    go ls.streamOutput(stdout, "stdout")
    go ls.streamOutput(stderr, "stderr")
    
    return nil
}

func (ls *LogStreamer) streamOutput(pipe io.ReadCloser, source string) {
    scanner := bufio.NewScanner(pipe)
    for scanner.Scan() {
        line := scanner.Text()
        
        // 1. WebSocketでリアルタイム送信
        ls.sendWebSocketLog(line, source)
        
        // 2. データベースに保存
        ls.saveToDatabase(line, source)
        
        // 3. ローカルログ出力
        log.Printf("[%s] %s", source, line)
    }
}
```

### 2. WebSocket通信

#### WebSocketメッセージ形式
```json
{
  "type": "log",
  "taskId": "task-123",
  "timestamp": "2024-01-01T12:00:00Z",
  "source": "stdout|stderr|agent",
  "level": "INFO|ERROR|DEBUG|WARN",
  "message": "ログメッセージ",
  "lineNumber": 42,
  "metadata": {
    "processId": 12345,
    "scriptPath": "/work/script.sh"
  }
}
```

#### WebSocket接続・送信
```go
func (ls *LogStreamer) sendWebSocketLog(message, source string) {
    logMessage := LogMessage{
        Type:      "log",
        TaskID:    ls.taskID,
        Timestamp: time.Now().Format(time.RFC3339),
        Source:    source,
        Level:     determineLogLevel(message),
        Message:   message,
    }
    
    if err := ls.wsClient.Send(logMessage); err != nil {
        log.Printf("WebSocket送信エラー: %v", err)
    }
}
```

### 3. データベース保存

#### ログエンティティ
```kotlin
@Entity
data class TaskLog(
    @Id
    val id: String = UUID.randomUUID().toString(),
    
    @Indexed
    val taskId: String,
    
    val timestamp: LocalDateTime,
    val source: String, // "stdout", "stderr", "agent"
    val level: String, // "INFO", "ERROR", "DEBUG", "WARN"
    val message: String,
    val lineNumber: Int? = null,
    val metadata: Map<String, Any> = emptyMap()
)
```

#### APIエンドポイント
```kotlin
@RestController
@RequestMapping("/api/v1/tasks")
class TaskLogController {
    
    @PostMapping("/{taskId}/logs")
    fun addLog(
        @PathVariable taskId: String,
        @RequestBody logRequest: LogRequest
    ): ResponseEntity<LogResponse> {
        val log = TaskLog(
            taskId = taskId,
            timestamp = LocalDateTime.now(),
            source = logRequest.source,
            level = logRequest.level,
            message = logRequest.message,
            metadata = logRequest.metadata
        )
        
        taskLogService.save(log)
        
        // WebSocketでクライアントに配信
        webSocketService.broadcastToTask(taskId, log)
        
        return ResponseEntity.ok(LogResponse(log.id))
    }
    
    @GetMapping("/{taskId}/logs")
    fun getLogs(
        @PathVariable taskId: String,
        @RequestParam(required = false) source: String?,
        @RequestParam(required = false) level: String?,
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "100") size: Int
    ): ResponseEntity<Page<TaskLog>> {
        val logs = taskLogService.findByTaskId(taskId, source, level, page, size)
        return ResponseEntity.ok(logs)
    }
}
```

### 4. クライアント配信

#### WebSocketハンドラー
```kotlin
@Component
class WebSocketHandler {
    
    private val sessions = ConcurrentHashMap<String, WebSocketSession>()
    
    fun handleMessage(session: WebSocketSession, message: String) {
        val logMessage = objectMapper.readValue(message, LogMessage::class.java)
        
        // タスクに関連するセッションに配信
        broadcastToTask(logMessage.taskId, logMessage)
    }
    
    fun broadcastToTask(taskId: String, message: Any) {
        val taskSessions = sessions.values.filter { session ->
            session.attributes["taskId"] == taskId
        }
        
        taskSessions.forEach { session ->
            try {
                session.send(TextMessage(objectMapper.writeValueAsString(message)))
            } catch (e: Exception) {
                log.error("WebSocket送信エラー", e)
            }
        }
    }
}
```

## 使用方法

### keruta-agentでの実行
```bash
# 基本的な実行（ログストリーミング自動有効）
keruta-agent execute --task-id task123

# ログレベル指定
keruta-agent execute --task-id task123 --log-level DEBUG

# ログストリーミング無効化
keruta-agent execute --task-id task123 --disable-log-streaming
```

### 管理パネルでの表示
```javascript
// WebSocket接続
const ws = new WebSocket(`ws://keruta-api:8080/ws/tasks/${taskId}`);

ws.onmessage = function(event) {
    const logMessage = JSON.parse(event.data);
    
    if (logMessage.type === 'log') {
        // ログをリアルタイム表示
        appendLog(logMessage);
    }
};

function appendLog(logMessage) {
    const logContainer = document.getElementById('log-container');
    const logElement = document.createElement('div');
    logElement.className = `log-entry log-${logMessage.level.toLowerCase()}`;
    logElement.innerHTML = `
        <span class="timestamp">${logMessage.timestamp}</span>
        <span class="source">[${logMessage.source}]</span>
        <span class="message">${logMessage.message}</span>
    `;
    logContainer.appendChild(logElement);
    logContainer.scrollTop = logContainer.scrollHeight;
}
```

## 設定

### 環境変数
| 変数名 | 説明 | デフォルト値 |
|--------|------|-------------|
| `KERUTA_LOG_STREAMING_ENABLED` | ログストリーミング有効化 | `true` |
| `KERUTA_LOG_BUFFER_SIZE` | ログバッファサイズ | `1000` |
| `KERUTA_LOG_BATCH_SIZE` | バッチ送信サイズ | `10` |
| `KERUTA_LOG_FLUSH_INTERVAL` | フラッシュ間隔（秒） | `1` |

### ログレベル
- `DEBUG`: デバッグ情報
- `INFO`: 一般情報
- `WARN`: 警告
- `ERROR`: エラー

## パフォーマンス考慮事項

### 1. バッファリング
- ログメッセージをバッファに蓄積
- バッチサイズまたは時間間隔で一括送信
- メモリ使用量の制御

### 2. レート制限
- WebSocket送信: 1000メッセージ/秒
- データベース保存: 1000レコード/秒
- クライアント配信: 500メッセージ/秒

### 3. エラーハンドリング
- WebSocket接続断時の再接続
- データベース接続エラー時のローカル保存
- クライアント接続断時のメッセージ破棄

## 関連ドキュメント
- [keruta-agent API仕様](./apiSpec.md)
- [keruta-agent コマンドリファレンス](./commandReference.md)
- [keruta-agent 実装例](./implementation.md)
- [Kubernetesログ収集仕様](../keruta/kubernetes/kubernetesLogCollection.md) 