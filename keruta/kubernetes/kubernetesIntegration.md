# Kubernetesインテグレーション概要

> **概要**: kerutaシステムとKubernetesの統合について、設計思想、設定方法、サンプルをまとめたドキュメントです。

## 目次
- [概要](#概要)
- [設計思想](#設計思想)
- [セットアップ](#セットアップ)
- [サンプル](#サンプル)
- [注意点](#注意点)
- [FAQ・トラブルシューティング](#faq・トラブルシューティング)
- [関連リンク](#関連リンク)

## 概要
kerutaシステムは、Kubernetesを利用してタスクを実行する機能を提供します。これにより、スケーラブルで堅牢なタスク実行環境を実現できます。

## 設計思想
- **コンテナ化**: 各タスクは独立したコンテナとして実行
- **リソース管理**: Kubernetesのリソース制限機能を活用
- **スケーラビリティ**: 必要に応じてPodを水平スケール
- **堅牢性**: Podの再起動やフェイルオーバーを自動処理
- **統合管理**: keruta-agentによるタスク実行の統合管理

## セットアップ
Kubernetes統合の設定はデータベースに保存され、初回起動時は`application.properties`のデフォルト値が使用されます。

```properties
# 例: application.properties
keruta.kubernetes.enabled=true
keruta.kubernetes.config-path=/path/to/kube/config
keruta.kubernetes.in-cluster=false
keruta.kubernetes.default-namespace=default
```

これらの設定は管理パネルの「Kubernetes設定」画面から変更可能です。

## サンプル
```yaml
apiVersion: batch/v1
kind: Job
spec:
  template:
    spec:
      containers:
        - name: main
          image: keruta-agent:latest # keruta-agentイメージ
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
      restartPolicy: Never
```

## 注意点
- kerutaサービスアカウントにはJob作成のためのRBAC権限が必要
- 機密情報はKubernetes Secretsで管理
- 適切なネットワークポリシーを適用
- 長い説明は環境変数の制限で切り詰められる場合あり
- 成果物収集機能を利用する場合、kerutaサービスアカウントにはPodに対する`exec`権限が必要です。
- Podが正常終了後すぐに削除される設定（例：`ttlSecondsAfterFinished`が短い）の場合、ファイル収集に失敗することがあります。
- 収集対象のファイルサイズには上限が設定されている場合があります。
- keruta-agentはタスクの実行状況を自動的にAPIサーバーに報告します。

## FAQ・トラブルシューティング
- **ポッド作成失敗**: クラスターへの接続設定と権限を確認
- **環境変数が設定されない**: タスク取得処理を確認
- **ポッドが起動しない**: イメージ名やリソース制限を確認

## 関連リンク
- [Kubernetes Job/Pod仕様](./kubernetesJobSpec.md)
- [Init Containerによる事前準備](./kubernetesInitContainer.md)
- [永続ボリューム(PVC)について](./kubernetesPVC.md)
- [keruta-agent コマンドリファレンス](../keruta-agent/commandReference.md)
