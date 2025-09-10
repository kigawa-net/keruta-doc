# Coderテンプレート利用制約仕様

## 概要

Kerutaシステムにおけるcoderテンプレートの利用は、環境変数で指定されたテンプレートのみに制限されます。この制約により、セキュリティの向上と運用の一貫性を確保します。

## 基本原則

### テンプレート利用制約

- **単一テンプレート使用**: Kerutaシステムで使用するcoderテンプレートは一つのみ
- **環境変数指定必須**: 使用するテンプレートは環境変数で明示的に指定
- **動的テンプレート選択禁止**: 実行時の自動テンプレート検索・選択は不可

## 環境変数仕様

### 設定項目

| 環境変数名 | 必須 | 説明 | 形式 | 例 |
|---|---|---|---|---|
| `CODER_TEMPLATE_ID` | ✓ | 使用するテンプレートID | 単一ID | `keruta-ubuntu-template` |
| `CODER_WORKSPACE_PREFIX` | ✓ | ワークスペース名のprefix | 文字列 | `dev`, `prod`, `test` |

### 設定例

```bash
# 使用するテンプレートIDの指定（必須）
export CODER_TEMPLATE_ID="keruta-ubuntu-22.04"

# ワークスペース名のprefix指定（必須）
export CODER_WORKSPACE_PREFIX="dev"
```

## 実装仕様

### テンプレート検証処理

1. **環境変数確認**
   - `CODER_TEMPLATE_ID`の存在確認
   - `CODER_WORKSPACE_PREFIX`の存在確認
   - 未設定の場合は起動時エラー

2. **テンプレート固定使用**
   - すべてのワークスペース作成で`CODER_TEMPLATE_ID`を使用
   - リクエストでのテンプレートID指定は無効化

3. **ワークスペース名生成**
   - `{CODER_WORKSPACE_PREFIX}-{sessionName}`の形式で生成
   - セッション名はサニタイズして使用

4. **テンプレート存在確認**
   - 起動時にCoderシステムでテンプレートの存在を確認
   - 存在しない場合は起動エラー

### エラーハンドリング

| エラーケース | HTTPステータス | エラーメッセージ |
|---|---|---|
| テンプレートID未設定 | 500 Internal Server Error | "CODER_TEMPLATE_ID environment variable is not configured" |
| Prefix未設定 | 500 Internal Server Error | "CODER_WORKSPACE_PREFIX environment variable is not configured" |
| テンプレート不存在 | 500 Internal Server Error | "Specified template ID '{templateId}' does not exist in Coder" |
| 不正なワークスペース名 | 400 Bad Request | "Invalid workspace name generated: '{workspaceName}'" |

## セキュリティ上の制約

### テンプレート内容制限

1. **実行可能コマンド制限**
   - システム管理コマンドの実行を制限
   - ネットワーク接続先の制限

2. **リソース制限**
   - CPU、メモリ使用量の上限設定
   - ディスク使用量の制限

3. **権限制限**
   - root権限での実行禁止
   - 特権コンテナの禁止

### 監査要件

1. **ログ記録**
   - テンプレート利用のログ記録
   - 拒否されたテンプレート利用の記録

2. **アクセス制御**
   - テンプレート作成・変更権限の制限
   - 管理者のみによるテンプレート管理

## 運用ガイド

### テンプレート追加手順

1. **テンプレート作成**
   - Coderシステムでテンプレートを作成
   - セキュリティチェックの実施

2. **環境変数更新**
   - `CODER_ALLOWED_TEMPLATES`に新しいテンプレートIDを追加
   - システムの再起動

3. **検証**
   - 新しいテンプレートでのワークスペース作成テスト
   - ログの確認

### トラブルシューティング

#### テンプレート利用エラー

**症状**: ワークスペース作成時にテンプレートエラーが発生

**確認手順**:
```bash
# 環境変数の確認
echo $CODER_TEMPLATE_ID
echo $CODER_WORKSPACE_PREFIX

# ログの確認
kubectl logs deployment/keruta-api -n keruta | grep -i template
kubectl logs deployment/keruta-api -n keruta | grep -i workspace
```

**対処法**:
1. 環境変数の設定確認
2. テンプレートIDの正確性確認
3. Coderシステムでのテンプレート存在確認

## API仕様への影響

### 関連エンドポイント

| エンドポイント | 変更内容 |
|---|---|
| `POST /api/v1/workspaces` | templateIdパラメータを無視、固定テンプレートを使用 |
| `GET /api/v1/workspaces/templates` | 単一テンプレートのみ返却 |

### リクエスト例

```json
{
  "name": "my-workspace",
  // templateIdは指定しても無視される
  "richParameterValues": []
}
```

### レスポンス例（正常時）

```json
{
  "id": "workspace-123",
  "name": "dev-my-session",  // {CODER_WORKSPACE_PREFIX}-{sessionName}の形式
  "templateId": "keruta-ubuntu-22.04",  // 環境変数で指定されたテンプレートを使用
  "status": "pending"
}
```

## 設定ファイル連携

### application.properties

```properties
# 環境変数からの読み込み
coder.template-id=${CODER_TEMPLATE_ID:}
coder.workspace-prefix=${CODER_WORKSPACE_PREFIX:}

# バリデーション設定
coder.template-validation.strict=true
coder.workspace-name-validation.strict=true
```

### Docker Compose設定例

```yaml
services:
  keruta-api:
    environment:
      - CODER_TEMPLATE_ID=keruta-ubuntu-22.04
      - CODER_WORKSPACE_PREFIX=dev
    # その他の設定...
```

## 監視・メトリクス

### ログ出力

```
[INFO] Using fixed template: templateId=keruta-ubuntu-22.04, workspacePrefix=dev
[INFO] Generated workspace name: workspaceName=dev-my-session
[ERROR] Environment variable CODER_TEMPLATE_ID is not configured
[ERROR] Environment variable CODER_WORKSPACE_PREFIX is not configured
[ERROR] Template does not exist in Coder: templateId=keruta-ubuntu-22.04
[ERROR] Invalid workspace name generated: workspaceName=dev-invalid@name
```

### メトリクス

- `coder.template.usage.count`: テンプレート利用回数
- `coder.workspace.creation.count`: ワークスペース作成回数

## 関連ドキュメント

- [Coderカスタムubuntuテンプレート設定ガイド](../coder-ubuntu-template.md)
- [Session-Workspace同期APIシステム](../../session-workspace-sync-system.md)
- [システム概要](./systemOverview.md)