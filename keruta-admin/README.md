# keruta-admin ドキュメント目次

本ディレクトリには、kerutaシステムの管理パネル機能に関する技術・運用ドキュメントが含まれています。

## 管理パネル関連ドキュメント
- [adminPanel.md](./adminPanel.md)  
  管理パネルの機能・画面・操作方法
- [adminPanelRemix.md](./adminPanelRemix.md)  
  Remix.jsを使った管理パネルの実装
- [adminPanelScriptGenerator.md](./adminPanelScriptGenerator.md)  
  インストールスクリプト作成機能
- [repositoryManagement.md](./repositoryManagement.md)  
  リポジトリ管理機能
- [backendConfiguration.md](./backendConfiguration.md)  
  環境変数を使用したバックエンド設定

---

各ドキュメントの詳細はリンク先をご参照ください。 

## 認証機能仕様

keruta-adminは、kerutaシステムの管理パネルとしてKeycloak認証を採用しており、WebインターフェースとバックエンドAPIの両方でエンタープライズグレードの認証・認可システムを提供します。

### Keycloak認証システム

#### 概要
エンタープライズグレードの認証・認可を実現するため、Keycloakとの連携により以下を提供します：
- **Webインターフェース認証**: 管理パネルへのブラウザベースアクセス
- **API認証**: バックエンドAPIへのプログラマティックアクセス  
- **SSO（Single Sign-On）**: kerutaエコシステム全体での統一認証体験
- **ID管理・アクセス制御**: 一元化されたユーザー管理と権限制御

#### 主要設定項目

##### Web認証用設定
```bash
# Keycloak環境変数（Web認証）
export KEYCLOAK_URL=https://keycloak.example.com
export KEYCLOAK_REALM=keruta-production
export KEYCLOAK_CLIENT_ID=keruta-admin-web
export KEYCLOAK_CLIENT_SECRET=your-web-client-secret
```

##### API認証用設定  
```bash
# Keycloak環境変数（API認証）
export KEYCLOAK_API_CLIENT_ID=keruta-admin-api
export KEYCLOAK_API_CLIENT_SECRET=your-api-client-secret
export KEYCLOAK_AUDIENCE=keruta-api
```

#### Spring Boot設定例

##### Web認証設定
```properties
# application.properties (Web認証)
spring.security.oauth2.client.registration.keycloak.client-id=${KEYCLOAK_CLIENT_ID}
spring.security.oauth2.client.registration.keycloak.client-secret=${KEYCLOAK_CLIENT_SECRET}
spring.security.oauth2.client.registration.keycloak.scope=openid,profile,email
spring.security.oauth2.client.registration.keycloak.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.keycloak.redirect-uri={baseUrl}/login/oauth2/code/{registrationId}
spring.security.oauth2.client.provider.keycloak.issuer-uri=${KEYCLOAK_URL}/realms/${KEYCLOAK_REALM}
spring.security.oauth2.client.provider.keycloak.user-name-attribute=preferred_username

keycloak.realm=${KEYCLOAK_REALM}
keycloak.auth-server-url=${KEYCLOAK_URL}
keycloak.resource=${KEYCLOAK_CLIENT_ID}
keycloak.public-client=false
keycloak.principal-attribute=preferred_username
```

##### API認証設定（JWT検証）
```properties
# application.properties (API認証)
spring.security.oauth2.resourceserver.jwt.issuer-uri=${KEYCLOAK_URL}/realms/${KEYCLOAK_REALM}
spring.security.oauth2.resourceserver.jwt.jwk-set-uri=${KEYCLOAK_URL}/realms/${KEYCLOAK_REALM}/protocol/openid_connect/certs

# APIクライアント設定  
keycloak.api.client-id=${KEYCLOAK_API_CLIENT_ID}
keycloak.api.client-secret=${KEYCLOAK_API_CLIENT_SECRET}
keycloak.api.audience=${KEYCLOAK_AUDIENCE}
```

#### サポート機能

##### Web認証機能
- **OpenID Connect準拠の認証フロー**: 標準的なOAuth 2.0/OpenID Connectプロトコルに基づくブラウザ認証
- **セッション管理**: 自動ログアウト、セッションタイムアウト、同時セッション制御
- **Single Sign-On（SSO）**: kerutaエコシステム全体での統一認証体験

##### API認証機能
- **JWT Bearer Token認証**: OAuth 2.0 JWTトークンによるAPIアクセス認証
- **Resource Server設定**: Spring SecurityのJWT検証によるAPIエンドポイント保護
- **スコープベース認可**: APIリソースへの細かなアクセス制御
- **プログラマティックアクセス**: クライアントアプリケーションからのトークン取得

##### 共通機能
- **ロールベースアクセス制御（RBAC）**: WebとAPI両方できめ細かな権限管理
- **マルチファクター認証（MFA）**: 追加のセキュリティレイヤーとしての二要素認証
- **監査ログ**: すべての認証・認可イベントの完全なログ記録

#### ロール・権限設定

Keycloakレルム内で以下のロールとグループを設定することを推奨します：

**管理者ロール（keruta-admin）**
- 全システム機能へのアクセス権限
- ユーザー管理、システム設定変更権限
- 監査ログ閲覧権限

**運用者ロール（keruta-operator）**
- タスク管理、ジョブ実行権限
- ログ閲覧、モニタリング権限
- 設定変更権限なし

**閲覧者ロール（keruta-viewer）**
- 読み取り専用アクセス権限
- ダッシュボード、レポート閲覧権限
- 操作権限なし

#### セキュリティ要件
- **HTTPS通信**: 本番環境では必ずHTTPS利用（TLS 1.2以上推奨）
- **クライアントシークレット管理**: 機密情報として適切に保護・管理
- **最小権限の原則**: ユーザーには必要最小限の権限のみ付与
- **セッションセキュリティ**: 適切なタイムアウト設定、セッション無効化機能
- **監査ログ**: 認証・認可イベントの完全なログ記録

### セットアップ手順

#### 1. Keycloakサーバーのセットアップ
```bash
# Dockerを使用した開発環境でのKeycloak起動
docker run -p 8180:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:latest start-dev
```

#### 2. レルムとクライアントの設定
1. **レルム作成**: `keruta` レルムを作成
2. **クライアント設定**:
   - クライアントID: `keruta-admin`
   - クライアントタイプ: `OpenID Connect`
   - アクセスタイプ: `confidential`
   - Valid Redirect URIs: `https://your-admin-domain.com/login/oauth2/code/keycloak`

#### 3. ユーザーとロールの設定
1. **ロール作成**: 前述の3つのロール（keruta-admin, keruta-operator, keruta-viewer）を作成
2. **ユーザー作成**: 必要なユーザーを作成し、適切なロールを割り当て
3. **グループ設定**: ロールベースのグループを作成してユーザー管理を効率化

### トラブルシューティング

#### よくあるエラーと解決方法

1. **認証エラー (401 Unauthorized)**
   - 原因: クライアントIDまたはクライアントシークレットが正しくない
   - 解決策: Keycloak管理コンソールでクライアント設定を再確認

2. **リダイレクトエラー**
   - 原因: Valid Redirect URIsの設定が不正
   - 解決策: アプリケーションのコールバックURLを正確に設定

3. **ロール権限エラー (403 Forbidden)**
   - 原因: ユーザーに適切なロールが割り当てられていない
   - 解決策: Keycloak管理コンソールでユーザーのロール割り当てを確認

4. **トークン検証エラー**
   - 原因: Keycloakサーバーのissuer-uriが正しくない
   - 解決策: レルム名とKeycloakサーバーのURLを確認

### 関連ドキュメント

詳細な設定方法については、以下のドキュメントを参照してください：
- [Keycloak連携設定](../keruta/auth/keycloakIntegration.md)
