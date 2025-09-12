# Queue-Workspace同期APIシステム

## 概要

本ドキュメントでは、KerutaにおけるQueue-Workspace同期APIシステムの包括的な実装詳細を説明します。このシステムは、キュー管理とCoderワークスペース管理を自動化し、シームレスな統合を実現します。

### システムの目的

- **自動化されたワークスペース管理**: キュー作成時の自動ワークスペース作成
- **状態同期の自動化**: キュー状態とワークスペース状態の自動同期
- **リアルタイム監視**: キューとワークスペースの状態をリアルタイムで監視
- **API駆動の管理**: RESTful APIを通じた統合管理

### アーキテクチャの変更点

従来のデータベース保存方式から、Coder API直接操作方式へ移行：

**以前**: Queue → Database Workspace → Coder API
**現在**: Queue → Coder API直接操作 (keruta-coder-provider経由)

この変更により、データの一貫性向上と管理の簡素化を実現しています。

## システムアーキテクチャ

### コンポーネント構成

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│                 │    │                  │    │                 │
│  keruta-admin   │───▶│   keruta-api     │───▶│keruta-coder-provider│
│  (Frontend UI)  │    │ (Queue API)      │    │ (Coder Manager) │
│                 │    │                  │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                │                         │
                                ▼                         ▼
                       ┌──────────────┐         ┌─────────────────┐
                       │   MongoDB    │         │   Coder Server  │
                       │ (Queue DB)   │         │  (Workspaces)   │
                       └──────────────┘         └─────────────────┘
```

### 主要コンポーネント

#### 1. keruta-api (Spring Boot API Server)
- **役割**: キュー管理とWorkspace APIプロキシ
- **機能**: 
  - キューCRUD操作
  - ワークスペースAPIプロキシ
  - ExecutorClient経由でのCoder操作
- **データストレージ**: MongoDB (キュー情報のみ)

#### 2. keruta-coder-provider (Coder Workspace Manager)
- **役割**: Coderワークスペース管理とキュー監視
- **機能**:
  - キュー状態監視 (30秒間隔)
  - ワークスペース自動作成・管理
  - Coder API直接操作
- **通信**: REST API経由でkeruta-apiと連携

#### 3. keruta-admin (Frontend UI)
- **役割**: フロントエンドUI
- **機能**: キュー・ワークスペース管理画面

### データフロー

```
キュー作成フロー:
1. Admin UI → keruta-api: キュー作成リクエスト
2. keruta-api → MongoDB: キュー情報保存 (PENDING状態)
3. keruta-coder-provider → keruta-api: PENDING状態キュー検索 (30秒間隔)
4. keruta-coder-provider → Coder API: ワークスペース作成
5. keruta-coder-provider → keruta-api: 関連タスク状況に応じてキュー状態を更新
   - 未完了タスクあり: ACTIVE状態に更新
   - タスクなし/全完了: COMPLETED状態に更新

ワークスペース・タスク監視フロー:
1. keruta-coder-provider → keruta-api: ACTIVE状態キュー検索 (60秒間隔)
2. keruta-coder-provider → Coder API: ワークスペース状態確認
3. keruta-coder-provider → keruta-api: 関連タスクの完了状況確認
4. keruta-coder-provider → keruta-api: タスク完了時にキュー状態をCOMPLETEDに更新
```

## 実装詳細

### キュー状態管理

#### 状態遷移

```
PENDING → ACTIVE → COMPLETED/TERMINATED
   ↑         ↑           ↑
   │         │           └─ 手動終了/全タスク完了
   │         └─ ワークスペース作成完了かつ未完了タスク存在時
   └─ キュー作成時の初期状態
```

#### 状態管理ルール

- **PENDING**: キュー作成直後、ワークスペース作成待ち
- **ACTIVE**: ワークスペース作成完了かつ未完了タスクが存在する状態
- **COMPLETED**: 関連する全タスクが完了した状態
- **TERMINATED**: 異常終了またはキャンセル

#### タスクベースの状態制御

キュー状態は関連タスクの状況によって自動制御されます：

- キューにタスクが関連付けられている場合、未完了タスクがある間はACTIVE状態を維持
- 関連するすべてのタスクが完了すると、キュー状態は自動的にCOMPLETEDに遷移
- タスクが存在しない場合は、ワークスペース作成完了後即座にACTIVE状態になる

**重要**: キュー状態の直接更新は403 Forbiddenで拒否され、システムによる自動管理のみ許可されています。

### Workspace自動作成機能

#### 作成プロセス

1. **キュー-ワークスペース関連付け**: キューIDをワークスペース名に含める
2. **テンプレート使用ロジック**:
   - 環境変数`CODER_TEMPLATE_ID`で指定された単一テンプレートを固定使用
   - リクエストでのテンプレート指定は無視
   - すべてのワークスペースで同じテンプレートを使用
3. **ワークスペース名生成**: `{CODER_WORKSPACE_PREFIX}-{sanitizedQueueName}`

#### 実装コード例

```kotlin
private fun getFixedTemplate(): String {
    val templateId = System.getenv("CODER_TEMPLATE_ID")
        ?: throw IllegalStateException("CODER_TEMPLATE_ID environment variable is not configured")
    
    // 起動時にテンプレートの存在を確認済み
    return templateId
}

private fun generateWorkspaceName(queueName: String): String {
    val prefix = System.getenv("CODER_WORKSPACE_PREFIX")
        ?: throw IllegalStateException("CODER_WORKSPACE_PREFIX environment variable is not configured")
    
    // キュー名をサニタイズ（英数字・ハイフンのみ許可）
    val sanitizedName = queueName.replace(Regex("[^a-zA-Z0-9\\-]"), "-")
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

### QueueMonitoringService動作詳細

#### 新規キュー監視 (`monitorNewQueues`)

- **実行間隔**: 30秒
- **対象**: PENDING状態のキュー
- **処理内容**:
  1. PENDINGキュー取得
  2. 既存ワークスペース確認
  3. ワークスペース作成 (存在しない場合)
  4. キュー状態をACTIVEに更新

#### アクティブキュー監視 (`monitorActiveQueues`)

- **実行間隔**: 60秒
- **対象**: ACTIVE状態のキュー
- **処理内容**:
  1. ACTIVEキュー取得
  2. ワークスペース状態確認
  3. 停止状態ワークスペースの自動開始

### テンプレート使用制約

#### 固定テンプレート使用

**重要**: 単一テンプレートを固定使用

1. **固定テンプレート**: `CODER_TEMPLATE_ID`で指定されたテンプレートのみ使用
2. **リクエスト無視**: APIリクエストでのテンプレート指定は無視される
3. **統一使用**: すべてのワークスペース作成で同じテンプレートを使用
4. **動的選択禁止**: キュータグや動的選択は無効

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

### Queue API Endpoints

#### キュー管理

| エンドポイント | メソッド | 説明 | リクエスト | レスポンス |
|---|---|---|---|---|
| `/api/v1/queues` | GET | 全キュー取得 | - | `List<QueueResponse>` |
| `/api/v1/queues` | POST | キュー作成 | `CreateQueueRequest` | `QueueResponse` |
| `/api/v1/queues/{id}` | GET | キュー詳細取得 | - | `QueueResponse` |
| `/api/v1/queues/{id}` | PUT | キュー更新 | `UpdateQueueRequest` | `QueueResponse` |
| `/api/v1/queues/{id}` | DELETE | キュー削除 | - | `204 No Content` |

#### キュー状態・フィルタ

| エンドポイント | メソッド | 説明 | パラメータ | レスポンス |
|---|---|---|---|---|
| `/api/v1/queues/status/{status}` | GET | 状態別取得 | status: PENDING/ACTIVE/COMPLETED | `List<QueueResponse>` |
| `/api/v1/queues/search` | GET | 名前検索 | name: String | `List<QueueResponse>` |
| `/api/v1/queues/tag/{tag}` | GET | タグ別取得 | tag: String | `List<QueueResponse>` |

#### キュー状態管理

| エンドポイント | メソッド | 説明 | 制限事項 |
|---|---|---|---|
| `/api/v1/queues/{id}/status` | PUT | 状態更新 | **403 Forbidden** (システム専用) |

#### ワークスペース関連

| エンドポイント | メソッド | 説明 | レスポンス |
|---|---|---|---|
| `/api/v1/queues/{id}/workspaces` | GET | キューワークスペース取得 | `List<CoderWorkspaceResponse>` |
| `/api/v1/queues/{id}/sync-status` | POST | 状態同期実行 | `Map<String, Any>` |

### Workspace API Endpoints (keruta-coder-provider)

#### ワークスペース管理

| エンドポイント | メソッド | 説明 | パラメータ | レスポンス |
|---|---|---|---|---|
| `/api/v1/workspaces` | GET | ワークスペース一覧 | queueId?: String | `List<CoderWorkspaceDto>` |
| `/api/v1/workspaces/{id}` | GET | ワークスペース詳細 | - | `CoderWorkspaceDto` |
| `/api/v1/workspaces` | POST | ワークスペース作成 | `CreateCoderWorkspaceRequest` | `CoderWorkspaceDto` |
| `/api/v1/workspaces/{id}/start` | POST | ワークスペース開始 | - | `CoderWorkspaceDto` |
| `/api/v1/workspaces/{id}/stop` | POST | ワークスペース停止 | - | `CoderWorkspaceDto` |
| `/api/v1/workspaces/{id}` | DELETE | ワークスペース削除 | - | `204 No Content` |
| `/api/v1/workspaces/templates` | GET | テンプレート一覧 | - | `List<CoderTemplateDto>` |

### sync-status エンドポイント詳細

同期状態確認エンドポイント `/api/v1/queues/{id}/sync-status` のレスポンス形式:

```json
{
  "queueId": "queue-123",
  "queueStatus": "ACTIVE",
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

#### QueueResponse

```kotlin
data class QueueResponse(
    val id: String,
    val name: String,
    val description: String?,
    val status: String,
    val tags: List<String>,
    val templateConfig: QueueTemplateConfigResponse?,
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

1. **キュー監視サービス**
   - `QueueMonitoringService`のスケジュール実行状況
   - 回路ブレーカーの状態 (オープン状態の確認)
   - リトライ機能の実行頻度

2. **API通信状況**
   - keruta-api ↔ keruta-coder-provider間の通信状態
   - keruta-coder-provider ↔ Coder API間の通信状態
   - Coderトークンの有効性 (24時間自動更新)

3. **ワークスペース状態**
   - 作成失敗したワークスペースの有無
   - 長時間PENDING状態のキュー
   - 停止状態で残っているワークスペース

#### パフォーマンス監視

1. **処理時間**
   - キュー作成からワークスペース作成完了までの時間
   - 監視サイクルの実行時間
   - API レスポンス時間

2. **リソース使用量**
   - キュー数とワークスペース数の比率
   - MongoDB接続プール状況
   - HTTP接続プール状況

### トラブルシューティング

#### よくある問題と対処法

1. **キューがPENDING状態から変わらない**
   
   **症状**: キュー作成後、長時間PENDING状態が続く
   
   **確認手順**:
   ```bash
   # keruta-coder-provider のログ確認
   docker-compose logs -f keruta-coder-provider
   
   # 特定キューの監視ログ確認
   grep "Processing queue: queueId=<QUEUE_ID>" logs
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
   curl -H "Coder-Queue-Token: <TOKEN>" \
        https://coder.example.com/api/v2/templates
   
   # keruta-coder-provider のCoder通信ログ確認
   grep "Failed to create Coder workspace" logs
   ```
   
   **対処法**:
   - テンプレートID確認
   - Coderサーバーの状態確認
   - ネットワーク接続確認

3. **キュー状態更新の拒否**
   
   **症状**: 手動でのキュー状態更新が403エラーになる
   
   **説明**: これは正常な動作です。キュー状態は自動管理されます。
   
   **確認方法**:
   ```bash
   # 状態同期の手動実行
   curl -X POST http://localhost:8080/api/v1/queues/<QUEUE_ID>/sync-status
   ```

### ログの見方

#### keruta-coder-provider ログレベル

```
INFO  - 正常な処理の進行状況
WARN  - 回復可能な問題 (リトライ、回路ブレーカー等)
ERROR - 処理継続可能なエラー
DEBUG - 詳細なデバッグ情報 (プロダクションでは通常無効)
```

#### 重要なログメッセージ

```
# キュー監視開始
"Monitoring new queues for workspace verification"

# キュー処理開始
"Processing queue: queueId={} name={}"

# ワークスペース作成成功
"Successfully created Coder workspace for queue: queueId={}"

# 回路ブレーカーオープン
"Circuit breaker opened for key: {} after {} failures"

# トークン自動更新
"Scheduled token refresh completed successfully"
```

### 設定可能なパラメータ

#### keruta-coder-provider設定

**application.properties**:
```properties
# API接続設定
keruta.coder-provider.api-base-url=http://localhost:8080

# Coder接続設定
keruta.coder-provider.coder.base-url=https://coder.example.com
keruta.coder-provider.coder.token=${KERUTA_CODER_PROVIDER_CODER_TOKEN}
keruta.coder-provider.coder.template-id=${CODER_TEMPLATE_ID}
keruta.coder-provider.coder.workspace-prefix=${CODER_WORKSPACE_PREFIX}

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
@Scheduled(fixedDelay = 30000) // 新規キュー監視: 30秒間隔
fun monitorNewQueues()

@Scheduled(fixedDelay = 60000) // アクティブキュー監視: 60秒間隔  
fun monitorActiveQueues()
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

#### キュー関連機能の追加

1. **ドメインモデルの変更**
   - `Queue.kt`への新フィールド追加
   - `QueueEntity.kt`でのマッピング追加
   - データベースマイグレーション (必要に応じて)

2. **API エンドポイントの追加**
   - `QueueController.kt`でのエンドポイント実装
   - DTOクラスの更新
   - Swagger文書の更新

3. **監視ロジックへの影響**
   - `QueueMonitoringService`での新フィールド考慮
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

# keruta-coder-provider テスト実行  
cd keruta-coder-provider && ./gradlew test
```

#### 統合テスト

```bash
# 全システム統合テスト
docker-compose up -d
# テストスクリプト実行
./test-integration.sh
```

#### キュー-ワークスペース同期テスト

1. **キュー作成テスト**
```bash
curl -X POST http://localhost:8080/api/v1/queues \
  -H "Content-Type: application/json" \
  -d '{"name": "test-queue", "tags": ["python"]}'
```

2. **状態遷移確認**
```bash
# 30秒後にPENDING → ACTIVE変化を確認
curl http://localhost:8080/api/v1/queues/{queueId}
```

3. **ワークスペース確認**
```bash
# 関連ワークスペース確認
curl http://localhost:8080/api/v1/queues/{queueId}/workspaces
```

### デバッグ情報

#### デバッグログの有効化

**application.properties**:
```properties
# keruta-coder-provider のデバッグログ有効化
logging.level.net.kigawa.keruta.coder_provider=DEBUG

# HTTP通信のデバッグ
logging.level.org.springframework.web.client.RestTemplate=DEBUG
```

#### 診断エンドポイント

```bash
# キュー同期状態の確認
curl -X POST http://localhost:8080/api/v1/queues/{queueId}/sync-status

# ワークスペース状態の直接確認
curl http://localhost:9090/api/v1/workspaces?queueId={queueId}
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

Queue-Workspace同期APIシステムは、キュー管理とCoderワークスペース管理を自動化する強力なシステムです。主な特徴は以下の通りです:

### 主要な利点

1. **自動化**: キュー作成と同時にワークスペースが自動作成される
2. **状態同期**: キューとワークスペースの状態が自動で同期される
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