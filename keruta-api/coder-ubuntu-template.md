# Coderカスタムubuntuテンプレート設定ガイド

このガイドでは、Kerutaで使用するカスタムUbuntuテンプレートをCoderで作成し、設定する方法を説明します。

## 概要

Kerutaは自動的に最適なCoderテンプレートを選択しますが、特定のカスタムUbuntuテンプレートを優先的に使用することができます。

## テンプレート作成手順

### 1. Coderダッシュボードにアクセス

```bash
# Coderにログイン
open https://coder.kigawa.net
```

### 2. 新しいテンプレートを作成

1. **Templates** タブに移動
2. **Create Template** をクリック
3. **Starter Template** から **Ubuntu** または **Docker** を選択

### 3. テンプレート設定

#### 基本情報
- **Name**: `keruta-ubuntu` (推奨)
- **Display Name**: `Keruta Ubuntu Environment`
- **Description**: `Custom Ubuntu environment optimized for Keruta development tasks`
- **Icon**: Ubuntu または カスタムアイコンのURL

#### Terraform設定例

```hcl
terraform {
  required_providers {
    coder = {
      source  = "coder/coder"
      version = "~> 0.21.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.30"
    }
  }
}

locals {
  username = data.coder_workspace.me.owner
}

data "coder_provisioner" "me" {
}

provider "kubernetes" {
  # クラスター内での実行を想定
}

data "coder_workspace" "me" {
}

resource "coder_agent" "main" {
  arch                   = data.coder_provisioner.me.arch
  os                     = "linux"
  startup_script_timeout = 180
  startup_script = <<-EOT
    set -e
    # 基本ツールのインストール
    sudo apt-get update
    sudo apt-get install -y curl git vim build-essential
    
    # ワークスペースディレクトリの作成
    mkdir -p /home/coder/workspace
    chown coder:coder /home/coder/workspace
  EOT

  # メタデータ
  metadata {
    display_name = "CPU使用率"
    key          = "0_cpu_usage"
    script       = "coder stat cpu"
    interval     = 10
    timeout      = 1
  }

  metadata {
    display_name = "RAM使用率"
    key          = "1_ram_usage"
    script       = "coder stat mem"
    interval     = 10
    timeout      = 1
  }

  metadata {
    display_name = "ディスク使用量"
    key          = "3_disk"
    script       = "coder stat disk --path /"
    interval     = 60
    timeout      = 1
  }
}

resource "kubernetes_persistent_volume_claim" "home" {
  metadata {
    name      = "coder-\${data.coder_workspace.me.id}-home"
    namespace = var.workspaces_namespace
    labels = {
      "app.kubernetes.io/name"     = "coder-pvc"
      "app.kubernetes.io/instance" = "coder-pvc-\${data.coder_workspace.me.id}"
      "app.kubernetes.io/part-of"  = "coder"
      "coder.workspace.id"         = data.coder_workspace.me.id
      "coder.workspace.owner"      = data.coder_workspace.me.owner
    }
  }
  wait_until_bound = false
  spec {
    access_modes = ["ReadWriteOnce"]
    resources {
      requests = {
        storage = "\${var.disk_size}Gi"
      }
    }
  }
}

resource "kubernetes_deployment" "main" {
  count = data.coder_workspace.me.start_count
  metadata {
    name      = "coder-\${data.coder_workspace.me.owner}-\${data.coder_workspace.me.name}"
    namespace = var.workspaces_namespace
    labels = {
      "app.kubernetes.io/name"     = "coder-workspace"
      "app.kubernetes.io/instance" = "coder-workspace-\${data.coder_workspace.me.owner}-\${data.coder_workspace.me.name}"
      "app.kubernetes.io/part-of"  = "coder"
      "coder.workspace.id"         = data.coder_workspace.me.id
      "coder.workspace.owner"      = data.coder_workspace.me.owner
    }
  }

  spec {
    replicas = 1
    selector {
      match_labels = {
        "app.kubernetes.io/name" = "coder-workspace"
      }
    }
    template {
      metadata {
        labels = {
          "app.kubernetes.io/name" = "coder-workspace"
        }
      }
      spec {
        security_context {
          run_as_user    = 1000
          run_as_group   = 1000
          fs_group       = 1000
          run_as_non_root = true
        }
        container {
          name  = "main"
          image = "ubuntu:22.04"
          command = ["sh", "-c", coder_agent.main.init_script]
          security_context {
            run_as_user                = 1000
            run_as_non_root            = true
            allow_privilege_escalation = false
            read_only_root_filesystem  = false
          }
          env {
            name  = "CODER_AGENT_TOKEN"
            value = coder_agent.main.token
          }
          resources {
            requests = {
              "cpu"    = "250m"
              "memory" = "512Mi"
            }
            limits = {
              "cpu"    = "\${var.cpu}"
              "memory" = "\${var.memory}Gi"
            }
          }
          volume_mount {
            mount_path = "/home/coder"
            name       = "home"
            read_only  = false
          }
        }

        volume {
          name = "home"
          persistent_volume_claim {
            claim_name = kubernetes_persistent_volume_claim.home.metadata.0.name
            read_only  = false
          }
        }

        affinity {
          pod_anti_affinity {
            preferred_during_scheduling_ignored_during_execution {
              weight = 1
              pod_affinity_term {
                topology_key = "kubernetes.io/hostname"
                label_selector {
                  match_expressions {
                    key      = "app.kubernetes.io/name"
                    operator = "In"
                    values   = ["coder-workspace"]
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}

variable "cpu" {
  description = "CPU (cores)"
  default     = 2
  validation {
    condition = contains([
      "1",
      "2",
      "4",
      "6",
      "8"
    ], var.cpu)
    error_message = "Invalid cpu type!"
  }
}

variable "memory" {
  description = "Memory (GB)"
  default     = 4
  validation {
    condition = contains([
      "1",
      "2",
      "4",
      "8"
    ], var.memory)
    error_message = "Invalid memory type!"
  }
}

variable "disk_size" {
  description = "Disk size (GB)"
  default     = 20
}

variable "workspaces_namespace" {
  description = "The namespace to create workspaces in (must exist prior to creating workspaces)"
  type        = string
  default     = "coder-workspaces"
}
```

### 4. テンプレートの発行

1. **Publish** をクリックしてテンプレートを保存
2. テンプレートが正常に作成されたことを確認

## Keruta側の設定

### 環境変数設定

```bash
# 使用するテンプレートID（必須）
export CODER_TEMPLATE_ID="keruta-ubuntu-22.04"

# ワークスペース名のprefix（必須）
export CODER_WORKSPACE_PREFIX="dev"
```

### application.propertiesの設定

```properties
# 使用するテンプレートID
coder.template-id=${CODER_TEMPLATE_ID:}

# ワークスペース名prefix
coder.workspace-prefix=${CODER_WORKSPACE_PREFIX:}

# テンプレート検証設定
coder.template-validation.strict=true
```

## テンプレート利用制約

**重要**: Kerutaは単一のテンプレートのみ使用します：

1. **単一テンプレート**: `CODER_TEMPLATE_ID`で指定されたテンプレートのみ使用
2. **固定使用**: すべてのワークスペース作成で同じテンプレートを使用
3. **リクエスト無視**: APIリクエストでテンプレートIDを指定しても無視される

### テンプレート使用ルール

1. **固定テンプレート**: 環境変数`CODER_TEMPLATE_ID`で指定されたテンプレートを常に使用
2. **設定必須**: 環境変数が未設定の場合はシステム起動エラー
3. **存在確認**: 起動時にCoderシステムでテンプレートの存在を確認

## テンプレートの確認

### ログでの確認

```bash
# Kerutaのログを確認
kubectl logs -f deployment/keruta-api -n keruta

# 以下のようなログが出力されます
INFO [keruta] [CoderService:238] - Using preferred template: abc123-def456 (keruta-ubuntu)
```

### APIでの確認

```bash
# 利用可能なテンプレート一覧を取得
curl -X GET "http://localhost:8080/api/v1/workspaces/templates" \\
  -H "Accept: application/json"
```

## トラブルシューティング

### テンプレートが見つからない場合

1. **Coderでテンプレートが正しく発行されているか確認**
2. **テンプレート名にタイポがないか確認**
3. **Coder APIの認証情報が正しいか確認**
4. **ログを確認して詳細なエラー情報を取得**

### パフォーマンスの最適化

```hcl
# リソース制限の調整
resources {
  requests = {
    "cpu"    = "500m"    # 最小CPU
    "memory" = "1Gi"     # 最小メモリ
  }
  limits = {
    "cpu"    = "2"       # 最大CPU
    "memory" = "4Gi"     # 最大メモリ
  }
}
```

## 関連ドキュメント

- [Coderテンプレート利用制約仕様](./system/coderTemplateSpec.md)
- [Coder Template Documentation](https://coder.com/docs/coder-oss/templates)
- [Kubernetes Provider for Terraform](https://registry.terraform.io/providers/hashicorp/kubernetes/latest/docs)
- [Session-Workspace同期APIシステム](../../session-workspace-sync-system.md)