# Adminパネル インストールスクリプト生成機能

> **概要**: keruta管理パネルからインストールスクリプトを視覚的に作成・編集できる機能のドキュメントです。

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

Adminパネルのインストールスクリプト生成機能は、リポジトリ管理画面からインストールスクリプトを視覚的に作成・編集できる機能です。コマンドラインでの直接編集ではなく、Webインターフェースを通じてスクリプトを管理できます。

### 主な特徴
- **視覚的編集**: コードエディタを使用した直感的なスクリプト編集
- **テンプレート機能**: プロジェクトタイプ別の事前定義テンプレート
- **リアルタイムプレビュー**: スクリプトの構文チェックとプレビュー機能
- **バージョン管理**: スクリプトの変更履歴とロールバック機能
- **検証機能**: スクリプトの構文エラー検出と修正提案

## 機能一覧

### 基本機能
- [x] スクリプトの作成・編集・削除
- [x] テンプレートからの新規作成
- [x] リアルタイム構文チェック
- [x] スクリプトプレビュー
- [x] 変更履歴管理

### 高度な機能
- [x] プロジェクトタイプ自動検出
- [x] 依存関係の自動追加
- [x] 環境変数の設定支援
- [x] スクリプトのテスト実行
- [ ] スクリプトの共有・エクスポート
- [ ] 複数リポジトリへの一括適用

## 画面構成

### 1. スクリプト一覧画面
```
リポジトリ管理 > スクリプト管理
├── スクリプト一覧テーブル
│   ├── リポジトリ名
│   ├── プロジェクトタイプ
│   ├── 最終更新日時
│   ├── ステータス（有効/無効）
│   └── 操作ボタン（編集/削除/プレビュー）
├── 新規作成ボタン
└── フィルター・検索機能
```

### 2. スクリプト編集画面
```
スクリプト編集
├── ヘッダー情報
│   ├── リポジトリ選択
│   ├── プロジェクトタイプ選択
│   └── スクリプト名
├── エディタ領域
│   ├── コードエディタ（Monaco Editor）
│   ├── 行番号表示
│   ├── 構文ハイライト
│   └── エラー表示
├── サイドパネル
│   ├── テンプレート選択
│   ├── 変数設定
│   ├── 依存関係管理
│   └── プレビュー
└── 操作ボタン
    ├── 保存
    ├── テスト実行
    ├── プレビュー
    └── キャンセル
```

### 3. テンプレート選択画面
```
テンプレート選択
├── カテゴリ別テンプレート
│   ├── Node.js
│   ├── Python
│   ├── Java (Maven/Gradle)
│   ├── Go
│   ├── Rust
│   └── カスタム
├── テンプレート詳細
│   ├── 説明
│   ├── 必要なファイル
│   ├── 実行時間目安
│   └── プレビュー
└── 適用ボタン
```

## 使用方法

### 1. スクリプトの新規作成

1. **リポジトリ管理画面**にアクセス
2. **「スクリプト管理」**タブを選択
3. **「新規作成」**ボタンをクリック
4. **リポジトリ**を選択
5. **テンプレート**を選択（または空のスクリプトから開始）
6. **スクリプトを編集**
7. **「保存」**ボタンで確定

### 2. 既存スクリプトの編集

1. **スクリプト一覧**から対象スクリプトを選択
2. **「編集」**ボタンをクリック
3. **コードエディタ**でスクリプトを修正
4. **リアルタイムで構文チェック**を確認
5. **「保存」**ボタンで変更を確定

### 3. スクリプトのテスト実行

1. **編集画面**で**「テスト実行」**ボタンをクリック
2. **テスト環境**でスクリプトが実行される
3. **実行結果**と**ログ**を確認
4. **問題があれば修正**して再実行

## スクリプトテンプレート

### Node.js テンプレート
```bash
#!/bin/sh
set -e

echo "Setting up Node.js project..."

# Check if package.json exists
if [ -f "package.json" ]; then
    echo "Installing npm dependencies..."
    npm install
    
    # Check if there are build scripts
    if npm run --silent build 2>/dev/null; then
        echo "Running build script..."
        npm run build
    fi
else
    echo "Warning: package.json not found"
fi

echo "Node.js setup completed"
```

### Python テンプレート
```bash
#!/bin/sh
set -e

echo "Setting up Python project..."

# Check if requirements.txt exists
if [ -f "requirements.txt" ]; then
    echo "Installing Python dependencies..."
    pip install -r requirements.txt
fi

# Check if setup.py exists
if [ -f "setup.py" ]; then
    echo "Installing package in development mode..."
    pip install -e .
fi

# Check if pyproject.toml exists
if [ -f "pyproject.toml" ]; then
    echo "Installing with pip..."
    pip install .
fi

echo "Python setup completed"
```

### Java (Maven) テンプレート
```bash
#!/bin/sh
set -e

echo "Setting up Java Maven project..."

# Check if pom.xml exists
if [ -f "pom.xml" ]; then
    echo "Building Maven project..."
    mvn clean install
    
    # Run tests if they exist
    if mvn test -q 2>/dev/null; then
        echo "Running tests..."
        mvn test
    fi
else
    echo "Warning: pom.xml not found"
fi

echo "Java Maven setup completed"
```

### Java (Gradle) テンプレート
```bash
#!/bin/sh
set -e

echo "Setting up Java Gradle project..."

# Check if gradle files exist
if [ -f "build.gradle" ] || [ -f "build.gradle.kts" ]; then
    echo "Building Gradle project..."
    ./gradlew build
    
    # Run tests if they exist
    if ./gradlew test -q 2>/dev/null; then
        echo "Running tests..."
        ./gradlew test
    fi
else
    echo "Warning: Gradle build files not found"
fi

echo "Java Gradle setup completed"
```

## API仕様

### スクリプト取得
```
GET /api/v1/repositories/{repositoryId}/script
```

**レスポンス例:**
```json
{
  "id": "script-123",
  "repositoryId": "repo-456",
  "name": "Node.js Setup Script",
  "content": "#!/bin/sh\nset -e\nnpm install",
  "projectType": "nodejs",
  "createdAt": "2024-01-01T00:00:00Z",
  "updatedAt": "2024-01-01T00:00:00Z",
  "version": 1
}
```

### スクリプト作成・更新
```
POST /api/v1/repositories/{repositoryId}/script
PUT /api/v1/repositories/{repositoryId}/script
```

**リクエスト例:**
```json
{
  "name": "Custom Setup Script",
  "content": "#!/bin/sh\nset -e\necho 'Custom setup'",
  "projectType": "custom"
}
```

### スクリプトテスト実行
```
POST /api/v1/repositories/{repositoryId}/script/test
```

**レスポンス例:**
```json
{
  "success": true,
  "output": "Setting up project...\nSetup completed",
  "exitCode": 0,
  "executionTime": 2.5
}
```

## 権限管理

| 操作 | 管理者 (admin) | 一般ユーザー (user) | 開発者 (developer) |
|------|:--------------:|:-------------------:|:------------------:|
| スクリプト閲覧 | ○ | ○ | ○ |
| スクリプト作成 | ○ | × | ○ |
| スクリプト編集 | ○ | × | ○ |
| スクリプト削除 | ○ | × | × |
| テスト実行 | ○ | × | ○ |
| テンプレート管理 | ○ | × | × |

## 技術仕様

### フロントエンド
- **エディタ**: Monaco Editor (VS Code と同じエンジン)
- **構文ハイライト**: Shell script対応
- **リアルタイム検証**: クライアントサイド構文チェック
- **UI フレームワーク**: Bootstrap 5 + Thymeleaf

### バックエンド
- **API**: Spring Boot REST API
- **スクリプト保存**: データベース（PostgreSQL）
- **テスト実行**: Kubernetes Job
- **ファイル管理**: Git リポジトリ内の `.keruta/install.sh`

### セキュリティ
- **入力検証**: スクリプト内容のサニタイゼーション
- **実行制限**: 危険なコマンドの実行制限
- **権限チェック**: リポジトリアクセス権限の確認
- **ログ記録**: スクリプト実行履歴の記録

## トラブルシューティング

### よくある問題

#### 1. スクリプトが保存できない
**原因**: 権限不足または構文エラー
**解決方法**: 
- ユーザー権限を確認
- 構文エラーを修正
- 特殊文字のエスケープを確認

#### 2. テスト実行が失敗する
**原因**: 依存関係不足または環境問題
**解決方法**:
- スクリプトの依存関係を確認
- テスト環境の設定を確認
- ログを詳細に確認

#### 3. テンプレートが適用されない
**原因**: プロジェクトタイプの不一致
**解決方法**:
- プロジェクトタイプを正しく選択
- 手動でテンプレートをコピー
- カスタムテンプレートを作成

### ログの確認方法
1. **Adminパネル** > **ログ管理**でスクリプト実行ログを確認
2. **Kubernetes** > **Pod ログ**で詳細な実行ログを確認
3. **リポジトリ管理** > **スクリプト履歴**で変更履歴を確認

## 今後の拡張予定

### 短期予定（1-2ヶ月）
- [ ] スクリプトの共有機能
- [ ] 複数リポジトリへの一括適用
- [ ] より多くのプロジェクトタイプ対応
- [ ] スクリプトのバージョン管理強化

### 中期予定（3-6ヶ月）
- [ ] ビジュアルスクリプトビルダー
- [ ] 条件分岐とループ機能
- [ ] 外部APIとの連携
- [ ] スクリプトのパフォーマンス分析

### 長期予定（6ヶ月以上）
- [ ] AI支援スクリプト生成
- [ ] スクリプトの自動最適化
- [ ] マルチクラウド対応
- [ ] エンタープライズ機能

---

## 関連ドキュメント

- [管理パネルドキュメント](./adminPanel.md)
- [セットアップスクリプト機能](./setupScript.md)
- [リポジトリ管理](./repositoryManagement.md)
- [Kubernetes統合](./kubernetes/kubernetesIntegration.md)

ご意見・ご要望はIssueまたはPRでお知らせください。 