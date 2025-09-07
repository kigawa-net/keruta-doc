# keruta-agent Integration

> **概要**: kerutaシステムでのkeruta-agentの統合に関するドキュメントです。

## 目次
- [概要](#概要)
- [実装詳細](#実装詳細)
- [使用方法](#使用方法)
- [関連リンク](#関連リンク)

## 概要
keruta-agentはkerutaシステムによって実行されるKubernetes Jobで使用されるCLIツールです。このドキュメントでは、kerutaシステムがGitHubから最新のkeruta-agentリリースをダウンロードして使用する方法について説明します。

## 実装詳細

### KerutaAgentService
`KerutaAgentService`はGitHubから最新のkeruta-agentリリースのURLを取得するためのサービスです。

```kotlin
interface KerutaAgentService {
    /**
     * Gets the URL of the latest release of keruta-agent from GitHub.
     *
     * @return The URL of the latest release
     */
    fun getLatestReleaseUrl(): String
}
```

実装クラス`KerutaAgentServiceImpl`はGitHub APIを使用して最新のリリースを取得します。

### 環境変数の設定
`KubernetesEnvironmentHandler`は、Kubernetes Jobの環境変数を設定します。最新のkeruta-agentリリースのURLを`KERUTA_AGENT_LATEST_RELEASE_URL`環境変数として追加します。

```kotlin
val latestReleaseUrl = kerutaAgentService.getLatestReleaseUrl()
EnvVar("KERUTA_AGENT_LATEST_RELEASE_URL", latestReleaseUrl, null)
```

### スクリプトの実行
`KubernetesScriptHandler`は、Kubernetes Jobで実行されるスクリプトを設定します。スクリプトは`KERUTA_AGENT_LATEST_RELEASE_URL`環境変数からkeruta-agentをダウンロードして実行可能にします。

```bash
# Download and install keruta-agent from the latest release
if [ -z "$KERUTA_AGENT_LATEST_RELEASE_URL" ]; then
  echo "KERUTA_AGENT_LATEST_RELEASE_URL is not set or empty. Skipping keruta-agent download."
else
  echo "Downloading keruta-agent from: $KERUTA_AGENT_LATEST_RELEASE_URL"
  if curl -sfL -o /usr/local/bin/keruta-agent "$KERUTA_AGENT_LATEST_RELEASE_URL"; then
    chmod +x /usr/local/bin/keruta-agent
    echo "keruta-agent downloaded and installed successfully"
    # Print version information
    keruta-agent --version || echo "Failed to get keruta-agent version"
  else
    echo "Failed to download keruta-agent from $KERUTA_AGENT_LATEST_RELEASE_URL"
  fi
fi
```

## 使用方法
この機能は自動的に動作します。Kubernetes Jobが作成されると、最新のkeruta-agentリリースがダウンロードされ、Jobで使用できるようになります。

## 関連リンク
- [keruta-agent GitHub リポジトリ](https://github.com/kigawa-net/keruta-agent/)
- [Kubernetes Job仕様](./kubernetesJobSpec.md)
- [Kubernetesインテグレーション](./kubernetesIntegration.md)