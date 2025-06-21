# keruta-agent 実装例

> **概要**: keruta-agentの実装例とサンプルコードを提供するドキュメントです。

## 目次
- [概要](#概要)
- [プロジェクト構造](#プロジェクト構造)
- [主要コンポーネント](#主要コンポーネント)
- [実装例](#実装例)
- [サンプルスクリプト](#サンプルスクリプト)
- [テスト](#テスト)
- [デプロイ](#デプロイ)

## 概要
このドキュメントでは、keruta-agentの実装例とサンプルコードを提供します。Go言語を使用したCLIツールの実装例を示します。

## プロジェクト構造
```
keruta-agent/
├── cmd/
│   └── keruta-agent/
│       └── main.go                 # メインエントリーポイント
├── internal/
│   ├── api/
│   │   ├── client.go               # APIクライアント
│   │   ├── models.go               # データモデル
│   │   └── errors.go               # エラーハンドリング
│   ├── commands/
│   │   ├── root.go                 # ルートコマンド
│   │   ├── start.go                # startコマンド
│   │   ├── success.go              # successコマンド
│   │   ├── fail.go                 # failコマンド
│   │   ├── progress.go             # progressコマンド
│   │   ├── log.go                  # logコマンド
│   │   ├── artifact.go             # artifactコマンド
│   │   ├── health.go               # healthコマンド
│   │   └── config.go               # configコマンド
│   ├── config/
│   │   └── config.go               # 設定管理
│   ├── logger/
│   │   └── logger.go               # ログ機能
│   └── utils/
│       ├── file.go                 # ファイル操作
│       └── metrics.go              # メトリクス収集
├── pkg/
│   ├── artifacts/
│   │   └── manager.go              # 成果物管理
│   ├── health/
│   │   └── checker.go              # ヘルスチェック
│   └── metrics/
│       └── collector.go            # メトリクス収集
├── scripts/
│   ├── build.sh                    # ビルドスクリプト
│   └── install.sh                  # インストールスクリプト
├── tests/
│   ├── integration/                # 統合テスト
│   └── unit/                       # 単体テスト
├── examples/
│   ├── basic-task.sh               # 基本的なタスク例
│   ├── error-handling.sh           # エラーハンドリング例
│   └── artifact-upload.sh          # 成果物アップロード例
├── Dockerfile                      # Dockerイメージ定義
├── go.mod                          # Goモジュール定義
├── go.sum                          # 依存関係チェックサム
├── Makefile                        # ビルド・テスト用Makefile
└── README.md                       # メインREADME
```

## 主要コンポーネント

### 1. APIクライアント (`internal/api/client.go`)
```go
package api

import (
    "bytes"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
)

type Client struct {
    baseURL    string
    token      string
    httpClient *http.Client
}

func NewClient(baseURL, token string) *Client {
    return &Client{
        baseURL: baseURL,
        token:   token,
        httpClient: &http.Client{
            Timeout: 30 * time.Second,
        },
    }
}

func (c *Client) UpdateTaskStatus(taskID, status, message string, progress int) error {
    data := map[string]interface{}{
        "status":   status,
        "message":  message,
        "progress": progress,
    }
    
    if status == "COMPLETED" || status == "FAILED" {
        data["completedAt"] = time.Now().Format(time.RFC3339)
    }
    
    return c.makeRequest("PATCH", fmt.Sprintf("/tasks/%s/status", taskID), data)
}

func (c *Client) SendLog(taskID, level, message string) error {
    data := map[string]interface{}{
        "level":     level,
        "message":   message,
        "timestamp": time.Now().Format(time.RFC3339),
        "metadata": map[string]string{
            "source":  "keruta-agent",
            "version": "1.0.0",
        },
    }
    
    return c.makeRequest("POST", fmt.Sprintf("/tasks/%s/logs", taskID), data)
}

func (c *Client) UploadArtifact(taskID, filePath, description string) error {
    // multipart/form-dataでのファイルアップロード実装
    return nil
}

func (c *Client) makeRequest(method, path string, data interface{}) error {
    jsonData, err := json.Marshal(data)
    if err != nil {
        return err
    }
    
    req, err := http.NewRequest(method, c.baseURL+path, bytes.NewBuffer(jsonData))
    if err != nil {
        return err
    }
    
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", "Bearer "+c.token)
    req.Header.Set("User-Agent", "keruta-agent/1.0.0")
    
    resp, err := c.httpClient.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode >= 400 {
        return fmt.Errorf("API request failed with status: %d", resp.StatusCode)
    }
    
    return nil
}
```

### 2. 設定管理 (`internal/config/config.go`)
```go
package config

import (
    "os"
    "strconv"
    "time"
    
    "github.com/spf13/viper"
)

type Config struct {
    API struct {
        URL     string        `mapstructure:"url"`
        Timeout time.Duration `mapstructure:"timeout"`
    } `mapstructure:"api"`
    
    Logging struct {
        Level  string `mapstructure:"level"`
        Format string `mapstructure:"format"`
    } `mapstructure:"logging"`
    
    Artifacts struct {
        MaxSize    int64  `mapstructure:"max_size"`
        Directory  string `mapstructure:"directory"`
    } `mapstructure:"artifacts"`
    
    ErrorHandling struct {
        AutoFix    bool `mapstructure:"auto_fix"`
        RetryCount int  `mapstructure:"retry_count"`
    } `mapstructure:"error_handling"`
}

func Load() (*Config, error) {
    // 環境変数から設定を読み込み
    viper.SetEnvPrefix("KERUTA")
    viper.AutomaticEnv()
    
    // 設定ファイルから読み込み
    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath("$HOME/.keruta")
    viper.AddConfigPath(".")
    
    // デフォルト値設定
    setDefaults()
    
    var config Config
    if err := viper.Unmarshal(&config); err != nil {
        return nil, err
    }
    
    return &config, nil
}

func setDefaults() {
    viper.SetDefault("api.url", "http://keruta-api:8080")
    viper.SetDefault("api.timeout", "30s")
    viper.SetDefault("logging.level", "INFO")
    viper.SetDefault("logging.format", "json")
    viper.SetDefault("artifacts.max_size", 100*1024*1024) // 100MB
    viper.SetDefault("artifacts.directory", "/.keruta/doc")
    viper.SetDefault("error_handling.auto_fix", true)
    viper.SetDefault("error_handling.retry_count", 3)
}
```

### 3. ログ機能 (`internal/logger/logger.go`)
```go
package logger

import (
    "os"
    
    "github.com/sirupsen/logrus"
)

var log *logrus.Logger

func Init(level, format string) {
    log = logrus.New()
    
    // ログレベル設定
    switch level {
    case "DEBUG":
        log.SetLevel(logrus.DebugLevel)
    case "INFO":
        log.SetLevel(logrus.InfoLevel)
    case "WARN":
        log.SetLevel(logrus.WarnLevel)
    case "ERROR":
        log.SetLevel(logrus.ErrorLevel)
    default:
        log.SetLevel(logrus.InfoLevel)
    }
    
    // ログ形式設定
    if format == "json" {
        log.SetFormatter(&logrus.JSONFormatter{})
    } else {
        log.SetFormatter(&logrus.TextFormatter{})
    }
    
    log.SetOutput(os.Stdout)
}

func Debug(args ...interface{}) {
    log.Debug(args...)
}

func Info(args ...interface{}) {
    log.Info(args...)
}

func Warn(args ...interface{}) {
    log.Warn(args...)
}

func Error(args ...interface{}) {
    log.Error(args...)
}

func Fatal(args ...interface{}) {
    log.Fatal(args...)
}
```

## 実装例

### 1. メインエントリーポイント (`cmd/keruta-agent/main.go`)
```go
package main

import (
    "os"
    
    "github.com/spf13/cobra"
    
    "keruta-agent/internal/commands"
    "keruta-agent/internal/config"
    "keruta-agent/internal/logger"
)

func main() {
    // 設定読み込み
    cfg, err := config.Load()
    if err != nil {
        os.Exit(1)
    }
    
    // ログ初期化
    logger.Init(cfg.Logging.Level, cfg.Logging.Format)
    
    // ルートコマンド作成
    rootCmd := commands.NewRootCommand(cfg)
    
    if err := rootCmd.Execute(); err != nil {
        logger.Error("Command execution failed:", err)
        os.Exit(1)
    }
}
```

### 2. ルートコマンド (`internal/commands/root.go`)
```go
package commands

import (
    "github.com/spf13/cobra"
    "keruta-agent/internal/config"
)

func NewRootCommand(cfg *config.Config) *cobra.Command {
    rootCmd := &cobra.Command{
        Use:   "keruta",
        Short: "keruta-agent - Kubernetes Pod内でタスクを実行するためのCLIツール",
        Long: `keruta-agentは、kerutaシステムによってKubernetes Jobとして実行されるPod内で動作するCLIツールです。
タスクの実行状況をkeruta APIサーバーに報告し、成果物の保存、ログの収集、エラーハンドリングなどの機能を提供します。`,
        Version: "1.0.0",
    }
    
    // サブコマンド追加
    rootCmd.AddCommand(
        NewStartCommand(cfg),
        NewSuccessCommand(cfg),
        NewFailCommand(cfg),
        NewProgressCommand(cfg),
        NewLogCommand(cfg),
        NewArtifactCommand(cfg),
        NewHealthCommand(cfg),
        NewConfigCommand(cfg),
    )
    
    return rootCmd
}
```

### 3. Startコマンド (`internal/commands/start.go`)
```go
package commands

import (
    "os"
    
    "github.com/spf13/cobra"
    "keruta-agent/internal/api"
    "keruta-agent/internal/config"
    "keruta-agent/internal/logger"
)

func NewStartCommand(cfg *config.Config) *cobra.Command {
    var taskID, apiURL, logLevel string
    
    cmd := &cobra.Command{
        Use:   "start",
        Short: "タスクの実行を開始し、ステータスをPROCESSINGに更新",
        RunE: func(cmd *cobra.Command, args []string) error {
            // 環境変数から値を取得
            if taskID == "" {
                taskID = os.Getenv("KERUTA_TASK_ID")
            }
            if apiURL == "" {
                apiURL = os.Getenv("KERUTA_API_URL")
            }
            if apiURL == "" {
                apiURL = cfg.API.URL
            }
            
            token := os.Getenv("KERUTA_API_TOKEN")
            if token == "" {
                logger.Fatal("KERUTA_API_TOKEN environment variable is required")
            }
            
            // APIクライアント作成
            client := api.NewClient(apiURL, token)
            
            // タスク開始
            err := client.UpdateTaskStatus(taskID, "PROCESSING", "タスクを開始します", 0)
            if err != nil {
                logger.Error("Failed to start task:", err)
                return err
            }
            
            logger.Info("Task started successfully")
            return nil
        },
    }
    
    cmd.Flags().StringVar(&taskID, "task-id", "", "タスクID（環境変数KERUTA_TASK_IDから自動取得）")
    cmd.Flags().StringVar(&apiURL, "api-url", "", "keruta APIのURL（環境変数KERUTA_API_URLから自動取得）")
    cmd.Flags().StringVar(&logLevel, "log-level", "INFO", "ログレベル（DEBUG, INFO, WARN, ERROR）")
    
    return cmd
}
```

## サンプルスクリプト

### 1. 基本的なタスク例 (`examples/basic-task.sh`)
```bash
#!/bin/bash
set -e

# 環境変数設定
export KERUTA_TASK_ID="123e4567-e89b-12d3-a456-426614174000"
export KERUTA_API_URL="http://keruta-api:8080"
export KERUTA_API_TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

echo "=== 基本的なタスク実行例 ==="

# タスク開始
echo "1. タスクを開始します..."
keruta start --log-level INFO

# 進捗報告
echo "2. 進捗を報告します..."
keruta progress 25 --message "データの読み込み中..."

# 処理実行（シミュレーション）
echo "3. データ処理を実行します..."
sleep 5

# 進捗報告
echo "4. 進捗を更新します..."
keruta progress 75 --message "データの処理中..."

# 成果物の作成
echo "5. 成果物を作成します..."
mkdir -p /.keruta/doc
echo "処理結果: データ処理が正常に完了しました" > /.keruta/doc/result.txt
echo "処理時間: 5秒" >> /.keruta/doc/result.txt

# タスク成功
echo "6. タスクを完了します..."
keruta success --message "データ処理が完了しました"

echo "=== タスク実行完了 ==="
```

### 2. エラーハンドリング例 (`examples/error-handling.sh`)
```bash
#!/bin/bash
set -e

# 環境変数設定
export KERUTA_TASK_ID="123e4567-e89b-12d3-a456-426614174000"
export KERUTA_API_URL="http://keruta-api:8080"
export KERUTA_API_TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

echo "=== エラーハンドリング例 ==="

# タスク開始
keruta start

# エラーハンドリング設定
trap 'keruta fail --message "予期しないエラーが発生しました: $?"' ERR

# 構造化ログ送信
keruta log INFO "処理を開始します"
keruta log DEBUG "設定ファイルを読み込み中..."

# リスクのある処理（エラーが発生する可能性）
echo "リスクのある処理を実行します..."
python risky_operation.py

# 成功時の処理
keruta log INFO "処理が完了しました"
keruta success --message "処理が正常に完了しました"

echo "=== エラーハンドリング例完了 ==="
```

### 3. 成果物アップロード例 (`examples/artifact-upload.sh`)
```bash
#!/bin/bash
set -e

# 環境変数設定
export KERUTA_TASK_ID="123e4567-e89b-12d3-a456-426614174000"
export KERUTA_API_URL="http://keruta-api:8080"
export KERUTA_API_TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

echo "=== 成果物アップロード例 ==="

# タスク開始
keruta start

# 成果物ディレクトリ作成
mkdir -p /.keruta/doc

# 成果物作成
echo "月次レポート" > /.keruta/doc/report.txt
echo "生成日時: $(date)" >> /.keruta/doc/report.txt

# 手動で成果物を追加
keruta artifact add /.keruta/doc/report.txt --description "月次レポート"

# 成果物一覧表示
keruta artifact list

# タスク完了
keruta success --message "成果物の作成とアップロードが完了しました"

echo "=== 成果物アップロード例完了 ==="
```

## テスト

### 1. 単体テスト (`tests/unit/api_test.go`)
```go
package api

import (
    "testing"
    "time"
)

func TestClient_UpdateTaskStatus(t *testing.T) {
    client := NewClient("http://test-api:8080", "test-token")
    
    // テストケース
    testCases := []struct {
        name     string
        taskID   string
        status   string
        message  string
        progress int
        wantErr  bool
    }{
        {
            name:     "正常なタスク開始",
            taskID:   "test-task-id",
            status:   "PROCESSING",
            message:  "タスクを開始します",
            progress: 0,
            wantErr:  false,
        },
        {
            name:     "正常なタスク完了",
            taskID:   "test-task-id",
            status:   "COMPLETED",
            message:  "タスクが完了しました",
            progress: 100,
            wantErr:  false,
        },
    }
    
    for _, tc := range testCases {
        t.Run(tc.name, func(t *testing.T) {
            err := client.UpdateTaskStatus(tc.taskID, tc.status, tc.message, tc.progress)
            if (err != nil) != tc.wantErr {
                t.Errorf("UpdateTaskStatus() error = %v, wantErr %v", err, tc.wantErr)
            }
        })
    }
}
```

### 2. 統合テスト (`tests/integration/basic_flow_test.go`)
```go
package integration

import (
    "os"
    "testing"
    "time"
    
    "keruta-agent/internal/api"
    "keruta-agent/internal/logger"
)

func TestBasicTaskFlow(t *testing.T) {
    // テスト環境設定
    os.Setenv("KERUTA_TASK_ID", "test-task-id")
    os.Setenv("KERUTA_API_URL", "http://test-api:8080")
    os.Setenv("KERUTA_API_TOKEN", "test-token")
    
    logger.Init("DEBUG", "text")
    
    client := api.NewClient("http://test-api:8080", "test-token")
    
    // 1. タスク開始
    err := client.UpdateTaskStatus("test-task-id", "PROCESSING", "テスト開始", 0)
    if err != nil {
        t.Fatalf("Failed to start task: %v", err)
    }
    
    // 2. 進捗更新
    err = client.UpdateTaskStatus("test-task-id", "PROCESSING", "テスト実行中", 50)
    if err != nil {
        t.Fatalf("Failed to update progress: %v", err)
    }
    
    // 3. ログ送信
    err = client.SendLog("test-task-id", "INFO", "テストログ")
    if err != nil {
        t.Fatalf("Failed to send log: %v", err)
    }
    
    // 4. タスク完了
    err = client.UpdateTaskStatus("test-task-id", "COMPLETED", "テスト完了", 100)
    if err != nil {
        t.Fatalf("Failed to complete task: %v", err)
    }
}
```

## デプロイ

### 1. Dockerfile
```dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app

# 依存関係コピー
COPY go.mod go.sum ./
RUN go mod download

# ソースコードコピー
COPY . .

# バイナリビルド
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o keruta-agent ./cmd/keruta-agent

# 実行イメージ
FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

# バイナリコピー
COPY --from=builder /app/keruta-agent .

# 実行権限付与
RUN chmod +x keruta-agent

# エントリーポイント設定
ENTRYPOINT ["./keruta-agent"]
```

### 2. Makefile
```makefile
.PHONY: build test clean docker-build docker-run

# ビルド
build:
	go build -o bin/keruta-agent ./cmd/keruta-agent

# テスト実行
test:
	go test ./...

# 統合テスト実行
test-integration:
	go test ./tests/integration/...

# クリーンアップ
clean:
	rm -rf bin/
	go clean

# Dockerイメージビルド
docker-build:
	docker build -t keruta-agent:latest .

# Dockerコンテナ実行
docker-run:
	docker run --rm \
		-e KERUTA_TASK_ID=test-task-id \
		-e KERUTA_API_URL=http://host.docker.internal:8080 \
		-e KERUTA_API_TOKEN=test-token \
		keruta-agent:latest

# インストール
install: build
	sudo cp bin/keruta-agent /usr/local/bin/
	sudo chmod +x /usr/local/bin/keruta-agent

# リリースビルド
release: clean
	GOOS=linux GOARCH=amd64 go build -o bin/keruta-agent-linux-amd64 ./cmd/keruta-agent
	GOOS=darwin GOARCH=amd64 go build -o bin/keruta-agent-darwin-amd64 ./cmd/keruta-agent
	GOOS=windows GOARCH=amd64 go build -o bin/keruta-agent-windows-amd64.exe ./cmd/keruta-agent
```

### 3. Kubernetes Job例
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: keruta-task-example
spec:
  template:
    spec:
      containers:
      - name: task-runner
        image: keruta-agent:latest
        env:
        - name: KERUTA_TASK_ID
          value: "123e4567-e89b-12d3-a456-426614174000"
        - name: KERUTA_API_URL
          value: "http://keruta-api:8080"
        - name: KERUTA_API_TOKEN
          valueFrom:
            secretKeyRef:
              name: keruta-api-token
              key: token
        command: ["/bin/bash", "-c"]
        args:
        - |
          #!/bin/bash
          set -e
          
          # keruta-agentを使用してタスク実行
          keruta start
          
          # 実際の処理
          echo "タスク処理を実行中..."
          sleep 10
          
          # 成果物作成
          mkdir -p /.keruta/doc
          echo "処理完了" > /.keruta/doc/result.txt
          
          # タスク完了
          keruta success --message "タスクが正常に完了しました"
      restartPolicy: Never
  backoffLimit: 3
```

この実装例により、keruta-agentの基本的な機能と使用方法を理解できます。実際の開発では、これらのコードをベースに、プロジェクトの要件に合わせてカスタマイズしてください。 