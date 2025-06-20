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
- リソースリクエスト・リミットはデフォルトまたはタスクごとに指定
- ServiceAccountやRBAC、Secretsで最小権限実行
- Podの標準出力・標準エラーはKubernetesログとして取得
- Jobの終了コードや状態でタスクの成否を判定

## サンプル
```yaml
apiVersion: batch/v1
kind: Job
spec:
  template:
    spec:
      initContainers:
        - name: prepare-docs
          image: curlimages/curl
          command:
            - "sh"
            - "-c"
            - |
              mkdir -p /work/.keruta
              # KERUTA_DOCUMENT_IDとKERUTA_API_ENDPOINTは環境変数から取得することを想定
              DOC_ID=${KERUTA_DOCUMENT_ID:-"default-readme"}
              API_ENDPOINT=${KERUTA_API_ENDPOINT:-"http://keruta-api.keruta.svc.cluster.local"}
              
              # APIからドキュメントを取得してファイルに保存
              # -s: サイレントモード, -f: エラー時にサイレントに失敗, -L: リダイレクトを追跡
              curl -sfL -o /work/.keruta/README.md "${API_ENDPOINT}/api/v1/documents/${DOC_ID}/content"
          volumeMounts:
            - name: workdir
              mountPath: /work
      containers:
        - name: main
          image: <TASK_IMAGE>
          command: ["run-task"]
          env:
            - name: KERUTA_TASK_ID
              value: "123"
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

## 関連リンク
- [kubernetesIntegration.md](./kubernetesIntegration.md)
- [kubernetesInitContainer.md](./kubernetesInitContainer.md)
- [kubernetesPVC.md](./kubernetesPVC.md) 