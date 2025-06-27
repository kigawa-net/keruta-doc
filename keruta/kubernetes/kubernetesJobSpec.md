# Kubernetes Job/Pod仕様

> **概要**: kerutaシステムで利用するKubernetes Job/Podの基本仕様、起動コマンド、リソース制限、セキュリティ設定などのポイントをまとめたドキュメントです。

## 目次
- [概要](#概要)
- [基本仕様](#基本仕様)
- [サンプル](#サンプル)
- [注意点](#注意点)
- [関連リンク](#関連リンク)

## 概要
kerutaシステムで利用するKubernetes Job/Podの設計ポイントをまとめます。

## 基本仕様
- デフォルトDockerイメージは管理パネルまたは設定で指定
- タスクごとにコマンドやエントリポイントを指定可能
- タスク情報は環境変数でPodに渡す
- 必要に応じて永続/一時ボリュームをマウント
  - 親タスクが存在する場合、子タスクは親タスクのPVC(永続ボリューム)を共有し、同じリポジトリデータ等にアクセスできます。詳細は[永続ボリューム(PVC)について](./kubernetesPVC.md)を参照してください。
- リソースリクエスト・リミットはデフォルトまたはタスクごとに指定
- ServiceAccountやRBAC、Secretsで最小権限実行
- Podの標準出力・標準エラーはKubernetesログとして取得
- Jobの終了コードや状態でタスクの成否を判定
- **keruta-agent**: タスク実行はkeruta-agentが行い、タスクステータスの更新、ログ収集、エラーハンドリングを統合的に管理します。
- **initコンテナ**: タスク実行前の準備処理（リポジトリのクローン、ファイルのダウンロード等）は、initコンテナ内でkeruta-agentが実行します。

## サンプル
```yaml
apiVersion: batch/v1
kind: Job
spec:
  template:
    spec:
      # initContainersでタスク実行前の準備を行います。
      # リポジトリのクローンや複数ステップに渡る準備処理の詳細は、
      # [Init Containerによる事前準備](./kubernetesInitContainer.md) を参照してください。
      initContainers:
        - name: keruta-agent-init
          image: <AGENT_IMAGE> # keruta-agentイメージ
          command:
            - "keruta-agent"
            - "init"
            - "--task-id"
            - "$(KERUTA_TASK_ID)"
            - "--api-url"
            - "$(KERUTA_API_URL)"
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
          volumeMounts:
            - name: workdir
              mountPath: /work
      containers:
        - name: main
          image: <AGENT_IMAGE> # keruta-agentイメージ
          command: 
            - "keruta-agent"
            - "run"
            - "--task-id"
            - "$(KERUTA_TASK_ID)"
            - "--api-url"
            - "$(KERUTA_API_URL)"
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
            # ...他の環境変数...
          volumeMounts:
            - name: workdir
              mountPath: /work
      volumes:
        - name: workdir
          emptyDir: {}
      restartPolicy: Never
```

## 注意点
- Jobに紐づくPodが`CrashLoopBackOff`等の異常状態で長時間継続した場合は失敗扱い
- セキュリティ要件に応じてRBACやSecretsを適切に設定
- keruta-agentはタスクの実行状況を自動的にAPIサーバーに報告します

## 関連リンク
- [Kubernetesインテグレーション概要](./kubernetesIntegration.md)
- [Init Containerによる事前準備](./kubernetesInitContainer.md)
- [keruta-agent コマンドリファレンス](../keruta-agent/commandReference.md)
- [keruta-agent 実装例](../keruta-agent/implementation.md)
- [永続ボリューム(PVC)について](./kubernetesPVC.md) 