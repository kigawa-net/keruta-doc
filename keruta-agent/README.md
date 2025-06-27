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
`keruta-agent`は、kerutaシステムによってKubernetes Jobとして実行されるPod内で動作するCLIツールです。タスクの実行状況をkeruta APIサーバーに報告し、ログの収集、エラーハンドリングなどの機能を提供します。

### 主な機能
- タスクステータスの更新（PROCESSING → COMPLETED/FAILED）
- 実行ログの収集と送信
- エラー発生時の自動修正タスク作成
- タスク実行時間の計測
- ヘルスチェック機能

## アーキテクチャ
```
┌─────────────────────────────────────────────────────────┐
│                Kubernetes Pod                           │
│                                                         │
│  ┌─────────────────┐    ┌─────────────────────────────┐ │
│  │   User Script   │    │      keruta-agent          │ │
│  │                 │    │                             │ │
│  │ #!/bin/bash     │───▶│  • タスクステータス更新     │ │
│  │ keruta start    │    │  • ログ収集                 │ │
│  │ # 処理実行      │    │  • エラーハンドリング       │ │
│  │ keruta success  │    │                             │ │
│  └─────────────────┘    └─────────────────────────────┘ │
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
                    └─────────────────────────┘
```

## 機能仕様

### 1. タスクライフサイクル管理
- **開始**: `keruta start` - タスクをPROCESSING状態に更新
- **成功**: `keruta success` - タスクをCOMPLETED状態に更新
- **失敗**: `keruta fail` - タスクをFAILED状態に更新
- **進捗**: `keruta progress <percentage>` - 進捗率を更新

### 2. ログ管理
- 標準出力・標準エラー出力の自動キャプチャ
- 構造化ログ（JSON形式）のサポート
- ログレベル制御（DEBUG, INFO, WARN, ERROR）
- ログローテーション機能

### 3. エラーハンドリング
- 予期しないエラー発生時の自動検出
- エラー詳細の自動収集
- 自動修正タスクの作成（設定可能）
- リトライ機能（設定可能）

### 4. 監視・メトリクス
- 実行時間の計測
- リソース使用量の監視
- ヘルスチェック機能
- メトリクスのkeruta APIへの送信

## コマンド仕様

### 基本コマンド

#### `keruta start`
タスクの実行を開始し、ステータスをPROCESSINGに更新します。

```bash
keruta start [options]
```

**オプション:**
- `--task-id <id>`: タスクIDを指定（環境変数から自動取得がデフォルト）
- `--api-url <url>`: keruta APIのURL（環境変数から自動取得がデフォルト）
- `--log-level <level>`: ログレベル（DEBUG, INFO, WARN, ERROR）

**例:**
```bash
#!/bin/bash
keruta start --log-level INFO
# タスク処理を実行
```

#### `keruta success`
タスクの成功を報告し、ステータスをCOMPLETEDに更新します。

```bash
keruta success [options]
```

**オプション:**
- `--message <message>`: 成功メッセージ

**例:**
```bash
# 処理完了後
keruta success --message "データ処理が正常に完了しました"
```

#### `keruta fail`
タスクの失敗を報告し、ステータスをFAILEDに更新します。

```bash
keruta fail [options]
```

**オプション:**
- `--message <message>`: エラーメッセージ
- `--error-code <code>`: エラーコード
- `--auto-fix`: 自動修正タスクを作成するかどうか

**例:**
```bash
# エラー発生時
keruta fail --message "データベース接続に失敗しました" --error-code DB_CONNECTION_ERROR
```

#### `keruta progress`
タスクの進捗率を更新します。

```bash
keruta progress <percentage> [options]
```

**引数:**
- `percentage`: 進捗率（0-100）

**例:**
```bash
keruta progress 50 --message "データ処理中..."
```

### ユーティリティコマンド

#### `keruta log`
構造化ログを送信します。

```bash
keruta log <level> <message> [options]
```

**引数:**
- `level`: ログレベル（DEBUG, INFO, WARN, ERROR）
- `message`: ログメッセージ

**例:**
```bash
keruta log INFO "データベースクエリを実行中..."
```

#### `keruta health`
ヘルスチェックを実行します。

```bash
keruta health [options]
```

**オプション:**
- `--check-api`: keruta APIとの接続確認
- `--check-disk`: ディスク容量確認
- `--check-memory`: メモリ使用量確認

#### `keruta config`
設定を表示・更新します。

```bash
keruta config show
keruta config set <key> <value>
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
EOF
```

## 使用方法

### 基本的な使用例
```bash
#!/bin/bash
set -e

# タスク開始
keruta start

# 進捗報告
keruta progress 25 --message "データの読み込み中..."

# 処理実行
python process_data.py

# 進捗報告
keruta progress 75 --message "データの処理中..."

# タスク成功
keruta success --message "データ処理が完了しました"
```

### エラーハンドリング例
```bash
#!/bin/bash
set -e

keruta start

# エラーハンドリング
trap 'keruta fail --message "予期しないエラーが発生しました: $?"' ERR

# 処理実行
python risky_operation.py

keruta success
```

### 構造化ログの使用例
```bash
#!/bin/bash
keruta start

keruta log INFO "処理を開始します"
keruta log DEBUG "設定ファイルを読み込み中..."

# 処理実行
python main.py

keruta log INFO "処理が完了しました"
keruta success
```

## 技術スタック
- **言語**: Go 1.21+
- **フレームワーク**: Cobra（CLIフレームワーク）
- **HTTP クライアント**: net/http（標準ライブラリ）
- **設定管理**: Viper
- **ログ**: logrus
- **テスト**: testify

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