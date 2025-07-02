# 共通API仕様 目次

このディレクトリにはkerutaプロジェクトのAPI仕様をAPI種別ごとに分割して記載しています。

## 目次
- [API共通仕様](apiSpec/common.md)
- [keruta-agent API仕様](apiSpec/agent.md)
- [タスク管理API仕様](apiSpec/task.md)
- [エージェント管理API仕様](apiSpec/agentManagement.md)
- [リポジトリ管理API仕様](apiSpec/repository.md)
- [ドキュメント管理API仕様](apiSpec/document.md)

# API仕様について

kerutaプロジェクトのAPI仕様は、上記の目次にあるように各カテゴリごとに分割されたファイルに記載されています。
各ファイルには、それぞれのAPIカテゴリに関する詳細な仕様、エンドポイント、リクエスト/レスポンス例が含まれています。

## 共通仕様の概要

[API共通仕様](apiSpec/common.md)には、以下の内容が含まれています：

- API設計方針
- 認証・認可
- エラーハンドリング
- 共通レスポンス形式
- バージョニング
- CORS (Cross-Origin Resource Sharing)
- その他共通事項

## 各APIカテゴリの概要

- [keruta-agent API仕様](apiSpec/agent.md): keruta-agentがkeruta APIサーバーと通信する際のAPI仕様
- [タスク管理API仕様](apiSpec/task.md): タスクの作成、取得、更新、削除に関するAPI
- [エージェント管理API仕様](apiSpec/agentManagement.md): エージェントの管理に関するAPI
- [リポジトリ管理API仕様](apiSpec/repository.md): リポジトリの管理に関するAPI
- [ドキュメント管理API仕様](apiSpec/document.md): ドキュメントの管理に関するAPI

## 関連ドキュメント
- [システム詳細・セットアップ](keruta/system/projectDetails.md)
- [データモデル定義](keruta/system/dataModel.md)