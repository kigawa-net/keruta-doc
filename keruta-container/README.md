# keruta-container

`keruta-container`は、kerutaのタスクを実行するために必要なツールやライブラリを含むコンテナイメージです。

## 概要

このコンテナイメージは、タスクの実行環境を統一し、再現性を高めることを目的としています。

## ベースイメージ

- (例: `ubuntu:22.04`)

## 主なツールとライブラリ

- (例: `python 3.10`)
- (例: `aws-cli`)
- (例: `kubectl`)

## ビルド方法

```bash
# ビルドコマンドの例
docker build -t keruta-container .
```

## 実行方法

```bash
# 実行コマンドの例
docker run --rm -it keruta-container /bin/bash
```

## 環境変数

タスクの実行に必要な環境変数は以下の通りです。

| 環境変数名 | 説明 | 必須 | デフォルト値 |
| :--- | :--- | :--- | :--- |
| `KERUTA_API_ENDPOINT` | keruta APIサーバーのエンドポイント | はい | |
| `KERUTA_TASK_ID` | 実行するタスクのID | はい | |
| `KERUTA_ACCESS_TOKEN` | keruta APIへのアクセストークン | はい | |

---

詳細はプロジェクトのDockerfileを参照してください。 