# keruta-github

このサブプロジェクトは、kerutaをGitHubから操作するためのツール/サービスを提供します。

## 概要
keruta本体とGitHubを連携し、タスク管理や進捗の自動反映、GitHub Actions/Webhookとの連携を目指す拡張モジュールです。

## 目的
- kerutaのタスクや状態をGitHubリポジトリ上から管理・操作できるようにする
- GitHub ActionsやWebhookとの連携を想定

## 主な機能案
- GitHub Issue/PRからタスク登録
- タスク進捗のGitHub上への自動反映
- GitHub認証・権限管理

## ディレクトリ構成（予定）
- `src/` : メイン実装
- `docs/` : ドキュメント

## 今後の予定
- 実装はこれから着手予定です（WIP）
- 詳細な利用方法やセットアップ手順は今後追記します

---

ご意見・ご要望はIssueまたはPRでお知らせください。

## GitHub Appsとしての実装仕様

### 1. アプリの目的
- keruta本体とGitHubリポジトリを連携し、タスク管理・進捗反映・自動化を実現する。
- GitHub上のIssue/PR操作を通じてkerutaのタスクを管理できるようにする。

### 2. 主な機能
- GitHub Issue/PRの作成・更新・クローズ時に、keruta側のタスクを自動生成・更新・完了状態に同期
- kerutaのタスク進捗をGitHub Issue/PRのコメントやラベル、ステータスに自動反映
- GitHub ActionsやWebhookと連携し、CI/CDや自動通知を実現
- GitHub認証・権限管理（必要なリポジトリ/組織単位でのインストール・権限付与）

### 3. GitHub Appsの権限
- Issues: 読み取り・書き込み
- Pull requests: 読み取り・書き込み
- Repository metadata: 読み取り
- Webhook: 受信
- Actions: 読み取り（必要に応じて）

### 4. イベントハンドリング
- 受信するイベント例:
  - issues（opened, edited, closed, reopened, labeled, unlabeled, assigned, unassigned）
  - pull_request（opened, edited, closed, reopened, labeled, unlabeled, assigned, unassigned, merged）
  - issue_comment（created, edited, deleted）
  - push（必要に応じて）
- これらのイベントを受信し、kerutaのAPIやDBと連携してタスク情報を同期

### 5. 認証・セキュリティ
- GitHub AppsのJWT認証を利用し、API呼び出し時に署名付きトークンを発行
- 必要に応じてユーザーごとのOAuth認証もサポート

### 6. UI/UX
- GitHub上のIssue/PRにkeruta連携用のコメントやチェックボックス、ラベルを自動付与
- 必要に応じてGitHub Appsの設定画面でkeruta連携先や動作オプションを指定可能

### 7. ディレクトリ構成（例）
- `src/` : GitHub Apps本体（Node.js/TypeScriptやPythonなどで実装）
- `src/webhook/` : Webhook受信・イベント処理
- `src/github_api/` : GitHub API連携
- `src/keruta_api/` : keruta本体API連携
- `docs/` : 仕様・セットアップ手順

### 8. セットアップ・運用
- GitHub Appsの登録・インストール手順をREADMEに記載
- keruta本体との連携設定（APIキーやエンドポイントなど）を環境変数や設定ファイルで管理 