# ログストリーミング 実装仕様

> **概要**: ログストリーミング機能の技術的実装詳細とコード例です。

## 概要

このドキュメントでは、ログストリーミング機能の実装における技術的詳細、コード例、設計パターンを説明します。

## keruta-agent 実装

### 1. ログキャプチャエンジン

#### LogStreamer構造体
```go
type LogStreamer struct {
    taskID    string
    apiClient *APIClient
    wsClient  *WebSocketClient
    buffer    *LogBuffer
    config    *LogConfig
}

type LogConfig struct {
    BufferSize     int           `json:"bufferSize"`
    FlushInterval  time.Duration `json:"flushInterval"`
    BatchSize      int           `json:"batchSize"`
    MaxRetries     int           `json:"maxRetries"`
    RetryInterval  time.Duration `json:"retryInterval"`
}
```

#### サブプロセス出力のリアルタイムキャプチャ
```go
func (ls *LogStreamer) CaptureAndStream(cmd *exec.Cmd) error {
    // 標準出力のキャプチャ
    stdout, err := cmd.StdoutPipe()
    if err != nil {
        return fmt.Errorf("stdout pipe error: %w", err)
    }
    
    // 標準エラー出力のキャプチャ
    stderr, err := cmd.StderrPipe()
    if err != nil {
        return fmt.Errorf("stderr pipe error: %w", err)
    }
    
    // 並行してログをストリーミング
    go ls.streamOutput(stdout, "stdout")
    go ls.streamOutput(stderr, "stderr")
    
    return nil
}

func (ls *LogStreamer) streamOutput(pipe io.ReadCloser, source string) {
    defer pipe.Close()
    
    scanner := bufio.NewScanner(pipe)
    lineNumber := 0
    
    for scanner.Scan() {
        lineNumber++
        line := scanner.Text()
        
        logMessage := &LogMessage{
            TaskID:    ls.taskID,
            Timestamp: time.Now().Format(time.RFC3339),
            Source:    source,
            Level:     ls.determineLogLevel(line),
            Message:   line,
            LineNumber: lineNumber,
            Metadata: map[string]interface{}{
                "processId": os.Getpid(),
                "source":    source,
            },
        }
        
        // 1. WebSocketでリアルタイム送信
        ls.sendWebSocketLog(logMessage)
        
        // 2. バッファに追加（データベース保存用）
        ls.buffer.Add(logMessage)
        
        // 3. ローカルログ出力
        log.Printf("[%s] %s", source, line)
    }
    
    if err := scanner.Err(); err != nil {
        log.Printf("Scanner error for %s: %v", source, err)
    }
}
```

### 2. WebSocket通信

#### WebSocketクライアント
```go
type WebSocketClient struct {
    conn     *websocket.Conn
    url      string
    taskID   string
    sendChan chan *LogMessage
    done     chan struct{}
}

func NewWebSocketClient(url, taskID string) (*WebSocketClient, error) {
    ws := &WebSocketClient{
        url:      url,
        taskID:   taskID,
        sendChan: make(chan *LogMessage, 1000),
        done:     make(chan struct{}),
    }
    
    if err := ws.connect(); err != nil {
        return nil, err
    }
    
    go ws.sendLoop()
    go ws.pingLoop()
    
    return ws, nil
}

func (ws *WebSocketClient) connect() error {
    dialer := websocket.Dialer{
        HandshakeTimeout: 10 * time.Second,
    }
    
    conn, _, err := dialer.Dial(ws.url, nil)
    if err != nil {
        return fmt.Errorf("websocket connection failed: %w", err)
    }
    
    ws.conn = conn
    return nil
}

func (ws *WebSocketClient) sendLoop() {
    for {
        select {
        case msg := <-ws.sendChan:
            if err := ws.sendMessage(msg); err != nil {
                log.Printf("WebSocket send error: %v", err)
                ws.reconnect()
            }
        case <-ws.done:
            return
        }
    }
}

func (ws *WebSocketClient) sendMessage(msg *LogMessage) error {
    data, err := json.Marshal(msg)
    if err != nil {
        return fmt.Errorf("message marshal error: %w", err)
    }
    
    return ws.conn.WriteMessage(websocket.TextMessage, data)
}

func (ws *WebSocketClient) reconnect() {
    for {
        log.Println("Attempting WebSocket reconnection...")
        if err := ws.connect(); err == nil {
            log.Println("WebSocket reconnected successfully")
            return
        }
        time.Sleep(5 * time.Second)
    }
}
```

### 3. ログバッファリング

#### LogBuffer実装
```go
type LogBuffer struct {
    logs     []*LogMessage
    mutex    sync.RWMutex
    maxSize  int
    flushCh  chan struct{}
}

func NewLogBuffer(maxSize int) *LogBuffer {
    buffer := &LogBuffer{
        logs:    make([]*LogMessage, 0, maxSize),
        maxSize: maxSize,
        flushCh: make(chan struct{}, 1),
    }
    
    go buffer.flushLoop()
    return buffer
}

func (b *LogBuffer) Add(log *LogMessage) {
    b.mutex.Lock()
    defer b.mutex.Unlock()
    
    b.logs = append(b.logs, log)
    
    if len(b.logs) >= b.maxSize {
        select {
        case b.flushCh <- struct{}{}:
        default:
        }
    }
}

func (b *LogBuffer) flushLoop() {
    ticker := time.NewTicker(1 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            b.flush()
        case <-b.flushCh:
            b.flush()
        }
    }
}

func (b *LogBuffer) flush() {
    b.mutex.Lock()
    if len(b.logs) == 0 {
        b.mutex.Unlock()
        return
    }
    
    logs := make([]*LogMessage, len(b.logs))
    copy(logs, b.logs)
    b.logs = b.logs[:0]
    b.mutex.Unlock()
    
    // データベースにバッチ保存
    if err := b.saveToDatabase(logs); err != nil {
        log.Printf("Database save error: %v", err)
    }
}
```

## keruta API 実装

### 1. Spring Boot コントローラー

#### TaskLogController
```kotlin
@RestController
@RequestMapping("/api/v1/tasks")
class TaskLogController(
    private val taskLogService: TaskLogService,
    private val webSocketService: WebSocketService
) {
    
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
            lineNumber = logRequest.lineNumber,
            metadata = logRequest.metadata
        )
        
        val savedLog = taskLogService.save(log)
        
        // WebSocketでクライアントに配信
        webSocketService.broadcastToTask(taskId, savedLog)
        
        return ResponseEntity.ok(LogResponse(savedLog.id))
    }
    
    @GetMapping("/{taskId}/logs")
    fun getLogs(
        @PathVariable taskId: String,
        @RequestParam(required = false) source: String?,
        @RequestParam(required = false) level: String?,
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "100") size: Int,
        @RequestParam(required = false) from: String?,
        @RequestParam(required = false) to: String?
    ): ResponseEntity<Page<TaskLog>> {
        val logs = taskLogService.findByTaskId(
            taskId, source, level, page, size, from, to
        )
        return ResponseEntity.ok(logs)
    }
    
    @GetMapping("/{taskId}/logs/stats")
    fun getLogStats(@PathVariable taskId: String): ResponseEntity<LogStats> {
        val stats = taskLogService.getStats(taskId)
        return ResponseEntity.ok(stats)
    }
    
    @DeleteMapping("/{taskId}/logs")
    fun deleteLogs(
        @PathVariable taskId: String,
        @RequestParam(required = false) before: String?
    ): ResponseEntity<DeleteResponse> {
        val deletedCount = taskLogService.deleteByTaskId(taskId, before)
        return ResponseEntity.ok(DeleteResponse(deletedCount))
    }
}
```

### 2. データモデル

#### TaskLogエンティティ
```kotlin
@Entity
@Table(name = "task_logs")
data class TaskLog(
    @Id
    val id: String = UUID.randomUUID().toString(),
    
    @Indexed
    val taskId: String,
    
    val timestamp: LocalDateTime,
    val source: String, // "stdout", "stderr", "agent"
    val level: String, // "DEBUG", "INFO", "WARN", "ERROR"
    val message: String,
    val lineNumber: Int? = null,
    val metadata: Map<String, Any> = emptyMap()
)

@Entity
@Table(name = "log_stats")
data class LogStats(
    @Id
    val taskId: String,
    val totalLogs: Long,
    val logCounts: Map<String, Long>,
    val levelCounts: Map<String, Long>,
    val timeRange: TimeRange,
    val lastUpdated: LocalDateTime = LocalDateTime.now()
)

data class TimeRange(
    val start: LocalDateTime,
    val end: LocalDateTime
)
```

### 3. WebSocketハンドラー

#### WebSocketHandler
```kotlin
@Component
class WebSocketHandler(
    private val objectMapper: ObjectMapper,
    private val taskLogService: TaskLogService
) {
    
    private val sessions = ConcurrentHashMap<String, WebSocketSession>()
    private val taskSubscriptions = ConcurrentHashMap<String, MutableSet<String>>()
    
    fun handleMessage(session: WebSocketSession, message: String) {
        try {
            val logMessage = objectMapper.readValue(message, LogMessage::class.java)
            
            when (logMessage.type) {
                "log" -> handleLogMessage(logMessage)
                "subscribe" -> handleSubscribe(session, logMessage)
                "unsubscribe" -> handleUnsubscribe(session, logMessage)
                "ping" -> handlePing(session)
            }
        } catch (e: Exception) {
            log.error("WebSocket message handling error", e)
            sendError(session, "INVALID_MESSAGE_FORMAT", e.message)
        }
    }
    
    private fun handleLogMessage(logMessage: LogMessage) {
        // データベースに保存
        val taskLog = TaskLog(
            taskId = logMessage.taskId,
            timestamp = LocalDateTime.parse(logMessage.timestamp),
            source = logMessage.source,
            level = logMessage.level,
            message = logMessage.message,
            lineNumber = logMessage.lineNumber,
            metadata = logMessage.metadata
        )
        
        taskLogService.save(taskLog)
        
        // タスクに関連するセッションに配信
        broadcastToTask(logMessage.taskId, logMessage)
    }
    
    private fun handleSubscribe(session: WebSocketSession, message: LogMessage) {
        val taskId = message.taskId
        val sessionId = session.id
        
        sessions[sessionId] = session
        taskSubscriptions.computeIfAbsent(taskId) { mutableSetOf() }.add(sessionId)
        
        // 接続確認メッセージを送信
        sendMessage(session, mapOf(
            "type" to "subscribed",
            "taskId" to taskId,
            "timestamp" to LocalDateTime.now().toString()
        ))
    }
    
    fun broadcastToTask(taskId: String, message: Any) {
        val sessionIds = taskSubscriptions[taskId] ?: return
        
        sessionIds.forEach { sessionId ->
            val session = sessions[sessionId]
            if (session?.isOpen == true) {
                try {
                    sendMessage(session, message)
                } catch (e: Exception) {
                    log.error("WebSocket broadcast error", e)
                    removeSession(sessionId)
                }
            } else {
                removeSession(sessionId)
            }
        }
    }
    
    private fun sendMessage(session: WebSocketSession, message: Any) {
        val json = objectMapper.writeValueAsString(message)
        session.send(TextMessage(json))
    }
    
    private fun removeSession(sessionId: String) {
        sessions.remove(sessionId)
        taskSubscriptions.values.forEach { it.remove(sessionId) }
    }
}
```

## パフォーマンス最適化

### 1. バッファリング戦略
```go
type BufferingStrategy struct {
    BufferSize     int
    FlushInterval  time.Duration
    BatchSize      int
    MaxRetries     int
    RetryInterval  time.Duration
}

func NewAdaptiveBuffering() *BufferingStrategy {
    return &BufferingStrategy{
        BufferSize:    1000,
        FlushInterval: 1 * time.Second,
        BatchSize:     100,
        MaxRetries:    3,
        RetryInterval: 5 * time.Second,
    }
}
```

### 2. レート制限
```go
type RateLimiter struct {
    limit    int
    interval time.Duration
    tokens   chan struct{}
}

func NewRateLimiter(limit int, interval time.Duration) *RateLimiter {
    rl := &RateLimiter{
        limit:    limit,
        interval: interval,
        tokens:   make(chan struct{}, limit),
    }
    
    go rl.refill()
    return rl
}

func (rl *RateLimiter) refill() {
    ticker := time.NewTicker(rl.interval)
    defer ticker.Stop()
    
    for range ticker.C {
        for i := 0; i < rl.limit; i++ {
            select {
            case rl.tokens <- struct{}{}:
            default:
                return
            }
        }
    }
}

func (rl *RateLimiter) Allow() bool {
    select {
    case <-rl.tokens:
        return true
    default:
        return false
    }
}
```

## エラーハンドリング

### 1. 再接続ロジック
```go
func (ws *WebSocketClient) reconnectWithBackoff() {
    backoff := time.Second
    maxBackoff := 30 * time.Second
    
    for {
        log.Printf("Attempting WebSocket reconnection in %v", backoff)
        time.Sleep(backoff)
        
        if err := ws.connect(); err == nil {
            log.Println("WebSocket reconnected successfully")
            return
        }
        
        backoff *= 2
        if backoff > maxBackoff {
            backoff = maxBackoff
        }
    }
}
```

### 2. データベースフォールバック
```go
func (b *LogBuffer) saveToDatabase(logs []*LogMessage) error {
    // まずWebSocketで送信を試行
    for _, log := range logs {
        if err := b.wsClient.Send(log); err != nil {
            // WebSocket送信失敗時はローカルファイルに保存
            b.saveToLocalFile(log)
        }
    }
    
    // データベース保存は非同期で実行
    go func() {
        if err := b.apiClient.BatchSaveLogs(logs); err != nil {
            log.Printf("Database save error: %v", err)
            // 失敗時はローカルファイルに保存
            for _, log := range logs {
                b.saveToLocalFile(log)
            }
        }
    }()
    
    return nil
}
```

## 関連ドキュメント
- [アーキテクチャ概要](./architecture.md)
- [API仕様](./api-specification.md)
- [使用方法](./usage-guide.md)
- [設定・パフォーマンス](./configuration-performance.md) 