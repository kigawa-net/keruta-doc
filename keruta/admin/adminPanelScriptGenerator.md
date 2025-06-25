# Adminパネル インストールスクリプト生成機能

> **概要**: keruta管理パネルからインストールスクリプトを作成・編集できる機能のドキュメントです。

## 目次
- [概要](#概要)
- [機能一覧](#機能一覧)
- [画面構成](#画面構成)
- [使用方法](#使用方法)
- [スクリプトテンプレート](#スクリプトテンプレート)
- [API仕様](#api仕様)
- [権限管理](#権限管理)
- [技術仕様](#技術仕様)
- [トラブルシューティング](#トラブルシューティング)
- [今後の拡張予定](#今後の拡張予定)

## 概要

Adminパネルのインストールスクリプト生成機能は、リポジトリ管理画面からインストールスクリプトを作成・編集できる機能です。Webインターフェースを通じてスクリプトを管理できます。

### 主な特徴
- **直感的編集**: コードエディタを使用したスクリプト編集
- **自動テンプレート**: 主要言語のセットアップを自動判定するデフォルトスクリプト
- **管理画面からの編集・保存**

## 機能一覧

### 実装済み機能
- [x] スクリプトの作成・編集・削除（管理画面から）
- [x] デフォルトスクリプトの自動挿入（リポジトリ作成時）

### 未実装・今後の機能
- [ ] テンプレート選択・複数テンプレート
- [ ] プロジェクトタイプ選択
- [ ] リアルタイム構文チェック
- [ ] スクリプトプレビュー
- [ ] 変更履歴管理・バージョン管理
- [ ] スクリプトの共有・エクスポート
- [ ] 複数リポジトリへの一括適用
- [ ] テスト実行（現状はダミー）

## 画面構成

### 1. スクリプト一覧画面
- リポジトリごとにスクリプト編集ボタン

### 2. スクリプト編集画面
- コードエディタ（Monaco Editor）
- 保存ボタン
- テスト実行ボタン（ダミー）

## 使用方法

### 1. スクリプトの新規作成
- リポジトリ作成時にデフォルトスクリプトが自動で挿入されます
- 編集は「編集」ボタンから

### 2. 既存スクリプトの編集
- スクリプト一覧から編集対象を選択し、編集・保存

### 3. スクリプトのテスト実行
- 編集画面で「テスト実行」ボタンを押すとダミー結果が返ります（本番実行は未実装）

## スクリプトテンプレート

> **現状**
> デフォルトスクリプトは1種類で、Node.js/Python/Java(Gradle/Maven)の主要なセットアップを自動判定して実行します。
> テンプレート選択やプロジェクトタイプごとの分岐は未実装です。

```bash
#!/bin/sh
# This is the default setup script for Keruta repositories.
# This script is executed after the repository is cloned.
# You can customize this script to install dependencies, set up the environment, etc.

set -e

echo "Starting setup script for repository"

# Check if package.json exists and run npm install
if [ -f "package.json" ]; then
    echo "Found package.json, running npm install"
    npm install
fi

# Check if requirements.txt exists and run pip install
if [ -f "requirements.txt" ]; then
    echo "Found requirements.txt, running pip install"
    pip install -r requirements.txt
fi

# Check if build.gradle or build.gradle.kts exists and run gradle build
if [ -f "build.gradle" ] || [ -f "build.gradle.kts" ]; then
    echo "Found Gradle build file, running gradle build"
    ./gradlew build
fi

# Check if pom.xml exists and run mvn install
if [ -f "pom.xml" ]; then
    echo "Found pom.xml, running mvn install"
    mvn install
fi

echo "Setup script completed successfully"
```

## API仕様

### スクリプト取得
```
GET /api/v1/repositories/{repositoryId}/script
```
- リポジトリのセットアップスクリプトを取得します。

### スクリプト保存
- 管理画面 `/admin/repositories/script/{id}` からのみ可能（REST APIは未実装）

### スクリプトテスト実行
- 管理画面 `/admin/repositories/script/{id}/test` からのみ可能（現状はダミー実装）

## 権限管理

| 操作 | 管理者 (admin) | 一般ユーザー (user) | 開発者 (developer) |
|------|:--------------:|:-------------------:|:------------------:|
| スクリプト閲覧 | ○ | ○ | ○ |
| スクリプト作成 | ○ | × | ○ |
| スクリプト編集 | ○ | × | ○ |
| スクリプト削除 | ○ | × | × |
| テスト実行 | ○ | × | ○ |
| テンプレート管理 | × | × | × |

## 技術仕様

### フロントエンド
- **エディタ**: Monaco Editor (VS Code と同じエンジン)
- **UI フレームワーク**: Bootstrap 5 + Thymeleaf

### バックエンド
- **API**: Spring Boot REST API
- **スクリプト保存**: データベース（PostgreSQL）
- **テスト実行**: 未実装（ダミー応答）
- **ファイル管理**: Git リポジトリ内の `.keruta/install.sh`（将来対応予定）

### セキュリティ
- **入力検証**: スクリプト内容のサニタイゼーション
- **実行制限**: 危険なコマンドの実行制限（将来対応）
- **権限チェック**: リポジトリアクセス権限の確認
- **ログ記録**: スクリプト実行履歴の記録（将来対応）

## トラブルシューティング

### よくある問題

#### 1. スクリプトが保存できない
- 権限不足または構文エラー
- ユーザー権限を確認、構文エラーを修正

#### 2. テスト実行が失敗する
- 現状はダミー応答のみ

#### 3. テンプレートが適用されない
- テンプレート選択機能は未実装

### ログの確認方法
- ログ管理機能は将来対応予定

## 今後の拡張予定

- テンプレート選択・複数テンプレート
- プロジェクトタイプ選択
- リアルタイム構文チェック
- スクリプトプレビュー
- 変更履歴管理・バージョン管理
- スクリプトの共有・エクスポート
- 複数リポジトリへの一括適用
- テスト実行の本番対応

---

## 関連ドキュメント

- [管理パネルドキュメント](./adminPanel.md)
- [セットアップスクリプト機能](./setupScript.md)
- [リポジトリ管理](./repositoryManagement.md)
- [Kubernetes統合](./kubernetes/kubernetesIntegration.md)

ご意見・ご要望はIssueまたはPRでお知らせください。 