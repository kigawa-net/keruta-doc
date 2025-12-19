# ktse - Keruta Task Server Engine

## 概要

KTCP サーバーの WebSocket 実装を提供するモジュール。タスクサーバーで使用される WebSocket エンジン。

## 責務

- WebSocket サーバーの具体的実装
- WebSocket 接続のライフサイクル管理
- メッセージの送受信処理
- 接続プーリング
- パフォーマンス最適化

## 主要機能

- WebSocket エンドポイント提供（`/ws`）
- 双方向メッセージング
- 接続状態管理
- メッセージのバッチ処理
- 圧縮・最適化
- エラーハンドリング

## エンドポイント

```
ws://host:port/ws
wss://host:port/ws（推奨）
```

## 依存関係

- `ktcp:server` - KTCP サーバーインターフェース
- `ktcp:model` - プロトコルメッセージモデル
- Ktor WebSocket
