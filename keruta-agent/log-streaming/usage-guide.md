# ログストリーミング 使用方法

> **概要**: ログストリーミング機能の使用方法、サンプルコード、ベストプラクティスを説明します。

## 概要

このドキュメントでは、ログストリーミング機能の実際の使用方法、コマンド例、クライアント側の実装例を説明します。

## keruta-agent での使用

### 基本的な実行

#### ログストリーミング有効で実行
```bash
# 基本的な実行（ログストリーミング自動有効）
keruta-agent execute --task-id task123 --script /path/to/script.sh

# 詳細なログ出力
keruta-agent execute --task-id task123 --script /path/to/script.sh --verbose
```

#### ログレベル指定
```bash
# デバッグレベルのログを有効化
keruta-agent execute --task-id task123 --script /path/to/script.sh --log-level DEBUG

# 警告以上のログのみ
keruta-agent execute --task-id task123 --script /path/to/script.sh --log-level WARN
```

#### ログストリーミング無効化
```bash
# ログストリーミングを無効化して実行
keruta-agent execute --task-id task123 --script /path/to/script.sh --disable-log-streaming
```

### 環境変数による設定

#### ログストリーミング設定
```bash
# ログストリーミング有効化
export KERUTA_LOG_STREAMING_ENABLED=true

# ログバッファサイズ設定
export KERUTA_LOG_BUFFER_SIZE=1000

# バッチ送信サイズ設定
export KERUTA_LOG_BATCH_SIZE=10

# フラッシュ間隔設定（秒）
export KERUTA_LOG_FLUSH_INTERVAL=1

# WebSocket接続URL
export KERUTA_WEBSOCKET_URL=ws://keruta-api:8080/ws/tasks

# API接続URL
export KERUTA_API_URL=http://keruta-api:8080/api/v1
```

#### 認証設定
```bash
# JWTトークン設定
export KERUTA_AUTH_TOKEN=your-jwt-token-here

# API Key設定
export KERUTA_API_KEY=your-api-key-here
```

### 設定ファイルによる設定

#### config.yaml
```yaml
logStreaming:
  enabled: true
  bufferSize: 1000
  batchSize: 10
  flushInterval: 1s
  maxRetries: 3
  retryInterval: 5s
  
websocket:
  url: ws://keruta-api:8080/ws/tasks
  reconnectInterval: 5s
  maxReconnectAttempts: 10
  
api:
  url: http://keruta-api:8080/api/v1
  timeout: 30s
  retryCount: 3
  
logging:
  level: INFO
  format: json
  output: stdout
```

## 管理パネルでの使用

### WebSocket接続

#### JavaScript実装例
```javascript
class LogStreamingClient {
    constructor(taskId, options = {}) {
        this.taskId = taskId;
        this.options = {
            url: 'ws://keruta-api:8080/ws/tasks',
            reconnectInterval: 5000,
            maxReconnectAttempts: 10,
            ...options
        };
        
        this.ws = null;
        this.reconnectAttempts = 0;
        this.listeners = new Map();
        
        this.connect();
    }
    
    connect() {
        try {
            this.ws = new WebSocket(`${this.options.url}/${this.taskId}`);
            
            this.ws.onopen = () => {
                console.log('WebSocket接続確立');
                this.reconnectAttempts = 0;
                
                // 購読メッセージを送信
                this.subscribe();
            };
            
            this.ws.onmessage = (event) => {
                this.handleMessage(JSON.parse(event.data));
            };
            
            this.ws.onclose = () => {
                console.log('WebSocket接続切断');
                this.reconnect();
            };
            
            this.ws.onerror = (error) => {
                console.error('WebSocketエラー:', error);
            };
            
        } catch (error) {
            console.error('WebSocket接続エラー:', error);
            this.reconnect();
        }
    }
    
    subscribe() {
        const subscribeMessage = {
            type: 'subscribe',
            taskId: this.taskId,
            options: {
                sources: ['stdout', 'stderr', 'agent'],
                levels: ['DEBUG', 'INFO', 'WARN', 'ERROR'],
                realtime: true
            }
        };
        
        this.ws.send(JSON.stringify(subscribeMessage));
    }
    
    handleMessage(message) {
        switch (message.type) {
            case 'log':
                this.emit('log', message);
                break;
            case 'ping':
                this.emit('ping', message);
                break;
            case 'error':
                this.emit('error', message);
                break;
            case 'subscribed':
                this.emit('subscribed', message);
                break;
            default:
                console.warn('不明なメッセージタイプ:', message.type);
        }
    }
    
    reconnect() {
        if (this.reconnectAttempts >= this.options.maxReconnectAttempts) {
            console.error('最大再接続回数に達しました');
            this.emit('maxReconnectAttemptsReached');
            return;
        }
        
        this.reconnectAttempts++;
        console.log(`再接続試行 ${this.reconnectAttempts}/${this.options.maxReconnectAttempts}`);
        
        setTimeout(() => {
            this.connect();
        }, this.options.reconnectInterval);
    }
    
    on(event, callback) {
        if (!this.listeners.has(event)) {
            this.listeners.set(event, []);
        }
        this.listeners.get(event).push(callback);
    }
    
    emit(event, data) {
        const callbacks = this.listeners.get(event) || [];
        callbacks.forEach(callback => callback(data));
    }
    
    disconnect() {
        if (this.ws) {
            this.ws.close();
        }
    }
}
```

### ログ表示コンポーネント

#### HTML/CSS実装例
```html
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ログストリーミング</title>
    <style>
        .log-container {
            background: #1e1e1e;
            color: #ffffff;
            font-family: 'Courier New', monospace;
            padding: 20px;
            height: 600px;
            overflow-y: auto;
            border-radius: 8px;
        }
        
        .log-entry {
            margin: 2px 0;
            padding: 2px 8px;
            border-radius: 4px;
            white-space: pre-wrap;
            word-break: break-all;
        }
        
        .log-entry.timestamp {
            color: #888888;
            font-size: 0.9em;
        }
        
        .log-entry.source {
            color: #4CAF50;
            font-weight: bold;
        }
        
        .log-entry.level-DEBUG {
            color: #2196F3;
        }
        
        .log-entry.level-INFO {
            color: #4CAF50;
        }
        
        .log-entry.level-WARN {
            color: #FF9800;
        }
        
        .log-entry.level-ERROR {
            color: #F44336;
            background-color: rgba(244, 67, 54, 0.1);
        }
        
        .log-controls {
            margin: 20px 0;
            display: flex;
            gap: 10px;
            align-items: center;
        }
        
        .log-controls button {
            padding: 8px 16px;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            font-size: 14px;
        }
        
        .log-controls .clear-btn {
            background: #f44336;
            color: white;
        }
        
        .log-controls .pause-btn {
            background: #ff9800;
            color: white;
        }
        
        .log-controls .resume-btn {
            background: #4caf50;
            color: white;
        }
        
        .log-stats {
            display: flex;
            gap: 20px;
            margin: 10px 0;
            font-size: 14px;
        }
        
        .log-stats .stat {
            background: #f5f5f5;
            padding: 8px 12px;
            border-radius: 4px;
        }
    </style>
</head>
<body>
    <div class="log-controls">
        <button class="clear-btn" onclick="clearLogs()">クリア</button>
        <button class="pause-btn" onclick="pauseLogs()">一時停止</button>
        <button class="resume-btn" onclick="resumeLogs()" style="display: none;">再開</button>
        <label>
            <input type="checkbox" id="autoScroll" checked> 自動スクロール
        </label>
        <label>
            <input type="checkbox" id="showTimestamp" checked> タイムスタンプ表示
        </label>
    </div>
    
    <div class="log-stats">
        <div class="stat">総ログ数: <span id="totalLogs">0</span></div>
        <div class="stat">INFO: <span id="infoCount">0</span></div>
        <div class="stat">WARN: <span id="warnCount">0</span></div>
        <div class="stat">ERROR: <span id="errorCount">0</span></div>
    </div>
    
    <div id="logContainer" class="log-container"></div>
    
    <script>
        let logClient;
        let isPaused = false;
        let logCounts = { INFO: 0, WARN: 0, ERROR: 0, total: 0 };
        
        // ログストリーミングクライアント初期化
        function initLogStreaming(taskId) {
            logClient = new LogStreamingClient(taskId, {
                url: 'ws://keruta-api:8080/ws/tasks'
            });
            
            logClient.on('log', handleLogMessage);
            logClient.on('error', handleError);
            logClient.on('subscribed', handleSubscribed);
        }
        
        // ログメッセージ処理
        function handleLogMessage(logMessage) {
            if (isPaused) return;
            
            const logContainer = document.getElementById('logContainer');
            const logElement = document.createElement('div');
            logElement.className = `log-entry level-${logMessage.level}`;
            
            let logText = '';
            
            if (document.getElementById('showTimestamp').checked) {
                logText += `[${new Date(logMessage.timestamp).toLocaleTimeString()}] `;
            }
            
            logText += `[${logMessage.source.toUpperCase()}] `;
            logText += `[${logMessage.level}] `;
            logText += logMessage.message;
            
            logElement.textContent = logText;
            logContainer.appendChild(logElement);
            
            // 自動スクロール
            if (document.getElementById('autoScroll').checked) {
                logContainer.scrollTop = logContainer.scrollHeight;
            }
            
            // 統計更新
            updateLogStats(logMessage.level);
        }
        
        // エラーハンドリング
        function handleError(errorMessage) {
            console.error('ログストリーミングエラー:', errorMessage);
            
            const logContainer = document.getElementById('logContainer');
            const errorElement = document.createElement('div');
            errorElement.className = 'log-entry level-ERROR';
            errorElement.textContent = `[ERROR] ${errorMessage.message}`;
            logContainer.appendChild(errorElement);
        }
        
        // 購読確認
        function handleSubscribed(message) {
            console.log('ログストリーミング購読開始:', message);
        }
        
        // ログ統計更新
        function updateLogStats(level) {
            logCounts[level]++;
            logCounts.total++;
            
            document.getElementById('totalLogs').textContent = logCounts.total;
            document.getElementById('infoCount').textContent = logCounts.INFO;
            document.getElementById('warnCount').textContent = logCounts.WARN;
            document.getElementById('errorCount').textContent = logCounts.ERROR;
        }
        
        // ログクリア
        function clearLogs() {
            document.getElementById('logContainer').innerHTML = '';
            logCounts = { INFO: 0, WARN: 0, ERROR: 0, total: 0 };
            updateLogStats();
        }
        
        // ログ一時停止
        function pauseLogs() {
            isPaused = true;
            document.querySelector('.pause-btn').style.display = 'none';
            document.querySelector('.resume-btn').style.display = 'inline-block';
        }
        
        // ログ再開
        function resumeLogs() {
            isPaused = false;
            document.querySelector('.pause-btn').style.display = 'inline-block';
            document.querySelector('.resume-btn').style.display = 'none';
        }
        
        // ページ読み込み時に初期化
        document.addEventListener('DOMContentLoaded', function() {
            // URLパラメータからタスクIDを取得
            const urlParams = new URLSearchParams(window.location.search);
            const taskId = urlParams.get('taskId') || 'demo-task';
            
            initLogStreaming(taskId);
        });
    </script>
</body>
</html>
```

## REST API 使用例

### cURL コマンド例

#### ログ取得
```bash
# タスクログ一覧取得
curl -X GET "http://keruta-api:8080/api/v1/tasks/task123/logs" \
  -H "Authorization: Bearer your-jwt-token"

# 特定のソースのログのみ取得
curl -X GET "http://keruta-api:8080/api/v1/tasks/task123/logs?source=stdout" \
  -H "Authorization: Bearer your-jwt-token"

# エラーログのみ取得
curl -X GET "http://keruta-api:8080/api/v1/tasks/task123/logs?level=ERROR" \
  -H "Authorization: Bearer your-jwt-token"

# ページネーション付きで取得
curl -X GET "http://keruta-api:8080/api/v1/tasks/task123/logs?page=0&size=50" \
  -H "Authorization: Bearer your-jwt-token"
```

#### ログ統計取得
```bash
# タスクログ統計取得
curl -X GET "http://keruta-api:8080/api/v1/tasks/task123/logs/stats" \
  -H "Authorization: Bearer your-jwt-token"
```

#### ログ削除
```bash
# タスクログ削除
curl -X DELETE "http://keruta-api:8080/api/v1/tasks/task123/logs" \
  -H "Authorization: Bearer your-jwt-token"

# 特定日時以前のログ削除
curl -X DELETE "http://keruta-api:8080/api/v1/tasks/task123/logs?before=2024-01-01T00:00:00Z" \
  -H "Authorization: Bearer your-jwt-token"
```

#### ログエクスポート
```bash
# JSON形式でエクスポート
curl -X GET "http://keruta-api:8080/api/v1/tasks/task123/logs/export?format=json" \
  -H "Authorization: Bearer your-jwt-token" \
  -o task123_logs.json

# CSV形式でエクスポート
curl -X GET "http://keruta-api:8080/api/v1/tasks/task123/logs/export?format=csv" \
  -H "Authorization: Bearer your-jwt-token" \
  -o task123_logs.csv
```

### Python クライアント例

```python
import requests
import json
import websocket
import threading
import time

class KerutaLogClient:
    def __init__(self, base_url, auth_token):
        self.base_url = base_url
        self.headers = {
            'Authorization': f'Bearer {auth_token}',
            'Content-Type': 'application/json'
        }
    
    def get_logs(self, task_id, source=None, level=None, page=0, size=100):
        """ログ一覧取得"""
        params = {
            'page': page,
            'size': size
        }
        if source:
            params['source'] = source
        if level:
            params['level'] = level
        
        response = requests.get(
            f"{self.base_url}/tasks/{task_id}/logs",
            headers=self.headers,
            params=params
        )
        response.raise_for_status()
        return response.json()
    
    def get_log_stats(self, task_id):
        """ログ統計取得"""
        response = requests.get(
            f"{self.base_url}/tasks/{task_id}/logs/stats",
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()
    
    def delete_logs(self, task_id, before=None):
        """ログ削除"""
        params = {}
        if before:
            params['before'] = before
        
        response = requests.delete(
            f"{self.base_url}/tasks/{task_id}/logs",
            headers=self.headers,
            params=params
        )
        response.raise_for_status()
        return response.json()
    
    def export_logs(self, task_id, format='json', output_file=None):
        """ログエクスポート"""
        params = {'format': format}
        
        response = requests.get(
            f"{self.base_url}/tasks/{task_id}/logs/export",
            headers=self.headers,
            params=params
        )
        response.raise_for_status()
        
        if output_file:
            with open(output_file, 'wb') as f:
                f.write(response.content)
            return output_file
        else:
            return response.content

class KerutaWebSocketClient:
    def __init__(self, task_id, ws_url, on_message=None):
        self.task_id = task_id
        self.ws_url = f"{ws_url}/{task_id}"
        self.on_message = on_message
        self.ws = None
    
    def connect(self):
        """WebSocket接続"""
        self.ws = websocket.WebSocketApp(
            self.ws_url,
            on_open=self.on_open,
            on_message=self.on_message,
            on_error=self.on_error,
            on_close=self.on_close
        )
        
        wst = threading.Thread(target=self.ws.run_forever)
        wst.daemon = True
        wst.start()
    
    def on_open(self, ws):
        print(f"WebSocket接続確立: {self.task_id}")
        # 購読メッセージ送信
        subscribe_message = {
            "type": "subscribe",
            "taskId": self.task_id,
            "options": {
                "sources": ["stdout", "stderr", "agent"],
                "levels": ["DEBUG", "INFO", "WARN", "ERROR"],
                "realtime": True
            }
        }
        ws.send(json.dumps(subscribe_message))
    
    def on_message(self, ws, message):
        data = json.loads(message)
        if self.on_message:
            self.on_message(data)
        else:
            print(f"ログ: {data}")
    
    def on_error(self, ws, error):
        print(f"WebSocketエラー: {error}")
    
    def on_close(self, ws, close_status_code, close_msg):
        print(f"WebSocket接続切断: {close_status_code} - {close_msg}")
    
    def disconnect(self):
        if self.ws:
            self.ws.close()

# 使用例
if __name__ == "__main__":
    # REST API クライアント
    client = KerutaLogClient(
        base_url="http://keruta-api:8080/api/v1",
        auth_token="your-jwt-token"
    )
    
    # ログ取得
    logs = client.get_logs("task123", source="stdout", level="ERROR")
    print(f"取得したログ数: {len(logs['content'])}")
    
    # ログ統計
    stats = client.get_log_stats("task123")
    print(f"総ログ数: {stats['totalLogs']}")
    
    # WebSocket クライアント
    def on_log_message(message):
        if message['type'] == 'log':
            print(f"[{message['timestamp']}] {message['source']}: {message['message']}")
    
    ws_client = KerutaWebSocketClient(
        task_id="task123",
        ws_url="ws://keruta-api:8080/ws/tasks",
        on_message=on_log_message
    )
    
    ws_client.connect()
    
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        ws_client.disconnect()
```

## ベストプラクティス

### 1. パフォーマンス最適化
- ログバッファサイズを適切に設定
- バッチ送信サイズを調整
- 不要なログレベルは無効化

### 2. エラーハンドリング
- WebSocket接続断時の再接続処理
- データベース接続エラー時のフォールバック
- レート制限の考慮

### 3. セキュリティ
- JWTトークンの適切な管理
- WebSocket接続の認証
- ログデータの暗号化

### 4. 監視・運用
- ログストリーミングの状態監視
- パフォーマンスメトリクスの収集
- エラー率の監視

## 関連ドキュメント
- [アーキテクチャ概要](./architecture.md)
- [API仕様](./api-specification.md)
- [実装仕様](./implementation-specification.md)
- [設定・パフォーマンス](./configuration-performance.md) 