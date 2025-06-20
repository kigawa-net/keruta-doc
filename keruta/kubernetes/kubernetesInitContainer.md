# Init Containerによるリポジトリクローン

> **概要**: タスク実行前にinit containerでリポジトリをクローンし、Pod内で利用する仕組みとそのサンプル構成をまとめたドキュメントです。

## 目次
- [概要](#概要)
- [仕様](#仕様)
- [サンプル](#サンプル)
- [注意点](#注意点)
- [関連リンク](#関連リンク)

## 概要
タスク実行前に、init containerを利用して指定リポジトリをPod内にクローンできます。これにより、タスク本体のコンテナは事前に用意されたソースコードやリソースを利用して処理を開始できます。

## 仕様
- init containerで`git clone`コマンドを実行し、リポジトリを永続ボリューム（emptyDir等）にクローンします。
- メインコンテナ（タスク実行用）は同じボリュームをマウントし、クローン済みリポジトリにアクセスできます。
- クローン先パスやリポジトリURLはタスク情報や設定から動的に指定可能です。
- 認証が必要な場合は、Kubernetes Secret等で認証情報を渡します。

## サンプル
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
      restartPolicy: Never
```

## 注意点
- init containerが失敗した場合、メインコンテナは起動しません。
- プライベートリポジトリの場合、SSHキーやアクセストークンをSecretで管理してください。

## 関連リンク
- [kubernetesIntegration.md](./kubernetesIntegration.md)
- [kubernetesJobSpec.md](./kubernetesJobSpec.md)
- [kubernetesPVC.md](./kubernetesPVC.md) 