# セッション関連フォーム仕様

## 概要

Keruta Admin パネルにおけるセッション管理フォームの詳細仕様書です。Remix.js と Tailwind CSS を使用したモダンな実装で、セッションの作成・編集・管理を行います。

## 技術スタック

- **フレームワーク**: Remix.js (React)
- **スタイリング**: Tailwind CSS
- **言語**: TypeScript
- **ビルドツール**: Remix CLI

## セッション管理画面構成

### 画面一覧

| 画面名 | パス | 説明 |
|---|---|---|
| セッション一覧 | `/sessions` | セッションの一覧表示・検索・フィルタリング |
| セッション詳細 | `/sessions/:id` | セッション詳細情報・関連ワークスペース表示 |
| セッション作成 | `/sessions/new` | 新規セッション作成フォーム |
| セッション編集 | `/sessions/:id/edit` | 既存セッション編集フォーム |

## セッション一覧画面仕様

### レイアウト

```tsx
// sessions/index.tsx
<div className="min-h-screen bg-gray-50">
  <div className="container mx-auto px-4 py-6">
    <PageHeader title="セッション管理" />
    <SessionFilters />
    <SessionTable />
    <Pagination />
  </div>
</div>
```

### フィルタリング機能

#### SessionFilters コンポーネント

```tsx
interface SessionFiltersProps {
  filters: SessionFilters;
  onFiltersChange: (filters: SessionFilters) => void;
}

interface SessionFilters {
  status: 'ALL' | 'PENDING' | 'ACTIVE' | 'COMPLETED' | 'TERMINATED';
  name: string;
  tags: string[];
  createdFrom: string;
  createdTo: string;
}
```

#### フィルタUI

| フィールド | 入力タイプ | 説明 |
|---|---|---|
| ステータス | ドロップダウン | セッションステータスで絞り込み |
| セッション名 | テキスト入力 | 部分一致検索 |
| タグ | タグセレクト | 複数選択可能 |
| 作成日（開始） | 日付選択 | 作成日範囲の開始日 |
| 作成日（終了） | 日付選択 | 作成日範囲の終了日 |

### テーブル表示

#### SessionTable コンポーネント

| カラム | 表示内容 | ソート | 説明 |
|---|---|---|---|
| ID | セッションID（8桁） | × | セッション識別子（短縮表示） |
| 名前 | セッション名 | ○ | リンククリックで詳細画面へ |
| ステータス | バッジ表示 | ○ | PENDING/ACTIVE/COMPLETED/TERMINATED |
| タグ | タグバッジ | × | 最大3個まで表示、超過分は "+N" |
| 作成日時 | 相対時間表示 | ○ | "2時間前", "3日前" 等 |
| ワークスペース | 関連ワークスペース数 | × | "1個のワークスペース" |
| アクション | ボタン群 | × | 詳細・編集・削除 |

#### ステータスバッジ

```tsx
const StatusBadge = ({ status }: { status: string }) => {
  const badgeClass = {
    PENDING: 'bg-yellow-400 text-black',
    ACTIVE: 'bg-green-600 text-white', 
    COMPLETED: 'bg-blue-500 text-white',
    TERMINATED: 'bg-red-600 text-white'
  }[status] || 'bg-gray-500 text-white';
  
  return (
    <span className={`inline-block px-2 py-1 rounded text-xs font-medium uppercase ${badgeClass}`}>
      {status}
    </span>
  );
};
```

## セッション作成フォーム仕様

### フォーム構成

#### CreateSessionForm コンポーネント

```tsx
interface CreateSessionRequest {
  name: string;
  description?: string;
  tags: string[];
}
```

### フォームフィールド

| フィールド | 入力タイプ | 必須 | バリデーション | 説明 |
|---|---|---|---|---|
| セッション名 | テキスト入力 | ✓ | 3-50文字、英数字・ハイフン・アンダースコア | ワークスペース名に使用 |
| 説明 | テキストエリア | - | 最大500文字 | セッション用途の説明 |
| タグ | タグ入力 | - | 各タグ1-20文字、最大10個 | 分類・検索用 |

### バリデーション仕様

#### フロントエンド検証

```tsx
const sessionFormSchema = {
  name: {
    required: true,
    minLength: 3,
    maxLength: 50,
    pattern: /^[a-zA-Z0-9\-_]+$/,
    message: 'セッション名は3-50文字の英数字・ハイフン・アンダースコアで入力してください'
  },
  description: {
    maxLength: 500,
    message: '説明は500文字以内で入力してください'
  },
  tags: {
    maxItems: 10,
    itemPattern: /^[a-zA-Z0-9\-_]+$/,
    itemMaxLength: 20,
    message: 'タグは20文字以内の英数字・ハイフン・アンダースコアで、最大10個まで'
  }
};
```

#### サーバーサイド検証

```tsx
// sessions/new.tsx action
export const action = async ({ request }: ActionFunctionArgs) => {
  const formData = await request.formData();
  const sessionData = Object.fromEntries(formData);
  
  // バリデーション
  const errors = validateSessionData(sessionData);
  if (Object.keys(errors).length > 0) {
    return json({ errors }, { status: 400 });
  }
  
  try {
    const response = await fetch(`${API_BASE_URL}/api/v1/sessions`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(sessionData)
    });
    
    if (!response.ok) {
      throw new Error('セッション作成に失敗しました');
    }
    
    return redirect('/sessions');
  } catch (error) {
    return json({ error: error.message }, { status: 500 });
  }
};
```

### フォームUI

#### レイアウト

```tsx
<Form method="post" className="max-w-2xl mx-auto p-8 bg-white shadow-lg rounded-lg">
  <div className="grid grid-cols-1 md:grid-cols-2 gap-4 mb-6">
    <div className="md:col-span-1">
      <label htmlFor="name" className="block text-sm font-medium text-gray-700 mb-2">
        セッション名 <span className="text-red-600">*</span>
      </label>
      <input
        type="text"
        className="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
        id="name"
        name="name"
        required
        maxLength={50}
      />
      <div className="mt-1 text-sm text-red-600 hidden peer-invalid:block">
        セッション名を入力してください
      </div>
    </div>
  </div>
  
  <div className="mb-6">
    <label htmlFor="description" className="block text-sm font-medium text-gray-700 mb-2">
      説明
    </label>
    <textarea
      className="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
      id="description"
      name="description"
      rows={3}
      maxLength={500}
    />
  </div>
  
  <div className="mb-6">
    <label htmlFor="tags" className="block text-sm font-medium text-gray-700 mb-2">
      タグ
    </label>
    <TagInput name="tags" maxTags={10} />
  </div>
  
  <div className="flex flex-col sm:flex-row justify-between gap-4 mt-8">
    <Link 
      to="/sessions" 
      className="px-6 py-2 text-gray-700 bg-gray-200 hover:bg-gray-300 rounded-md text-center transition-colors"
    >
      キャンセル
    </Link>
    <button 
      type="submit" 
      className="px-6 py-2 text-white bg-blue-600 hover:bg-blue-700 rounded-md transition-colors"
    >
      作成
    </button>
  </div>
</Form>
```

## セッション詳細画面仕様

### 詳細情報表示

#### SessionDetail コンポーネント

```tsx
interface SessionDetailProps {
  session: SessionResponse;
  workspaces: CoderWorkspaceResponse[];
}
```

### 表示項目

| セクション | 項目 | 表示形式 |
|---|---|---|
| 基本情報 | セッションID | 全桁表示 |
|  | セッション名 | テキスト |
|  | 説明 | テキスト（改行対応） |
|  | ステータス | バッジ |
|  | タグ | タグバッジ |
|  | 作成日時 | ISO形式 + 相対時間 |
|  | 更新日時 | ISO形式 + 相対時間 |
| ワークスペース | 関連ワークスペース一覧 | テーブル |
|  | ワークスペースアクセス | リンクボタン |

### ワークスペーステーブル

| カラム | 表示内容 | 説明 |
|---|---|---|
| 名前 | ワークスペース名 | Coder へのリンク |
| ステータス | 実行状態 | running/stopped/starting/stopping |
| ヘルス | 健全性 | healthy/unhealthy/unknown |
| 最終使用 | 最終アクセス時間 | 相対時間表示 |
| アクション | 操作ボタン | 開始・停止・アクセス |

## セッション編集フォーム仕様

### 編集制約

#### 編集可能項目

| 項目 | 編集可否 | 理由 |
|---|---|---|
| セッション名 | × | ワークスペース名に影響するため |
| 説明 | ○ | 変更可能 |
| タグ | ○ | 変更可能 |
| ステータス | × | システムが自動管理 |

#### EditSessionForm コンポーネント

```tsx
interface UpdateSessionRequest {
  description?: string;
  tags: string[];
}
```

### 編集フォームUI

```tsx
<Form method="put" className="max-w-2xl mx-auto p-8 bg-white shadow-lg rounded-lg">
  <div className="mb-6">
    <label className="block text-sm font-medium text-gray-700 mb-2">
      セッション名
    </label>
    <input
      type="text"
      className="w-full px-3 py-2 bg-gray-50 border border-gray-300 rounded-md cursor-not-allowed text-gray-600"
      value={session.name}
      disabled
      readOnly
    />
    <div className="mt-1 text-sm text-gray-600">
      セッション名は変更できません
    </div>
  </div>
  
  <div className="mb-6">
    <label htmlFor="description" className="block text-sm font-medium text-gray-700 mb-2">
      説明
    </label>
    <textarea
      className="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500"
      id="description"
      name="description"
      defaultValue={session.description}
      rows={3}
      maxLength={500}
    />
  </div>
  
  <div className="mb-6">
    <label htmlFor="tags" className="block text-sm font-medium text-gray-700 mb-2">
      タグ
    </label>
    <TagInput 
      name="tags" 
      defaultValue={session.tags}
      maxTags={10} 
    />
  </div>
  
  <div className="flex flex-col sm:flex-row justify-between gap-4 mt-8">
    <Link 
      to={`/sessions/${session.id}`} 
      className="px-6 py-2 text-gray-700 bg-gray-200 hover:bg-gray-300 rounded-md text-center transition-colors"
    >
      キャンセル
    </Link>
    <button 
      type="submit" 
      className="px-6 py-2 text-white bg-blue-600 hover:bg-blue-700 rounded-md transition-colors"
    >
      更新
    </button>
  </div>
</Form>
```

## 共通コンポーネント仕様

### TagInput コンポーネント

```tsx
interface TagInputProps {
  name: string;
  defaultValue?: string[];
  maxTags?: number;
  placeholder?: string;
}

const TagInput: React.FC<TagInputProps> = ({ 
  name, 
  defaultValue = [], 
  maxTags = 10,
  placeholder = "タグを入力してEnter" 
}) => {
  const [tags, setTags] = useState<string[]>(defaultValue);
  const [inputValue, setInputValue] = useState('');
  
  const handleKeyPress = (e: KeyboardEvent) => {
    if (e.key === 'Enter' && inputValue.trim()) {
      e.preventDefault();
      if (tags.length < maxTags && !tags.includes(inputValue.trim())) {
        setTags([...tags, inputValue.trim()]);
        setInputValue('');
      }
    }
  };
  
  return (
    <div className="space-y-2">
      <div className="flex flex-wrap gap-2 min-h-[24px]">
        {tags.map((tag, index) => (
          <span 
            key={index} 
            className="inline-flex items-center gap-1 px-2 py-1 bg-gray-200 text-gray-700 rounded-full text-sm"
          >
            {tag}
            <button
              type="button"
              className="ml-1 text-gray-500 hover:text-gray-700 hover:bg-gray-300 rounded-full w-4 h-4 flex items-center justify-center text-xs"
              onClick={() => setTags(tags.filter((_, i) => i !== index))}
              aria-label="タグを削除"
            >
              ×
            </button>
          </span>
        ))}
      </div>
      <input
        type="text"
        className="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-blue-500 disabled:bg-gray-50 disabled:cursor-not-allowed disabled:text-gray-600"
        value={inputValue}
        onChange={(e) => setInputValue(e.target.value)}
        onKeyPress={handleKeyPress}
        placeholder={placeholder}
        disabled={tags.length >= maxTags}
      />
      {tags.map((tag, index) => (
        <input
          key={index}
          type="hidden"
          name={name}
          value={tag}
        />
      ))}
    </div>
  );
};
```

## API連携仕様

### エンドポイント対応

| 画面操作 | HTTPメソッド | エンドポイント | 説明 |
|---|---|---|---|
| 一覧取得 | GET | `/api/v1/sessions` | フィルタリング・ページング対応 |
| 詳細取得 | GET | `/api/v1/sessions/:id` | セッション詳細情報 |
| 作成 | POST | `/api/v1/sessions` | 新規セッション作成 |
| 更新 | PUT | `/api/v1/sessions/:id` | セッション情報更新（制限あり） |
| 削除 | DELETE | `/api/v1/sessions/:id` | セッション削除 |
| ワークスペース取得 | GET | `/api/v1/sessions/:id/workspaces` | 関連ワークスペース |

### エラーハンドリング

#### API エラー対応

```tsx
// sessions/routes.tsx
export const loader = async ({ request }: LoaderFunctionArgs) => {
  try {
    const url = new URL(request.url);
    const params = new URLSearchParams({
      page: url.searchParams.get('page') || '0',
      size: url.searchParams.get('size') || '20',
      status: url.searchParams.get('status') || '',
      name: url.searchParams.get('name') || ''
    });
    
    const response = await fetch(`${API_BASE_URL}/api/v1/sessions?${params}`);
    
    if (!response.ok) {
      throw new Response('セッション一覧の取得に失敗しました', { 
        status: response.status 
      });
    }
    
    return json(await response.json());
  } catch (error) {
    console.error('Sessions loader error:', error);
    throw new Response('サーバーエラーが発生しました', { status: 500 });
  }
};
```

#### フォームエラー表示

```tsx
// ErrorMessage コンポーネント
const ErrorMessage: React.FC<{ error?: string }> = ({ error }) => {
  if (!error) return null;
  
  return (
    <div className="flex items-center gap-2 p-4 bg-red-50 border border-red-200 rounded-md text-red-700 mb-4" role="alert">
      <span className="text-xl">⚠️</span>
      {error}
    </div>
  );
};
```

## アクセス権限

### ロール別操作権限

| 操作 | 管理者 (admin) | 開発者 (developer) | 一般ユーザー (user) |
|---|:-:|:-:|:-:|
| セッション一覧閲覧 | ○ | ○ | ○ |
| セッション詳細閲覧 | ○ | ○ | ○ |
| セッション作成 | ○ | ○ | △（制限あり） |
| セッション編集 | ○ | ○ | △（自分のみ） |
| セッション削除 | ○ | × | × |
| ワークスペース操作 | ○ | ○ | △（自分のみ） |

### 認証・認可実装

```tsx
// app/utils/auth.server.ts
export const requireUser = async (request: Request) => {
  const session = await getSession(request.headers.get('Cookie'));
  const token = session.get('access_token');
  
  if (!token) {
    throw redirect('/auth/login');
  }
  
  return getUserFromToken(token);
};

export const requireRole = (allowedRoles: string[]) => {
  return async (request: Request) => {
    const user = await requireUser(request);
    
    if (!allowedRoles.some(role => user.roles.includes(role))) {
      throw new Response('アクセス権限がありません', { status: 403 });
    }
    
    return user;
  };
};
```

## レスポンシブ対応

### ブレークポイント対応

| デバイス | 画面幅 | レイアウト調整 |
|---|---|---|
| スマートフォン | < 576px | テーブル → カード表示、フィルタ → モーダル |
| タブレット | 576px - 768px | 2カラム → 1カラム、サイドバー → 折りたたみ |
| デスクトップ | > 768px | フルレイアウト |

### モバイル最適化

```tsx
// SessionCard コンポーネント（モバイル用）
const SessionCard: React.FC<{ session: SessionResponse }> = ({ session }) => (
  <div className="bg-white border border-gray-200 rounded-lg mb-4 block md:hidden">
    <div className="p-4">
      <div className="flex justify-between items-start mb-2">
        <h6 className="text-lg font-semibold text-gray-900">{session.name}</h6>
        <StatusBadge status={session.status} />
      </div>
      
      <p className="text-gray-600 text-sm mb-4 leading-relaxed">{session.description}</p>
      
      <div className="flex justify-between items-center">
        <small className="text-gray-600 text-sm">
          {formatDistanceToNow(new Date(session.createdAt))}前
        </small>
        <div className="flex flex-col sm:flex-row gap-2">
          <Link 
            to={`/sessions/${session.id}`} 
            className="px-3 py-1.5 text-blue-600 border border-blue-600 hover:bg-blue-50 rounded text-center text-sm transition-colors"
          >
            詳細
          </Link>
          <Link 
            to={`/sessions/${session.id}/edit`} 
            className="px-3 py-1.5 text-gray-600 border border-gray-600 hover:bg-gray-50 rounded text-center text-sm transition-colors"
          >
            編集
          </Link>
        </div>
      </div>
    </div>
  </div>
);
```

## パフォーマンス考慮事項

### データ取得最適化

- **ページング**: デフォルト20件、最大100件
- **遅延読み込み**: ワークスペース情報は詳細画面で取得
- **キャッシュ**: セッション一覧は5分間キャッシュ
- **検索デバウンス**: 名前検索は300ms遅延

### フォーム最適化

- **クライアントサイド検証**: リアルタイム検証でUX向上
- **プログレッシブエンハンスメント**: JavaScript無効時も動作
- **楽観的更新**: 成功想定でUI即座に更新

## 関連ドキュメント

- [Session-Workspace同期APIシステム](../../session-workspace-sync-system.md)
- [管理パネルドキュメント](./adminPanel.md)
- [管理パネルのRemix実装](./adminPanelRemix.md)
- [Coderテンプレート利用制約仕様](../keruta-api/system/coderTemplateSpec.md)