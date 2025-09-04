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
│   │   ├── execute.go              # executeコマンド（統合実行）
│   │   ├── log.go                  # logコマンド
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
│   └── error-handling.sh           # エラーハンドリング例
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

func (c *Client) GetTask(taskID string) (*Task, error) {
    req, err := http.NewRequest("GET", c.baseURL+"/tasks/"+taskID, nil)
    if err != nil {
        return nil, err
    }
    
    req.Header.Set("Authorization", "Bearer "+c.token)
    req.Header.Set("User-Agent", "keruta-agent/1.0.0")
    
    resp, err := c.httpClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode >= 400 {
        return nil, fmt.Errorf("API request failed with status: %d", resp.StatusCode)
    }
    
    var task Task
    if err := json.NewDecoder(resp.Body).Decode(&task); err != nil {
        return nil, err
    }
    
    return &task, nil
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

func (c *Client) UploadDocument(taskID, filePath string) error {
    file, err := os.Open(filePath)
    if err != nil {
        return err
    }
    defer file.Close()
    
    body := &bytes.Buffer{}
    writer := multipart.NewWriter(body)
    part, err := writer.CreateFormFile("file", filepath.Base(filePath))
    if err != nil {
        return err
    }
    
    if _, err := io.Copy(part, file); err != nil {
        return err
    }
    writer.Close()
    
    req, err := http.NewRequest("POST", c.baseURL+"/tasks/"+taskID+"/documents", body)
    if err != nil {
        return err
    }
    
    req.Header.Set("Content-Type", writer.FormDataContentType())
    req.Header.Set("Authorization", "Bearer "+c.token)
    
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

type Task struct {
    ID           string `json:"id"`
    Title        string `json:"title"`
    Description  string `json:"description"`
    Status       string `json:"status"`
    RepositoryID string `json:"repositoryId"`
    DocumentID   string `json:"documentId"`
    CreatedAt    string `json:"createdAt"`
    UpdatedAt    string `json:"updatedAt"`
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
    viper.SetDefault("api.timeout", 30*time.Second)
    viper.SetDefault("logging.level", "INFO")
    viper.SetDefault("logging.format", "text")
    viper.SetDefault("error_handling.auto_fix", true)
    viper.SetDefault("error_handling.retry_count", 3)
}
```

### 3. 統合実行コマンド (`internal/commands/execute.go`)
```go
package commands

import (
    "fmt"
    "os"
    "os/exec"
    "path/filepath"
    
    "keruta-agent/internal/api"
    "keruta-agent/internal/logger"
)

type ExecuteCommand struct {
    taskID       string
    apiURL       string
    workDir      string
    autoStart    bool
    skipInit     bool
    autoCleanup  bool
}

func NewExecuteCommand() *ExecuteCommand {
    return &ExecuteCommand{
        workDir:     "/work",
        autoStart:   true,
        skipInit:    false,
        autoCleanup: true,
    }
}

func (c *ExecuteCommand) Execute() error {
    logger.Info("keruta-agent統合実行を開始します")
    
    // APIクライアントの初期化
    client := api.NewClient(c.apiURL, os.Getenv("KERUTA_API_TOKEN"))
    
    // タスク情報の取得
    task, err := client.GetTask(c.taskID)
    if err != nil {
        return fmt.Errorf("タスク情報の取得に失敗しました: %w", err)
    }
    
    logger.Info("タスク情報を取得しました", "task_id", task.ID, "title", task.Title)
    
    // 初期化処理
    if !c.skipInit {
        if err := c.initialize(client, task); err != nil {
            return fmt.Errorf("初期化処理に失敗しました: %w", err)
        }
    }
    
    // タスク開始
    if c.autoStart {
        if err := client.UpdateTaskStatus(c.taskID, "PROCESSING", "タスクを開始します", 0); err != nil {
            return fmt.Errorf("タスク開始に失敗しました: %w", err)
        }
    }
    
    // タスク実行
    if err := c.executeTask(client, task); err != nil {
        return fmt.Errorf("タスク実行に失敗しました: %w", err)
    }
    
    // クリーンアップ処理
    if c.autoCleanup {
        if err := c.cleanup(client, task); err != nil {
            return fmt.Errorf("クリーンアップ処理に失敗しました: %w", err)
        }
    }
    
    logger.Info("keruta-agent統合実行が完了しました")
    return nil
}

func (c *ExecuteCommand) initialize(client *api.Client, task *api.Task) error {
    logger.Info("初期化処理を開始します")
    
    // 作業ディレクトリの作成
    if err := os.MkdirAll(c.workDir, 0755); err != nil {
        return err
    }
    
    // リポジトリのクローン
    if task.RepositoryID != "" {
        if err := c.cloneRepository(task.RepositoryID); err != nil {
            return err
        }
    }
    
    // インストールスクリプトの実行
    if err := c.runInstallScript(); err != nil {
        return err
    }
    
    // ドキュメントの取得
    if task.DocumentID != "" {
        if err := c.fetchDocument(client, task.DocumentID); err != nil {
            return err
        }
    }
    
    logger.Info("初期化処理が完了しました")
    return nil
}

func (c *ExecuteCommand) cloneRepository(repositoryID string) error {
    logger.Info("リポジトリをクローン中...", "repository_id", repositoryID)
    
    // git cloneコマンドの実行
    cmd := exec.Command("git", "clone", "--depth", "1", "--single-branch", 
        fmt.Sprintf("https://github.com/example/%s.git", repositoryID), c.workDir)
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    
    if err := cmd.Run(); err != nil {
        return err
    }
    
    // git-exclude設定
    excludeFile := filepath.Join(c.workDir, ".git", "info", "exclude")
    if err := os.MkdirAll(filepath.Dir(excludeFile), 0755); err != nil {
        return err
    }
    
    f, err := os.OpenFile(excludeFile, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        return err
    }
    defer f.Close()
    
    if _, err := f.WriteString("/.keruta\n"); err != nil {
        return err
    }
    
    logger.Info("リポジトリのクローンが完了しました")
    return nil
}

func (c *ExecuteCommand) runInstallScript() error {
    installScript := filepath.Join(c.workDir, ".keruta", "install.sh")
    
    if _, err := os.Stat(installScript); os.IsNotExist(err) {
        logger.Info("インストールスクリプトが見つかりません。スキップします。")
        return nil
    }
    
    logger.Info("インストールスクリプトを実行中...")
    
    cmd := exec.Command("sh", installScript)
    cmd.Dir = c.workDir
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    
    return cmd.Run()
}

func (c *ExecuteCommand) fetchDocument(client *api.Client, documentID string) error {
    logger.Info("ドキュメントを取得中...", "document_id", documentID)
    
    // APIからドキュメントを取得する処理
    // 実装は省略
    
    logger.Info("ドキュメントの取得が完了しました")
    return nil
}

func (c *ExecuteCommand) executeTask(client *api.Client, task *api.Task) error {
    logger.Info("タスク実行を開始します")
    
    // 進捗更新
    if err := client.UpdateTaskStatus(c.taskID, "PROCESSING", "タスク実行中", 50); err != nil {
        return err
    }
    
    // 実際のタスク処理
    // ここではサンプルとしてsleepを実行
    cmd := exec.Command("sleep", "10")
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    
    if err := cmd.Run(); err != nil {
        return err
    }
    
    // タスク完了
    if err := client.UpdateTaskStatus(c.taskID, "COMPLETED", "タスクが正常に完了しました", 100); err != nil {
        return err
    }
    
    logger.Info("タスク実行が完了しました")
    return nil
}

func (c *ExecuteCommand) cleanup(client *api.Client, task *api.Task) error {
    logger.Info("クリーンアップ処理を開始します")
    
    // 成果物の収集とアップロード
    docDir := filepath.Join(c.workDir, ".keruta", "doc")
    if _, err := os.Stat(docDir); !os.IsNotExist(err) {
        files, err := os.ReadDir(docDir)
        if err != nil {
            return err
        }
        
        for _, file := range files {
            if !file.IsDir() {
                filePath := filepath.Join(docDir, file.Name())
                if err := client.UploadDocument(c.taskID, filePath); err != nil {
                    logger.Warn("成果物のアップロードに失敗しました", "file", file.Name(), "error", err)
                } else {
                    logger.Info("成果物をアップロードしました", "file", file.Name())
                }
            }
        }
    }
    
    logger.Info("クリーンアップ処理が完了しました")
    return nil
}
```

## サンプルスクリプト

### 1. 基本的なタスク実行 (`examples/basic-task.sh`)
```bash
#!/bin/bash
set -e

# 環境変数設定
export KERUTA_TASK_ID="test-task-id"
export KERUTA_API_URL="http://keruta-api:8080"
export KERUTA_API_TOKEN="test-token"

# 統合的なタスク実行（タスク情報はAPIから自動取得）
keruta execute --task-id "$KERUTA_TASK_ID"

echo "タスクが正常に完了しました"
```

### 2. エラーハンドリング (`examples/error-handling.sh`)
```bash
#!/bin/bash
set -e

# エラーハンドリング
trap 'keruta log ERROR "予期しないエラーが発生しました: $?"' ERR

# 統合的なタスク実行
keruta execute --task-id "$KERUTA_TASK_ID"

echo "タスクが正常に完了しました"
```

## テスト

### 1. 単体テスト (`tests/unit/api_client_test.go`)
```go
package unit

import (
    "testing"
    
    "keruta-agent/internal/api"
)

func TestGetTask(t *testing.T) {
    client := api.NewClient("http://test-api:8080", "test-token")
    
    task, err := client.GetTask("test-task-id")
    if err != nil {
        t.Fatalf("GetTask() error = %v", err)
    }
    
    if task.ID != "test-task-id" {
        t.Errorf("Expected task ID 'test-task-id', got '%s'", task.ID)
    }
}

func TestUpdateTaskStatus(t *testing.T) {
    client := api.NewClient("http://test-api:8080", "test-token")
    
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
    
    // 1. タスク情報取得
    task, err := client.GetTask("test-task-id")
    if err != nil {
        t.Fatalf("Failed to get task: %v", err)
    }
    
    // 2. タスク開始
    err = client.UpdateTaskStatus("test-task-id", "PROCESSING", "テスト開始", 0)
    if err != nil {
        t.Fatalf("Failed to start task: %v", err)
    }
    
    // 3. 進捗更新
    err = client.UpdateTaskStatus("test-task-id", "PROCESSING", "テスト実行中", 50)
    if err != nil {
        t.Fatalf("Failed to update progress: %v", err)
    }
    
    // 4. ログ送信
    err = client.SendLog("test-task-id", "INFO", "テストログ")
    if err != nil {
        t.Fatalf("Failed to send log: %v", err)
    }
    
    // 5. タスク完了
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

RUN apk --no-cache add ca-certificates git

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
		keruta-agent:latest execute

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
        command: ["keruta-agent", "execute"]
        args:
        - "--task-id"
        - "$(KERUTA_TASK_ID)"
        - "--api-url"
        - "$(KERUTA_API_URL)"
        - "--work-dir"
        - "/work"
        - "--auto-cleanup"
      restartPolicy: Never
  backoffLimit: 3
```

この実装例により、keruta-agentの統合的な機能と使用方法を理解できます。実際の開発では、これらのコードをベースに、プロジェクトの要件に合わせてカスタマイズしてください。 