# Kubernetes Integration - タスク環境変数によるJob作成

## 概要

kerutaシステムは、最新のタスク情報を環境変数として設定したKubernetes Jobリソースを作成する機能を提供します。この機能により、タスクキューから取得したタスクの情報をKubernetes Jobに渡し、タスクに関連する処理を自動化することができます。Jobは1回限りのバッチ処理としてPodを生成し、タスクの実行を管理します。Jobの完了・失敗状態はタスクの成否判定に利用されます。

## 機能詳細

### 環境変数マッピング

タスクの各フィールドは以下のように環境変数にマッピングされます：

| タスクフィールド    | 環境変数名                   | 説明                                                    |
|-------------|-------------------------|-------------------------------------------------------|
| id          | KERUTA_TASK_ID          | タスクの一意識別子                                             |
| title       | KERUTA_TASK_TITLE       | タスクのタイトル                                              |
| description | KERUTA_TASK_DESCRIPTION | タスクの詳細説明                                              |
| priority    | KERUTA_TASK_PRIORITY    | タスクの優先度（数値）                                           |
| status      | KERUTA_TASK_STATUS      | タスクのステータス（PENDING, IN_PROGRESS, COMPLETED, CANCELLED） |
| createdAt   | KERUTA_TASK_CREATED_AT  | タスク作成日時（ISO-8601形式）                                   |
| updatedAt   | KERUTA_TASK_UPDATED_AT  | タスク最終更新日時（ISO-8601形式）                                 |

### 設定

Kubernetes統合の設定はデータベースに保存されます。初回起動時には、以下のデフォルト設定が使用されます：

1. `application.properties`ファイルのデフォルト設定：

```properties
# Kubernetes設定のデフォルト値
keruta.kubernetes.enabled=true
keruta.kubernetes.config-path=/path/to/kube/config
keruta.kubernetes.in-cluster=false
keruta.kubernetes.default-namespace=default
```

これらの設定は管理パネルの「Kubernetes設定」画面から変更できます。変更された設定はデータベースに保存され、アプリケーションの再起動後も保持されます。

| 設定項目                | 説明                                  |
|---------------------|-------------------------------------|
| 有効/無効              | Kubernetes統合機能の有効/無効（true/false）    |
| 設定ファイルパス           | kubeconfig ファイルのパス（クラスター外で実行する場合）   |
| クラスター内実行           | クラスター内で実行する場合はtrue、外部から接続する場合はfalse |
| デフォルトネームスペース       | デフォルトのネームスペース                       |
| デフォルトイメージ          | タスク実行用のデフォルトDockerイメージ               |
| プロセッサーネームスペース      | ジョブプロセッサー用のネームスペース                   |

### セキュリティ考慮事項

1. **RBAC権限**: kerutaサービスアカウントには、Job作成のための適切なRBAC権限が必要です。
2. **機密情報**: 機密情報を含むタスクを処理する場合は、Kubernetes Secretsの使用を検討してください。
3. **ネットワークポリシー**: 作成されたJobに紐づくPodに適切なネットワークポリシーを適用してください。

## 制限事項

1. 現在のバージョンでは、一度に1つのJobのみ作成できます。
2. 長いタスク説明は環境変数の制限により切り詰められる可能性があります。

## トラブルシューティング

1. **ポッド作成失敗**: Kubernetesクラスターへの接続設定と権限を確認してください。
2. **環境変数が設定されない**: タスクが正しく取得されているか確認してください。
3. **ポッドが起動しない**: イメージ名とリソース制限を確認してください。
4. **ポッドがCrashLoopBackOff状態になる**: コンテナの起動時エラーが原因です。この状態が長時間続いた場合、タスクは失敗として処理されます。

## 制限事項

1. 現在のバージョンでは、一度に1つのJobのみ作成できます。
2. 長いタスク説明は環境変数の制限により切り詰められる可能性があります。

## Job/Pod仕様

### ベースイメージ
- デフォルトでは、管理パネルまたは設定で指定されたDockerイメージを使用します。
- イメージはタスク実行に必要なアプリケーションやランタイムが含まれている必要があります。

### 起動コマンド
- タスクごとに指定されたコマンドまたはエントリポイントでPodが起動します。
- コマンドや引数は、タスク情報や設定から動的に指定可能です。

### 環境変数
- タスク情報（id, title, description, priority, status, createdAt, updatedAt）は環境変数として渡されます（詳細は前述の「環境変数マッピング」参照）。

### マウント・ボリューム
- 必要に応じて、永続ボリュームや一時ボリュームをマウントできます（例：成果物の保存や一時ファイルの利用）。

### リソース制限
- CPUやメモリのリソースリクエスト・リミットは、デフォルト値またはタスクごとに指定可能です。

### セキュリティ
- 必要に応じてServiceAccountやRBAC、Secretsを利用し、最小権限で実行されます。

### ログ
- Jobにより生成されたPodの標準出力・標準エラーはKubernetesのPodログとして取得できます。

### 終了条件
- タスクが完了するとPodは自動的に終了し、Jobも完了状態となります。
- Jobの終了コードや状態はタスクの成否判定に利用されます。
- Jobに紐づくPodが `CrashLoopBackOff` などの異常状態になり、長時間その状態が続いた場合、実行の失敗として処理されます。

## ログ収集仕様

### ログの収集方法
- kerutaシステムは、タスク実行Jobにより生成されたPodのログを自動的に収集します。
- 収集対象は標準出力（stdout）および標準エラー（stderr）です。
- ログはタスクIDと紐付けて保存されます。

### ログの構造
- タイムスタンプ
- ログレベル（INFO, WARN, ERROR等）
- タスクID
- Job名
- Pod名
- メッセージ本文

### ログの保存
- ログはkerutaのデータベースに保存されます。
- 保存期間は設定可能です（デフォルト: 30日）。
- 長期保存が必要なログは外部ストレージへエクスポート可能です。

### ログの参照
管理者はkerutaの管理パネルから、実行されたタスクのJobに紐づくPodログを直接確認できます。

1. **タスク一覧画面へアクセス**:
   - 管理パネルにログインし、ナビゲーションメニューから「タスク管理」を選択します。
2. **タスクの選択**:
   - ログを閲覧したいタスクを一覧から探し、対象タスクの「Logs」ボタンをクリックします。
3. **ログ閲覧**:
   - ログ閲覧用のモーダルウィンドウが開き、Jobに紐づくPodの標準出力および標準エラーが表示されます。
4. **ログの操作**:
   - **自動更新**: ログはリアルタイムでストリーミング表示されます。
   - **ダウンロード**: ログ全体をテキストファイルとしてダウンロードするボタンが提供されます。
   - **フィルタリング**: ログレベルやキーワードでログを絞り込む機能も利用できます。

また、APIエンドポイント `/api/tasks/{taskId}/log` を通じてもログデータを取得できます。

### ログレベル
- INFO: 通常の実行状況
- WARN: 警告（実行は継続可能）
- ERROR: エラー（実行継続不可）
- DEBUG: デバッグ情報（開発時のみ）

### ログローテーション
- ログサイズに基づくローテーション
- 時間に基づくローテーション
- 古いログの自動アーカイブ

### エラー通知
- ERRORレベルのログ発生時に管理者へ通知可能
- 通知方法は設定可能（Email, Slack等）

## init containerによるリポジトリクローン

### 概要
タスク実行前に、init containerを利用して指定リポジトリをPod内にクローンできます。これにより、タスク本体のコンテナは事前に用意されたソースコードやリソースを利用して処理を開始できます。

### 仕様
- init containerで`git clone`コマンドを実行し、リポジトリを永続ボリューム（emptyDir等）にクローンします。
- メインコンテナ（タスク実行用）は同じボリュームをマウントし、クローン済みリポジトリにアクセスできます。
- クローン先パスやリポジトリURLはタスク情報や設定から動的に指定可能です。
- 認証が必要な場合は、Kubernetes Secret等で認証情報を渡します。

### サンプル構成
```yaml
apiVersion: batch/v1
kind: Job
spec:
  template:
    spec:
      volumes:
        - name: repo-volume
          emptyDir: {}
      initContainers:
        - name: git-clone
          image: alpine/git
          command: ["git", "clone", "<REPO_URL>", "/repo"]
          volumeMounts:
            - name: repo-volume
              mountPath: /repo
      containers:
        - name: main
          image: <TASK_IMAGE>
          volumeMounts:
            - name: repo-volume
              mountPath: /repo
          # ...他の設定...
```

### 注意事項
- init containerが失敗した場合、メインコンテナは起動しません。
- プライベートリポジトリの場合、SSHキーやアクセストークンをSecretで管理してください。

## PVCによるgitリポジトリの永続化

### 概要
init containerでクローンしたgitリポジトリを、emptyDirではなくPersistentVolumeClaim（PVC）を利用して永続化することができます。これにより、Podの再起動や複数Pod間でリポジトリデータを共有したい場合に有効です。

### 仕様
- 事前にPersistentVolumeClaim（PVC）を作成しておきます。
- init containerでgit cloneを実行し、クローン先をPVCでマウントしたディレクトリに指定します。
- メインコンテナも同じPVCをマウントし、クローン済みリポジトリにアクセスします。
- PVCのストレージクラスやアクセスモード（ReadWriteOnce/ReadWriteMany）は運用要件に応じて選択してください。

### サンプル構成
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: git-repo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: batch/v1
kind: Job
spec:
  template:
    spec:
      volumes:
        - name: git-repo
          persistentVolumeClaim:
            claimName: git-repo-pvc
      initContainers:
        - name: git-clone
          image: alpine/git
          command: ["git", "clone", "<REPO_URL>", "/git-repo"]
          volumeMounts:
            - name: git-repo
              mountPath: /git-repo
      containers:
        - name: main
          image: <TASK_IMAGE>
          volumeMounts:
            - name: git-repo
              mountPath: /git-repo
          # ...他の設定...
```

### 注意事項
- PVCはPod削除後もデータが保持されます。リポジトリの更新や削除運用に注意してください。
- 複数Podから同時に書き込みが発生しないよう、アクセスモードを適切に設定してください。
- プライベートリポジトリの場合、認証情報の管理はSecret等を利用してください。
