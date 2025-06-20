# PVCによるgitリポジトリの永続化

## 概要
init containerでクローンしたgitリポジトリを、emptyDirではなくPersistentVolumeClaim（PVC）を利用して永続化することができます。これにより、Podの再起動や複数Pod間でリポジトリデータを共有したい場合に有効です。

## 仕様
- 事前にPersistentVolumeClaim（PVC）を作成しておきます。
- init containerでgit cloneを実行し、クローン先をPVCでマウントしたディレクトリに指定します。
- メインコンテナも同じPVCをマウントし、クローン済みリポジトリにアクセスします。
- PVCのストレージクラスやアクセスモード（ReadWriteOnce/ReadWriteMany）は運用要件に応じて選択してください。

## サンプル構成
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

## 注意事項
- PVCはPod削除後もデータが保持されます。リポジトリの更新や削除運用に注意してください。
- 複数Podから同時に書き込みが発生しないよう、アクセスモードを適切に設定してください。
- プライベートリポジトリの場合、認証情報の管理はSecret等を利用してください。 