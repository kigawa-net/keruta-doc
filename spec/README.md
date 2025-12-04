# keruta プロトコル仕様

## 概要

keruta はクラウドベースのバッチ処理システムであり、複数の worker を経由してタスクを処理する分散アーキテクチャを採用しています。本ディレクトリでは、keruta
システムの主要なプロトコル仕様を定義しています。

### 仕様の種類

- **タスク関連プロトコル**: タスクのスケジューリング、実行、管理に関するプロトコル
- **ドキュメント関連プロトコル**: ドキュメントの管理、共有、処理に関するプロトコル

## 仕様書一覧

### タスク関連

* [keruta task server](./task-server.md)
    - タスクサーバーの仕様

* [keruta task client protocol](./task-client-protocol.md)
    - KTCP (keruta task client protocol) 仕様
    - タスクのスケジューリングを制御するためのサーバー・クライアント間プロトコル

* [keruta task server protocol](./task-server-protocol.md)
    - タスクサーバープロトコルの仕様

### ドキュメント関連

* [keruta document server](./document-server.md)
    - ドキュメントサーバーの仕様

* [keruta document protocol](./document-protocol.md)
    - ドキュメントプロトコルの仕様

## 開発ガイドライン

- プロトコル仕様は明確で実装可能なものとする
- バージョン管理を考慮した設計を行う
- セキュリティ要件を明記する
- 互換性と拡張性を確保する
