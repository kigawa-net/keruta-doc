# keruta-agent コマンド一覧

> **概要**: keruta-agentで利用可能なすべてのコマンドとその使用方法をまとめたリファレンスドキュメントです。

## 目次
- [概要](#概要)
- [メインコマンド](#メインコマンド)
- [ユーティリティコマンド](#ユーティリティコマンド)
- [使用例](#使用例)
- [環境変数](#環境変数)

## 概要

keruta-agentは、kerutaシステムによってKubernetes Jobとして実行されるPod内で動作するCLIツールです。タスクの実行状況をkeruta APIサーバーに報告し、ログの収集、エラーハンドリングなどの機能を提供します。

## メインコマンド

### `keruta-agent execute`

指定されたタスクIDのスクリプトをkeruta APIから取得し、サブプロセスとして実行します。入力待ち状態を自動検出します。

#### 構文
```bash
keruta-agent execute [options]
```

#### オプション
| オプション | 説明 | デフォルト値 |
|------------|------|-------------|
| `--task-id <id>` | タスクIDを指定 | 環境変数`KERUTA_TASK_ID`から取得 |
| `--api-url <url>` | keruta APIのURL | 環境変数`KERUTA_API_URL`から取得 |
| `--work-dir <path>` | 作業ディレクトリ | `/work` |
| `--log-level <level>` | ログレベル（DEBUG, INFO, WARN, ERROR） | `INFO` |
| `--auto-detect-input` | 入力待ち状態の自動検出 | `true` |
| `--timeout <seconds>` | サブプロセスのタイムアウト時間（秒） | `0`（無制限） |
| `--help, -h` | ヘルプを表示 | - |

#### 使用例
```bash
# 基本的な実行
keruta-agent execute --task-id task123

# カスタム設定での実行
keruta-agent execute \
    --task-id task123 \
    --api-url http://keruta-api:8080 \
    --work-dir /work \
    --log-level DEBUG

# タイムアウト付きで実行
keruta-agent execute \
    --timeout 300 \
    --task-id task123
```

#### 動作
1. タスクIDからkeruta APIでスクリプトを取得
2. タスク状態を`PROCESSING`に更新
3. 取得したスクリプトをサブプロセスとして実行
4. サブプロセスの標準出力・標準エラー出力をリアルタイムでキャプチャ
5. サブプロセスが入力待ち状態になった場合、タスク状態を`WAITING_FOR_INPUT`に更新
6. 管理パネルからの標準入力送信を待機
7. サブプロセスが正常終了した場合、タスク状態を`COMPLETED`に更新
8. サブプロセスが異常終了した場合、タスク状態を`FAILED`に更新

## ユーティリティコマンド

### `keruta log`
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
keruta log DEBUG "設定ファイルを読み込み中..."
keruta log WARN "リソース使用量が高いです"
keruta log ERROR "予期しないエラーが発生しました"
```

### `keruta health`
ヘルスチェックを実行します。

```bash
keruta health [options]
```

**オプション:**
- `--check-api`: keruta APIとの接続確認
- `--check-disk`: ディスク容量確認
- `--check-memory`: メモリ使用量確認

**例:**
```bash
keruta health --check-api --check-disk
```

### `keruta config`
設定を表示・更新します。

#### `keruta config show`
現在の設定を表示します。

```bash
keruta config show
```

#### `keruta config set`
設定を更新します。

```bash
keruta config set <key> <value>
```

**引数:**
- `key`: 設定キー
- `value`: 設定値

**例:**
```bash
keruta config set log_level DEBUG
```

## 使用例

### 基本的なスクリプト実行
```bash
# keruta-agentでタスクを実行（スクリプトはAPIから取得）
keruta-agent execute --task-id task123
```

### エラーハンドリング付きスクリプト
```bash
# エラーハンドリング付きで実行
keruta-agent execute --task-id task123
```

### ログ付きスクリプト
```bash
# ログ付きで実行
keruta-agent execute --task-id task123
```

### Kubernetes Jobでの使用
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
            - name: KERUTA_LOG_LEVEL
              value: "INFO"
      restartPolicy: Never
```

## 環境変数

keruta-agentは以下の環境変数を利用します：

### 必須環境変数
- `KERUTA_TASK_ID`: タスクID
- `KERUTA_API_URL`: keruta APIのURL
- `KERUTA_API_TOKEN`: keruta APIの認証トークン

### オプション環境変数
- `KERUTA_LOG_LEVEL`: ログレベル（デフォルト: INFO）
- `KERUTA_WORK_DIR`: 作業ディレクトリ（デフォルト: /work）

### APIから自動取得される情報
- リポジトリID
- ドキュメントID
- タスク設定
- 実行パラメータ

---

## 関連リンク
- [keruta-agent README](README.md)
- [keruta-agent API仕様](apiSpec.md)
- [keruta-agent 実装例](implementation.md)
- [keruta プロジェクト詳細](../keruta/projectDetails.md) 