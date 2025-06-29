# ログストリーミング機能

> **概要**: keruta-agentからkeruta本体、データベース、クライアントへスクリプトの実行ログをリアルタイムでストリーミングする機能です。

## 概要

ログストリーミング機能は、Kubernetes Job内で実行されるスクリプトの標準出力・標準エラー出力をリアルタイムでキャプチャし、以下の3つの宛先にストリーミングします：

1. **keruta本体**: WebSocket経由でリアルタイム送信
2. **データベース**: 構造化ログとして永続化
3. **クライアント**: 管理パネルでのリアルタイム表示

## 機能特徴

### 🚀 リアルタイム配信
- WebSocketによる低遅延ログ配信
- スクリプト実行中のリアルタイム表示
- 接続断時の自動再接続

### 📊 構造化ログ
- JSON形式での構造化ログ保存
- ログレベル、ソース、メタデータの管理
- 高度な検索・フィルタリング機能

### 🔧 柔軟な設定
- 環境変数・設定ファイルによる設定
- バッファサイズ・バッチサイズの調整
- ログレベルの制御

### 📈 パフォーマンス最適化
- 適応的バッファリング
- レート制限機能
- メモリ管理最適化

### 🛡️ セキュリティ
- JWT認証・API Key認証
- ログフィルタリング（機密情報除外）
- SSL/TLS暗号化

## クイックスタート

### 1. 基本的な使用

```bash
# ログストリーミング有効でスクリプト実行
keruta-agent execute --task-id task123 --script /path/to/script.sh

# デバッグレベルのログを有効化
keruta-agent execute --task-id task123 --script /path/to/script.sh --log-level DEBUG
```

### 2. 環境変数設定

```bash
# 基本設定
export KERUTA_LOG_STREAMING_ENABLED=true
export KERUTA_LOG_BUFFER_SIZE=1000
export KERUTA_LOG_BATCH_SIZE=10

# WebSocket接続設定
export KERUTA_WEBSOCKET_URL=ws://keruta-api:8080/ws/tasks
export KERUTA_API_URL=http://keruta-api:8080/api/v1

# 認証設定
export KERUTA_AUTH_TOKEN=your-jwt-token-here
```

### 3. 管理パネルでの表示

```javascript
// WebSocket接続
const ws = new WebSocket(`ws://keruta-api:8080/ws/tasks/${taskId}`);

ws.onmessage = function(event) {
    const logMessage = JSON.parse(event.data);
    if (logMessage.type === 'log') {
        console.log(`[${logMessage.timestamp}] ${logMessage.message}`);
    }
};
```

## アーキテクチャ

```
┌─────────────────────────────────────────────────────────┐
│                Kubernetes Pod                           │
│                                                         │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              keruta-agent                          │ │
│  │                                                     │
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

## API エンドポイント

### REST API

| メソッド | エンドポイント | 説明 |
|---------|---------------|------|
| `GET` | `/api/v1/tasks/{taskId}/logs` | ログ一覧取得 |
| `GET` | `/api/v1/tasks/{taskId}/logs/stats` | ログ統計取得 |
| `DELETE` | `/api/v1/tasks/{taskId}/logs` | ログ削除 |
| `GET` | `/api/v1/tasks/{taskId}/logs/export` | ログエクスポート |

### WebSocket API

| エンドポイント | 説明 |
|---------------|------|
| `ws://keruta-api:8080/ws/tasks/{taskId}` | リアルタイムログ配信 |

## 設定オプション

### 環境変数

| 変数名 | 説明 | デフォルト値 |
|--------|------|-------------|
| `KERUTA_LOG_STREAMING_ENABLED` | ログストリーミング有効化 | `true` |
| `KERUTA_LOG_BUFFER_SIZE` | ログバッファサイズ | `1000` |
| `KERUTA_LOG_BATCH_SIZE` | バッチ送信サイズ | `10` |
| `KERUTA_LOG_FLUSH_INTERVAL` | フラッシュ間隔（秒） | `1` |
| `KERUTA_WEBSOCKET_URL` | WebSocket接続URL | `ws://localhost:8080/ws/tasks` |
| `KERUTA_API_URL` | API接続URL | `http://localhost:8080/api/v1` |

### 設定ファイル

```yaml
logStreaming:
  enabled: true
  bufferSize: 1000
  batchSize: 10
  flushInterval: 1s
  level: INFO

websocket:
  url: ws://keruta-api:8080/ws/tasks
  timeout: 30s
  reconnectInterval: 5s

api:
  url: http://keruta-api:8080/api/v1
  timeout: 30s
  retryCount: 3
```

## 使用例

### keruta-agent での実行

```bash
# 基本的な実行
keruta-agent execute --task-id task123 --script /path/to/script.sh

# ログレベル指定
keruta-agent execute --task-id task123 --script /path/to/script.sh --log-level DEBUG

# ログストリーミング無効化
keruta-agent execute --task-id task123 --script /path/to/script.sh --disable-log-streaming
```

### REST API 使用例

```bash
# ログ取得
curl -X GET "http://keruta-api:8080/api/v1/tasks/task123/logs" \
  -H "Authorization: Bearer your-jwt-token"

# エラーログのみ取得
curl -X GET "http://keruta-api:8080/api/v1/tasks/task123/logs?level=ERROR" \
  -H "Authorization: Bearer your-jwt-token"

# ログ統計取得
curl -X GET "http://keruta-api:8080/api/v1/tasks/task123/logs/stats" \
  -H "Authorization: Bearer your-jwt-token"
```

### JavaScript クライアント

```javascript
class LogStreamingClient {
    constructor(taskId) {
        this.taskId = taskId;
        this.ws = new WebSocket(`ws://keruta-api:8080/ws/tasks/${taskId}`);
        this.setupEventHandlers();
    }
    
    setupEventHandlers() {
        this.ws.onopen = () => {
            console.log('WebSocket接続確立');
            this.subscribe();
        };
        
        this.ws.onmessage = (event) => {
            const message = JSON.parse(event.data);
            if (message.type === 'log') {
                this.handleLogMessage(message);
            }
        };
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
    
    handleLogMessage(logMessage) {
        console.log(`[${logMessage.timestamp}] ${logMessage.source}: ${logMessage.message}`);
    }
}

// 使用例
const client = new LogStreamingClient('task123');
```

## パフォーマンス

### 推奨設定

| 項目 | 推奨値 | 説明 |
|------|--------|------|
| バッファサイズ | 1000-5000 | メモリ使用量とレイテンシのバランス |
| バッチサイズ | 10-100 | ネットワーク効率の最適化 |
| フラッシュ間隔 | 1-5秒 | リアルタイム性とパフォーマンスのバランス |
| レート制限 | 1000 msg/sec | システム負荷の制御 |

### 監視メトリクス

- **ログ処理数**: 1分間あたりの処理ログ数
- **エラー率**: ログ処理エラーの割合
- **レイテンシ**: ログ生成から配信までの時間
- **メモリ使用量**: バッファ使用メモリ量
- **接続数**: アクティブなWebSocket接続数

## トラブルシューティング

### よくある問題

#### WebSocket接続エラー
```bash
# 接続確認
curl -I http://keruta-api:8080/health

# ログ確認
tail -f /var/log/keruta-agent.log | grep websocket
```

#### メモリ不足
```bash
# メモリ使用量確認
ps aux | grep keruta-agent

# ガベージコレクション強制実行
curl -X POST http://localhost:8080/debug/gc
```

#### パフォーマンス問題
```bash
# メトリクス確認
curl -X GET http://localhost:8080/metrics

# プロファイル取得
curl -X GET http://localhost:8080/debug/pprof/profile
```

## 関連ドキュメント

- [アーキテクチャ概要](./architecture.md) - システム設計の詳細
- [API仕様](./api-specification.md) - エンドポイントとデータモデル
- [実装仕様](./implementation-specification.md) - 技術的実装詳細
- [使用方法](./usage-guide.md) - 使用例とサンプルコード
- [設定・パフォーマンス](./configuration-performance.md) - 設定と最適化

## ライセンス

このプロジェクトはMITライセンスの下で公開されています。

## 貢献

バグ報告や機能要望は、GitHubのIssueでお知らせください。プルリクエストも歓迎します。

## サポート

技術的な質問やサポートが必要な場合は、以下の方法でお問い合わせください：

- GitHub Issues: [keruta-doc/issues](https://github.com/your-org/keruta-doc/issues)
- ドキュメント: [keruta-doc/docs](https://github.com/your-org/keruta-doc/docs)
- メール: support@keruta.example.com 