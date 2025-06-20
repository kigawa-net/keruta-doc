# Kubernetes連携仕様

> **概要**: kerutaシステムでタスク情報をKubernetes Jobに環境変数として渡し、自動化・連携を実現するための設計・設定方法をまとめたドキュメントです。

## 目次
- [概要](#概要)
- [機能詳細](#機能詳細)
- [セットアップ](#セットアップ)
- [サンプル](#サンプル)
- [注意点](#注意点)
- [FAQ・トラブルシューティング](#faqトラブルシューティング)
- [関連リンク](#関連リンク)

## 概要
kerutaシステムは、タスク情報を環境変数としてKubernetes Jobリソースに渡し、バッチ処理を自動化します。Jobの完了・失敗状態はタスクの成否判定に利用されます。

## 機能詳細
- タスク情報（id, title, description, priority, status, createdAt, updatedAt）を環境変数としてPodに渡す
- Jobは1回限りのバッチ処理としてPodを生成
- Jobの状態でタスクの成否を判定

### 環境変数マッピング
| タスクフィールド    | 環境変数名                   | 説明                |
|-------------|-------------------------|---------------------|
| id          | KERUTA_TASK_ID          | タスクの一意識別子   |
| title       | KERUTA_TASK_TITLE       | タスクのタイトル     |
| description | KERUTA_TASK_DESCRIPTION | タスクの詳細説明     |
| priority    | KERUTA_TASK_PRIORITY    | タスクの優先度      |
| status      | KERUTA_TASK_STATUS      | タスクのステータス   |
| createdAt   | KERUTA_TASK_CREATED_AT  | タスク作成日時      |
| updatedAt   | KERUTA_TASK_UPDATED_AT  | タスク最終更新日時   |

### 成果物収集機能
- Jobが正常に完了した際に、Pod内の`/.keruta/doc`ディレクトリ配下のファイルをkerutaシステムに保存します。
- Job終了後、クリーンアップ用のJobが実行され、keruta-apiを利用してドキュメントを更新します。

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
          image: <TASK_IMAGE>
          # 例: 処理を実行し、/.keruta/doc ディレクトリに成果物を作成
          command: ["/bin/sh", "-c", "mkdir -p /.keruta/doc && echo 'Job completed successfully.' > /.keruta/doc/README.md"]
          env:
            - name: KERUTA_TASK_ID
              value: "123"
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

## FAQ・トラブルシューティング
- **ポッド作成失敗**: クラスターへの接続設定と権限を確認
- **環境変数が設定されない**: タスク取得処理を確認
- **ポッドが起動しない**: イメージ名やリソース制限を確認
- **CrashLoopBackOff**: 起動時エラー。長時間続く場合はタスク失敗扱い

## 関連リンク
- [kubernetesJobSpec.md](./kubernetesJobSpec.md)
- [kubernetesInitContainer.md](./kubernetesInitContainer.md)
- [kubernetesPVC.md](./kubernetesPVC.md)
- [kubernetesLogCollection.md](./kubernetesLogCollection.md)
