# keruta-builder WebSocket通信仕様

このドキュメントは、keruta-builderとkeruta本体間で行われるWebSocket通信の仕様についてまとめたものです。

## 概要

- keruta-builderは起動時にkeruta本体へWebSocket接続を行います。
- keruta本体はWebSocket経由でビルド・pushリクエストを送信します。
- builderはリクエストを受信し、Dockerイメージのビルド・push処理を実行します。
- ビルドやpushの進捗・結果もWebSocket経由でkeruta本体へ通知します。

## 通信の流れ

1. keruta-builderがkeruta本体のWebSocketサーバーに接続する
2. keruta本体がpushリクエストメッセージを送信する
3. builderがリクエストを受信し、ビルド・push処理を開始
4. builderが進捗・結果をWebSocketでkeruta本体に送信

## メッセージ例

### pushリクエスト（keruta本体→builder）
```json
{
  "type": "push-request",
  "repository": "example/app",
  "tag": "v1.0.0",
  "dockerfile": "...Dockerfileの内容...",
  "buildArgs": {"KEY": "VALUE"}
}
```

### 進捗通知（builder→keruta本体）
```json
{
  "type": "progress",
  "status": "building",
  "message": "イメージをビルド中..."
}
```

### 結果通知（builder→keruta本体）
```json
{
  "type": "result",
  "status": "success",
  "imageUrl": "registry.example.com/example/app:v1.0.0"
}
```

## エラーハンドリング

エラー発生時の対応方針は以下の通りです。

### 主なエラー種別と対応

- **ビルド失敗（Dockerfileエラー、依存取得失敗など）**
  - status: "failed"
  - エラーメッセージを含めてkeruta本体に通知
  - 原因が一時的なものであれば自動再ビルドを最大N回まで試行

- **push失敗（認証エラー、レジストリ到達不可など）**
  - status: "failed"
  - エラー内容を通知
  - 一時的なネットワーク障害等の場合は自動再試行

- **WebSocket通信エラー**
  - 再接続を自動で試行
  - 一定回数失敗時はエラー通知し、タスクを"unknown"または"failed"として扱う

- **不明なエラー・例外**
  - status: "error"
  - 詳細な例外内容を通知
  - 必要に応じて管理者へアラート

### エラー通知の例

```json
{
  "type": "result",
  "status": "failed",
  "errorType": "build",
  "message": "Dockerfileの構文エラー: ..."
}
```

### 備考

- エラー内容・再試行回数・通知方法は今後運用に応じて調整可能です。
- 重大な障害時は管理者への通知やログ出力も行います。

## 備考

- 実際のメッセージ仕様やフィールドは今後変更される可能性があります。
- 認証やエラーハンドリングの詳細は別途定義予定です。
- 異常終了時は、エラーの原因に応じてタスクの結果（status）を変更したり、必要に応じて自動的に再ビルドを行う設計とします。 