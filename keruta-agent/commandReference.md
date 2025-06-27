# keruta-agent コマンド一覧

> **概要**: keruta-agentで利用可能なすべてのコマンドとその使用方法をまとめたリファレンスドキュメントです。

## 目次
- [概要](#概要)
- [基本コマンド](#基本コマンド)
- [Kubernetes統合コマンド](#kubernetes統合コマンド)
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

## Kubernetes統合コマンド

### `keruta init`
Init Containerで実行される初期化処理を行います。

```bash
keruta init [options]
```

**オプション:**
- `--repository-id <id>`: リポジトリID（環境変数KERUTA_REPOSITORY_IDから自動取得がデフォルト）
- `--document-id <id>`: ドキュメントID（環境変数KERUTA_DOCUMENT_IDから自動取得がデフォルト）
- `--api-url <url>`: keruta APIのURL（環境変数KERUTA_API_ENDPOINTから自動取得がデフォルト）
- `--work-dir <path>`: 作業ディレクトリ（デフォルト: /work）
- `--log-level <level>`: ログレベル

**機能:**
- リポジトリのクローン
- git-exclude設定の追加
- インストールスクリプトの取得と実行
- ドキュメントの取得

**例:**
```bash
keruta init --repository-id repo123 --work-dir /work
```

### `keruta run`
メインコンテナで実行されるタスク処理を行います。

```bash
keruta run [options]
```

**オプション:**
- `--task-id <id>`: タスクID（環境変数KERUTA_TASK_IDから自動取得がデフォルト）
- `--api-url <url>`: keruta APIのURL（環境変数KERUTA_API_URLから自動取得がデフォルト）
- `--work-dir <path>`: 作業ディレクトリ（デフォルト: /work）
- `--log-level <level>`: ログレベル
- `--auto-start`: 自動的にタスク開始を実行（デフォルト: true）

**機能:**
- タスクの開始（--auto-start=trueの場合）
- タスク実行の監視
- ログの自動収集
- エラーハンドリング

**例:**
```bash
keruta run --task-id task123 --work-dir /work
```

### `keruta cleanup`
クリーンアップジョブで実行される後処理を行います。

```bash
keruta cleanup [options]
```

**オプション:**
- `--task-id <id>`: タスクID（環境変数KERUTA_TASK_IDから自動取得がデフォルト）
- `--api-url <url>`: keruta APIのURL（環境変数KERUTA_API_URLから自動取得がデフォルト）
- `--source-pod <name>`: 成果物を収集するソースPod名（環境変数KERUTA_SOURCE_PODから自動取得がデフォルト）
- `--log-level <level>`: ログレベル

**機能:**
- メインジョブのPodから成果物を収集
- 成果物をAPIサーバーにアップロード
- タスクステータスの更新

**例:**
```bash
keruta cleanup --task-id task123 --source-pod keruta-job-task123-pod-xyz
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

### Kubernetes統合実行例
```bash
# Init Containerでの実行
keruta init --repository-id repo123 --work-dir /work

# メインコンテナでの実行
keruta run --task-id task123 --work-dir /work

# クリーンアップジョブでの実行
keruta cleanup --task-id task123 --source-pod keruta-job-task123-pod-xyz
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

keruta-agentは以下の環境変数を利用します：

### 必須環境変数
- `KERUTA_TASK_ID`: タスクID
- `KERUTA_API_URL`: keruta APIのURL
- `KERUTA_API_TOKEN`: keruta APIの認証トークン

### オプション環境変数
- `KERUTA_REPOSITORY_ID`: リポジトリID（initコマンド用）
- `KERUTA_DOCUMENT_ID`: ドキュメントID（initコマンド用）
- `KERUTA_API_ENDPOINT`: keruta APIのエンドポイント（initコマンド用）
- `KERUTA_SOURCE_POD`: ソースPod名（cleanupコマンド用）
- `KERUTA_LOG_LEVEL`: ログレベル（デフォルト: INFO）

---

## 関連リンク
- [keruta-agent README](README.md)
- [keruta-agent API仕様](apiSpec.md)
- [keruta-agent 実装例](implementation.md)
- [keruta プロジェクト詳細](../keruta/projectDetails.md) 