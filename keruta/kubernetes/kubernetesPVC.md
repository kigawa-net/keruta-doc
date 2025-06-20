# PVCによるgitリポジトリの永続化

> **概要**: init containerでクローンしたgitリポジトリをPVCで永続化・共有する方法や、Kotlinによるマニフェスト自動生成例をまとめたドキュメントです。

## 目次
- [概要](#概要)
- [仕様](#仕様)
- [サンプル](#サンプル)
- [Kotlinによるマニフェスト生成例](#kotlinによるマニフェスト生成例)
- [注意点](#注意点)
- [関連リンク](#関連リンク)

## 概要
init containerでクローンしたgitリポジトリを、PersistentVolumeClaim（PVC）を利用して永続化できます。Podの再起動や複数Pod間でリポジトリデータを共有したい場合に有効です。

## 仕様
- 事前にPVCを作成
- init containerでgit cloneを実行し、クローン先をPVCでマウントしたディレクトリに指定
- メインコンテナも同じPVCをマウントし、クローン済みリポジトリにアクセス
- 親タスクが存在する場合は親タスクのPVCを子タスクでも利用

## サンプル
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
            claimName: {{ 親タスクが存在する場合: 親タスクのPVC名 | 親タスクがない場合: 新規PVC名 }}
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
      restartPolicy: Never
```

## Kotlinによるマニフェスト生成例
```kotlin
// build.gradle.kts の依存追加例
// implementation("com.fasterxml.jackson.dataformat:jackson-dataformat-yaml:2.15.2")
// implementation("com.fasterxml.jackson.module:jackson-module-kotlin:2.15.2")

fun generateJobYaml(pvcName: String, isNewPvc: Boolean): String {
    // ...（省略: 既存のサンプルコードを簡潔に記載）...
}
```

## 注意点
- PVCはPod削除後もデータが保持される
- 複数Podから同時書き込みが発生しないようアクセスモードを適切に設定
- プライベートリポジトリの場合、認証情報はSecret等で管理
- PVC名はDNS-1123サブドメイン名規約に従う

## 関連リンク
- [kubernetesIntegration.md](./kubernetesIntegration.md)
- [kubernetesJobSpec.md](./kubernetesJobSpec.md)
- [kubernetesInitContainer.md](./kubernetesInitContainer.md) 