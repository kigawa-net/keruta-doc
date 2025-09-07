# Keycloak認証・認可設定

> **概要**: kerutaシステムの認証・認可をKeycloakで実現するためのセットアップ手順、設定例、セキュリティ考慮事項、トラブルシューティングをまとめたドキュメントです。

## 目次
- [概要](#概要)
- [セットアップ](#セットアップ)
- [設定例](#設定例)
- [セキュリティ考慮事項](#セキュリティ考慮事項)
- [FAQ・トラブルシューティング](#faqトラブルシューティング)
- [関連リンク](#関連リンク)

## 概要
kerutaシステムはKeycloakを用いてユーザー認証・認可を管理します。SSOやID管理、アクセス制御を一元化できます。

## セットアップ
### Keycloakサーバーのインストール
```bash
docker run -p 8180:8080 -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:latest start-dev
```
- 管理コンソール: http://localhost:8180/admin/
- 初期認証: admin / admin

### レルム・クライアント・ユーザー・ロール作成
- レルム名: keruta
- クライアントID: keruta（openid-connect, confidential）
- ユーザー作成・パスワード設定
- ロール: admin, user
- ユーザーにロール割当

## 設定例
### application.properties
```properties
spring.security.oauth2.client.registration.keycloak.client-id=keruta
spring.security.oauth2.client.registration.keycloak.client-secret=your-client-secret
spring.security.oauth2.client.registration.keycloak.scope=openid,profile,email
spring.security.oauth2.client.registration.keycloak.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.keycloak.redirect-uri={baseUrl}/login/oauth2/code/{registrationId}
spring.security.oauth2.client.provider.keycloak.issuer-uri=http://localhost:8180/realms/keruta
spring.security.oauth2.client.provider.keycloak.user-name-attribute=preferred_username
keycloak.realm=keruta
keycloak.auth-server-url=http://localhost:8180
keycloak.resource=keruta
keycloak.public-client=true
keycloak.principal-attribute=preferred_username
```

### 環境変数例
```bash
export KEYCLOAK_URL=https://keycloak.example.com
export KEYCLOAK_REALM=keruta-production
export KEYCLOAK_CLIENT_ID=keruta-app
export KEYCLOAK_CLIENT_SECRET=your-production-client-secret
```

## セキュリティ考慮事項
- クライアントシークレットは機密情報として管理
- 本番環境は必ずHTTPSを利用
- JWTトークンの署名・有効期限を検証
- ユーザーには最小権限のみ付与
- セッションタイムアウト・無効化を適切に設定

## FAQ・トラブルシューティング
- **認証エラー**: クライアントID/シークレットを確認
- **リダイレクトエラー**: Valid Redirect URIs設定を確認
- **ロールマッピングの問題**: ユーザーに適切なロールが割当てられているか確認
- **トークン検証失敗**: Keycloakサーバーのアドレス・レルム名を確認
- **Keycloakサーバーダウン時**: 認証サービスは利用不可

## 関連リンク
- [公式Keycloakドキュメント](https://www.keycloak.org/documentation)