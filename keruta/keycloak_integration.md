# Keycloak 統合 - 認証・認可の設定

## 概要

kerutaシステムは、Keycloakを使用してユーザー認証と認可を管理しています。Keycloakは、シングルサインオン（SSO）、ID管理、およびアクセス管理機能を提供するオープンソースのIdentity and Access Management（IAM）ソリューションです。

## セットアップ

### Keycloakサーバーのインストール

1. **Dockerを使用したKeycloakのセットアップ**:

```bash
docker run -p 8180:8080 -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:latest start-dev
```

2. **Keycloak管理コンソールへのアクセス**:
   - URL: http://localhost:8180/admin/
   - 初期認証情報:
     - ユーザー名: admin
     - パスワード: admin

### レルムの作成

1. 管理コンソールにログイン後、「Create realm」ボタンをクリックします。
2. レルム名に「keruta」を入力し、「Create」をクリックします。

### クライアントの設定

1. 左側のメニューから「Clients」を選択し、「Create client」をクリックします。
2. 以下の情報を入力します：
   - Client ID: keruta
   - Client Protocol: openid-connect
   - Root URL: http://localhost:8080/
3. 「Save」をクリックします。
4. クライアント設定画面で以下の設定を行います：
   - Access Type: confidential
   - Valid Redirect URIs: http://localhost:8080/*
   - Web Origins: http://localhost:8080
5. 「Save」をクリックします。
6. 「Credentials」タブを開き、クライアントシークレットを確認します（この値は後で使用します）。

### ユーザーの作成

1. 左側のメニューから「Users」を選択し、「Add user」をクリックします。
2. 以下の情報を入力します：
   - Username: kerutauser
   - Email: user@example.com
   - First Name: Keruta
   - Last Name: User
3. 「Create」をクリックします。
4. 「Credentials」タブを開き、「Set Password」をクリックします。
5. パスワードを入力し、「Temporary」をオフにして「Save」をクリックします。

### ロールの設定

1. 左側のメニューから「Roles」を選択し、「Create role」をクリックします。
2. 以下のロールを作成します：
   - admin: 管理者権限
   - user: 一般ユーザー権限
3. 「Users」メニューに戻り、作成したユーザーを選択します。
4. 「Role Mappings」タブを開き、適切なロールを割り当てます。

## 設定

### application.propertiesの設定

Keycloak統合を使用するには、以下の設定が必要です：

```properties
# Keycloak OAuth2クライアント設定
spring.security.oauth2.client.registration.keycloak.client-id=${KEYCLOAK_CLIENT_ID:keruta}
spring.security.oauth2.client.registration.keycloak.client-secret=${KEYCLOAK_CLIENT_SECRET:your-client-secret}
spring.security.oauth2.client.registration.keycloak.scope=${KEYCLOAK_SCOPE:openid,profile,email}
spring.security.oauth2.client.registration.keycloak.authorization-grant-type=${KEYCLOAK_GRANT_TYPE:authorization_code}
spring.security.oauth2.client.registration.keycloak.redirect-uri=${KEYCLOAK_REDIRECT_URI:{baseUrl}/login/oauth2/code/{registrationId}}

# Keycloakプロバイダー設定
spring.security.oauth2.client.provider.keycloak.issuer-uri=${KEYCLOAK_URL:http://localhost:8180}/realms/${KEYCLOAK_REALM:keruta}
spring.security.oauth2.client.provider.keycloak.user-name-attribute=${KEYCLOAK_USERNAME_ATTRIBUTE:preferred_username}

# Keycloakアダプター設定
keycloak.realm=${KEYCLOAK_REALM:keruta}
keycloak.auth-server-url=${KEYCLOAK_URL:http://localhost:8180}
keycloak.resource=${KEYCLOAK_CLIENT_ID:keruta}
keycloak.public-client=${KEYCLOAK_PUBLIC_CLIENT:true}
keycloak.principal-attribute=${KEYCLOAK_USERNAME_ATTRIBUTE:preferred_username}
```

| プロパティ                       | 説明                                |
|-----------------------------|-----------------------------------|
| KEYCLOAK_CLIENT_ID          | Keycloakで設定したクライアントID             |
| KEYCLOAK_CLIENT_SECRET      | Keycloakで生成されたクライアントシークレット        |
| KEYCLOAK_SCOPE              | 要求するスコープ（通常はopenid,profile,email） |
| KEYCLOAK_GRANT_TYPE         | 認可グラントタイプ（通常はauthorization_code）  |
| KEYCLOAK_REDIRECT_URI       | 認証後のリダイレクトURI                     |
| KEYCLOAK_URL                | KeycloakサーバーのURL                  |
| KEYCLOAK_REALM              | 使用するKeycloakレルム                   |
| KEYCLOAK_USERNAME_ATTRIBUTE | ユーザー名として使用する属性                    |
| KEYCLOAK_PUBLIC_CLIENT      | パブリッククライアントかどうか                   |

### 環境変数による設定

本番環境では、セキュリティを確保するために環境変数を使用して設定することをお勧めします：

```bash
export KEYCLOAK_URL=https://keycloak.example.com
export KEYCLOAK_REALM=keruta-production
export KEYCLOAK_CLIENT_ID=keruta-app
export KEYCLOAK_CLIENT_SECRET=your-production-client-secret
```

## セキュリティ考慮事項

1. **クライアントシークレットの保護**: クライアントシークレットは機密情報として扱い、ソースコードやバージョン管理システムに保存しないでください。

2. **HTTPS**: 本番環境では、すべての通信をHTTPS経由で行うようにしてください。

3. **トークン検証**: JWTトークンの署名と有効期限を常に検証してください。

4. **最小権限の原則**: ユーザーには必要最小限の権限のみを付与してください。

5. **セッション管理**: 適切なセッションタイムアウトを設定し、不要になったセッションを無効化してください。

## トラブルシューティング

1. **認証エラー**: クライアントIDとシークレットが正しいことを確認してください。

2. **リダイレクトエラー**: Valid Redirect URIsの設定が正しいことを確認してください。

3. **ロールマッピングの問題**: ユーザーに適切なロールが割り当てられていることを確認してください。

4. **トークン検証の失敗**: Keycloakサーバーのアドレスとレルム名が正しいことを確認してください。

## 制限事項

1. Keycloakサーバーがダウンしている場合、認証サービスは利用できません。

2. カスタム認証フローを実装する場合は、追加の設定が必要になる場合があります。