# Session-Workspace同期APIシステム

## 概要

本ドキュメントでは、KerutaにおけるSession-Workspace同期APIシステムの包括的な実装詳細を説明します。このシステムは、セッション管理とCoderワークスペース管理を自動化し、シームレスな統合を実現します。

### システムの目的

- **自動化されたワークスペース管理**: セッション作成時の自動ワークスペース作成
- **状態同期の自動化**: セッション状態とワークスペース状態の自動同期
- **リアルタイム監視**: セッションとワークスペースの状態をリアルタイムで監視
- **API駆動の管理**: RESTful APIを通じた統合管理

### アーキテクチャの変更点

従来のデータベース保存方式から、Coder API直接操作方式へ移行：

**以前**: Session → Database Workspace → Coder API
**現在**: Session → Coder API直接操作 (keruta-executor経由)

この変更により、データの一貫性向上と管理の簡素化を実現しています。

## システムアーキテクチャ

### コンポーネント構成

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│                 │    │                  │    │                 │
│  keruta-admin   │───▶│   keruta-api     │───▶│ keruta-executor │
│  (Frontend UI)  │    │ (Session API)    │    │ (Coder Manager) │
│                 │    │                  │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                │                         │
                                ▼                         ▼
                       ┌──────────────┐         ┌─────────────────┐
                       │   MongoDB    │         │   Coder Server  │
                       │ (Session DB) │         │  (Workspaces)   │
                       └──────────────┘         └─────────────────┘
```

### 主要コンポーネント

#### 1. keruta-api (Spring Boot API Server)
- **役割**: セッション管理とWorkspace APIプロキシ
- **機能**: 
  - セッションCRUD操作
  - ワークスペースAPIプロキシ
  - ExecutorClient経由でのCoder操作
- **データストレージ**: MongoDB (セッション情報のみ)

#### 2. keruta-executor (Coder Workspace Manager)
- **役割**: Coderワークスペース管理とセッション監視
- **機能**:
  - セッション状態監視 (30秒間隔)
  - ワークスペース自動作成・管理
  - Coder API直接操作
- **通信**: REST API経由でkeruta-apiと連携

#### 3. keruta-admin (Frontend UI)
- **役割**: フロントエンドUI
- **機能**: セッション・ワークスペース管理画面

### データフロー

```
セッション作成フロー:
1. Admin UI → keruta-api: セッション作成リクエスト
2. keruta-api → MongoDB: セッション情報保存 (PENDING状態)
3. keruta-executor → keruta-api: PENDING状態セッション検索 (30秒間隔)
4. keruta-executor → Coder API: ワークスペース作成
5. keruta-executor → keruta-api: セッション状態をACTIVEに更新

ワークスペース監視フロー:
1. keruta-executor → keruta-api: ACTIVE状態セッション検索 (60秒間隔)
2. keruta-executor → Coder API: ワークスペース状態確認
3. keruta-executor → Coder API: 必要に応じてワークスペース開始
```

## 実装詳細

### セッション状態管理

#### 状態遷移

```
PENDING → ACTIVE → COMPLETED/TERMINATED
   ↑         ↑           ↑
   │         │           └─ 手動終了/自動終了
   │         └─ ワークスペース作成完了時
   └─ セッション作成時の初期状態
```

#### 状態管理ルール

- **PENDING**: セッション作成直後、ワークスペース作成待ち
- **ACTIVE**: ワークスペース作成完了、利用可能状態
- **COMPLETED**: 正常終了
- **TERMINATED**: 異常終了またはキャンセル

**重要**: セッション状態の直接更新は403 Forbiddenで拒否され、システムによる自動管理のみ許可されています。

### Workspace自動作成機能

#### 作成プロセス

1. **セッション-ワークスペース関連付け**: セッションIDをワークスペース名に含める
2. **テンプレート使用ロジック**:
   - 環境変数`CODER_TEMPLATE_ID`で指定された単一テンプレートを固定使用
   - リクエストでのテンプレート指定は無視
   - すべてのワークスペースで同じテンプレートを使用
3. **ワークスペース名生成**: `{CODER_WORKSPACE_PREFIX}-{sanitizedSessionName}`

#### 実装コード例

```kotlin
private fun getFixedTemplate(): String {
    val templateId = System.getenv("CODER_TEMPLATE_ID")
        ?: throw IllegalStateException("CODER_TEMPLATE_ID environment variable is not configured")
    
    // 起動時にテンプレートの存在を確認済み
    return templateId
}

private fun generateWorkspaceName(sessionName: String): String {
    val prefix = System.getenv("CODER_WORKSPACE_PREFIX")
        ?: throw IllegalStateException("CODER_WORKSPACE_PREFIX environment variable is not configured")
    
    // セッション名をサニタイズ（英数字・ハイフンのみ許可）
    val sanitizedName = sessionName.replace(Regex("[^a-zA-Z0-9\\-]"), "-")
        .lowercase()
        .trim('-')
    
    val workspaceName = "$prefix-$sanitizedName"
    
    // 名前の長さ制限（Coderの制限に合わせて63文字以下）
    return if (workspaceName.length > 63) {
        workspaceName.substring(0, 63).trimEnd('-')
    } else {
        workspaceName
    }
}
```

### SessionMonitoringService動作詳細

#### 新規セッション監視 (`monitorNewSessions`)

- **実行間隔**: 30秒
- **対象**: PENDING状態のセッション
- **処理内容**:
  1. PENDINGセッション取得
  2. 既存ワークスペース確認
  3. ワークスペース作成 (存在しない場合)
  4. セッション状態をACTIVEに更新

#### アクティブセッション監視 (`monitorActiveSessions`)

- **実行間隔**: 60秒
- **対象**: ACTIVE状態のセッション
- **処理内容**:
  1. ACTIVEセッション取得
  2. ワークスペース状態確認
  3. 停止状態ワークスペースの自動開始

### テンプレート使用制約

#### 固定テンプレート使用

**重要**: 単一テンプレートを固定使用

1. **固定テンプレート**: `CODER_TEMPLATE_ID`で指定されたテンプレートのみ使用
2. **リクエスト無視**: APIリクエストでのテンプレート指定は無視される
3. **統一使用**: すべてのワークスペース作成で同じテンプレートを使用
4. **動的選択禁止**: セッションタグや動的選択は無効

### エラーハンドリングと回路ブレーカーパターン

#### 回路ブレーカー設定

```kotlin
private val maxFailures = 5              // 最大失敗回数
private val circuitOpenDuration = 60000L // 回路オープン時間 (1分)
private val resetTimeout = 120000L       // リセットタイムアウト (2分)
```

#### リトライ機能

- **指数バックオフ**: 1秒 → 2秒 → 4秒 → 8秒...
- **ジッター追加**: ランダム遅延でリクエスト分散
- **4xxエラー**: リトライなし (即座に失敗)
- **5xxエラー**: 最大3回リトライ

#### 実装例

```kotlin
private fun <T> executeWithRetry(operation: String, maxRetries: Int, block: () -> T): T {
    repeat(maxRetries) { attempt ->
        try {
            return block()
        } catch (e: HttpClientErrorException) {
            throw e // 4xxエラーはリトライしない
        } catch (e: Exception) {
            if (attempt == maxRetries - 1) throw e
            val delay = calculateBackoffDelay(attempt)
            Thread.sleep(delay)
        }
    }
}

private fun calculateBackoffDelay(attempt: Int): Long {
    val baseDelay = 1000L * (1L shl attempt) // 指数バックオフ
    val jitter = Random.nextLong(0, baseDelay / 2) // ジッター
    return baseDelay + jitter
}
```

## API仕様

### Session API Endpoints

#### セッション管理

| エンドポイント | メソッド | 説明 | リクエスト | レスポンス |
|---|---|---|---|---|
| `/api/v1/sessions` | GET | 全セッション取得 | - | `List<SessionResponse>` |
| `/api/v1/sessions` | POST | セッション作成 | `CreateSessionRequest` | `SessionResponse` |
| `/api/v1/sessions/{id}` | GET | セッション詳細取得 | - | `SessionResponse` |
| `/api/v1/sessions/{id}` | PUT | セッション更新 | `UpdateSessionRequest` | `SessionResponse` |
| `/api/v1/sessions/{id}` | DELETE | セッション削除 | - | `204 No Content` |

#### セッション状態・フィルタ

| エンドポイント | メソッド | 説明 | パラメータ | レスポンス |
|---|---|---|---|---|
| `/api/v1/sessions/status/{status}` | GET | 状態別取得 | status: PENDING/ACTIVE/COMPLETED | `List<SessionResponse>` |
| `/api/v1/sessions/search` | GET | 名前検索 | name: String | `List<SessionResponse>` |
| `/api/v1/sessions/tag/{tag}` | GET | タグ別取得 | tag: String | `List<SessionResponse>` |

#### セッション状態管理

| エンドポイント | メソッド | 説明 | 制限事項 |
|---|---|---|---|
| `/api/v1/sessions/{id}/status` | PUT | 状態更新 | **403 Forbidden** (システム専用) |

#### ワークスペース関連

| エンドポイント | メソッド | 説明 | レスポンス |
|---|---|---|---|
| `/api/v1/sessions/{id}/workspaces` | GET | セッションワークスペース取得 | `List<CoderWorkspaceResponse>` |
| `/api/v1/sessions/{id}/sync-status` | POST | 状態同期実行 | `Map<String, Any>` |

### Workspace API Endpoints (keruta-executor)

#### ワークスペース管理

| エンドポイント | メソッド | 説明 | パラメータ | レスポンス |
|---|---|---|---|---|
| `/api/v1/workspaces` | GET | ワークスペース一覧 | sessionId?: String | `List<CoderWorkspaceDto>` |
| `/api/v1/workspaces/{id}` | GET | ワークスペース詳細 | - | `CoderWorkspaceDto` |
| `/api/v1/workspaces` | POST | ワークスペース作成 | `CreateCoderWorkspaceRequest` | `CoderWorkspaceDto` |
| `/api/v1/workspaces/{id}/start` | POST | ワークスペース開始 | - | `CoderWorkspaceDto` |
| `/api/v1/workspaces/{id}/stop` | POST | ワークスペース停止 | - | `CoderWorkspaceDto` |
| `/api/v1/workspaces/{id}` | DELETE | ワークスペース削除 | - | `204 No Content` |
| `/api/v1/workspaces/templates` | GET | テンプレート一覧 | - | `List<CoderTemplateDto>` |

### sync-status エンドポイント詳細

同期状態確認エンドポイント `/api/v1/sessions/{id}/sync-status` のレスポンス形式:

```json
{
  "sessionId": "session-123",
  "sessionStatus": "ACTIVE",
  "workspaceCount": 1,
  "workspaces": [
    {
      "id": "workspace-456",
      "name": "dev-my-project",
      "status": "running",
      "health": "healthy"
    }
  ],
  "syncedAt": 1703097600000
}
```

### DTOクラス構造

#### SessionResponse

```kotlin
data class SessionResponse(
    val id: String,
    val name: String,
    val description: String?,
    val status: String,
    val tags: List<String>,
    val templateConfig: SessionTemplateConfigResponse?,
    val createdAt: String,
    val updatedAt: String
)
```

#### CoderWorkspaceResponse

```kotlin
data class CoderWorkspaceResponse(
    val id: String,
    val name: String,
    val ownerId: String,
    val ownerName: String,
    val templateId: String,
    val templateName: String,
    val status: String,
    val health: String,
    val accessUrl: String?,
    val autoStart: Boolean,
    val lastUsedAt: String?,
    val createdAt: String,
    val updatedAt: String
)
```

## 運用ガイド

### 監視ポイント

#### システム健全性

1. **セッション監視サービス**
   - `SessionMonitoringService`のスケジュール実行状況
   - 回路ブレーカーの状態 (オープン状態の確認)
   - リトライ機能の実行頻度

2. **API通信状況**
   - keruta-api ↔ keruta-executor間の通信状態
   - keruta-executor ↔ Coder API間の通信状態
   - Coderトークンの有効性 (24時間自動更新)

3. **ワークスペース状態**
   - 作成失敗したワークスペースの有無
   - 長時間PENDING状態のセッション
   - 停止状態で残っているワークスペース

#### パフォーマンス監視

1. **処理時間**
   - セッション作成からワークスペース作成完了までの時間
   - 監視サイクルの実行時間
   - API レスポンス時間

2. **リソース使用量**
   - セッション数とワークスペース数の比率
   - MongoDB接続プール状況
   - HTTP接続プール状況

### トラブルシューティング

#### よくある問題と対処法

1. **セッションがPENDING状態から変わらない**
   
   **症状**: セッション作成後、長時間PENDING状態が続く
   
   **確認手順**:
   ```bash
   # keruta-executor のログ確認
   docker-compose logs -f keruta-executor
   
   # 特定セッションの監視ログ確認
   grep "Processing session: sessionId=<SESSION_ID>" logs
   ```
   
   **一般的な原因と対処**:
   - Coderトークンの期限切れ → トークン再生成
   - テンプレートが見つからない → 利用可能テンプレート確認
   - 回路ブレーカーがオープン状態 → 時間経過を待つ

2. **ワークスペース作成エラー**
   
   **症状**: ワークスペース作成時にエラーが発生
   
   **確認手順**:
   ```bash
   # Coder API直接確認
   curl -H "Coder-Session-Token: <TOKEN>" \
        https://coder.example.com/api/v2/templates
   
   # keruta-executor のCoder通信ログ確認
   grep "Failed to create Coder workspace" logs
   ```
   
   **対処法**:
   - テンプレートID確認
   - Coderサーバーの状態確認
   - ネットワーク接続確認

3. **セッション状態更新の拒否**
   
   **症状**: 手動でのセッション状態更新が403エラーになる
   
   **説明**: これは正常な動作です。セッション状態は自動管理されます。
   
   **確認方法**:
   ```bash
   # 状態同期の手動実行
   curl -X POST http://localhost:8080/api/v1/sessions/<SESSION_ID>/sync-status
   ```

### ログの見方

#### keruta-executor ログレベル

```
INFO  - 正常な処理の進行状況
WARN  - 回復可能な問題 (リトライ、回路ブレーカー等)
ERROR - 処理継続可能なエラー
DEBUG - 詳細なデバッグ情報 (プロダクションでは通常無効)
```

#### 重要なログメッセージ

```
# セッション監視開始
"Monitoring new sessions for workspace verification"

# セッション処理開始
"Processing session: sessionId={} name={}"

# ワークスペース作成成功
"Successfully created Coder workspace for session: sessionId={}"

# 回路ブレーカーオープン
"Circuit breaker opened for key: {} after {} failures"

# トークン自動更新
"Scheduled token refresh completed successfully"
```

### 設定可能なパラメータ

#### keruta-executor設定

**application.properties**:
```properties
# API接続設定
keruta.executor.api-base-url=http://localhost:8080

# Coder接続設定
keruta.executor.coder.base-url=https://coder.example.com
keruta.executor.coder.token=${KERUTA_EXECUTOR_CODER_TOKEN}
keruta.executor.coder.template-id=${CODER_TEMPLATE_ID}
keruta.executor.coder.workspace-prefix=${CODER_WORKSPACE_PREFIX}

# 監視間隔調整 (application.yml)
spring:
  task:
    scheduling:
      pool:
        size: 10
```

#### 監視間隔の調整

監視間隔は `@Scheduled` アノテーションで制御されています:

```kotlin
@Scheduled(fixedDelay = 30000) // 新規セッション監視: 30秒間隔
fun monitorNewSessions()

@Scheduled(fixedDelay = 60000) // アクティブセッション監視: 60秒間隔  
fun monitorActiveSessions()
```

変更する場合は、ソースコード修正と再デプロイが必要です。

#### 回路ブレーカー設定

```kotlin
private val maxFailures = 5              // 最大失敗回数
private val circuitOpenDuration = 60000L // 回路オープン時間 (1分)
private val resetTimeout = 120000L       // リセットタイムアウト (2分)
```

## 開発者向け情報

### 新機能追加時の注意点

#### セッション関連機能の追加

1. **ドメインモデルの変更**
   - `Session.kt`への新フィールド追加
   - `SessionEntity.kt`でのマッピング追加
   - データベースマイグレーション (必要に応じて)

2. **API エンドポイントの追加**
   - `SessionController.kt`でのエンドポイント実装
   - DTOクラスの更新
   - Swagger文書の更新

3. **監視ロジックへの影響**
   - `SessionMonitoringService`での新フィールド考慮
   - 必要に応じて監視ロジックの調整

#### ワークスペース関連機能の追加

1. **Coder API との整合性**
   - Coder APIバージョンとの互換性確認
   - 新しいCoder機能の活用可能性検討

2. **エラーハンドリングの強化**
   - 新しいエラーパターンの考慮
   - リトライロジックの調整

### テスト方法

#### 単体テスト

```bash
# keruta-api テスト実行
cd keruta-api && ./gradlew test

# keruta-executor テスト実行  
cd keruta-executor && ./gradlew test
```

#### 統合テスト

```bash
# 全システム統合テスト
docker-compose up -d
# テストスクリプト実行
./test-integration.sh
```

#### セッション-ワークスペース同期テスト

1. **セッション作成テスト**
```bash
curl -X POST http://localhost:8080/api/v1/sessions \
  -H "Content-Type: application/json" \
  -d '{"name": "test-session", "tags": ["python"]}'
```

2. **状態遷移確認**
```bash
# 30秒後にPENDING → ACTIVE変化を確認
curl http://localhost:8080/api/v1/sessions/{sessionId}
```

3. **ワークスペース確認**
```bash
# 関連ワークスペース確認
curl http://localhost:8080/api/v1/sessions/{sessionId}/workspaces
```

### デバッグ情報

#### デバッグログの有効化

**application.properties**:
```properties
# keruta-executor のデバッグログ有効化
logging.level.net.kigawa.keruta.executor=DEBUG

# HTTP通信のデバッグ
logging.level.org.springframework.web.client.RestTemplate=DEBUG
```

#### 診断エンドポイント

```bash
# セッション同期状態の確認
curl -X POST http://localhost:8080/api/v1/sessions/{sessionId}/sync-status

# ワークスペース状態の直接確認
curl http://localhost:9090/api/v1/workspaces?sessionId={sessionId}
```

#### 手動でのワークスペース操作

```bash
# 手動ワークスペース作成
curl -X POST http://localhost:9090/api/v1/workspaces \
  -H "Content-Type: application/json" \
  -d '{
    "name": "manual-workspace",
    "templateId": "template-id",
    "richParameterValues": []
  }'

# 手動ワークスペース開始
curl -X POST http://localhost:9090/api/v1/workspaces/{workspaceId}/start
```

## まとめ

Session-Workspace同期APIシステムは、セッション管理とCoderワークスペース管理を自動化する強力なシステムです。主な特徴は以下の通りです:

### 主要な利点

1. **自動化**: セッション作成と同時にワークスペースが自動作成される
2. **状態同期**: セッションとワークスペースの状態が自動で同期される
3. **障害耐性**: 回路ブレーカーパターンとリトライ機能による高い可用性
4. **監視機能**: リアルタイムでの状態監視と自動復旧

### 技術的特徴

1. **API駆動**: RESTful APIによる統合管理
2. **マイクロサービス**: 各コンポーネントの独立性と拡張性
3. **イベント駆動**: スケジュールベースの非同期処理
4. **エラーハンドリング**: 包括的なエラー処理と回復メカニズム

### 今後の発展性

- **スケーラビリティ**: 水平スケーリングに対応した設計
- **拡張性**: 新しいCoder機能への対応が容易
- **監視性**: 詳細なログとメトリクスによる運用支援

このシステムにより、開発者は手動でのワークスペース管理から解放され、効率的な開発環境の利用が可能になります。