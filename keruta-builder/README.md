# keruta-builder

keruta-builderは、kerutaプロジェクトのサブプロジェクトであり、指定されたDockerfileをビルドし、Dockerイメージをコンテナレジストリにpushするサーバーです。

## 主な役割

- Dockerfileを受け取り、Dockerイメージをビルド
- ビルドしたイメージを指定のレジストリへpush
- ビルド・pushの進捗や結果をAPI経由で取得可能

## 想定利用シーン

- CI/CDパイプラインの一部として、アプリケーションのDockerイメージを自動ビルド・デプロイ
- WebUIやAPI経由でDockerイメージのビルド・配布を管理

## 通信方式

- keruta-builderは起動後、keruta本体に対してWebSocket接続を行います。
- keruta本体はWebSocket経由でbuilderにpushリクエストを送信します。
- builderはリクエストを受信し、Dockerイメージのビルド・push処理を実行します。
- ビルドやpushの進捗・結果もWebSocket経由で通知します。

## 備考

- keruta本体や他サブプロジェクトと連携して利用されます
- 詳細なAPI仕様やセットアップ手順は今後追記予定です 