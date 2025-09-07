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

## 認証機能仕様

keruta-adminは、React Router v7で構成されるフルスタックWebアプリケーションとして、kerutaシステムの管理パネル機能を提供します。React Router v7のSSR（Server-Side Rendering）機能を活用し、Keycloak認証によりサーバーサイドとクライアントサイドの両方でエンタープライズグレードの認証・認可システムを実現します。

## アプリケーション構成

### 技術スタック
- **フロントエンド**: React + TypeScript
- **ルーティング**: React Router v7（SSR対応）
- **レンダリング**: サーバーサイドレンダリング + クライアントハイドレーション
- **認証**: Keycloak（サーバーサイド・クライアントサイド両対応）
- **APIクライアント**: openapi-typescript + Fetch API（型安全なAPIクライアント）
- **型生成**: OpenAPI仕様からのTypeScript型自動生成
- **状態管理**: React Context / Redux（必要に応じて）
- **バンドラー**: Vite（React Router v7のデフォルト）

### Keycloak認証システム

#### 概要
React Router v7のSSR機能とKeycloakを連携し、以下を提供します：
- **サーバーサイド認証**: 初回ページ読み込み時のサーバーでの認証状態確認
- **クライアントサイド認証**: ハイドレーション後のブラウザでの認証フロー
- **ハイブリッド認証**: サーバーとクライアント間の認証状態同期
- **トークンベースAPI認証**: JWTトークンを使用したkerutaバックエンドAPIへのアクセス  
- **SSO（Single Sign-On）**: kerutaエコシステム全体での統一認証体験
- **SEO最適化**: 認証済みコンテンツのサーバーサイドレンダリング

#### 主要設定項目

##### サーバー・クライアント共通環境変数
```bash
# Keycloak設定（SSR対応）
export VITE_KEYCLOAK_URL=https://keycloak.example.com
export VITE_KEYCLOAK_REALM=keruta-production
export VITE_KEYCLOAK_CLIENT_ID=keruta-admin-ssr
export VITE_BACKEND_API_URL=https://api.keruta.example.com

# OpenAPI設定
export VITE_OPENAPI_SPEC_URL=https://api.keruta.example.com/openapi.json

# サーバーサイド専用設定
export KEYCLOAK_CLIENT_SECRET=your-ssr-client-secret
export SESSION_SECRET=your-session-secret-key
```

##### OpenAPI TypeScript生成設定（package.json）
```json
{
  "scripts": {
    "generate:api": "openapi-typescript ${VITE_OPENAPI_SPEC_URL} -o ./app/types/api.ts",
    "generate:api-local": "openapi-typescript ./openapi.json -o ./app/types/api.ts",
    "dev": "npm run generate:api && react-router dev",
    "build": "npm run generate:api && react-router build"
  },
  "devDependencies": {
    "openapi-typescript": "^7.0.0"
  }
}
```

##### SSR用Keycloak設定ファイル（app/keycloak.server.ts）
```typescript
// SSR用のKeycloak設定
export const keycloakConfig = {
  realm: process.env.VITE_KEYCLOAK_REALM!,
  'auth-server-url': process.env.VITE_KEYCLOAK_URL!,
  'ssl-required': 'external',
  resource: process.env.VITE_KEYCLOAK_CLIENT_ID!,
  'confidential-port': 0,
  'public-client': false, // SSRでは confidential client
  credentials: {
    secret: process.env.KEYCLOAK_CLIENT_SECRET!
  }
};
```

##### クライアント用設定ファイル（public/keycloak.json）
```json
{
  "realm": "keruta-production",
  "auth-server-url": "https://keycloak.example.com",
  "ssl-required": "external",
  "resource": "keruta-admin-ssr",
  "public-client": true,
  "confidential-port": 0
}
```

#### React/TypeScript実装例

##### SSR対応認証システム実装

###### サーバーサイド認証（app/auth.server.ts）
```typescript
// React Router v7 SSR対応認証
import { redirect, type LoaderFunctionArgs } from "react-router";
import { getSession, commitSession } from "./sessions.server";
import jwt from "jsonwebtoken";

export async function requireAuth(request: Request) {
  const session = await getSession(request.headers.get("Cookie"));
  const token = session.get("access_token");

  if (!token) {
    // Keycloak認証へリダイレクト
    const keycloakUrl = `${process.env.VITE_KEYCLOAK_URL}/realms/${process.env.VITE_KEYCLOAK_REALM}/protocol/openid_connect/auth`;
    const params = new URLSearchParams({
      client_id: process.env.VITE_KEYCLOAK_CLIENT_ID!,
      redirect_uri: `${new URL(request.url).origin}/auth/callback`,
      response_type: "code",
      scope: "openid profile email",
      state: new URL(request.url).pathname,
    });

    throw redirect(`${keycloakUrl}?${params}`);
  }

  try {
    // JWT検証（簡略化）
    const decoded = jwt.decode(token);
    return { token, user: decoded };
  } catch (error) {
    // トークンが無効な場合は再認証
    throw redirect("/auth/login");
  }
}

// OAuth2コールバックハンドラー
export async function handleAuthCallback(request: Request) {
  const url = new URL(request.url);
  const code = url.searchParams.get("code");
  const state = url.searchParams.get("state");

  if (!code) {
    throw new Error("認証コードが取得できませんでした");
  }

  // Keycloakでトークン交換
  const tokenResponse = await fetch(
    `${process.env.VITE_KEYCLOAK_URL}/realms/${process.env.VITE_KEYCLOAK_REALM}/protocol/openid_connect/token`,
    {
      method: "POST",
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({
        grant_type: "authorization_code",
        client_id: process.env.VITE_KEYCLOAK_CLIENT_ID!,
        client_secret: process.env.KEYCLOAK_CLIENT_SECRET!,
        code,
        redirect_uri: `${url.origin}/auth/callback`,
      }),
    }
  );

  const tokens = await tokenResponse.json();
  
  // セッションに保存
  const session = await getSession(request.headers.get("Cookie"));
  session.set("access_token", tokens.access_token);
  session.set("refresh_token", tokens.refresh_token);

  return redirect(state || "/", {
    headers: { "Set-Cookie": await commitSession(session) },
  });
}
```

###### クライアントサイド認証（app/auth.client.ts）
```typescript
// ブラウザでのKeycloak初期化
import Keycloak from 'keycloak-js';

let keycloak: Keycloak | null = null;

export const initKeycloakClient = async (): Promise<Keycloak> => {
  if (typeof window === 'undefined') {
    throw new Error('クライアントサイドでのみ使用可能');
  }

  if (keycloak) return keycloak;

  keycloak = new Keycloak('/keycloak.json');

  try {
    const authenticated = await keycloak.init({
      onLoad: 'check-sso',
      silentCheckSsoRedirectUri: window.location.origin + '/silent-check-sso.html',
      pkceMethod: 'S256',
      checkLoginIframe: false,
    });

    if (authenticated) {
      // サーバーセッションと同期
      await syncWithServerSession();
    }

    return keycloak;
  } catch (error) {
    console.error('Keycloak初期化エラー:', error);
    throw error;
  }
};

async function syncWithServerSession() {
  // サーバーサイドセッションとクライアントサイドトークンを同期
  if (keycloak?.token) {
    await fetch('/auth/sync', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ token: keycloak.token }),
    });
  }
}

export { keycloak };
```

##### SSR対応認証コンテキスト
```typescript
// app/contexts/AuthContext.tsx
import React, { createContext, useContext } from 'react';
import { useLoaderData } from 'react-router';

interface AuthContextType {
  isAuthenticated: boolean;
  user: any;
  token: string | undefined;
  hasRole: (role: string) => boolean;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  // サーバーサイドから渡されるユーザー情報を取得
  const { user, token } = useLoaderData() as { user?: any; token?: string };

  const hasRole = (role: string): boolean => {
    if (!user || !user.realm_access?.roles) return false;
    return user.realm_access.roles.includes(role);
  };

  return (
    <AuthContext.Provider value={{
      isAuthenticated: !!user,
      user,
      token,
      hasRole
    }}>
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuthはAuthProvider内で使用してください');
  }
  return context;
};
```

##### React Router v7ローダー実装
```typescript
// app/routes/_index.tsx
import type { LoaderFunction } from "react-router";
import { json } from "react-router";
import { requireAuth } from "~/auth.server";

export const loader: LoaderFunction = async ({ request }) => {
  // サーバーサイドで認証状態を確認
  const auth = await requireAuth(request);
  
  return json({
    user: auth.user,
    token: auth.token,
    timestamp: new Date().toISOString(),
  });
};

export default function Dashboard() {
  const { user, timestamp } = useLoaderData<typeof loader>();

  return (
    <div>
      <h1>ダッシュボード</h1>
      <p>ようこそ、{user.preferred_username}さん</p>
      <p>最終更新: {timestamp} (SSR)</p>
    </div>
  );
}
```

#### React Router構成

##### アプリケーションルーティング設定
```typescript
// src/App.tsx
import React from 'react';
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { AuthProvider, useAuth } from './contexts/AuthContext';
import PrivateRoute from './components/PrivateRoute';

// ページコンポーネント
import Dashboard from './pages/Dashboard';
import TaskManagement from './pages/TaskManagement';
import RepositoryManagement from './pages/RepositoryManagement';
import UserManagement from './pages/UserManagement';
import Settings from './pages/Settings';
import Layout from './components/Layout';

const AppRoutes: React.FC = () => {
  return (
    <Routes>
      <Route path="/" element={<PrivateRoute><Layout /></PrivateRoute>}>
        <Route index element={<Navigate to="/dashboard" replace />} />
        <Route path="dashboard" element={<Dashboard />} />
        <Route path="tasks" element={<TaskManagement />} />
        <Route path="repositories" element={<RepositoryManagement />} />
        <Route path="users" element={<PrivateRoute requiredRole="keruta-admin"><UserManagement /></PrivateRoute>} />
        <Route path="settings" element={<Settings />} />
      </Route>
      <Route path="*" element={<Navigate to="/" replace />} />
    </Routes>
  );
};

const App: React.FC = () => {
  return (
    <AuthProvider>
      <BrowserRouter>
        <AppRoutes />
      </BrowserRouter>
    </AuthProvider>
  );
};

export default App;
```

##### 認証保護されたルート
```typescript
// src/components/PrivateRoute.tsx
import React from 'react';
import { useAuth } from '../contexts/AuthContext';

interface PrivateRouteProps {
  children: React.ReactNode;
  requiredRole?: string;
}

const PrivateRoute: React.FC<PrivateRouteProps> = ({ children, requiredRole }) => {
  const { isAuthenticated, hasRole } = useAuth();

  if (!isAuthenticated) {
    return <div>認証が必要です</div>;
  }

  if (requiredRole && !hasRole(requiredRole)) {
    return <div>アクセス権限がありません</div>;
  }

  return <>{children}</>;
};

export default PrivateRoute;
```

#### サポート機能

##### SPA認証機能
- **Keycloak JavaScript Adapter**: ブラウザベースのOpenID Connect認証フロー
- **PKCE対応**: セキュリティ強化されたSPA向け認証フロー
- **自動トークン更新**: リフレッシュトークンによるシームレスなセッション延長
- **ルートベース認証**: React Routerと連携した認証保護されたページアクセス

##### クライアントサイドAPI認証
- **JWT Bearer Token**: フロントエンドからのAPIアクセス認証
- **Axiosインターセプター**: 自動的なトークン添付とエラーハンドリング
- **トークン有効性チェック**: API呼び出し前の自動トークン検証
- **ロールベース表示制御**: ユーザーロールに基づくUI要素の表示・非表示

##### 共通機能
- **ロールベースアクセス制御（RBAC）**: フロントエンドとAPIの両方での権限管理
- **マルチファクター認証（MFA）**: Keycloakによる二要素認証サポート
- **レスポンシブデザイン**: React Routerによるモバイル対応のSPA構成

#### ロール・権限設定

Keycloakレルム内で以下のロールとグループを設定することを推奨します：

**管理者ロール（keruta-admin）**
- 全システム機能へのアクセス権限
- ユーザー管理、システム設定変更権限
- 監査ログ閲覧権限

**運用者ロール（keruta-operator）**
- タスク管理、ジョブ実行権限
- ログ閲覧、モニタリング権限
- 設定変更権限なし

**閲覧者ロール（keruta-viewer）**
- 読み取り専用アクセス権限
- ダッシュボード、レポート閲覧権限
- 操作権限なし

#### セキュリティ要件
- **HTTPS通信**: 本番環境では必ずHTTPS利用（TLS 1.2以上推奨）
- **クライアントシークレット管理**: 機密情報として適切に保護・管理
- **最小権限の原則**: ユーザーには必要最小限の権限のみ付与
- **セッションセキュリティ**: 適切なタイムアウト設定、セッション無効化機能
- **監査ログ**: 認証・認可イベントの完全なログ記録

### セットアップ手順

#### 1. Keycloakサーバーのセットアップ
```bash
# Dockerを使用した開発環境でのKeycloak起動
docker run -p 8180:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:latest start-dev
```

#### 2. レルムとクライアントの設定
1. **レルム作成**: `keruta` レルムを作成

2. **SSR認証クライアント設定** (`keruta-admin-ssr`):
   - クライアントID: `keruta-admin-ssr`
   - クライアントタイプ: `OpenID Connect`
   - アクセスタイプ: `confidential`（SSRでは confidential client）
   - Standard Flow Enabled: `On`
   - Implicit Flow Enabled: `Off`
   - Direct Access Grants Enabled: `Off`
   - Service Accounts Enabled: `On`（サーバーサイド用）
   - Valid Redirect URIs: `https://your-admin-domain.com/auth/callback`
   - Valid Post Logout Redirect URIs: `https://your-admin-domain.com/*`
   - Web Origins: `https://your-admin-domain.com`
   - Root URL: `https://your-admin-domain.com`

#### 3. ユーザーとロールの設定
1. **ロール作成**: 前述の3つのロール（keruta-admin, keruta-operator, keruta-viewer）を作成
2. **ユーザー作成**: 必要なユーザーを作成し、適切なロールを割り当て
3. **グループ設定**: ロールベースのグループを作成してユーザー管理を効率化

### 型安全APIクライアント実装（openapi-typescript + fetch）

#### 生成される型定義（app/types/api.ts）
```typescript
// openapi-typescript によって自動生成される型定義
export interface paths {
  "/api/v1/tasks": {
    get: {
      responses: {
        200: {
          content: {
            "application/json": components["schemas"]["TaskListResponse"];
          };
        };
        401: {
          content: {
            "application/json": components["schemas"]["ErrorResponse"];
          };
        };
      };
    };
    post: {
      requestBody: {
        content: {
          "application/json": components["schemas"]["CreateTaskRequest"];
        };
      };
      responses: {
        201: {
          content: {
            "application/json": components["schemas"]["Task"];
          };
        };
      };
    };
  };
  "/api/v1/tasks/{id}": {
    get: {
      parameters: {
        path: {
          id: string;
        };
      };
      responses: {
        200: {
          content: {
            "application/json": components["schemas"]["Task"];
          };
        };
      };
    };
    delete: {
      parameters: {
        path: {
          id: string;
        };
      };
      responses: {
        204: never;
      };
    };
  };
}

export interface components {
  schemas: {
    Task: {
      id: string;
      name: string;
      status: "pending" | "running" | "completed" | "failed";
      createdAt: string;
      updatedAt: string;
    };
    TaskListResponse: {
      tasks: components["schemas"]["Task"][];
      total: number;
    };
    CreateTaskRequest: {
      name: string;
      description?: string;
    };
    ErrorResponse: {
      message: string;
      code: string;
    };
  };
}
```

#### 型安全なAPIクライアント実装
```typescript
// app/lib/api-client.ts
import type { paths, components } from "~/types/api";

type ApiResponse<T> = T extends { content: { "application/json": infer U } } ? U : never;

class TypeSafeApiClient {
  private baseURL: string;
  private getAuthToken: () => Promise<string | null>;

  constructor(baseURL: string, getAuthToken: () => Promise<string | null>) {
    this.baseURL = baseURL;
    this.getAuthToken = getAuthToken;
  }

  private async request<T>(
    path: string,
    options: RequestInit = {}
  ): Promise<T> {
    const token = await this.getAuthToken();
    
    const response = await fetch(`${this.baseURL}${path}`, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...(token && { Authorization: `Bearer ${token}` }),
        ...options.headers,
      },
    });

    if (!response.ok) {
      if (response.status === 401) {
        // 認証エラー時の処理（SSR対応）
        if (typeof window !== 'undefined') {
          window.location.href = '/auth/login';
        }
        throw new Error('認証が必要です');
      }
      
      const error = await response.json().catch(() => ({ message: 'Unknown error' }));
      throw new Error(error.message || `HTTP ${response.status}`);
    }

    if (response.status === 204) {
      return undefined as T;
    }

    return response.json();
  }

  // GET /api/v1/tasks
  async getTasks(): Promise<ApiResponse<paths["/api/v1/tasks"]["get"]["responses"][200]>> {
    return this.request<ApiResponse<paths["/api/v1/tasks"]["get"]["responses"][200]>>(
      "/api/v1/tasks"
    );
  }

  // POST /api/v1/tasks
  async createTask(
    data: paths["/api/v1/tasks"]["post"]["requestBody"]["content"]["application/json"]
  ): Promise<ApiResponse<paths["/api/v1/tasks"]["post"]["responses"][201]>> {
    return this.request<ApiResponse<paths["/api/v1/tasks"]["post"]["responses"][201]>>(
      "/api/v1/tasks",
      {
        method: "POST",
        body: JSON.stringify(data),
      }
    );
  }

  // GET /api/v1/tasks/{id}
  async getTask(
    id: string
  ): Promise<ApiResponse<paths["/api/v1/tasks/{id}"]["get"]["responses"][200]>> {
    return this.request<ApiResponse<paths["/api/v1/tasks/{id}"]["get"]["responses"][200]>>(
      `/api/v1/tasks/${id}`
    );
  }

  // DELETE /api/v1/tasks/{id}
  async deleteTask(id: string): Promise<void> {
    return this.request<void>(`/api/v1/tasks/${id}`, {
      method: "DELETE",
    });
  }

  // 汎用メソッド：任意のパスとメソッドで型安全なリクエスト
  async typedRequest<
    Path extends keyof paths,
    Method extends keyof paths[Path],
    Response extends paths[Path][Method] extends { responses: infer R } ? R : never,
    Success extends keyof Response = 200 extends keyof Response ? 200 : keyof Response
  >(
    path: Path,
    method: Method,
    options: {
      params?: paths[Path][Method] extends { parameters: { path: infer P } } ? P : never;
      query?: paths[Path][Method] extends { parameters: { query: infer Q } } ? Q : never;
      body?: paths[Path][Method] extends { requestBody: { content: { "application/json": infer B } } } ? B : never;
    } = {}
  ): Promise<ApiResponse<Response[Success]>> {
    let url = path as string;
    
    // パスパラメータの置換
    if (options.params) {
      Object.entries(options.params as Record<string, string>).forEach(([key, value]) => {
        url = url.replace(`{${key}}`, encodeURIComponent(value));
      });
    }

    // クエリパラメータの追加
    if (options.query) {
      const searchParams = new URLSearchParams(options.query as Record<string, string>);
      url += `?${searchParams.toString()}`;
    }

    return this.request<ApiResponse<Response[Success]>>(url, {
      method: method as string,
      ...(options.body && { body: JSON.stringify(options.body) }),
    });
  }
}

export { TypeSafeApiClient };
```

#### APIクライアント使用例

##### サービス層の実装
```typescript
// app/services/task.service.ts
import { TypeSafeApiClient } from "~/lib/api-client";
import type { components } from "~/types/api";

// 生成された型を再エクスポート
export type Task = components["schemas"]["Task"];
export type CreateTaskRequest = components["schemas"]["CreateTaskRequest"];
export type TaskListResponse = components["schemas"]["TaskListResponse"];

export class TaskService {
  constructor(private apiClient: TypeSafeApiClient) {}

  async getAllTasks(): Promise<TaskListResponse> {
    return this.apiClient.getTasks();
  }

  async createTask(request: CreateTaskRequest): Promise<Task> {
    return this.apiClient.createTask(request);
  }

  async getTask(id: string): Promise<Task> {
    return this.apiClient.getTask(id);
  }

  async deleteTask(id: string): Promise<void> {
    return this.apiClient.deleteTask(id);
  }

  // 汎用メソッドを使用した例
  async getTasksWithPagination(page: number, limit: number) {
    return this.apiClient.typedRequest("/api/v1/tasks", "get", {
      query: { page: page.toString(), limit: limit.toString() },
    });
  }
}

// APIクライアントのファクトリー
export function createTaskService(getAuthToken: () => Promise<string | null>): TaskService {
  const apiClient = new TypeSafeApiClient(
    import.meta.env.VITE_BACKEND_API_URL,
    getAuthToken
  );
  return new TaskService(apiClient);
}
```

##### SSR対応のAPIクライアント初期化
```typescript
// app/lib/api-client.server.ts
import { TypeSafeApiClient } from "./api-client";

export function createServerApiClient(token: string): TypeSafeApiClient {
  return new TypeSafeApiClient(
    process.env.VITE_BACKEND_API_URL!,
    async () => token
  );
}

// app/lib/api-client.client.ts
import { TypeSafeApiClient } from "./api-client";
import { keycloak } from "~/auth/keycloak.client";

export function createClientApiClient(): TypeSafeApiClient {
  return new TypeSafeApiClient(
    import.meta.env.VITE_BACKEND_API_URL,
    async () => {
      if (!keycloak || !keycloak.authenticated) {
        return null;
      }
      
      try {
        await keycloak.updateToken(30);
        return keycloak.token || null;
      } catch (error) {
        console.error("トークン更新エラー:", error);
        return null;
      }
    }
  );
}
```

#### React Router v7 + 型安全APIクライアントの使用例

##### ローダーでのサーバーサイドデータ取得
```typescript
// app/routes/tasks._index.tsx
import type { LoaderFunction } from "react-router";
import { json } from "react-router";
import { useLoaderData } from "react-router";
import { requireAuth } from "~/auth.server";
import { createServerApiClient } from "~/lib/api-client.server";
import { createTaskService, type TaskListResponse } from "~/services/task.service";

export const loader: LoaderFunction = async ({ request }) => {
  const auth = await requireAuth(request);
  
  // サーバーサイドでAPIからデータ取得
  const apiClient = createServerApiClient(auth.token);
  const taskService = createTaskService(async () => auth.token);
  
  const tasksData = await taskService.getAllTasks();

  return json({
    user: auth.user,
    tasks: tasksData,
  });
};

export default function TasksPage() {
  const { tasks, user } = useLoaderData<{
    tasks: TaskListResponse;
    user: any;
  }>();

  return (
    <div>
      <h1>タスク管理</h1>
      <p>合計: {tasks.total}件</p>
      
      <div>
        {tasks.tasks.map(task => (
          <div key={task.id}>
            <h3>{task.name}</h3>
            <p>状態: {task.status}</p>
            <p>作成日: {new Date(task.createdAt).toLocaleDateString()}</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

##### クライアントサイドでの動的操作
```typescript
// app/components/TaskManager.tsx
import React, { useState } from 'react';
import { useNavigate } from 'react-router';
import { createClientApiClient } from '~/lib/api-client.client';
import { createTaskService, type Task, type CreateTaskRequest } from '~/services/task.service';
import { useAuth } from '~/contexts/AuthContext';

const taskService = createTaskService(() => 
  Promise.resolve(document.cookie.includes('access_token') ? 'token' : null)
);

interface TaskManagerProps {
  initialTasks: Task[];
}

export function TaskManager({ initialTasks }: TaskManagerProps) {
  const [tasks, setTasks] = useState<Task[]>(initialTasks);
  const [loading, setLoading] = useState(false);
  const { hasRole } = useAuth();
  const navigate = useNavigate();

  const handleCreateTask = async (request: CreateTaskRequest) => {
    try {
      setLoading(true);
      const newTask = await taskService.createTask(request);
      setTasks([...tasks, newTask]);
    } catch (error) {
      console.error('タスク作成エラー:', error);
      if (error instanceof Error && error.message.includes('認証')) {
        navigate('/auth/login');
      }
    } finally {
      setLoading(false);
    }
  };

  const handleDeleteTask = async (taskId: string) => {
    try {
      setLoading(true);
      await taskService.deleteTask(taskId);
      setTasks(tasks.filter(task => task.id !== taskId));
    } catch (error) {
      console.error('タスク削除エラー:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      {(hasRole('keruta-admin') || hasRole('keruta-operator')) && (
        <div>
          <button 
            onClick={() => handleCreateTask({ 
              name: 'New Task',
              description: '新しいタスクです'
            })}
            disabled={loading}
          >
            {loading ? '作成中...' : 'タスク作成'}
          </button>
        </div>
      )}

      <div>
        {tasks.map(task => (
          <div key={task.id} style={{ border: '1px solid #ccc', margin: '10px', padding: '10px' }}>
            <h3>{task.name}</h3>
            <p>状態: <span style={{ 
              color: task.status === 'completed' ? 'green' : 
                     task.status === 'failed' ? 'red' : 'orange' 
            }}>{task.status}</span></p>
            
            {hasRole('keruta-admin') && (
              <button 
                onClick={() => handleDeleteTask(task.id)}
                disabled={loading}
                style={{ backgroundColor: '#ff4444', color: 'white' }}
              >
                削除
              </button>
            )}
          </div>
        ))}
      </div>
    </div>
  );
}
```

##### 型安全性の利点
```typescript
// 型安全性の例 - コンパイル時にエラーを検出
const example = async () => {
  const taskService = createTaskService(() => Promise.resolve('token'));
  
  // ✅ 正しい使用法 - 型チェックパス
  const task = await taskService.createTask({
    name: "Valid Task",
    description: "Optional description"
  });
  
  // ❌ 型エラー - 必須フィールドが不足
  // const invalidTask = await taskService.createTask({
  //   description: "Missing name field"
  // });
  
  // ✅ レスポンスの型も自動で推論
  console.log(task.id);        // string
  console.log(task.status);    // "pending" | "running" | "completed" | "failed"
  console.log(task.createdAt); // string
  
  // ❌ 存在しないプロパティアクセスはコンパイルエラー
  // console.log(task.invalidProperty);
};
```

### トラブルシューティング

#### よくあるエラーと解決方法

1. **SPA認証エラー (401 Unauthorized)**
   - 原因: SPAクライアントIDが正しくないまたはKeycloak設定が不正
   - 解決策: Keycloak管理コンソールでSPAクライアント設定を再確認

2. **CORS エラー**
   - 原因: Keycloakのクライアント設定でWeb Originsが正しく設定されていない
   - 解決策: Keycloakクライアント設定でWeb Originsにフロントエンドドメインを追加

3. **トークン更新エラー**
   - 原因: リフレッシュトークンが期限切れまたは無効
   - 解決策: Keycloak設定でセッションアイドルタイムアウトを確認、必要に応じて再ログイン

4. **ルートアクセスエラー (403 Forbidden)**
   - 原因: ユーザーに適切なロールが割り当てられていない
   - 解決策: Keycloak管理コンソールでユーザーのロール割り当てを確認

5. **PKCEエラー**
   - 原因: Keycloakクライアント設定でPKCE設定が正しくない
   - 解決策: ブラウザ開発者ツールでPKCEパラメータを確認、Keycloak設定を見直し

6. **API呼び出しエラー**
   - 原因: バックエンドAPIでのJWTトークン検証失敗
   - 解決策: バックエンドAPIのKeycloak設定（issuer-uri、jwk-set-uri）を確認

7. **ローカル開発でのHTTPS要求エラー**
   - 原因: 本番環境ではHTTPS必須だが開発環境でHTTP使用
   - 解決策: 開発用Keycloakレルム設定で「SSL required」を「external requests」に変更

### 関連ドキュメント

詳細な設定方法については、以下のドキュメントを参照してください：
- [Keycloak連携設定](../keruta-api/auth/keycloakIntegration.md)
- [バックエンド設定](./backendConfiguration.md)
