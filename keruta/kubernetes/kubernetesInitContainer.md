# Kubernetes Init Container仕様

> **概要**: kerutaシステムで利用するKubernetes Init Containerの仕様、設定、サンプルをまとめたドキュメントです。

## 目次
- [概要](#概要)
- [機能詳細](#機能詳細)
- [設定](#設定)
- [サンプル](#サンプル)
- [注意点](#注意点)
- [関連リンク](#関連リンク)

## 概要
Init Containerは、メインコンテナが起動する前に実行されるコンテナです。kerutaシステムでは、タスク実行前の準備処理（リポジトリのクローン、インストールスクリプトの実行、APIからのファイル取得など）を担当します。

## 機能詳細
Init Containerは、以下の機能を提供します。

### 責務
- **リポジトリのクローン**: 指定されたGitリポジトリをワークディレクトリにクローン
- **git-exclude設定**: `.git/info/exclude`ファイルにkeruta関連の除外設定を追加
- **インストールスクリプト実行**: APIから取得したインストールスクリプト（`/.keruta/install.sh`）の実行
- **ドキュメント取得**: APIからタスク関連のドキュメントを取得
- **ディレクトリ準備**: 必要なディレクトリ構造の作成

### 実行タイミング
- メインコンテナが起動する前に実行されます
- すべてのInit Containerが正常に完了した場合のみ、メインコンテナが起動します
- いずれかのInit Containerが失敗した場合、Pod全体が失敗扱いになります

### 利用するAPI
Init Containerは、keruta-apiの以下のエンドポイントを利用します。
- `GET /api/v1/repositories/{repositoryId}/script`: インストールスクリプトの取得
- `GET /api/v1/documents/{documentId}/content`: ドキュメントの取得

## 設定
Init Containerの挙動は、以下の環境変数で制御されます。

### 必須環境変数
- **KERUTA_REPOSITORY_ID**: タスクに紐づくリポジトリID
- **KERUTA_DOCUMENT_ID**: タスクに紐づくドキュメントID
- **KERUTA_API_ENDPOINT**: keruta-apiのエンドポイント

### 認証
- プライベートリポジトリへのアクセスや、認証が必要なAPIの利用には、Kubernetes Secretを利用して認証情報を安全にコンテナに渡します。

## サンプル
```yaml
apiVersion: batch/v1
kind: Job
spec:
  template:
    spec:
      volumes:
        - name: work-volume
          emptyDir: {}
      initContainers:
        - name: keruta-agent-init
          image: keruta-agent:latest # keruta-agentイメージ
          workingDir: /work
          env:
            # タスクに紐づくリポジトリID. ConfigMap等から動的に設定することを想定
            - name: KERUTA_REPOSITORY_ID
              valueFrom:
                configMapKeyRef:
                  name: task-metadata
                  key: repositoryId
            # タスクに紐づくドキュメントID
            - name: KERUTA_DOCUMENT_ID
              valueFrom:
                configMapKeyRef:
                  name: task-metadata
                  key: documentId
            # keruta-apiのエンドポイント
            - name: KERUTA_API_ENDPOINT
              value: "http://keruta-api.keruta.svc.cluster.local"
            # API認証トークン
            - name: KERUTA_API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: keruta-api-token
                  key: token
          command:
            - "keruta-agent"
            - "init"
            - "--repository-id"
            - "$(KERUTA_REPOSITORY_ID)"
            - "--document-id"
            - "$(KERUTA_DOCUMENT_ID)"
            - "--api-url"
            - "$(KERUTA_API_ENDPOINT)"
            - "--work-dir"
            - "/work"
          volumeMounts:
            - name: work-volume
              mountPath: /work
      containers:
        - name: main
          image: keruta-agent:latest # keruta-agentイメージ
          workingDir: /work
          command:
            - "keruta-agent"
            - "run"
            - "--task-id"
            - "$(KERUTA_TASK_ID)"
            - "--api-url"
            - "$(KERUTA_API_ENDPOINT)"
          env:
            - name: KERUTA_TASK_ID
              value: "123"
            - name: KERUTA_API_ENDPOINT
              value: "http://keruta-api.keruta.svc.cluster.local"
            - name: KERUTA_API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: keruta-api-token
                  key: token
          volumeMounts:
            - name: work-volume
              mountPath: /work
      restartPolicy: Never
```

## 注意点
- init containerが失敗した場合、メインコンテナは起動しません。
- プライベートリポジトリの場合、SSHキーやアクセストークンをSecretで管理してください。
- keruta-agentは自動的にリポジトリのクローン、インストールスクリプトの実行、ドキュメントの取得を行います。

## 関連リンク
- [ローカル環境での除外設定](../git/gitExcludeSpec.md)
- [Kubernetes Job/Pod仕様](./kubernetesJobSpec.md)
- [Kubernetesインテグレーション概要](./kubernetesIntegration.md)
- [永続ボリューム(PVC)について](./kubernetesPVC.md)
- [keruta-agent コマンドリファレンス](../keruta-agent/commandReference.md) 
