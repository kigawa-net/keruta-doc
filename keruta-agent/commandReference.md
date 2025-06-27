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

### `keruta execute`
タスクを統合的に実行します。タスク情報はkerutaのAPIから自動取得し、初期化から実行、ログ収集まで統合的に管理します。

```bash
keruta execute [options]
```

**オプション:**
- `--task-id <id>`: タスクID（環境変数KERUTA_TASK_IDから自動取得がデフォルト）
- `--api-url <url>`: keruta APIのURL（環境変数KERUTA_API_URLから自動取得がデフォルト）
- `--work-dir <path>`: 作業ディレクトリ（デフォルト: /work）
- `--log-level <level>`: ログレベル（DEBUG, INFO, WARN, ERROR）
- `--skip-init`: 初期化処理をスキップ（デフォルト: false）
- `--auto-cleanup`: 自動的にクリーンアップを実行（デフォルト: true）

**機能:**
1. **タスク情報取得**: kerutaのAPIからタスク情報（リポジトリID、ドキュメントID等）を自動取得
2. **初期化処理**: リポジトリのクローン、git-exclude設定、インストールスクリプトの取得と実行、ドキュメントの取得
3. **タスク実行**: タスクの開始、実行、監視
4. **ログ収集**: 実行ログの自動収集と送信
5. **エラーハンドリング**: エラー発生時の適切な処理
6. **クリーンアップ**: 成果物の収集とアップロード（--auto-cleanup=trueの場合）

**例:**
```bash
# 基本的な実行
keruta execute --task-id task123

# カスタム設定での実行
keruta execute \
    --task-id task123 \
    --api-url http://keruta-api:8080 \
    --work-dir /work \
    --log-level DEBUG
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

# 統合的なタスク実行（タスク情報はAPIから自動取得）
keruta execute --task-id task123
```

### カスタム設定での実行例
```bash
#!/bin/bash
set -e

# カスタム設定での統合実行
keruta execute \
    --task-id task123 \
    --api-url http://keruta-api:8080 \
    --work-dir /work \
    --log-level DEBUG \
    --auto-cleanup
```

### エラーハンドリング例
```bash
#!/bin/bash
set -e

# エラーハンドリング
trap 'keruta log ERROR "予期しないエラーが発生しました: $?"' ERR

# 統合的なタスク実行
keruta execute --task-id task123
```

### 構造化ログの使用例
```bash
#!/bin/bash

# 統合的なタスク実行
keruta execute --task-id task123

# 追加のログ送信
keruta log INFO "追加の処理を実行中..."
keruta log DEBUG "詳細なデバッグ情報"
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