# keruta 仕様

## 概要

keruta はクラウドベースのバッチ処理システムであり、複数の worker を経由してタスクを処理する分散アーキテクチャを採用しています。本ディレクトリでは、keruta
システムの主要な仕様を定義しています。

### 仕様の種類

- **タスク関連**: タスクのスケジューリング、実行、管理
- **プロバイダー関連**: タスクの実行を行うモジュール
- **ログ関連**: ログフォーマット、ストリーミング、保持ポリシー

## 仕様書一覧

### タスク関連

* [keruta task server](./task-server.md)
    - タスクサーバーの仕様

* [keruta task client protocol](./task-client-protocol.md)
    - KTCP (keruta task client protocol) 仕様
    - タスクのスケジューリングを制御するためのサーバー・クライアント間プロトコル

* [keruta task server protocol](./task-server-protocol.md)
    - タスクサーバープロトコルの仕様

### プロバイダー関連

* [keruta coder provider](./coder-provider.md)
    - coder 環境管理プロバイダーの仕様
    - KTCP を使用した環境起動管理

### ログ関連

* [keruta log](./log.md)
    - ログフォーマット、ログレベル、ログストリーミングの仕様

### モジュール仕様

モジュール間の依存関係は [モジュール構成図](./module/modules.png) を参照してください。

* [kodel:core](./module/kodel:core.md)
    - DI、エラーハンドリング、ログ管理を提供する汎用ライブラリのコアモジュール

* [kodel:api](./module/kodel:api.md)
    - API 層でのエラーハンドリング、ログ統合、DI 統合を提供

* [ktcp:model](./module/ktcp:model.md)
    - KTCP プロトコルのデータモデル定義

* [ktcp:client](./module/ktcp:client.md)
    - KTCP クライアント実装（プロバイダー側）

* [ktcp:server](./module/ktcp:server.md)
    - KTCP サーバー実装（タスクサーバー側）

* [ktse](./module/ktse.md)
    - Ktor WebSocket を使用した KTCP サーバーエンジン

## 開発ガイドライン

- プロトコル仕様は明確で実装可能なものとする
- バージョン管理を考慮した設計を行う
- セキュリティ要件を明記する
- 互換性と拡張性を確保する
