# Keruta Logging Configuration

> **概要**: このドキュメントでは、Kerutaアプリケーションのログ設定について詳細に説明します。

## 目次
- [概要](#概要)
- [ログレベル](#ログレベル)
- [ログフォーマット](#ログフォーマット)
- [MDC (Mapped Diagnostic Context)](#mdc-mapped-diagnostic-context)
- [リクエストとレスポンスのログ](#リクエストとレスポンスのログ)
- [セキュリティ関連のログ](#セキュリティ関連のログ)
- [ログの例](#ログの例)

## 概要

Kerutaアプリケーションは、デバッグ、監視、トラブルシューティングを容易にするために詳細なログを提供します。ログ設定は、アプリケーションの各モジュールの`application.properties`ファイルで構成されています。

## ログレベル

以下のログレベルが設定されています：

```properties
# 基本的なコンポーネントのログレベル
logging.level.org.springframework.data.mongodb=INFO
logging.level.net.kigawa.keruta=DEBUG

# 特定のコンポーネントのログレベル
logging.level.net.kigawa.keruta.core.usecase.task.background.BackgroundTaskProcessor=WARN
logging.level.net.kigawa.keruta.core.usecase.task.TaskServiceImpl=WARN
logging.level.net.kigawa.keruta.infra.app.kubernetes.KubernetesServiceImpl=WARN

# セキュリティ関連のコンポーネントのログレベル
logging.level.org.springframework.security=DEBUG
logging.level.net.kigawa.keruta.infra.security=DEBUG
logging.level.org.keycloak=DEBUG

# HTTPリクエストとレスポンスのログレベル
logging.level.org.springframework.web=DEBUG
logging.level.org.springframework.web.filter.CommonsRequestLoggingFilter=DEBUG
```

## ログフォーマット

ログは以下のフォーマットで出力されます：

```properties
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss.SSS} %5p [${spring.application.name}] [%X{requestId}] [%X{userId}] [%t] [%c{1}:%L] - %m%n
logging.pattern.file=%d{yyyy-MM-dd HH:mm:ss.SSS} %5p [${spring.application.name}] [%X{requestId}] [%X{userId}] [%t] [%c{1}:%L] - %m%n
```

このフォーマットには以下の情報が含まれます：
- タイムスタンプ（ミリ秒まで）
- ログレベル（5文字に調整）
- アプリケーション名
- リクエストID（MDCから）
- ユーザーID（MDCから）
- スレッド名
- クラス名と行番号
- ログメッセージ

## MDC (Mapped Diagnostic Context)

MDCは、ログメッセージに追加のコンテキスト情報を提供するために使用されます。Kerutaアプリケーションでは、以下の情報がMDCに追加されます：

- `requestId`: リクエストを一意に識別するID（`X-Request-ID`ヘッダーから取得、または自動生成）
- `userId`: リクエストを行ったユーザーのID（認証されている場合）
- `method`: HTTPメソッド（GET、POST、PUT、DELETEなど）
- `path`: リクエストパス
- `ip`: クライアントのIPアドレス

これらの情報は、`LoggingFilter`によって各リクエストのMDCに追加されます。

## リクエストとレスポンスのログ

HTTPリクエストとレスポンスの詳細は、以下の2つのフィルターによってログに記録されます：

1. `RequestLoggingFilter`: リクエストとレスポンスの詳細（ヘッダー、ボディ、ステータスなど）をログに記録します。
2. `CommonsRequestLoggingFilter`: Spring Frameworkの組み込みフィルターで、リクエストの詳細をログに記録します。

これらのフィルターは、静的リソース（CSS、JavaScript、画像など）のリクエストに対してはログを記録しないように設定されています。

## セキュリティ関連のログ

セキュリティ関連のログは、以下のコンポーネントによって記録されます：

- `org.springframework.security`: Spring Securityのログ
- `net.kigawa.keruta.infra.security`: Kerutaアプリケーションのセキュリティモジュールのログ
- `org.keycloak`: Keycloakのログ

これらのログは、認証、認可、セキュリティフィルターなどの情報を提供します。

## ログの例

以下は、ログの例です：

```
2025-07-04 10:15:30.123 DEBUG [keruta] [550e8400-e29b-41d4-a716-446655440000] [john.doe] [http-nio-8080-exec-1] [LoggingFilter:48] - Request: GET /api/v1/tasks from IP: 192.168.1.100
2025-07-04 10:15:30.125 DEBUG [keruta] [550e8400-e29b-41d4-a716-446655440000] [john.doe] [http-nio-8080-exec-1] [RequestLoggingFilter:31] - Request Headers: host: localhost:8080, connection: keep-alive, accept: application/json, Authorization: [REDACTED], 
2025-07-04 10:15:30.150 DEBUG [keruta] [550e8400-e29b-41d4-a716-446655440000] [john.doe] [http-nio-8080-exec-1] [JwtAuthenticationFilter:26] - JWT token validated successfully
2025-07-04 10:15:30.200 DEBUG [keruta] [550e8400-e29b-41d4-a716-446655440000] [john.doe] [http-nio-8080-exec-1] [TaskController:45] - Getting all tasks for user: john.doe
2025-07-04 10:15:30.250 DEBUG [keruta] [550e8400-e29b-41d4-a716-446655440000] [john.doe] [http-nio-8080-exec-1] [TaskServiceImpl:78] - Found 5 tasks for user: john.doe
2025-07-04 10:15:30.300 DEBUG [keruta] [550e8400-e29b-41d4-a716-446655440000] [john.doe] [http-nio-8080-exec-1] [RequestLoggingFilter:51] - Response Status: 200, Headers: content-type: application/json, x-request-id: 550e8400-e29b-41d4-a716-446655440000, Duration: 180ms
2025-07-04 10:15:30.301 DEBUG [keruta] [550e8400-e29b-41d4-a716-446655440000] [john.doe] [http-nio-8080-exec-1] [RequestLoggingFilter:58] - Response Body: {"tasks":[{"id":"1","title":"Task 1"},{"id":"2","title":"Task 2"},{"id":"3","title":"Task 3"},{"id":"4","title":"Task 4"},{"id":"5","title":"Task 5"}]}
```

このログから、以下の情報を読み取ることができます：
- リクエストは2025年7月4日10時15分30秒に行われました
- リクエストIDは`550e8400-e29b-41d4-a716-446655440000`です
- ユーザーID（認証されたユーザー）は`john.doe`です
- リクエストは`GET /api/v1/tasks`で、IPアドレス`192.168.1.100`から来ています
- JWTトークンは正常に検証されました
- ユーザー`john.doe`のタスクが5つ見つかりました
- レスポンスのステータスは200で、処理時間は180msです
- レスポンスボディにはタスクのリストが含まれています
