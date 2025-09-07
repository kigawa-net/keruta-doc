# Kubernetes永続ボリューム(PVC)仕様

> **概要**: kerutaシステムで利用するKubernetes永続ボリューム(PVC)の仕様、設定、サンプルをまとめたドキュメントです。

## 目次
- [概要](#概要)
- [機能詳細](#機能詳細)
- [設定](#設定)
- [サンプル](#サンプル)
- [注意点](#注意点)
- [関連リンク](#関連リンク)

## 概要
kerutaシステムでは、タスク間でリポジトリデータを共有するために、Kubernetesの永続ボリューム(PVC)を利用します。これにより、親タスクと子タスク間で同じリポジトリデータにアクセスできます。

## 機能詳細
PVCは、以下の機能を提供します。

### 責務
- **データ永続化**: リポジトリのクローン結果やタスク実行結果を永続化
- **タスク間共有**: 親タスクと子タスク間でリポジトリデータを共有
- **効率的なリソース利用**: 同じリポジトリを複数回クローンする必要がない

### 命名規則
- PVC名は `git-repo-pvc-{taskId}` の形式
- 親タスクが存在する場合、子タスクは親タスクのPVCを継承

### アクセスモード
- **ReadWriteOnce**: 単一ノードでの読み書き（デフォルト）
- **ReadWriteMany**: 複数ノードでの読み書き（対応ストレージクラスが必要）

## 設定
PVCの挙動は、以下のパラメータで制御されます。

### 必須パラメータ
- **taskId**: タスクID（PVC名の生成に使用）
- **pvcStorageSize**: ストレージサイズ（デフォルト: "1Gi"）
- **pvcAccessMode**: アクセスモード（デフォルト: "ReadWriteOnce"）

### オプションパラメータ
- **parentTaskId**: 親タスクID（指定された場合、親タスクのPVCを継承）
- **pvcStorageClass**: ストレージクラス名（省略時はデフォルトクラスを使用）

## サンプル

### 新規PVC作成例
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: git-repo-pvc-task123
  labels:
    app: keruta
    task-id: task123
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard  # ストレージクラスを指定（省略可）
---
apiVersion: batch/v1
kind: Job
metadata:
  name: keruta-job-task123
spec:
  template:
    spec:
      volumes:
        - name: git-repo
          persistentVolumeClaim:
            # 親タスクがある場合は親のPVC名、ない場合は新規PVC名を指定
            claimName: git-repo-pvc-task123
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
          env:
            - name: KERUTA_TASK_ID
              value: "task123"
            - name: KERUTA_API_URL
              value: "http://keruta-api:8080"
            - name: KERUTA_API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: keruta-api-token
                  key: token
          volumeMounts:
            - name: git-repo
              mountPath: /work
      restartPolicy: Never
```

### 既存PVCをマウントする例
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: keruta-job-with-existing-pvc
spec:
  template:
    spec:
      volumes:
        - name: pvc-volume
          persistentVolumeClaim:
            claimName: existing-pvc-name  # 既存のPVC名
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
          env:
            - name: KERUTA_TASK_ID
              value: "task456"
            - name: KERUTA_API_URL
              value: "http://keruta-api:8080"
            - name: KERUTA_API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: keruta-api-token
                  key: token
          volumeMounts:
            - name: pvc-volume
              mountPath: /work
      restartPolicy: Never
```

## 注意点
- PVCの削除は手動で行う必要があります（Job削除時には自動削除されません）
- ストレージクラスによっては、アクセスモードの制限があります
- 親タスクのPVCを継承する場合、親タスクが完了していてもPVCは残ります
- keruta-agentはタスク情報をAPIから自動取得し、リポジトリのクローンと管理を自動的に行います

## 関連リンク
- [Kubernetes Job/Pod仕様](./kubernetesJobSpec.md)
- [Kubernetesインテグレーション概要](./kubernetesIntegration.md)
- [keruta-agent コマンドリファレンス](../keruta-agent/commandReference.md)
