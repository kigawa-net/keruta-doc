# Init Containerによる事前準備

> **概要**: タスク実行前にinit containerを利用して、リポジトリのクローン、セットアップスクリプトの実行、必要なファイルのダウンロードなどを行い、タスク実行環境を準備する仕組みについてまとめたドキュメントです。

## 目次
- [概要](#概要)
- [仕様](#仕様)
- [サンプル](#サンプル)
- [注意点](#注意点)
- [関連リンク](#関連リンク)

## 概要
タスク実行前に、一つまたは複数のinit containerを利用して、タスク実行に必要な準備を行います。これにより、タスク本体のコンテナは、責務を事前準備処理から分離できます。

## 仕様
- **リポジトリのクローン**: `git`コマンドが利用可能なイメージを使い、指定されたリポジトリを共有ボリュームにクローンします。
- **Gitの除外設定**: クローン後に`.git/info/exclude`ファイルに`.keruta`ディレクトリを追加し、Gitの管理対象から除外します。
- **セットアップスクリプトの実行**: クローンしたリポジトリに含まれるセットアップスクリプト（例: `.keruta/install.sh`）を実行し、依存関係のインストールや環境設定を行います。
- **ファイルダウンロード**: `curl`などのツールを使い、APIエンドポイントからドキュメントや設定ファイルを取得し、共有ボリュームに配置します。
- **共有ボリューム**: `initContainers`とメインコンテナ間でファイルを共有するために、`emptyDir`などのボリュームを利用します。
- **実行順序**: `initContainers`は定義された順序で実行されます。後のコンテナは、前のコンテナが準備したファイルやディレクトリにアクセスできます。
- **認証**: プライベートリポジトリへのアクセスや、認証が必要なAPIの利用には、Kubernetes Secretを利用して認証情報を安全にコンテナに渡します。

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
        - name: git-clone
          image: alpine/git # gitコマンドが使えるイメージ
          command:
            - /bin/sh
            - -c
            - |
              set -e
              git clone --depth 1 --single-branch <REPO_URL> /work
              echo 'Setting up git exclusions'
              echo '/.keruta' >> /work/.git/info/exclude
              echo 'Git exclusions configured'
          volumeMounts:
            - name: work-volume
              mountPath: /work
        - name: setup
          image: curlimages/curl # curlやshが使えるイメージ
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
          command:
            - "sh"
            - "-c"
            - |
              set -e
              mkdir -p ./.keruta

              # APIからインストールスクリプトを取得して実行
              if [ -n "$KERUTA_REPOSITORY_ID" ]; then
                echo "Fetching install script for repository: $KERUTA_REPOSITORY_ID"
                SCRIPT_URL="${KERUTA_API_ENDPOINT}/api/v1/repositories/${KERUTA_REPOSITORY_ID}/script"

                if curl -sfL -o ./.keruta/install.sh "${SCRIPT_URL}" && [ -s ./.keruta/install.sh ]; then
                  echo "Running downloaded setup script..."
                  chmod +x ./.keruta/install.sh
                  ./.keruta/install.sh
                else
                  echo "Install script not found or is empty. Skipping execution."
                fi
              elif [ -f ./.keruta/install.sh ]; then
                echo "Running local setup script..."
                sh ./.keruta/install.sh
              fi

              # APIからドキュメントを取得
              if [ -n "$KERUTA_DOCUMENT_ID" ]; then
                echo "Fetching document: $KERUTA_DOCUMENT_ID"
                DOC_URL="${KERUTA_API_ENDPOINT}/api/v1/documents/${KERUTA_DOCUMENT_ID}/content"
                curl -sfL -o ./.keruta/README.md "$DOC_URL"
              fi
          volumeMounts:
            - name: work-volume
              mountPath: /work
      containers:
        - name: main
          image: <TASK_IMAGE>
          workingDir: /work
          volumeMounts:
            - name: work-volume
              mountPath: /work
      restartPolicy: Never
```

## 注意点
- init containerが失敗した場合、メインコンテナは起動しません。
- プライベートリポジトリの場合、SSHキーやアクセストークンをSecretで管理してください。

## 関連リンク
- [ローカル環境での除外設定](../gitExcludeSpec.md)
- [Kubernetes Job/Pod仕様](./kubernetesJobSpec.md)
- [Kubernetesインテグレーション概要](./kubernetesIntegration.md)
- [永続ボリューム(PVC)について](./kubernetesPVC.md) 
