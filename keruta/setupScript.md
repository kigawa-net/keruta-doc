# セットアップスクリプト機能

## 概要

セットアップスクリプト機能は、リポジトリがクローンされた後に自動的に実行されるスクリプトを提供します。このスクリプトは、依存関係のインストールや環境設定など、タスク実行に必要な準備を行います。

## 仕様

- **スクリプト形式**: シェルスクリプト（`#!/bin/sh`）
- **実行タイミング**: リポジトリがクローンされた後、タスク実行前
- **実行環境**: Kubernetesのinit container内
- **デフォルトスクリプト**: リポジトリ作成時に自動的に設定される
- **カスタマイズ**: リポジトリの更新時にスクリプトを変更可能
- **APIエンドポイント**: `/api/v1/repositories/{id}/script`でスクリプトを取得可能

## デフォルトスクリプト

デフォルトのセットアップスクリプトは、一般的なプロジェクトタイプを自動検出し、適切な依存関係のインストールを行います：

- **Node.js**: `package.json`が存在する場合、`npm install`を実行
- **Python**: `requirements.txt`が存在する場合、`pip install -r requirements.txt`を実行
- **Gradle**: `build.gradle`または`build.gradle.kts`が存在する場合、`./gradlew build`を実行
- **Maven**: `pom.xml`が存在する場合、`mvn install`を実行

## サンプルスクリプト

プロジェクトルートの`.keruta/install.sh`にサンプルスクリプトが用意されています。このスクリプトをカスタマイズして、特定のプロジェクトに必要な設定を行うことができます。

```sh
#!/bin/sh
# This is a sample setup script for Keruta repositories.
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

## APIの使用方法

セットアップスクリプトは、以下のAPIエンドポイントを通じて取得できます：

```
GET /api/v1/repositories/{id}/script
```

このエンドポイントは、指定されたリポジトリIDに対応するセットアップスクリプトを返します。

## 関連リンク

- [Init Containerによる事前準備](./kubernetes/kubernetesInitContainer.md)
- [Gitの除外設定](./gitExcludeSpec.md)