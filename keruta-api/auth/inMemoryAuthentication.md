# In-Memory Authentication Configuration

> **概要**: kerutaシステムの内部APIアクセス用の認証メカニズムについて説明します。このドキュメントでは、システム内部で使用される認証ユーザーの設定方法を解説します。

## 目次
- [概要](#概要)
- [認証ユーザー](#認証ユーザー)
- [設定方法](#設定方法)
- [トラブルシューティング](#トラブルシューティング)

## 概要
kerutaシステムは、内部APIアクセスのために簡易的な認証メカニズムを提供しています。この認証メカニズムは、システム内部のコンポーネント間の通信に使用され、Keycloak認証とは別に動作します。

## 認証ユーザー
システムには以下の認証ユーザーが定義されています：

1. **管理者ユーザー (admin)**
   - 用途: システム管理者用のアクセス権限
   - デフォルトユーザー名: `admin`
   - デフォルトパスワード: `password`
   - 権限: `ADMIN`

2. **API用ユーザー (keruta-api)**
   - 用途: システム内部のAPI間通信用
   - デフォルトユーザー名: `keruta-api`
   - デフォルトパスワード: `api-password`
   - 権限: `API`

## 設定方法
認証ユーザーの設定は、アプリケーションプロパティまたは環境変数で行います。

### application.properties での設定
```properties
# 管理者ユーザー設定
auth.admin.username=admin
auth.admin.password=password

# API用ユーザー設定
auth.api.username=keruta-api
auth.api.password=api-password
```

### 環境変数での設定
```bash
# 管理者ユーザー設定
export AUTH_ADMIN_USERNAME=admin
export AUTH_ADMIN_PASSWORD=secure-admin-password

# API用ユーザー設定
export AUTH_API_USERNAME=keruta-api
export AUTH_API_PASSWORD=secure-api-password
```

## トラブルシューティング

### よくあるエラー

1. **UsernameNotFoundException: User not found**
   - 原因: 認証に使用しているユーザー名が存在しない
   - 解決策: 
     - application.propertiesまたは環境変数で正しいユーザー名が設定されているか確認
     - UserServiceが正しく初期化されているか確認

2. **認証失敗エラー**
   - 原因: パスワードが一致しない
   - 解決策:
     - application.propertiesまたは環境変数で正しいパスワードが設定されているか確認
     - 環境変数の値が優先されるため、環境変数とプロパティファイルの両方を確認

3. **権限エラー**
   - 原因: ユーザーに必要な権限がない
   - 解決策:
     - ユーザーに適切な権限が割り当てられているか確認
     - APIユーザーには`API`権限、管理者ユーザーには`ADMIN`権限が必要
