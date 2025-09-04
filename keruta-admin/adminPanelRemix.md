# 管理パネルのRemix実装

> **概要**: keruta管理パネルのRemix.jsを使った実装に関するドキュメントです。

## 目次
- [概要](#概要)
- [技術スタック](#技術スタック)
- [プロジェクト構成](#プロジェクト構成)
- [ルーティング](#ルーティング)
- [開発方法](#開発方法)
- [デプロイ方法](#デプロイ方法)
- [APIとの連携](#apiとの連携)

## 概要
keruta管理パネルは、従来のSpring Boot + Thymeleafによる実装から、Remix.jsを使ったモダンなフロントエンド実装に移行しました。Remix.jsは、Reactベースのフルスタックウェブフレームワークで、優れたユーザー体験と開発者体験を提供します。

## 技術スタック
- **フレームワーク**: Remix.js (React)
- **スタイリング**: Bootstrap 5
- **言語**: TypeScript
- **ビルドツール**: Remix CLI

## プロジェクト構成
```
keruta-admin/
├── app/                    # アプリケーションコード
│   ├── components/         # 共通コンポーネント
│   ├── routes/             # ルート定義
│   └── root.tsx            # ルートレイアウト
├── public/                 # 静的ファイル
├── package.json            # 依存関係
├── remix.config.js         # Remix設定
└── tsconfig.json           # TypeScript設定
```

## ルーティング
Remixでは、ファイルベースのルーティングを採用しています。主要なルートは以下の通りです：

- `/` - ダッシュボード
- `/tasks` - タスク管理
- `/documents` - ドキュメント管理
- `/repositories` - リポジトリ管理
- `/kubernetes` - Kubernetes設定
- `/agents` - エージェント管理

## 開発方法
### 環境構築
1. 依存関係のインストール
```bash
npm install
```

2. 開発サーバーの起動
```bash
npm run dev
```

3. ブラウザで http://localhost:3000 にアクセス

### コンポーネント開発
共通コンポーネントは `app/components` ディレクトリに配置します。例えば、共通レイアウトは `app/components/Layout.tsx` に定義されています。

## デプロイ方法
### ビルド
```bash
npm run build
```

### 本番環境での実行
```bash
npm start
```

### Dockerを使ったデプロイ
```bash
docker build -t keruta-admin .
docker run -p 3000:3000 keruta-admin
```

## APIとの連携
Remix.jsでは、サーバーサイドのローダー関数とアクション関数を使用してAPIと連携します。

### ローダー関数の例
```typescript
export const loader = async ({ request }: LoaderFunctionArgs) => {
  const apiUrl = process.env.API_URL || "http://localhost:8080/api";
  const response = await fetch(`${apiUrl}/tasks`);
  
  if (!response.ok) {
    throw new Response("APIからのデータ取得に失敗しました", { status: 500 });
  }
  
  return json(await response.json());
};
```

### アクション関数の例
```typescript
export const action = async ({ request }: ActionFunctionArgs) => {
  const formData = await request.formData();
  const apiUrl = process.env.API_URL || "http://localhost:8080/api";
  
  const response = await fetch(`${apiUrl}/tasks`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
    },
    body: JSON.stringify(Object.fromEntries(formData)),
  });
  
  if (!response.ok) {
    return json({ error: "タスクの作成に失敗しました" }, { status: 400 });
  }
  
  return redirect("/tasks");
};
```

---

このドキュメントは、keruta管理パネルのRemix.js実装に関する基本的な情報を提供しています。詳細な実装やAPI仕様については、コードベースやAPI仕様書を参照してください。