# kerutaシステム全体構成図

このドキュメントは、kerutaシステム全体の構成をMermaidダイアグラムで視覚化したものです。

## システム全体アーキテクチャ

```mermaid
graph TB
    %% External Components
    subgraph "External Services"
        GH[GitHub Repository]
        KC[Keycloak Server]
    end

    %% User Interfaces
    subgraph "User Interfaces"
        WEB[keruta-admin Web UI]
        API_EXT[External API Clients]
    end

    %% Kubernetes Cluster
    subgraph K8S["Kubernetes Cluster"]
        subgraph "Ingress Layer"
            ING[Ingress Controller]
        end
        
        subgraph "Core Services"
            API[keruta-api Service<br/>Spring Boot/Kotlin]
            ADMIN[keruta-admin Service<br/>React Router SSR]
        end
        
        subgraph "Executor Layer"
            EXEC[keruta-coder-provider<br/>Worker Manager]
        end
        
        subgraph "Worker Layer"
            WP1[Worker Pod 1<br/>Coder + keruta-agent]
            WP2[Worker Pod 2<br/>Coder + keruta-agent]
            WP3[Worker Pod N...<br/>Coder + keruta-agent]
        end
        
        subgraph "Task Execution"
            JP1[Task Job Pod 1<br/>Coder Environment]
            JP2[Task Job Pod 2<br/>Coder Environment]
            JP3[Task Job Pod N...<br/>Coder Environment]
        end
        
        subgraph "Storage"
            PVC[Persistent Volume<br/>Shared Storage]
            LOGS[Log Collection]
        end
        
    end

    %% Database
    subgraph "Data Layer"
        MONGO[(MongoDB<br/>Database)]
    end

    %% Authentication & Authorization Flow
    KC -.->|OIDC Authentication| WEB
    KC -.->|JWT Validation| API
    KC -.->|Service Account| EXEC
    KC -.->|Service Account| WP1
    KC -.->|Service Account| WP2
    KC -.->|Service Account| WP3

    %% Ingress Routing
    WEB -->|HTTP/HTTPS| ING
    API_EXT -->|HTTP/HTTPS| ING
    ING -->|REST API| API
    ING --> ADMIN

    %% Service Communications
    API <-->|Task Management| MONGO
    API -->|Task Queue Management| EXEC
    EXEC -->|Manage Workers| WP1
    EXEC -->|Manage Workers| WP2
    EXEC -->|Manage Workers| WP3
    
    %% Worker Operations
    WP1 -->|Create Task Jobs| JP1
    WP2 -->|Create Task Jobs| JP2
    WP3 -->|Create Task Jobs| JP3
    
    WP1 -->|Task Status Update via REST API| ING
    WP2 -->|Task Status Update via REST API| ING
    WP3 -->|Task Status Update via REST API| ING

    %% Job Pod Operations
    JP1 -->|Clone Repository| GH
    JP2 -->|Clone Repository| GH
    JP3 -->|Clone Repository| GH
    

    %% Storage Access
    JP1 -->|Read/Write| PVC
    JP2 -->|Read/Write| PVC
    JP3 -->|Read/Write| PVC
    
    JP1 -->|Log Output| LOGS
    JP2 -->|Log Output| LOGS
    JP3 -->|Log Output| LOGS



    %% Admin Panel Data Flow
    ADMIN <-->|User Management| API
    ADMIN <-->|Task Monitoring| API
    ADMIN <-->|Repository Config| API

    %% Styling
    classDef userInterface fill:#e1f5fe
    classDef coreService fill:#f3e5f5
    classDef executor fill:#e8eaf6
    classDef worker fill:#fff3e0
    classDef storage fill:#e8f5e8
    classDef external fill:#fce4ec
    classDef auth fill:#fff8e1

    class WEB,API_EXT userInterface
    class API,ADMIN coreService
    class EXEC executor
    class WP1,WP2,WP3,JP1,JP2,JP3 worker
    class MONGO,PVC,LOGS storage
    class GH,KC external
    class KC auth
```

## コンポーネント詳細

### 外部サービス
- **GitHub Repository**: ソースコード管理、タスク実行対象
- **Keycloak Server**: 認証・認可サーバー（OIDC/OAuth2）

### ユーザーインターフェース
- **keruta-admin Web UI**: Web管理パネル（React Router SSR）
- **External API Clients**: 外部システム連携

### Kubernetesクラスター
- **Ingress Controller**: 外部トラフィック制御
- **keruta-api Service**: メインAPIサーバー（Spring Boot/Kotlin）
- **keruta-admin Service**: Web UIホスティング（React Router SSR）
- **keruta-coder-provider**: Worker Pod管理・オーケストレーション
- **Worker Pods**: Coder + keruta-agentによるタスクキュー処理
- **Task Job Pods**: Coder環境での実際のタスク実行

### データ層
- **MongoDB**: メインデータストア（タスク、ユーザー、ドキュメント）
- **Persistent Volume**: 共有ストレージ
- **Log Collection**: ログ集約システム

## データフロー

### 認証フロー
1. ユーザーがKeycloakで認証
2. JWTトークンを取得
3. APIリクエスト時にトークン検証
4. オフラインモード対応

### タスク実行フロー
1. ユーザーがWeb UIでタスク作成
2. APIがタスクをMongoDBに保存
3. keruta-coder-providerがタスクキューを監視・管理
4. keruta-coder-providerがWorker Podにタスクを割り当て
5. Worker Pod（Coder + keruta-agent）がタスク処理
6. Coder環境でJob Podを作成してタスク実行
7. keruta-agentが実行結果を収集
8. 結果をAPIサーバー経由でMongoDBに保存

### Git連携フロー
1. Job Pod作成時にGitリポジトリをclone
2. タスク実行環境セットアップ
3. 共有ストレージに結果保存
4. ログ収集・監視

## 特徴

- **マイクロサービス設計**: 各コンポーネントが独立
- **Kubernetes Native**: コンテナオーケストレーション活用
- **統一認証**: Keycloak SSO対応
- **スケーラブル**: Worker Pod水平スケーリング対応
- **Git統合**: リポジトリ直接連携
- **ログ・監視**: 運用観点の考慮