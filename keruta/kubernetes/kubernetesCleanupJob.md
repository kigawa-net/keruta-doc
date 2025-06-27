# Kubernetesクリーンアップジョブ仕様

> **概要**: このドキュメントは、kerutaシステムでメインのJobが完了した後に実行されるクリーンアップ用のJobに関する仕様を定義します。

## 目次
- [概要](#概要)
- [機能詳細](#機能詳細)
- [設定](#設定)
- [サンプル](#サンプル)
- [注意点](#注意点)
- [関連リンク](#関連リンク)

## 概要
クリーンアップジョブは、メインのKubernetes Jobが完了した後に実行される補助的なジョブです。主な責務は、メインジョブによって生成された成果物をkerutaシステムに登録し、関連するタスクの状態を更新することです。

## 機能詳細
クリーンアップジョブは、以下の機能を提供します。

### 責務
- メインジョブのPodから成果物（`/.keruta/doc` 内のファイル）を収集する。
- 収集した成果物を`keruta-api`経由でアップロードし、タスクに関連付ける。
- タスクのステータスを更新する。
- （オプション）メインジョブが使用したリソース（例: PVC）をクリーンアップする。

### 実行タイミング
- メインジョブが `Succeeded` 状態になった直後にトリガーされます。
- メインジョブが `Failed` 状態になった場合は実行されません。

### 利用するAPI
クリーンアップジョブは、keruta-apiの以下のエンドポイントを利用して、タスクドキュメントの更新を行います。
- `POST /api/tasks/{taskId}/documents`

## 設定
クリーンアップジョブの挙動は、kerutaシステムの管理パネルから設定可能です。特別な設定は通常不要ですが、使用するコンテナイメージやリソース制限は環境に応じて調整が必要になる場合があります。

## サンプル
以下は、クリーンアップジョブのサンプルマニフェストです。このジョブはkeruta-agentを利用して成果物をアップロードする例です。

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: cleanup-job-for-task-${KERUTA_TASK_ID}
spec:
  template:
    spec:
      containers:
        - name: cleanup-container
          image: keruta-agent:latest # keruta-agentイメージ
          command:
            - "keruta-agent"
            - "cleanup"
            - "--task-id"
            - "$(KERUTA_TASK_ID)"
            - "--api-url"
            - "$(KERUTA_API_URL)"
            - "--source-pod"
            - "$(KERUTA_SOURCE_POD)"
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
            - name: KERUTA_SOURCE_POD
              value: "keruta-job-task123-pod-xyz"
      restartPolicy: Never
```

## 注意点
- クリーンアップジョブが使用するサービスアカウントには、`keruta-api`へのアクセス権限が必要です。
- ネットワークポリシーで`keruta-api`への通信が許可されている必要があります。
- 成果物の収集ロジックは、メインジョブが成果物をどの様に出力するかに依存します。keruta-agentは自動的に成果物の収集とアップロードを行います。
- keruta-agentは、メインジョブのPodから成果物を収集し、APIサーバーに送信します。

## 関連リンク
- [kubernetesIntegration.md](./kubernetesIntegration.md)
- [kubernetesJobSpec.md](./kubernetesJobSpec.md)
- [kubernetesPVC.md](./kubernetesPVC.md)
- [keruta-agent コマンドリファレンス](../keruta-agent/commandReference.md) 