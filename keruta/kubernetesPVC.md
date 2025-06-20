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

## リポジトリとPVCの紐付け

各リポジトリごとに専用のPVCを作成し、リポジトリ単位で永続化・管理する構成も可能です。

### PVCの命名規則
PVCの`metadata.name`にリポジトリ名や一意なIDを含めることで、どのリポジトリ用のPVCかを明確にします。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: git-repo-pvc-my-repository # "my-repository"部分をリポジトリ名などに
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### Job/PodでのPVC参照
JobやPodの`volumes`定義で、対象リポジトリ用のPVC名を指定します。

```yaml
# ... Job Spec ...
      volumes:
        - name: git-repo
          persistentVolumeClaim:
            claimName: git-repo-pvc-my-repository # 対象リポジトリ用のPVCを指定
# ...
```

## 注意事項
- PVCはPod削除後もデータが保持されます。リポジトリの更新や削除運用に注意してください。
- 複数Podから同時に書き込みが発生しないよう、アクセスモードを適切に設定してください。
- プライベートリポジトリの場合、認証情報の管理はSecret等を利用してください。
- PVC名に使用できる文字は[DNS-1123サブドメイン名](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/#dns-subdomain-names)の規約に従う必要があります。リポジトリ名に`/`などが含まれる場合は、使用可能な文字に変換してください。
- 複数リポジトリを扱う場合、リポジトリごとにPVCを動的に作成・管理する仕組み（例: Helm、Kustomize、Operatorなど）の導入を検討すると運用が容易になります。 