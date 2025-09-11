# Keycloak認証・認可設定 - システム全体統合

> **概要**: kerutaシステム全体の認証・認可をKeycloakで統一実現するためのセットアップ手順、設定例、セキュリティ考慮事項、トラブルシューティングをまとめたドキュメントです。全コンポーネントでKeycloak認証を使用し、OIDCオフラインモードにも対応します。

## 目次
- [概要](#概要)
- [システム全体アーキテクチャ](#システム全体アーキテクチャ)
- [セットアップ](#セットアップ)
- [設定例](#設定例)
- [OIDCオフラインモード](#oidcオフラインモード)
- [コンポーネント別設定](#コンポーネント別設定)
- [セキュリティ考慮事項](#セキュリティ考慮事項)
- [FAQ・トラブルシューティング](#faqトラブルシューティング)
- [関連リンク](#関連リンク)

## 概要
kerutaシステム全体でKeycloakを用いてユーザー認証・認可を統一管理します。従来のinMemory認証から完全移行し、すべてのコンポーネント（API、Admin、Agent、Builder）でSSOやID管理、アクセス制御を一元化します。OIDCオフラインモードにも対応し、ネットワーク断絶時でも限定的な認証を可能にします。

## システム全体アーキテクチャ

```
┌─────────────────────────────────────────────────────────────┐
│                    Keycloak Server                          │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │   Realm:    │ │  Client:    │ │   Users &   │           │
│  │   keruta    │ │  keruta-*   │ │   Roles     │           │
│  └─────────────┘ └─────────────┘ └─────────────┘           │
└─────────────────────────┬───────────────────────────────────┘
                          │ OIDC Protocol
                          │ (Online + Offline Mode)
    ┌─────────────────────┼─────────────────────┐
    │                     │                     │
    ▼                     ▼                     ▼
┌─────────┐         ┌─────────┐         ┌─────────┐
│ keruta  │         │ keruta  │         │ keruta  │
│   API   │◄────────┤  Admin  │◄────────┤  Agent  │
│ Service │         │  Panel  │         │   CLI   │
└─────────┘         └─────────┘         └─────────┘
    │                     │                     │
    ▼                     ▼                     ▼
┌─────────┐         ┌─────────┐         ┌─────────┐
│ MongoDB │         │  React  │         │ Worker  │
│Database │         │Frontend │         │  Pods   │
└─────────┘         └─────────┘         └─────────┘
```

## セットアップ
### Keycloakサーバーのインストール
```bash
docker run -p 8180:8080 -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:latest start-dev
```
- 管理コンソール: http://localhost:8180/admin/
- 初期認証: admin / admin

### レルム・クライアント・ユーザー・ロール作成

#### レルム作成
- レルム名: `keruta`
- 設定: SSOセッション最大時間、リフレッシュトークン有効期限等を調整

#### クライアント作成（各コンポーネント用）
1. **keruta-api** (Server-side API用)
   - プロトコル: openid-connect
   - アクセスタイプ: confidential
   - サービスアカウント有効: ON
   - オフラインアクセス有効: ON

2. **keruta-admin** (Admin UI用)
   - プロトコル: openid-connect  
   - アクセスタイプ: public
   - リダイレクトURI: http://localhost:3000/*, https://admin.keruta.local/*
   - オフラインアクセス有効: ON

3. **keruta-agent** (CLI Agent用)
   - プロトコル: openid-connect
   - アクセスタイプ: public
   - デバイスフロー有効: ON
   - オフラインアクセス有効: ON

#### ロール・ユーザー作成
- ロール: `system-admin`, `project-admin`, `developer`, `viewer`
- ユーザー作成・パスワード設定
- ユーザーにロール割当

## 設定例

### keruta-api application.properties
```properties
# Keycloak OAuth2 設定
spring.security.oauth2.client.registration.keycloak.client-id=keruta-api
spring.security.oauth2.client.registration.keycloak.client-secret=${KEYCLOAK_CLIENT_SECRET}
spring.security.oauth2.client.registration.keycloak.scope=openid,profile,email,offline_access
spring.security.oauth2.client.registration.keycloak.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.keycloak.redirect-uri={baseUrl}/login/oauth2/code/{registrationId}
spring.security.oauth2.client.provider.keycloak.issuer-uri=${KEYCLOAK_SERVER_URL}/realms/keruta
spring.security.oauth2.client.provider.keycloak.user-name-attribute=preferred_username

# Keycloak Resource Server 設定
spring.security.oauth2.resourceserver.jwt.issuer-uri=${KEYCLOAK_SERVER_URL}/realms/keruta
spring.security.oauth2.resourceserver.jwt.jwk-set-uri=${KEYCLOAK_SERVER_URL}/realms/keruta/protocol/openid_connect/certs

# オフラインモード設定
keycloak.offline-mode.enabled=true
keycloak.offline-mode.cache-duration=3600
keycloak.offline-mode.fallback-roles=viewer
```

### 環境変数例
```bash
# 本番環境
export KEYCLOAK_SERVER_URL=https://keycloak.example.com
export KEYCLOAK_CLIENT_SECRET=your-production-client-secret

# 開発環境
export KEYCLOAK_SERVER_URL=http://localhost:8180
export KEYCLOAK_CLIENT_SECRET=dev-client-secret
```

## OIDCオフラインモード

### 概要
OIDCオフラインモードは、Keycloakサーバーに接続できない場合でも限定的な認証・認可を継続できる機能です。リフレッシュトークンとキャッシュされたユーザー情報を使用します。

### 動作仕様

#### オンライン時（通常動作）
1. ユーザー認証をKeycloakサーバーで実行
2. アクセストークン、リフレッシュトークン、オフライントークンを取得
3. ユーザー情報とロール情報をローカルキャッシュに保存
4. トークン有効期限監視とリフレッシュ処理

#### オフライン時（断絶対応）
1. Keycloakサーバーへの接続失敗を検知
2. ローカルキャッシュからユーザー情報とロール情報を取得
3. オフライントークンの有効性確認（署名検証）
4. キャッシュ有効期限内であれば認証継続
5. 制限モード：読み取り専用操作のみ許可

### 設定項目
```properties
# オフラインモード有効化
keycloak.offline-mode.enabled=true

# キャッシュ保持期間（秒）
keycloak.offline-mode.cache-duration=3600

# オフライン時のフォールバック権限
keycloak.offline-mode.fallback-roles=viewer

# オフライン時の許可操作
keycloak.offline-mode.allowed-operations=READ,LIST

# キャッシュ暗号化キー
keycloak.offline-mode.cache-encryption-key=${CACHE_ENCRYPTION_KEY}
```

### 制限事項
- 新規ユーザー登録・権限変更は不可
- キャッシュ有効期限切れ後は認証不可
- パスワード変更等の重要操作は不可
- 監査ログは蓄積されサーバー復旧時に同期

## コンポーネント別設定

### keruta-api (Spring Boot)
```properties
# JWT検証設定
spring.security.oauth2.resourceserver.jwt.issuer-uri=${KEYCLOAK_SERVER_URL}/realms/keruta
spring.security.oauth2.resourceserver.jwt.cache-duration=PT5M

# ロールマッピング
keycloak.role-mappings.system-admin=ADMIN
keycloak.role-mappings.project-admin=PROJECT_ADMIN
keycloak.role-mappings.developer=DEVELOPER
keycloak.role-mappings.viewer=VIEWER

# API保護設定
keycloak.protected-endpoints=/api/admin/**,/api/project/**
keycloak.public-endpoints=/api/health,/api/public/**
```

### keruta-admin (React SPA)
```javascript
// keycloak-js設定
const keycloakConfig = {
  url: process.env.REACT_APP_KEYCLOAK_URL,
  realm: 'keruta',
  clientId: 'keruta-admin',
  onLoad: 'login-required',
  checkLoginIframe: false,
  enableLogging: true,
  // オフラインアクセス有効化
  scope: 'openid profile email offline_access'
};

// オフラインモード対応
const offlineConfig = {
  enabled: true,
  storageKey: 'keruta-offline-token',
  maxAge: 3600000 // 1時間
};
```

### keruta-agent (CLI)
```yaml
# CLI設定ファイル (~/.keruta/config.yaml)
auth:
  keycloak:
    server_url: "${KEYCLOAK_SERVER_URL}"
    realm: "keruta"
    client_id: "keruta-agent"
    device_flow: true
    offline_access: true
  
  offline_mode:
    enabled: true
    token_file: "~/.keruta/offline_token"
    cache_duration: 3600
    fallback_role: "viewer"

# デバイスフロー認証コマンド
# keruta auth login --device-flow
```

### keruta-builder (Build System)
```properties
# サービスアカウント認証
keycloak.service-account.client-id=keruta-api
keycloak.service-account.client-secret=${BUILDER_CLIENT_SECRET}
keycloak.service-account.scope=build,deploy

# ビルド実行権限
keycloak.required-roles.build=developer,project-admin
keycloak.required-roles.deploy=project-admin,system-admin
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