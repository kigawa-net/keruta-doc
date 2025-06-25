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
- 既存のPVCをマウントする場合は、タスクの環境変数で指定可能
  - `KERUTA_PVC_NAME`: マウントするPVCの名前
  - `KERUTA_PVC_MOUNT_PATH`: マウントするパス（デフォルト: `/pvc`）
- ストレージクラス（StorageClass）を指定可能
  - システム全体のデフォルト設定
  - リポジトリごとの設定
  - タスクごとの設定（優先度: タスク > リポジトリ > システムデフォルト）

## サンプル

### リポジトリ用PVCの例
```yaml
# 親タスクがない場合は新しいPVCを作成
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: git-repo-pvc-task123  # タスクIDに基づいたPVC名
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
      initContainers:
        - name: git-clone
          image: alpine/git
          command: ["git", "clone", "https://github.com/example/repo.git", "/git-repo"]
          volumeMounts:
            - name: git-repo
              mountPath: /git-repo
      containers:
        - name: main
          image: nginx:latest  # タスク実行用イメージ
          volumeMounts:
            - name: git-repo
              mountPath: /git-repo
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
          image: nginx:latest
          env:
            - name: KERUTA_PVC_NAME
              value: "existing-pvc-name"
            - name: KERUTA_PVC_MOUNT_PATH
              value: "/data"  # マウントパスを指定（省略時は/pvc）
          volumeMounts:
            - name: pvc-volume
              mountPath: /data
      restartPolicy: Never
```

## Kotlinによるマニフェスト生成例
```kotlin
// build.gradle.kts の依存追加例
// implementation("com.fasterxml.jackson.dataformat:jackson-dataformat-yaml:2.15.2")
// implementation("com.fasterxml.jackson.module:jackson-module-kotlin:2.15.2")
// implementation("io.fabric8:kubernetes-client:6.8.1")

// 必要なimport文は省略

/**
 * PVCを使用したKubernetesジョブのYAMLを生成する
 *
 * @param taskId タスクID
 * @param repoUrl Gitリポジトリのクローン元URL
 * @param image 実行するDockerイメージ
 * @param parentTaskId 親タスクID（存在する場合）
 * @param pvcStorageSize PVCのストレージサイズ（例: "1Gi"）
 * @param pvcAccessMode PVCのアクセスモード（例: "ReadWriteOnce"）
 * @param pvcStorageClass PVCのストレージクラス（例: "standard"）
 * @return 生成されたYAML文字列
 */
fun generateJobWithPvcYaml(
    taskId: String,
    repoUrl: String,
    image: String,
    parentTaskId: String? = null,
    pvcStorageSize: String = "1Gi",
    pvcAccessMode: String = "ReadWriteOnce",
    pvcStorageClass: String = ""
): String {
    val mapper = ObjectMapper(YAMLFactory()).registerKotlinModule()
    val result = StringBuilder()

    // PVC名の決定（親タスクがある場合は親のPVCを使用）
    val pvcName = if (parentTaskId != null) {
        "git-repo-pvc-$parentTaskId"
    } else {
        "git-repo-pvc-$taskId"
    }

    // 親タスクがない場合は新しいPVCを作成
    if (parentTaskId == null) {
        val pvc = PersistentVolumeClaimBuilder()
            .withNewMetadata()
                .withName(pvcName)
                .addToLabels("app", "keruta")
                .addToLabels("task-id", taskId)
            .endMetadata()
            .withNewSpec()
                .withAccessModes(pvcAccessMode)
                .withNewResources()
                    .addToRequests("storage", Quantity(pvcStorageSize))
                .endResources()
                .apply {
                    // ストレージクラスが指定されている場合は設定
                    if (pvcStorageClass.isNotBlank()) {
                        withStorageClassName(pvcStorageClass)
                    }
                }
            .endSpec()
            .build()

        result.append(mapper.writeValueAsString(pvc))
        result.append("\n---\n")
    }

    // Jobの作成
    val job = JobBuilder()
        .withNewMetadata()
            .withName("keruta-job-$taskId")
            .addToLabels("app", "keruta")
            .addToLabels("task-id", taskId)
        .endMetadata()
        .withNewSpec()
            .withBackoffLimit(4)
            .withNewTemplate()
                .withNewMetadata()
                    .addToLabels("app", "keruta")
                    .addToLabels("task-id", taskId)
                .endMetadata()
                .withNewSpec()
                    .addNewVolume()
                        .withName("repo-volume")
                        .withNewPersistentVolumeClaim()
                            .withClaimName(pvcName)
                        .endPersistentVolumeClaim()
                    .endVolume()
                    .addNewInitContainer()
                        .withName("git-clone")
                        .withImage("alpine/git")
                        .withCommand("git", "clone", repoUrl, "/repo")
                        .addNewVolumeMount()
                            .withName("repo-volume")
                            .withMountPath("/repo")
                        .endVolumeMount()
                    .endInitContainer()
                    .addNewContainer()
                        .withName("main")
                        .withImage(image)
                        .addNewVolumeMount()
                            .withName("repo-volume")
                            .withMountPath("/repo")
                        .endVolumeMount()
                    .endContainer()
                    .withRestartPolicy("Never")
                .endSpec()
            .endTemplate()
        .endSpec()
        .build()

    result.append(mapper.writeValueAsString(job))
    return result.toString()
}
```

## 注意点
- PVCはPod削除後もデータが保持される
- 複数Podから同時書き込みが発生しないようアクセスモードを適切に設定
- プライベートリポジトリの場合、認証情報はSecret等で管理
- PVC名はDNS-1123サブドメイン名規約に従う
- ストレージクラスはKubernetesクラスタに存在するものを指定する必要がある
  - 存在しないストレージクラスを指定するとPVC作成が失敗する
  - クラスタのデフォルトストレージクラスを使用する場合は空文字列を指定

## 関連リンク
- [kubernetesIntegration.md](./kubernetesIntegration.md)
- [kubernetesJobSpec.md](./kubernetesJobSpec.md)
- [kubernetesInitContainer.md](./kubernetesInitContainer.md) 
