# keruta 共通ドキュメント管理仕様

> **概要**: kerutaシステムのすべてのタスクで共通して利用されるドキュメントを管理する仕組みについて説明します。

## 目次
- [概要](#概要)
- [ディレクトリ構造](#ディレクトリ構造)
- [ドキュメント分類](#ドキュメント分類)
- [管理方法](#管理方法)
- [アクセス方法](#アクセス方法)
- [更新・バージョン管理](#更新バージョン管理)
- [関連リンク](#関連リンク)

## 概要
`commonDocuments/` ディレクトリには、kerutaシステムのすべてのタスクで共通して参照・利用されるドキュメントが保存されます。これらのドキュメントは、タスク実行時に自動的に利用可能になり、一貫性のある情報提供を実現します。

## ディレクトリ構造
```
commonDocuments/
├── README.md                    # このファイル（共通ドキュメント管理仕様）
├── system/                      # システム関連ドキュメント
│   ├── architecture.md         # システムアーキテクチャ
│   ├── api-reference.md        # APIリファレンス
│   └── deployment-guide.md     # デプロイメントガイド
├── development/                 # 開発関連ドキュメント
│   ├── coding-standards.md     # コーディング規約
│   ├── testing-guidelines.md   # テストガイドライン
│   └── troubleshooting.md      # トラブルシューティング
├── templates/                   # テンプレートファイル
│   ├── task-template.md        # タスク作成テンプレート
│   ├── report-template.md      # レポートテンプレート
│   └── script-template.sh      # スクリプトテンプレート
└── assets/                      # 静的アセット
    ├── images/                  # 画像ファイル
    ├── diagrams/                # 図表ファイル
    └── examples/                # サンプルファイル
```

## ドキュメント分類

### 1. システム関連 (`system/`)
- **architecture.md**: システム全体のアーキテクチャ図と説明
- **api-reference.md**: 全APIエンドポイントの詳細仕様
- **deployment-guide.md**: 環境構築・デプロイ手順

### 2. 開発関連 (`development/`)
- **coding-standards.md**: コーディング規約・ベストプラクティス
- **testing-guidelines.md**: テスト作成・実行ガイドライン
- **troubleshooting.md**: よくある問題と解決方法

### 3. テンプレート (`templates/`)
- **task-template.md**: タスク作成時の標準テンプレート
- **report-template.md**: 成果物レポートの標準フォーマット
- **script-template.sh**: 各種スクリプトの基本テンプレート

### 4. アセット (`assets/`)
- **images/**: ドキュメントで使用する画像ファイル
- **diagrams/**: システム図・フローチャート等
- **examples/**: サンプルコード・設定ファイル

## 管理方法

### アクセス権限
| 操作 | 管理者 (admin) | 一般ユーザー (user) | 開発者 (developer) |
|------|:--------------:|:-------------------:|:------------------:|
| ドキュメント閲覧 | ○ | ○ | ○ |
| ドキュメント作成・編集 | ○ | × | ○ |
| ドキュメント削除 | ○ | × | × |
| テンプレート管理 | ○ | × | ○ |

### 更新ルール
1. **バージョン管理**: すべての変更はGitで管理
2. **レビュー**: 重要な変更はプルリクエストでレビュー
3. **通知**: 共通ドキュメント更新時は関係者に通知
4. **互換性**: 既存タスクへの影響を考慮した更新

## アクセス方法

### 管理パネルからのアクセス
- 管理パネルの「ドキュメント管理」画面から共通ドキュメントを閲覧・編集可能
- タスク実行時に自動的に共通ドキュメントが参照可能

### APIからのアクセス
```bash
# 共通ドキュメント一覧取得
GET /api/v1/common-documents

# 特定の共通ドキュメント取得
GET /api/v1/common-documents/{category}/{filename}

# 共通ドキュメント更新
PUT /api/v1/common-documents/{category}/{filename}
```

### タスク実行時の自動利用
- タスク実行時に `/.keruta/common-documents/` ディレクトリが自動的にマウント
- 環境変数 `KERUTA_COMMON_DOCS_PATH` でパスが提供される

## 更新・バージョン管理

### 更新手順
1. 変更内容の検討・レビュー
2. 既存タスクへの影響確認
3. 管理パネルまたはAPI経由で更新
4. 変更通知の送信
5. 必要に応じてタスクの再実行

### バージョン管理
- 各ドキュメントの変更履歴をGitで管理
- 重要な変更はタグ付けでバージョン管理
- 後方互換性を保った更新を推奨

## 関連リンク
- [管理パネルドキュメント](../adminPanel.md)
- [タスクキューシステム設計](../taskQueueSystemDesign.md)
- [Kubernetes統合](../kubernetes/kubernetesIntegration.md)
- [リポジトリ管理](../repositoryManagement.md)

---

ご意見・ご要望はIssueまたはPRでお知らせください。