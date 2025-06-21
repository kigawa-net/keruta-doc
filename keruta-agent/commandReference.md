# keruta-agent コマンド一覧

> **概要**: keruta-agentで利用可能なすべてのコマンドとその使用方法をまとめたリファレンスドキュメントです。

## 目次
- [概要](#概要)
- [基本コマンド](#基本コマンド)
- [ユーティリティコマンド](#ユーティリティコマンド)
- [使用例](#使用例)
- [環境変数](#環境変数)

## 概要

keruta-agentは、kerutaシステムによってKubernetes Jobとして実行されるPod内で動作するCLIツールです。タスクの実行状況をkeruta APIサーバーに報告し、ログの収集、エラーハンドリングなどの機能を提供します。

## 基本コマンド

### `keruta start`
タスクの実行を開始し、ステータスをPROCESSINGに更新します。

```bash
keruta start [options]
```

**オプション:**
- `--task-id <id>`: タスクIDを指定（環境変数KERUTA_TASK_IDから自動取得がデフォルト）
- `--api-url <url>`: keruta APIのURL（環境変数KERUTA_API_URLから自動取得がデフォルト）
- `--log-level <level>`: ログレベル（DEBUG, INFO, WARN, ERROR）

**例:**
```bash
#!/bin/bash
keruta start --log-level INFO
# タスク処理を実行
```

### `keruta success`
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

### `keruta fail`
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

### `keruta progress`
タスクの進捗率を更新します。

```bash
keruta progress <percentage> [options]
```

**引数:**
- `percentage`: 進捗率（0-100）

**オプション:**
- `--message <message>`: 進捗メッセージ

**例:**
```bash
keruta progress 50 --message "データ処理中..."
```

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

### 基本的なタスク実行例
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

---

## 関連リンク
- [keruta-agent README](README.md)
- [keruta-agent API仕様](apiSpec.md)
- [keruta-agent 実装例](implementation.md)
- [keruta プロジェクト詳細](../keruta/projectDetails.md) 