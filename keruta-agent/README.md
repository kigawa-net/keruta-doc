# keruta-agent

> **概要**: kerutaによって実行されるjobで利用できるkerutaコマンドを実装するサブプロジェクトです。Kubernetes Pod内でタスクを実行する際に使用されるCLIツールです。

## 目次
- [概要](#概要)
- [インストール](#インストール)
- [アーキテクチャ](#アーキテクチャ)
- [機能仕様](#機能仕様)
- [コマンド仕様](#コマンド仕様)
- [環境変数](#環境変数)
- [セットアップ](#セットアップ)
- [使用方法](#使用方法)
- [技術スタック](#技術スタック)
- [プロジェクト構造](#プロジェクト構造)
- [関連リンク](#関連リンク)

## インストール

keruta-agentはシェルスクリプトで簡単にインストールできます。

### 例: インストール用シェルスクリプト

```bash
#!/bin/bash
set -e

# 最新バージョンのURL（例: v1.0.0、適宜修正）
AGENT_VERSION="v1.0.0"
AGENT_URL="https://github.com/your-org/keruta-agent/releases/download/${AGENT_VERSION}/keruta-agent-linux-amd64"
INSTALL_PATH="/usr/local/bin/keruta-agent"

# ダウンロード
curl -L -o keruta-agent "${AGENT_URL}"

# 実行権限付与
chmod +x keruta-agent

# 配置
sudo mv keruta-agent "${INSTALL_PATH}"

# パス確認
if ! echo "$PATH" | grep -q "/usr/local/bin"; then
  echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
  export PATH=$PATH:/usr/local/bin
fi

echo "keruta-agent installed to ${INSTALL_PATH}"
keruta-agent --version
```

## 概要
`keruta-agent`は、kerutaシステムによってKubernetes Jobとして実行されるPod内で動作するCLIツールです。keruta APIからスクリプトを取得し、サブプロセスとして実行します。サブプロセスが入力待ち状態になった場合、自動的にタスクの状態を入力待ち状態に変更し、管理パネルからの標準入力送信を可能にします。

### 主な機能
- keruta APIからのスクリプト取得
- サブプロセスとしてスクリプトの実行
- サブプロセスの入力待ち状態の自動検出
- タスクステータスの更新（PROCESSING → COMPLETED/FAILED/WAITING_FOR_INPUT）
- 実行ログの収集と送信
- エラー発生時の自動修正タスク作成
- タスク実行時間の計測
- ヘルスチェック機能
- 管理パネルからの標準入力送信

## アーキテクチャ
```
┌─────────────────────────────────────────────────────────┐
│                Kubernetes Pod                           │
│                                                         │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              keruta-agent                          │ │
│  │                                                     │ │
│  │  ┌─────────────────┐    ┌─────────────────────────┐ │ │
│  │  │   User Script   │    │   Input Detection      │ │ │
│  │  │   (Subprocess)  │◄───│   & State Management   │ │ │
│  │  │                 │    │                         │ │ │
│  │  │ #!/bin/bash     │    │  • タスクステータス更新 │ │ │
│  │  │ read input      │    │  • ログ収集             │ │ │
│  │  │ echo "hello"    │    │  • エラーハンドリング   │ │ │
│  │  │ exit 0          │    │  • 入力待ち検出         │ │ │
│  │  └─────────────────┘    └─────────────────────────┘ │ │
│  │                                                     │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                         │
└─────────────────────────────────────────────────────────┘
                                │
                                ▼
                    ┌─────────────────────────┐
                    │     keruta API          │
                    │   (Spring Boot)         │
                    │                         │
                    │  • タスク状態更新       │
                    │  • ログ保存             │
                    │  • 標準入力送信         │
                    └─────────────────────────┘
```

## 機能仕様

### 1. スクリプト取得
- タスクIDからkeruta APIでスクリプトを取得
- スクリプトの言語、ファイル名、パラメータを取得
- 作業ディレクトリにスクリプトファイルを作成
- 実行権限を付与

### 2. サブプロセス実行
- 取得したスクリプトをサブプロセスとして実行
- 標準出力・標準エラー出力のリアルタイムキャプチャ
- プロセス終了コードの監視

### 3. 入力待ち状態の自動検出
- サブプロセスが標準入力を待機している状態の検出
- 入力待ち状態になった場合、タスク状態を`WAITING_FOR_INPUT`に変更
- 管理パネルに通知して入力フォームを表示

### 4. 標準入力送信
- 管理パネルからサブプロセスの標準入力にテキストを送信
- 実行中のいつでも送信可能
- 送信されたテキストはサブプロセスに即座に反映

### 5. タスクライフサイクル管理
- **開始**: タスクをPROCESSING状態に更新
- **入力待ち**: サブプロセスの入力待ちを検出し、WAITING_FOR_INPUT状態に更新
- **成功**: サブプロセスが正常終了した場合、COMPLETED状態に更新
- **失敗**: サブプロセスが異常終了した場合、FAILED状態に更新

### 6. ログ管理
- サブプロセスの標準出力・標準エラー出力の自動キャプチャ
- 構造化ログ（JSON形式）のサポート
- ログレベル制御（DEBUG, INFO, WARN, ERROR）
- ログローテーション機能

### 7. エラーハンドリング
- サブプロセスの異常終了の検出
- エラー詳細の自動収集
- 自動修正タスクの作成（設定可能）
- リトライ機能（設定可能）

### 8. 監視・メトリクス
- 実行時間の計測
- リソース使用量の監視
- ヘルスチェック機能
- メトリクスのkeruta APIへの送信

## コマンド仕様

### メインコマンド

#### `keruta-agent execute`
指定されたタスクIDのスクリプトをkeruta APIから取得し、サブプロセスとして実行します。入力待ち状態を自動検出します。

```bash
keruta-agent execute [options]
```

**オプション:**
- `--task-id <id>`: タスクIDを指定（環境変数から自動取得がデフォルト）
- `--api-url <url>`: keruta APIのURL（環境変数から自動取得がデフォルト）
- `--work-dir <path>`: 作業ディレクトリ（デフォルト: /work）
- `--log-level <level>`: ログレベル（DEBUG, INFO, WARN, ERROR）
- `--auto-detect-input`: 入力待ち状態の自動検出（デフォルト: true）

**例:**
```bash
# 基本的な実行
keruta-agent execute --task-id task123

# カスタム設定での実行
keruta-agent execute \
    --task-id task123 \
    --api-url http://keruta-api:8080 \
    --work-dir /work \
    --log-level DEBUG
```

### ユーティリティコマンド

#### `keruta-agent log`
構造化ログを送信します。

```bash
keruta-agent log <level> <message> [options]
```

**引数:**
- `level`: ログレベル（DEBUG, INFO, WARN, ERROR）
- `message`: ログメッセージ

**例:**
```bash
keruta-agent log INFO "データベースクエリを実行中..."
```

#### `keruta-agent health`
ヘルスチェックを実行します。

```bash
keruta-agent health [options]
```

**オプション:**
- `--check-api`: keruta APIとの接続確認
- `--check-disk`: ディスク容量確認
- `--check-memory`: メモリ使用量確認

#### `keruta-agent config`
設定を表示・更新します。

```bash
keruta-agent config show
keruta-agent config set <key> <value>
```

## 環境変数

### 必須環境変数
| 変数名 | 説明 | 例 |
|--------|------|-----|
| `KERUTA_TASK_ID` | タスクの一意識別子 | `123e4567-e89b-12d3-a456-426614174000` |
| `KERUTA_API_URL` | keruta APIのURL | `http://keruta-api:8080` |
| `KERUTA_API_TOKEN` | API認証トークン | `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...` |

### オプション環境変数
| 変数名 | 説明 | デフォルト値 |
|--------|------|-------------|
| `KERUTA_LOG_LEVEL` | ログレベル | `INFO` |
| `KERUTA_AUTO_FIX_ENABLED` | 自動修正タスク作成 | `true` |
| `KERUTA_RETRY_COUNT` | リトライ回数 | `3` |
| `KERUTA_TIMEOUT` | API呼び出しタイムアウト（秒） | `30` |
| `KERUTA_AUTO_DETECT_INPUT` | 入力待ち状態の自動検出 | `true` |

## セットアップ

### 1. ビルド
```bash
# 依存関係のインストール
go mod download

# バイナリのビルド
go build -o keruta-agent ./cmd/keruta-agent

# Dockerイメージのビルド
docker build -t keruta-agent:latest .
```

### 2. インストール
```bash
# システム全体にインストール
sudo cp keruta-agent /usr/local/bin/
sudo chmod +x /usr/local/bin/keruta-agent

# または、PATHに追加
export PATH=$PATH:/path/to/keruta-agent
```

### 3. 設定
```bash
# 設定ファイルの作成
mkdir -p ~/.keruta
cat > ~/.keruta/config.yaml << EOF
api:
  url: http://keruta-api:8080
  timeout: 30s
logging:
  level: INFO
  format: json
error_handling:
  auto_fix: true
  retry_count: 3
input_detection:
  auto_detect: true
EOF
```

## 使用方法

### 基本的な使用例
```bash
#!/bin/bash
# script.sh - 入力待ちを含むサンプルスクリプト

echo "処理を開始します"
echo "名前を入力してください:"
read -r name

echo "こんにちは、$name さん！"
echo "年齢を入力してください:"
read -r age

echo "あなたは $age 歳ですね"
echo "処理が完了しました"
```

```bash
# keruta-agentでスクリプトを実行
keruta-agent execute --task-id task123
```

### Kubernetes Jobでの使用例
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: keruta-task-example
spec:
  template:
    spec:
      containers:
        - name: main
          image: keruta-agent:latest
          command: 
            - "keruta-agent"
            - "execute"
            - "--task-id"
            - "$(KERUTA_TASK_ID)"
            - "--api-url"
            - "$(KERUTA_API_URL)"
          env:
            - name: KERUTA_TASK_ID
              value: "123"
            - name: KERUTA_API_URL
              value: "http://keruta-api:8080"
            - name: KERUTA_API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: keruta-api-token
                  key: token
      restartPolicy: Never
```

### エラーハンドリング例
```bash
#!/bin/bash
# error-handling-script.sh

set -e

echo "エラーハンドリングテストを開始します"

# 存在しないファイルにアクセス（エラーを発生させる）
cat /nonexistent/file

echo "この行は実行されません"
```

```bash
# エラーハンドリング付きで実行
keruta-agent execute --task-id task123
```

### 構造化ログの使用例
```bash
#!/bin/bash
# logging-script.sh

echo "処理を開始します"
keruta-agent log INFO "スクリプト実行開始"

echo "設定ファイルを読み込み中..."
keruta-agent log DEBUG "設定ファイル読み込み中"

echo "データ処理中..."
keruta-agent log INFO "データ処理実行中"

echo "処理が完了しました"
keruta-agent log INFO "スクリプト実行完了"
```

```bash
# ログ付きで実行
keruta-agent execute --task-id task123
```

## 技術スタック
- **言語**: Go 1.21+
- **フレームワーク**: Cobra（CLIフレームワーク）
- **HTTP クライアント**: net/http（標準ライブラリ）
- **設定管理**: Viper
- **ログ**: logrus
- **テスト**: testify
- **プロセス管理**: os/exec（標準ライブラリ）

## プロジェクト構造
```
keruta-agent/
├── cmd/
│   └── keruta-agent/          # メインエントリーポイント
├── internal/
│   ├── api/                   # keruta APIクライアント
│   ├── commands/              # CLIコマンド実装
│   ├── config/                # 設定管理
│   ├── logger/                # ログ機能
│   ├── process/               # サブプロセス管理
│   └── utils/                 # ユーティリティ関数
├── pkg/
│   ├── health/                # ヘルスチェック
│   └── metrics/               # メトリクス収集
├── scripts/                   # ビルド・デプロイスクリプト
├── tests/                     # テストファイル
├── Dockerfile                 # Dockerイメージ定義
├── go.mod                     # Goモジュール定義
├── go.sum                     # 依存関係チェックサム
└── README.md                  # このファイル
```

## 関連リンク
- [keruta プロジェクト詳細](../keruta/projectDetails.md)
- [keruta Kubernetes連携](../keruta/kubernetes/kubernetesIntegration.md)
- [keruta タスクキューシステム設計](../keruta/taskQueueSystemDesign.md)
- [keruta API仕様](../keruta/apiSpec.md)
- [keruta-agent API仕様](apiSpec.md) 