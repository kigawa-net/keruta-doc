# ログ仕様

## 概要

kerutaシステムのログは、システムの監視、デバッグ、トラブルシューティングを目的とした構造化されたログを提供します。ログは以下のコンポーネントで生成されます：

- keruta-api: Spring Bootアプリケーションのログ
- keruta-agent: Kubernetes Job実行時のログ
- ログストリーミング: リアルタイムログ配信

## ログレベル

以下のログレベルを使用します：

- **ERROR**: システムエラー、回復不能な問題
- **WARN**: 警告、潜在的な問題
- **INFO**: 一般的な情報、重要なイベント
- **DEBUG**: デバッグ情報、開発時使用
- **TRACE**: 詳細なトレース情報

## ログフォーマット

### APIログフォーマット

```
%d{yyyy-MM-dd HH:mm:ss.SSS} %5p [${spring.application.name}] [%X{requestId}] [%X{userId}] [%t] [%c{1}:%L] - %m%n
```

含まれる情報：
- タイムスタンプ（ミリ秒まで）
- ログレベル
- アプリケーション名
- リクエストID
- ユーザーID
- スレッド名
- クラス名と行番号
- ログメッセージ

### エージェントログフォーマット

JSON形式の構造化ログ：

```json
{
  "timestamp": "2025-12-05T10:15:30.123Z",
  "level": "INFO",
  "taskId": "task123",
  "source": "stdout",
  "message": "Script execution started",
  "metadata": {
    "podName": "keruta-job-abc123",
    "namespace": "default"
  }
}
```

## ログストリーミング

ログはリアルタイムで以下の宛先にストリーミングされます：

1. **WebSocket**: クライアントへのリアルタイム配信
2. **データベース**: 永続化ストレージ
3. **ファイル**: ローカルログファイル

### WebSocketメッセージフォーマット

```json
{
  "type": "log",
  "taskId": "task123",
  "timestamp": "2025-12-05T10:15:30.123Z",
  "level": "INFO",
  "source": "stdout",
  "message": "Processing complete",
  "sequence": 42
}
```

## MDC (Mapped Diagnostic Context)

APIログでは以下のコンテキスト情報がMDCに追加されます：

- `requestId`: リクエスト識別子
- `userId`: 認証ユーザーID
- `method`: HTTPメソッド
- `path`: リクエストパス
- `ip`: クライアントIPアドレス

## ログ保持ポリシー

- **APIログ**: 30日保持、自動ローテーション
- **エージェントログ**: タスク完了後90日保持
- **ストリーミングログ**: データベースに永続化、1年保持

## セキュリティ考慮事項

- 機密情報（パスワード、トークン）はログから除外
- ログフィルタリングによりセンシティブデータのマスキング
- ログアクセスは認証済みユーザーのみ