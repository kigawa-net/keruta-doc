# ログストリーミング 設定・パフォーマンス

> **概要**: ログストリーミング機能の設定オプション、パフォーマンス最適化、監視設定を説明します。

## 概要

このドキュメントでは、ログストリーミング機能の設定方法、パフォーマンス最適化、監視・運用に関する詳細を説明します。

## 設定オプション

### 環境変数設定

#### 基本設定
| 変数名 | 説明 | デフォルト値 | 推奨値 |
|--------|------|-------------|--------|
| `KERUTA_LOG_STREAMING_ENABLED` | ログストリーミング有効化 | `true` | `true` |
| `KERUTA_LOG_BUFFER_SIZE` | ログバッファサイズ | `1000` | `1000-5000` |
| `KERUTA_LOG_BATCH_SIZE` | バッチ送信サイズ | `10` | `10-100` |
| `KERUTA_LOG_FLUSH_INTERVAL` | フラッシュ間隔（秒） | `1` | `1-5` |
| `KERUTA_LOG_LEVEL` | ログレベル | `INFO` | `INFO/DEBUG` |

#### WebSocket設定
| 変数名 | 説明 | デフォルト値 | 推奨値 |
|--------|------|-------------|--------|
| `KERUTA_WEBSOCKET_URL` | WebSocket接続URL | `ws://localhost:8080/ws/tasks` | 環境に応じて |
| `KERUTA_WEBSOCKET_TIMEOUT` | 接続タイムアウト（秒） | `30` | `30-60` |
| `KERUTA_WEBSOCKET_RECONNECT_INTERVAL` | 再接続間隔（秒） | `5` | `5-10` |
| `KERUTA_WEBSOCKET_MAX_RECONNECT_ATTEMPTS` | 最大再接続回数 | `10` | `10-20` |

#### API設定
| 変数名 | 説明 | デフォルト値 | 推奨値 |
|--------|------|-------------|--------|
| `KERUTA_API_URL` | API接続URL | `http://localhost:8080/api/v1` | 環境に応じて |
| `KERUTA_API_TIMEOUT` | APIタイムアウト（秒） | `30` | `30-60` |
| `KERUTA_API_RETRY_COUNT` | API再試行回数 | `3` | `3-5` |
| `KERUTA_API_RETRY_INTERVAL` | 再試行間隔（秒） | `5` | `5-10` |

#### 認証設定
| 変数名 | 説明 | デフォルト値 | 推奨値 |
|--------|------|-------------|--------|
| `KERUTA_AUTH_TOKEN` | JWT認証トークン | - | 必須 |
| `KERUTA_API_KEY` | API Key | - | サービス間通信用 |
| `KERUTA_AUTH_TYPE` | 認証タイプ | `bearer` | `bearer/api-key` |

### 設定ファイル

#### YAML設定例
```yaml
# config.yaml
logStreaming:
  enabled: true
  bufferSize: 1000
  batchSize: 10
  flushInterval: 1s
  maxRetries: 3
  retryInterval: 5s
  level: INFO
  
  # パフォーマンス設定
  performance:
    maxConcurrentStreams: 10
    rateLimitPerSecond: 1000
    memoryLimitMB: 512
    
  # フィルタリング設定
  filters:
    excludePatterns:
      - ".*password.*"
      - ".*secret.*"
      - ".*token.*"
    includePatterns:
      - ".*"
    maxMessageLength: 8192

websocket:
  url: ws://keruta-api:8080/ws/tasks
  timeout: 30s
  reconnectInterval: 5s
  maxReconnectAttempts: 10
  heartbeatInterval: 30s
  
  # セキュリティ設定
  security:
    enableSSL: true
    verifyCert: true
    certPath: /path/to/cert.pem

api:
  url: http://keruta-api:8080/api/v1
  timeout: 30s
  retryCount: 3
  retryInterval: 5s
  
  # レート制限
  rateLimit:
    requestsPerMinute: 1000
    burstSize: 100

logging:
  level: INFO
  format: json
  output: stdout
  file:
    enabled: false
    path: /var/log/keruta-agent.log
    maxSize: 100MB
    maxBackups: 5

monitoring:
  enabled: true
  metrics:
    - logCount
    - errorRate
    - latency
    - memoryUsage
  prometheus:
    enabled: false
    port: 9090
```

#### JSON設定例
```json
{
  "logStreaming": {
    "enabled": true,
    "bufferSize": 1000,
    "batchSize": 10,
    "flushInterval": "1s",
    "maxRetries": 3,
    "retryInterval": "5s",
    "level": "INFO",
    "performance": {
      "maxConcurrentStreams": 10,
      "rateLimitPerSecond": 1000,
      "memoryLimitMB": 512
    },
    "filters": {
      "excludePatterns": [
        ".*password.*",
        ".*secret.*",
        ".*token.*"
      ],
      "includePatterns": [
        ".*"
      ],
      "maxMessageLength": 8192
    }
  },
  "websocket": {
    "url": "ws://keruta-api:8080/ws/tasks",
    "timeout": "30s",
    "reconnectInterval": "5s",
    "maxReconnectAttempts": 10,
    "heartbeatInterval": "30s",
    "security": {
      "enableSSL": true,
      "verifyCert": true,
      "certPath": "/path/to/cert.pem"
    }
  },
  "api": {
    "url": "http://keruta-api:8080/api/v1",
    "timeout": "30s",
    "retryCount": 3,
    "retryInterval": "5s",
    "rateLimit": {
      "requestsPerMinute": 1000,
      "burstSize": 100
    }
  },
  "logging": {
    "level": "INFO",
    "format": "json",
    "output": "stdout",
    "file": {
      "enabled": false,
      "path": "/var/log/keruta-agent.log",
      "maxSize": "100MB",
      "maxBackups": 5
    }
  },
  "monitoring": {
    "enabled": true,
    "metrics": [
      "logCount",
      "errorRate",
      "latency",
      "memoryUsage"
    ],
    "prometheus": {
      "enabled": false,
      "port": 9090
    }
  }
}
```

## パフォーマンス最適化

### 1. バッファリング戦略

#### 適応的バッファリング
```go
type AdaptiveBuffer struct {
    baseSize     int
    maxSize      int
    currentSize  int
    loadFactor   float64
    mutex        sync.RWMutex
}

func (ab *AdaptiveBuffer) adjustSize(currentLoad float64) {
    ab.mutex.Lock()
    defer ab.mutex.Unlock()
    
    if currentLoad > 0.8 {
        // 高負荷時はバッファサイズを増加
        ab.currentSize = min(ab.currentSize*2, ab.maxSize)
    } else if currentLoad < 0.3 {
        // 低負荷時はバッファサイズを減少
        ab.currentSize = max(ab.currentSize/2, ab.baseSize)
    }
}
```

#### バッチ処理最適化
```go
type BatchProcessor struct {
    batchSize    int
    flushInterval time.Duration
    processor    func([]*LogMessage) error
    buffer       chan *LogMessage
}

func (bp *BatchProcessor) Start() {
    go func() {
        ticker := time.NewTicker(bp.flushInterval)
        defer ticker.Stop()
        
        batch := make([]*LogMessage, 0, bp.batchSize)
        
        for {
            select {
            case msg := <-bp.buffer:
                batch = append(batch, msg)
                if len(batch) >= bp.batchSize {
                    bp.processBatch(batch)
                    batch = batch[:0]
                }
            case <-ticker.C:
                if len(batch) > 0 {
                    bp.processBatch(batch)
                    batch = batch[:0]
                }
            }
        }
    }()
}
```

### 2. レート制限

#### トークンバケット実装
```go
type TokenBucket struct {
    capacity     int
    tokens       int
    refillRate   float64
    lastRefill   time.Time
    mutex        sync.Mutex
}

func (tb *TokenBucket) Allow() bool {
    tb.mutex.Lock()
    defer tb.mutex.Unlock()
    
    now := time.Now()
    elapsed := now.Sub(tb.lastRefill).Seconds()
    
    // トークンを補充
    newTokens := int(elapsed * tb.refillRate)
    tb.tokens = min(tb.tokens+newTokens, tb.capacity)
    tb.lastRefill = now
    
    if tb.tokens > 0 {
        tb.tokens--
        return true
    }
    return false
}
```

#### 階層的レート制限
```go
type HierarchicalRateLimiter struct {
    globalLimit  *TokenBucket
    perTaskLimit *TokenBucket
    perSourceLimit *TokenBucket
}

func (hrl *HierarchicalRateLimiter) Allow(taskID, source string) bool {
    // グローバル制限チェック
    if !hrl.globalLimit.Allow() {
        return false
    }
    
    // タスク別制限チェック
    if !hrl.perTaskLimit.Allow() {
        return false
    }
    
    // ソース別制限チェック
    if !hrl.perSourceLimit.Allow() {
        return false
    }
    
    return true
}
```

### 3. メモリ管理

#### メモリプール
```go
type LogMessagePool struct {
    pool sync.Pool
}

func NewLogMessagePool() *LogMessagePool {
    return &LogMessagePool{
        pool: sync.Pool{
            New: func() interface{} {
                return &LogMessage{
                    Metadata: make(map[string]interface{}),
                }
            },
        },
    }
}

func (lmp *LogMessagePool) Get() *LogMessage {
    return lmp.pool.Get().(*LogMessage)
}

func (lmp *LogMessagePool) Put(msg *LogMessage) {
    // メッセージをリセット
    msg.TaskID = ""
    msg.Message = ""
    msg.Metadata = make(map[string]interface{})
    lmp.pool.Put(msg)
}
```

#### ガベージコレクション最適化
```go
type MemoryManager struct {
    maxMemoryMB  int
    currentUsage int64
    mutex        sync.RWMutex
}

func (mm *MemoryManager) CheckMemoryLimit() bool {
    mm.mutex.RLock()
    defer mm.mutex.RUnlock()
    
    var m runtime.MemStats
    runtime.ReadMemStats(&m)
    
    currentMB := int(m.Alloc / 1024 / 1024)
    return currentMB < mm.maxMemoryMB
}

func (mm *MemoryManager) ForceGC() {
    runtime.GC()
    debug.FreeOSMemory()
}
```

## 監視・メトリクス

### 1. パフォーマンスメトリクス

#### メトリクス定義
```go
type Metrics struct {
    LogCount        int64
    ErrorCount      int64
    Latency         time.Duration
    MemoryUsage     int64
    BufferSize      int
    ConnectionCount int
    ReconnectCount  int
}

type MetricsCollector struct {
    metrics *Metrics
    mutex   sync.RWMutex
    startTime time.Time
}

func (mc *MetricsCollector) RecordLog() {
    atomic.AddInt64(&mc.metrics.LogCount, 1)
}

func (mc *MetricsCollector) RecordError() {
    atomic.AddInt64(&mc.metrics.ErrorCount, 1)
}

func (mc *MetricsCollector) RecordLatency(duration time.Duration) {
    mc.mutex.Lock()
    defer mc.mutex.Unlock()
    mc.metrics.Latency = duration
}

func (mc *MetricsCollector) GetMetrics() *Metrics {
    mc.mutex.RLock()
    defer mc.mutex.RUnlock()
    
    return &Metrics{
        LogCount:        atomic.LoadInt64(&mc.metrics.LogCount),
        ErrorCount:      atomic.LoadInt64(&mc.metrics.ErrorCount),
        Latency:         mc.metrics.Latency,
        MemoryUsage:     mc.getMemoryUsage(),
        BufferSize:      mc.metrics.BufferSize,
        ConnectionCount: mc.metrics.ConnectionCount,
        ReconnectCount:  mc.metrics.ReconnectCount,
    }
}
```

### 2. Prometheus統合

#### メトリクスエクスポート
```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    logCounter = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "keruta_logs_total",
            Help: "Total number of logs processed",
        },
        []string{"source", "level", "task_id"},
    )
    
    logLatency = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "keruta_log_latency_seconds",
            Help:    "Log processing latency",
            Buckets: prometheus.DefBuckets,
        },
        []string{"source"},
    )
    
    bufferSize = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "keruta_buffer_size",
            Help: "Current buffer size",
        },
        []string{"type"},
    )
    
    errorCounter = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "keruta_errors_total",
            Help: "Total number of errors",
        },
        []string{"type"},
    )
)

func init() {
    prometheus.MustRegister(logCounter)
    prometheus.MustRegister(logLatency)
    prometheus.MustRegister(bufferSize)
    prometheus.MustRegister(errorCounter)
}
```

### 3. ヘルスチェック

#### ヘルスチェックエンドポイント
```go
type HealthChecker struct {
    wsClient    *WebSocketClient
    apiClient   *APIClient
    metrics     *MetricsCollector
}

func (hc *HealthChecker) CheckHealth() HealthStatus {
    status := HealthStatus{
        Status:    "healthy",
        Timestamp: time.Now(),
        Checks:    make(map[string]CheckResult),
    }
    
    // WebSocket接続チェック
    wsCheck := hc.checkWebSocket()
    status.Checks["websocket"] = wsCheck
    
    // API接続チェック
    apiCheck := hc.checkAPI()
    status.Checks["api"] = apiCheck
    
    // メモリ使用量チェック
    memoryCheck := hc.checkMemory()
    status.Checks["memory"] = memoryCheck
    
    // 全体のステータス判定
    if !wsCheck.Healthy || !apiCheck.Healthy || !memoryCheck.Healthy {
        status.Status = "unhealthy"
    }
    
    return status
}

type HealthStatus struct {
    Status    string                  `json:"status"`
    Timestamp time.Time              `json:"timestamp"`
    Checks    map[string]CheckResult `json:"checks"`
}

type CheckResult struct {
    Healthy   bool        `json:"healthy"`
    Message   string      `json:"message"`
    Latency   time.Duration `json:"latency,omitempty"`
    Error     string      `json:"error,omitempty"`
}
```

## 運用設定

### 1. ログローテーション

#### ログローテーション設定
```yaml
logging:
  file:
    enabled: true
    path: /var/log/keruta-agent.log
    maxSize: 100MB
    maxBackups: 5
    maxAge: 30d
    compress: true
    format: json
```

### 2. アラート設定

#### アラートルール
```yaml
alerts:
  - name: "high_error_rate"
    condition: "error_rate > 0.05"
    duration: "5m"
    severity: "warning"
    
  - name: "high_latency"
    condition: "latency > 1s"
    duration: "2m"
    severity: "warning"
    
  - name: "memory_usage_high"
    condition: "memory_usage > 80%"
    duration: "1m"
    severity: "critical"
    
  - name: "websocket_disconnected"
    condition: "websocket_status != 'connected'"
    duration: "30s"
    severity: "critical"
```

### 3. バックアップ設定

#### ログバックアップ
```yaml
backup:
  enabled: true
  schedule: "0 2 * * *"  # 毎日午前2時
  retention:
    days: 30
    maxSize: "10GB"
  storage:
    type: "s3"
    bucket: "keruta-logs"
    region: "ap-northeast-1"
```

## トラブルシューティング

### 1. よくある問題と解決策

#### WebSocket接続エラー
```bash
# 接続確認
curl -I http://keruta-api:8080/health

# WebSocket接続テスト
wscat -c ws://keruta-api:8080/ws/tasks/task123

# ログ確認
tail -f /var/log/keruta-agent.log | grep websocket
```

#### メモリ不足
```bash
# メモリ使用量確認
ps aux | grep keruta-agent

# ガベージコレクション強制実行
curl -X POST http://localhost:8080/debug/gc

# メモリプロファイル取得
curl -X GET http://localhost:8080/debug/pprof/heap
```

#### パフォーマンス問題
```bash
# メトリクス確認
curl -X GET http://localhost:8080/metrics

# プロファイル取得
curl -X GET http://localhost:8080/debug/pprof/profile

# トレース取得
curl -X GET http://localhost:8080/debug/pprof/trace
```

### 2. デバッグ設定

#### デバッグモード有効化
```yaml
debug:
  enabled: true
  level: DEBUG
  pprof:
    enabled: true
    port: 6060
  trace:
    enabled: false
    output: /tmp/trace.out
```

## 関連ドキュメント
- [アーキテクチャ概要](./architecture.md)
- [API仕様](./api-specification.md)
- [実装仕様](./implementation-specification.md)
- [使用方法](./usage-guide.md) 