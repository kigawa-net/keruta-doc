# concept

## name

keruta

## description

workerを使ったバッチ処理をするツール
すべてクラウドで動作する

* task を queue に入れて worker で処理する
* 複数の worker を経由して動作する

### task

* task ごとの関連付けがされている
* task はアクションを起こす必要がある物
* palin text

### queue

* queue
* FIFO


### worker

* task を処理する
* 1worker - 1処理
* AIのコーディング、機械的な処理、人間による承認など

## 実装

### コアサーバー

* タスクの管理をする

### プロバイダー

* worker の種類ごとに動作する
