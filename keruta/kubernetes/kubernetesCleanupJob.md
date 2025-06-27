# Kubernetesクリーンアップジョブ仕様

> **概要**: このドキュメントは、kerutaシステムでの成果物収集とクリーンアップ処理について説明します。現在はkeruta-agentのexecuteコマンド内で自動的に実行されます。

## 目次
- [概要](#概要)
- [機能詳細](#機能詳細)
- [自動クリーンアップ](#自動クリーンアップ)
- [注意点](#注意点)
- [関連リンク](#関連リンク)

## 概要
kerutaシステムでは、タスク実行後の成果物収集とクリーンアップ処理は、keruta-agentの`execute`コマンド内で自動的に実行されます。これにより、別途のクリーンアップジョブは不要になりました。

## 機能詳細
クリーンアップ処理は、以下の機能を提供します。

### 責務
- メインジョブのPodから成果物（`/.keruta/doc` 内のファイル）を収集する。
- 収集した成果物を`keruta-api`経由でアップロードし、タスクに関連付ける。
- タスクのステータスを更新する。
- （オプション）メインジョブが使用したリソース（例: PVC）をクリーンアップする。

### 実行タイミング
- メインジョブの`keruta execute`コマンドが正常に完了した直後に自動実行されます。
- メインジョブが失敗した場合は実行されません。
- `--auto-cleanup=false`オプションで無効化可能です。

### 利用するAPI
クリーンアップ処理は、keruta-apiの以下のエンドポイントを利用して、タスクドキュメントの更新を行います。
- `POST /api/tasks/{taskId}/documents`

## 自動クリーンアップ

### keruta-agent executeコマンドでの自動実行
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: keruta-task-example
spec:
  template:
    spec:
      containers:
        - name: main
          image: keruta-agent:latest # keruta-agentイメージ
          command: 
            - "keruta-agent"
            - "execute"
            - "--task-id"
            - "$(KERUTA_TASK_ID)"
            - "--api-url"
            - "$(KERUTA_API_URL)"
            - "--auto-cleanup"  # 自動クリーンアップを有効化（デフォルト）
          env:
            - name: KERUTA_TASK_ID
              value: "123"
            - name: KERUTA_API_URL
              value: "http://keruta-api:8080"
            - name: KERUTA_API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: keruta-api-token
                  key: token
      restartPolicy: Never
```

### クリーンアップを無効化する場合
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: keruta-task-no-cleanup
spec:
  template:
    spec:
      containers:
        - name: main
          image: keruta-agent:latest
          command: 
            - "keruta-agent"
            - "execute"
            - "--task-id"
            - "$(KERUTA_TASK_ID)"
            - "--api-url"
            - "$(KERUTA_API_URL)"
            - "--auto-cleanup=false"  # 自動クリーンアップを無効化
          env:
            - name: KERUTA_TASK_ID
              value: "123"
            - name: KERUTA_API_URL
              value: "http://keruta-api:8080"
            - name: KERUTA_API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: keruta-api-token
                  key: token
      restartPolicy: Never
```

## 注意点
- keruta-agentが使用するサービスアカウントには、`keruta-api`へのアクセス権限が必要です。
- ネットワークポリシーで`keruta-api`への通信が許可されている必要があります。
- 成果物の収集ロジックは、メインジョブが成果物をどの様に出力するかに依存します。keruta-agentは自動的に成果物の収集とアップロードを行います。
- 自動クリーンアップが無効化されている場合、手動で成果物を収集する必要があります。

## 関連リンク
- [kubernetesIntegration.md](./kubernetesIntegration.md)
- [kubernetesJobSpec.md](./kubernetesJobSpec.md)
- [kubernetesPVC.md](./kubernetesPVC.md)
- [keruta-agent コマンドリファレンス](../keruta-agent/commandReference.md) 