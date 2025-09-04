# keruta-admin ドキュメント目次

本ディレクトリには、kerutaシステムの管理パネル機能に関する技術・運用ドキュメントが含まれています。

## 管理パネル関連ドキュメント
- [adminPanel.md](./adminPanel.md)  
  管理パネルの機能・画面・操作方法
- [adminPanelRemix.md](./adminPanelRemix.md)  
  Remix.jsを使った管理パネルの実装
- [adminPanelScriptGenerator.md](./adminPanelScriptGenerator.md)  
  インストールスクリプト作成機能
- [repositoryManagement.md](./repositoryManagement.md)  
  リポジトリ管理機能
- [backendConfiguration.md](./backendConfiguration.md)  
  環境変数を使用したバックエンド設定

---

各ドキュメントの詳細はリンク先をご参照ください。 

## keruta本体と同じSecretを利用したJWT署名

keruta-adminがkeruta本体APIにアクセスする際は、keruta本体と同じKubernetes Secretに格納されたsecretを利用できます。このsecretはJWT署名の検証に使用されます。

### 運用例
- Kubernetes Secret（例: `keruta-secrets`）にsecretを格納し、Podの環境変数として注入します。
- これにより、keruta本体と同じ認証情報でAPIアクセスが可能です。
- JWT署名の検証に同じsecretが使用されます。


#### 備考
- Secretの作成・管理、Podへの注入方法はKubernetesの標準的な方法に従ってください。
- より高度な認証が必要な場合はKeycloak等の外部IDプロバイダ連携も可能です。
