# リポジトリ管理ドキュメント

> **概要**: kerutaのリポジトリ管理機能の使用方法、設定オプション、認証方法などをまとめたドキュメントです。

## 目次
- [概要](#概要)
- [リポジトリ管理画面](#リポジトリ管理画面)
- [リポジトリの作成](#リポジトリの作成)
- [リポジトリの編集](#リポジトリの編集)
- [リポジトリの削除](#リポジトリの削除)
- [認証設定](#認証設定)
- [インストールスクリプト](#インストールスクリプト)
- [技術情報](#技術情報)
- [トラブルシューティング](#トラブルシューティング)

## 概要
リポジトリ管理機能は、kerutaシステム内でGitリポジトリを管理するための機能です。リポジトリのURL、認証情報、インストールスクリプトなどを設定し、タスク実行時に利用することができます。

## リポジトリ管理画面
リポジトリ管理画面では、登録されているすべてのリポジトリの一覧を表示します。各リポジトリについて以下の情報が表示されます：

- リポジトリ名
- URL
- 説明
- 有効性ステータス（URLが有効かどうか）
- 作成日時
- 更新日時

また、各リポジトリに対して編集、削除などの操作を行うことができます。

### アクセス方法
1. 管理パネルにログイン
2. サイドメニューから「Repositories」を選択

## リポジトリの作成
新しいリポジトリを作成するには：

1. リポジトリ管理画面で「Create Repository」ボタンをクリック
2. 以下の情報を入力：
   - **Name**: リポジトリの名前（必須）
   - **URL**: GitリポジトリのURL（必須）
   - **Description**: リポジトリの説明（任意）
3. 「Save」ボタンをクリック

## リポジトリの編集
既存のリポジトリを編集するには：

1. リポジトリ管理画面でリポジトリの「Edit」ボタンをクリック
2. 必要な情報を更新
3. 「Save」ボタンをクリック

## リポジトリの削除
リポジトリを削除するには：

1. リポジトリ管理画面でリポジトリの「Delete」ボタンをクリック
2. 確認ダイアログで「OK」をクリック

## 認証設定
リポジトリにアクセスするための認証情報を設定できます。以下の認証方法がサポートされています：

### ユーザー名/パスワード認証
プライベートリポジトリにアクセスするためのユーザー名とパスワードを設定します。

### SSH認証
SSHキーを使用してリポジトリにアクセスする場合は、秘密鍵を設定します。

## インストールスクリプト
各リポジトリには、クローン後に実行されるインストールスクリプトを設定できます。このスクリプトは、依存関係のインストールやビルドなどの初期設定を自動化するために使用されます。

### デフォルトスクリプト
新しいリポジトリを作成すると、以下のようなデフォルトのインストールスクリプトが自動的に設定されます：

```sh
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

このスクリプトは、一般的なプロジェクトタイプ（Node.js、Python、Gradle、Maven）を自動検出し、適切な依存関係のインストールコマンドを実行します。

### スクリプトのカスタマイズ
インストールスクリプトは、リポジトリの編集画面で自由にカスタマイズできます。プロジェクトの特定の要件に合わせてスクリプトを変更してください。

## 技術情報
- リポジトリ情報はMongoDBに保存されます
- URLの有効性は、HTTPリクエストを送信して確認されます
- 認証情報は暗号化されて保存されます

## トラブルシューティング
- **リポジトリURLが無効と表示される**: URLが正しいこと、およびサーバーからアクセス可能であることを確認してください
- **認証エラー**: 認証情報が正しいことを確認してください
- **インストールスクリプトが失敗する**: スクリプトのログを確認し、必要なツールがインストールされていることを確認してください

---
ご意見・ご要望はIssueまたはPRでお知らせください。