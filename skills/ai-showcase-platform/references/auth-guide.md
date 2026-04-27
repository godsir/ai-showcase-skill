# 用户认证方案（Supabase Auth + 游客模式）

## 架构概览

```
前端（React）
  ├─ Supabase Auth（邮箱/密码 or OAuth）→ 正式用户
  └─ Guest Mode（localStorage mock User）→ 游客体验用户
      ↓
NavBar / ProfilePage / CartPage 等通过 useAuth() 获取统一 user 对象
      ↓
请求 Cloud Function 时携带 Authorization: Bearer <token>（仅正式用户）
```

**核心设计**：AuthContext 同时管理正式用户和游客用户，对外暴露统一的 `user` 对象，组件无需关心用户来源。

---

## Supabase 项目初始化

1. 访问 [https://supabase.com](https://supabase.com) 创建新项目
2. 进入 **Project Settings → API**，复制：
   - `VITE_SUPABASE_URL`（Project URL）
   - `VITE_SUPABASE_ANON_KEY`（anon/public key）
   - `SUPABASE_SERVICE_ROLE_KEY`（service_role key，**仅用于服务端**）
3. 进入 **Authentication → Providers** 确认 Email 已启用
4. （可选）启用 GitHub OAuth，填写 Client ID/Secret

### Supabase 数据库表（在 SQL Editor 中执行）

```sql
-- 订单表
create table public.orders (
  id           uuid default gen_random_uuid() primary key,
  user_id      uuid references auth.users(id) on delete cascade not null,
  stripe_session_id text unique not null,
  items        jsonb not null,               -- [{productId, name, price, qty}]
  amount_total integer not null,             -- 单位：分（cents）
  currency     text not null default 'usd',
  status       text not null default 'pending', -- pending / paid / cancelled
  created_at   timestamptz default now()
);

-- RLS 策略：用户只能查看自己的订单
alter table public.orders enable row level security;

create policy "users can view own orders"
  on public.orders for select
  using (auth.uid() = user_id);

create policy "service role can insert orders"
  on public.orders for insert
  with check (true);   -- Cloud Function 用 service_role 绕过 RLS 插入
```

---

## 前端实现

### `src/lib/supabase.ts`

```typescript
import { createClient } from '@supabase/supabase-js';

const supabaseUrl  = import.meta.env.VITE_SUPABASE_URL as string;
const supabaseAnon = import.meta.env.VITE_SUPABASE_ANON_KEY as string;

export const isSupabaseConfigured = !!(supabaseUrl && supabaseAnon);

// 未配置时不创建 client，避免报错
export const supabase = isSupabaseConfigured
  ? createClient(supabaseUrl, supabaseAnon)
  : null as never;
```

> ⚠️ **关键**：`isSupabaseConfigured` 必须导出，AuthContext 依赖它判断是否走 Supabase 认证。
> ⚠️ **不能 throw Error** — 未配置时应优雅降级，让游客模式正常工作。

### `src/hooks/useAuth.tsx`（AuthContext + 游客模式）

```typescript
import { createContext, useContext, useEffect, useState, useCallback, type ReactNode } from 'react';
import type { User, Session } from '@supabase/supabase-js';
import { supabase, isSupabaseConfigured } from '@/lib/supabase';

const GUEST_STORAGE_KEY = 'mindforge_guest_user';
const DEMO_EMAIL = 'demo@mindforge.ai';
const DEMO_USER_ID = 'guest-demo-001';

/** 构造一个 mock User 对象，字段与 Supabase User 对齐 */
function createMockUser(): User {
  return {
    id: DEMO_USER_ID,
    email: DEMO_EMAIL,
    aud: 'authenticated',
    role: 'authenticated',
    app_metadata: {},
    user_metadata: { name: '体验用户' },
    created_at: new Date().toISOString(),
    // @ts-expect-error — Supabase 内部字段，mock 不需要完整
    identities: [],
  } as User;
}

interface AuthContextType {
  user: User | null;
  session: Session | null;
  loading: boolean;
  isConfigured: boolean;
  isGuest: boolean;
  signIn: (email: string, password: string) => Promise<{ error: Error | null }>;
  signUp: (email: string, password: string) => Promise<{ error: Error | null }>;
  signOut: () => Promise<void>;
  updatePassword: (newPassword: string) => Promise<{ error: Error | null }>;
  guestSignIn: () => void;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [session, setSession] = useState<Session | null>(null);
  const [loading, setLoading] = useState(true);
  const [isGuest, setIsGuest] = useState(false);

  useEffect(() => {
    // 优先检查游客身份
    const guestData = localStorage.getItem(GUEST_STORAGE_KEY);
    if (guestData) {
      try {
        const mockUser = createMockUser();
        mockUser.created_at = JSON.parse(guestData).created_at || mockUser.created_at;
        setUser(mockUser);
        setIsGuest(true);
        setLoading(false);
        return;
      } catch {
        localStorage.removeItem(GUEST_STORAGE_KEY);
      }
    }

    if (!isSupabaseConfigured) {
      setLoading(false);
      return;
    }

    supabase.auth.getSession().then(({ data }) => {
      setSession(data.session);
      setUser(data.session?.user ?? null);
      setLoading(false);
    });

    const { data: { subscription } } = supabase.auth.onAuthStateChange((_event, session) => {
      setSession(session);
      setUser(session?.user ?? null);
    });

    return () => subscription.unsubscribe();
  }, []);

  const signIn = useCallback(async (email: string, password: string) => {
    const { error } = await supabase.auth.signInWithPassword({ email, password });
    return { error: error as Error | null };
  }, []);

  const signUp = useCallback(async (email: string, password: string) => {
    const { error } = await supabase.auth.signUp({ email, password });
    return { error: error as Error | null };
  }, []);

  const signOut = useCallback(async () => {
    localStorage.removeItem(GUEST_STORAGE_KEY);
    setIsGuest(false);
    setUser(null);    // ⚠️ 必须！否则 NavBar 不会刷新
    setSession(null); // ⚠️ 必须！
    if (isSupabaseConfigured) {
      await supabase.auth.signOut();
    }
  }, []);

  const updatePassword = useCallback(async (newPassword: string) => {
    if (isGuest) {
      return { error: new Error('体验账号不支持修改密码') };
    }
    const { error } = await supabase.auth.updateUser({ password: newPassword });
    return { error: error as Error | null };
  }, []);

  const guestSignIn = useCallback(() => {
    const mockUser = createMockUser();
    localStorage.setItem(GUEST_STORAGE_KEY, JSON.stringify({
      created_at: mockUser.created_at,
      email: DEMO_EMAIL,
    }));
    setUser(mockUser);
    setIsGuest(true);
  }, []);

  return (
    <AuthContext.Provider value={{
      user, session, loading, isConfigured: isSupabaseConfigured,
      isGuest,
      signIn, signUp, signOut, updatePassword, guestSignIn,
    }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within AuthProvider');
  return ctx;
}
```

### `src/pages/LoginPage.tsx`（含游客体验入口）

```tsx
import { useState } from 'react';
import { useNavigate, useLocation, Link } from 'react-router-dom';
import { useAuth } from '@/hooks/useAuth';
import { cn } from '@/lib/cn';
import { Sparkles } from 'lucide-react';

type Tab = 'login' | 'register';

export function LoginPage() {
  const { signIn, signUp, isConfigured, guestSignIn } = useAuth();
  const navigate = useNavigate();
  const location = useLocation();
  const from = (location.state as { from?: string })?.from ?? '/';

  const [tab, setTab] = useState<Tab>('login');
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [confirm, setConfirm] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');
    if (tab === 'register' && password !== confirm) {
      setError('两次密码不一致');
      return;
    }
    setLoading(true);
    try {
      const { error: err } = tab === 'login'
        ? await signIn(email, password)
        : await signUp(email, password);
      if (err) throw err;
      if (tab === 'register') {
        setError('注册成功！请查收验证邮件后登录。');
        setTab('login');
      } else {
        navigate(from, { replace: true });
      }
    } catch (err: unknown) {
      setError(err instanceof Error ? err.message : '操作失败，请重试');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="min-h-screen bg-white flex items-center justify-center px-4 pt-16">
      <div className="w-full max-w-sm">
        <h1 className="text-3xl font-bold text-black mb-8 text-center">
          {tab === 'login' ? '欢迎回来' : '创建账号'}
        </h1>

        {/* Tab 切换 */}
        <div className="flex border border-black rounded-full p-1 mb-8">
          {(['login', 'register'] as Tab[]).map((t) => (
            <button key={t} onClick={() => { setTab(t); setError(''); }}
              className={cn(
                'flex-1 py-2 text-sm font-medium rounded-full transition-all duration-200',
                tab === t ? 'bg-black text-white' : 'text-black/50 hover:text-black'
              )}>
              {t === 'login' ? '登录' : '注册'}
            </button>
          ))}
        </div>

        {/* 未配置 Supabase 提示 */}
        {!isConfigured && tab !== 'login' && (
          <div className="mb-4 p-3 bg-yellow-50 text-yellow-700 text-xs rounded-xl">
            ⚠️ 认证服务未配置，请先在 .env.local 中填写 Supabase 凭证。或使用下方游客模式体验。
          </div>
        )}

        {/* 表单 */}
        <form onSubmit={handleSubmit} className="space-y-4">
          <div>
            <label className="block text-sm font-medium text-black mb-1.5">邮箱</label>
            <input type="email" required value={email}
              onChange={(e) => setEmail(e.target.value)} placeholder="you@example.com"
              className="w-full border border-black rounded-xl px-4 py-3 text-sm focus:outline-none focus:ring-2 focus:ring-black/20" />
          </div>
          <div>
            <label className="block text-sm font-medium text-black mb-1.5">密码</label>
            <input type="password" required value={password}
              onChange={(e) => setPassword(e.target.value)} placeholder="••••••••"
              className="w-full border border-black rounded-xl px-4 py-3 text-sm focus:outline-none focus:ring-2 focus:ring-black/20" />
          </div>
          {tab === 'register' && (
            <div>
              <label className="block text-sm font-medium text-black mb-1.5">确认密码</label>
              <input type="password" required value={confirm}
                onChange={(e) => setConfirm(e.target.value)} placeholder="••••••••"
                className={cn('w-full border rounded-xl px-4 py-3 text-sm focus:outline-none focus:ring-2',
                  confirm && password !== confirm ? 'border-red-500 focus:ring-red-200' : 'border-black focus:ring-black/20')} />
            </div>
          )}

          {error && (
            <p className={cn('text-sm px-3 py-2 rounded-lg',
              error.includes('成功') ? 'bg-green-50 text-green-700' : 'bg-red-50 text-red-600')}>{error}</p>
          )}

          <button type="submit" disabled={loading}
            className="w-full py-3 bg-black text-white font-medium rounded-full hover:bg-black/80 transition-colors disabled:opacity-60">
            {loading ? '处理中...' : tab === 'login' ? '登录' : '创建账号'}
          </button>
        </form>

        {/* ===== 游客体验入口 ===== */}
        <div className="relative my-6">
          <div className="absolute inset-0 flex items-center">
            <div className="w-full border-t border-black/10" />
          </div>
          <div className="relative flex justify-center text-xs">
            <span className="bg-white px-3 text-black/30">或者</span>
          </div>
        </div>

        <button
          type="button"
          onClick={() => { guestSignIn(); navigate(from, { replace: true }); }}
          className="w-full py-3 border border-black/20 text-black font-medium rounded-full hover:border-black/50 hover:bg-black/[0.02] transition-colors flex items-center justify-center gap-2"
        >
          <Sparkles size={16} />
          游客模式体验
        </button>

        {/* GitHub OAuth（可选） */}
        {isConfigured && (
          <>
            <div className="flex items-center gap-3 my-6">
              <div className="flex-1 h-px bg-black/10" />
              <span className="text-xs text-black/30">或</span>
              <div className="flex-1 h-px bg-black/10" />
            </div>
            <button
              onClick={() => useAuth().signInWithGitHub?.()}
              className="w-full py-3 border border-black rounded-full text-sm font-medium text-black hover:bg-black hover:text-white transition-colors flex items-center justify-center gap-2"
            >
              GitHub 登录
            </button>
          </>
        )}
      </div>
    </div>
  );
}
```

### `src/components/ProtectedRoute.tsx`

```tsx
import { Navigate, useLocation } from 'react-router-dom';
import { useAuth } from '@/hooks/useAuth';

export function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { user, loading } = useAuth();
  const location = useLocation();

  if (loading) return <div className="min-h-screen flex items-center justify-center">...</div>;
  if (!user) return <Navigate to="/login" state={{ from: location.pathname }} replace />;

  return <>{children}</>;
}
```

### `src/App.tsx` 路由配置

```tsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { AuthProvider } from '@/hooks/useAuth';
import { CartProvider } from '@/hooks/useCart';
import { HomePage } from '@/pages/HomePage';
import { LoginPage } from '@/pages/LoginPage';
import { ProfilePage } from '@/pages/ProfilePage';
import { CartPage } from '@/pages/CartPage';
import { CheckoutPage } from '@/pages/CheckoutPage';

export function App() {
  return (
    <BrowserRouter>
      <AuthProvider>
        <CartProvider>
          <Routes>
            <Route path="/" element={<HomePage />} />
            <Route path="/login" element={<LoginPage />} />
            <Route path="/profile" element={<ProfilePage />} />
            <Route path="/cart" element={<CartPage />} />
            <Route path="/checkout" element={<CheckoutPage />} />
          </Routes>
        </CartProvider>
      </AuthProvider>
    </BrowserRouter>
  );
}
```

> ⚠️ NavBar 和 Footer 应在各个页面内部渲染（HomePage 自带），非全局包裹。

---

## ⚠️ 踩坑经验（实际验证过的）

### 1. signOut 必须清除所有状态
```typescript
// ❌ 错误：只清 localStorage，NavBar 不刷新
const signOut = async () => {
  localStorage.removeItem(GUEST_STORAGE_KEY);
  setIsGuest(false);
};

// ✅ 正确：必须同时 setUser(null) + setSession(null)
const signOut = async () => {
  localStorage.removeItem(GUEST_STORAGE_KEY);
  setIsGuest(false);
  setUser(null);
  setSession(null);
  if (isSupabaseConfigured) await supabase.auth.signOut();
};
```

### 2. NavBar 判断用户不能用 isConfigured && user
```typescript
// ❌ 错误：游客 isConfigured=false，永远不显示头像
{isConfigured && user ? <UserAvatar /> : <LoginButton />}

// ✅ 正确：只判断 user，游客和正式用户统一处理
{user ? <UserAvatar /> : <LoginButton />}
```

### 3. 个人中心用户信息卡片不要 overflow-hidden
```tsx
// ❌ 错误：-mt-12 的头像被 overflow-hidden 裁掉
<div className="border border-black rounded-2xl overflow-hidden">
  <div className="h-28 bg-gradient-to-r ..." />
  <div className="-mt-12">...</div>
</div>

// ✅ 正确：去掉 overflow-hidden，渐变背景用 rounded-t-2xl
<div className="border border-black rounded-2xl">
  <div className="h-28 bg-gradient-to-r ... rounded-t-2xl" />
  <div className="-mt-12 relative z-10">...</div>
</div>
```

### 4. supabase 未配置时不能 throw Error
```typescript
// ❌ 错误：未配 .env 直接白屏
if (!supabaseUrl || !supabaseAnon) throw new Error('Missing env');

// ✅ 正确：优雅降级，游客模式可用
export const isSupabaseConfigured = !!(supabaseUrl && supabaseAnon);
export const supabase = isSupabaseConfigured ? createClient(...) : null as never;
```

### 5. JSX 条件渲染内不能有多个根元素
```tsx
// ❌ 错误：{!isGuest && (<button>...</button><div>...</div>)}  → PARSE_ERROR
// ✅ 正确：包裹一个 div
{!isGuest && (<div><button>...</button><div>...</div></div>)}
```

---

## 常见错误处理

| 错误 | 原因 | 解决 |
|------|------|------|
| `Invalid API key` | anon key 填写错误 | 检查 `.env.local` 中的 `VITE_SUPABASE_ANON_KEY` |
| `Email not confirmed` | 邮件未验证就登录 | 告知用户先验证邮件，或在 Supabase 控制台关闭邮件验证（仅开发环境） |
| `User already registered` | 邮箱已注册 | 提示切换到登录 Tab |
| CORS 错误 | Supabase URL 配置问题 | 检查 `VITE_SUPABASE_URL` 是否正确（不要有结尾 `/`） |
| 退出后 NavBar 仍显示头像 | signOut 未调 setUser(null) | 参见踩坑经验 #1 |
| 游客登录后 NavBar 不变 | NavBar 用了 `isConfigured && user` | 参见踩坑经验 #2 |
| 个人中心头像被渐变裁切 | overflow-hidden + 负 margin 冲突 | 参见踩坑经验 #3 |
