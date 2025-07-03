# バックエンド設定

このドキュメントでは、keruta管理パネルのバックエンド設定について説明します。

## 環境変数によるバックエンド設定

keruta管理パネルは、環境変数を使用してバックエンドAPIの接続先を設定できます。これにより、開発環境、テスト環境、本番環境など、異なる環境で異なるバックエンドを使用することが可能になります。

### 利用可能な環境変数

以下の環境変数が利用可能です：

| 環境変数 | 説明 | デフォルト値 |
|----------|------|------------|
| `BACKEND_URL` | バックエンドAPIのベースURL | `http://localhost:3001/api` |
| `API_VERSION` | APIバージョン | `v1` |
| `JWT_SECRET` | JWT トークン生成用の秘密鍵 | なし |
| `AUTH_TOKEN` | 認証トークン（JWT_SECRETが設定されていない場合に使用） | なし |


### 環境変数の設定方法

環境変数は以下の方法で設定できます：

1. `.env`ファイルを使用する（開発環境向け）
   ```
   # .envファイルの例
   BACKEND_URL=https://api.example.com
   API_VERSION=v2
   JWT_SECRET=your-jwt-secret-key
   AUTH_TOKEN=your-auth-token
   ```

2. システムの環境変数として設定する（本番環境向け）
   ```bash
   # Linuxの場合
   export BACKEND_URL=https://api.example.com
   export API_VERSION=v2
   export JWT_SECRET=your-jwt-secret-key
   export AUTH_TOKEN=your-auth-token

   # Windowsの場合
   set BACKEND_URL=https://api.example.com
   set API_VERSION=v2
   set JWT_SECRET=your-jwt-secret-key
   set AUTH_TOKEN=your-auth-token
   ```

3. コンテナ環境（Docker、Kubernetes）で環境変数を設定する
   ```yaml
   # Dockerの例（docker-compose.yml）
   services:
     keruta-admin:
       image: keruta-admin
       environment:
         - BACKEND_URL=https://api.example.com
         - API_VERSION=v2
         - JWT_SECRET=your-jwt-secret-key
         - AUTH_TOKEN=your-auth-token

   # Kubernetesの例（deployment.yaml）
   spec:
     containers:
     - name: keruta-admin
       image: keruta-admin
       env:
       - name: BACKEND_URL
         value: "https://api.example.com"
       - name: API_VERSION
         value: "v2"
       - name: JWT_SECRET
         value: "your-jwt-secret-key"

   ```

## APIユーティリティの使用方法

keruta管理パネルには、バックエンドAPIとの通信を簡単に行うためのAPIユーティリティが用意されています。

### APIユーティリティのインポート

```typescript
import { apiGet, apiPost, apiPut, apiDelete } from '~/utils/api';
```

### GETリクエストの例

```typescript
// データの取得
const fetchData = async () => {
  try {
    const result = await apiGet<YourDataType[]>('endpoint');
    // resultを使用する処理
  } catch (err) {
    // エラー処理
    console.error('Error fetching data:', err);
  }
};
```

### POSTリクエストの例

```typescript
// データの作成
const createData = async (data: YourDataType) => {
  try {
    const result = await apiPost<YourResponseType>('endpoint', data);
    // resultを使用する処理
  } catch (err) {
    // エラー処理
    console.error('Error creating data:', err);
  }
};
```

### PUTリクエストの例

```typescript
// データの更新
const updateData = async (id: string, data: YourDataType) => {
  try {
    const result = await apiPut<YourResponseType>(`endpoint/${id}`, data);
    // resultを使用する処理
  } catch (err) {
    // エラー処理
    console.error('Error updating data:', err);
  }
};
```

### DELETEリクエストの例

```typescript
// データの削除
const deleteData = async (id: string) => {
  try {
    await apiDelete<void>(`endpoint/${id}`);
    // 削除成功時の処理
  } catch (err) {
    // エラー処理
    console.error('Error deleting data:', err);
  }
};
```

## エラーハンドリング

APIユーティリティは、リクエストが失敗した場合に`ApiError`をスローします。この例外には、エラーメッセージとHTTPステータスコードが含まれています。

```typescript
import { ApiError } from '~/utils/api';

try {
  const data = await apiGet<YourDataType>('endpoint');
  // 成功時の処理
} catch (err) {
  if (err instanceof ApiError) {
    console.error(`API Error: ${err.message}, Status: ${err.status}`);

    // ステータスコードに基づいた処理
    if (err.status === 401) {
      // 認証エラーの処理
    } else if (err.status === 404) {
      // リソースが見つからない場合の処理
    } else {
      // その他のエラーの処理
    }
  } else {
    // APIエラー以外のエラー処理
    console.error('Unexpected error:', err);
  }
}
```

## 実装の詳細

バックエンド設定とAPIユーティリティの実装は以下のファイルにあります：

- `app/utils/backendConfig.ts`: バックエンド設定
- `app/utils/api.ts`: APIユーティリティ

これらのファイルを参照することで、実装の詳細を確認できます。
