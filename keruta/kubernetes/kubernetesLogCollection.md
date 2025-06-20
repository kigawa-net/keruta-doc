# Kubernetesログ収集・参照仕様

> **概要**: kerutaシステムでのPodログ自動収集・参照方法、管理パネルやAPIでの閲覧方法をまとめたドキュメントです。

## 目次
- [概要](#概要)
- [収集方法](#収集方法)
- [保存・参照](#保存参照)
- [ログレベル・ローテーション](#ログレベルローテーション)
- [エラー通知](#エラー通知)
- [関連リンク](#関連リンク)

## 概要
タスク実行Jobにより生成されたPodの標準出力・標準エラーを自動収集し、タスクIDと紐付けて保存・参照できます。

## 収集方法
- 収集対象: stdout, stderr
- ログはkerutaのDBに保存
- 保存期間は設定可能（デフォルト30日）
- 長期保存は外部ストレージへエクスポート可能

## 保存・参照
- 管理パネルからタスクごとにPodログを閲覧可能
- ログはリアルタイムでストリーミング表示・ダウンロード・フィルタリング可能
- APIエンドポイント `/api/tasks/{taskId}/log` でも取得可能

## ログレベル・ローテーション
- ログレベル: INFO, WARN, ERROR, DEBUG
- ログローテーション: サイズ・時間で自動アーカイブ

## エラー通知
- ERROR発生時に管理者へ通知（Email, Slack等）

## 関連リンク
- [kubernetesIntegration.md](./kubernetesIntegration.md)
- [kubernetesJobSpec.md](./kubernetesJobSpec.md)
- [kubernetesInitContainer.md](./kubernetesInitContainer.md)
- [kubernetesPVC.md](./kubernetesPVC.md) 