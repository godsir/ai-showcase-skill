# 用户认证方案（Supabase Auth）

## 架构概览

```
前端（React）
  ↓ supabase-js（邮箱/密码 or OAuth）
Supabase Auth
  ↓ 返回 JWT access_token
前端存储（localStorage / supabase 内置）
  ↓ 请求 Cloud Function 时携带 Authorization: Bearer <token>
EdgeOne Cloud Function
  ↓ 用 SUPABASE_SERVICE_ROLE_KEY 验证 token
  ↓ 执行业务逻辑（查订单、写订单等）
```

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

if (!supabaseUrl || !supabaseAnon) {
  throw new Error('Missing Supabase env vars. Check .env.local');
}

export const supabase = createClient(supabaseUrl, supabaseAnon);
```

### `src/hooks/useAuth.ts`

```typescript
import { useEffect, useState } from 'react';
import type { User, Session } from '@supabase/supabase-js';
import { supabase } from '@/lib/supabase';

export function useAuth() {
  const [user, setUser] = useState<User | null>(null);
  const [session, setSession] = useState<Session | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // 初始读取 session
    supabase.auth.getSession().then(({ data }) => {
      setSession(data.session);
      setUser(data.session?.user ?? null);
      setLoading(false);
    });

    // 监听登录/退出事件
    const { data: { subscription } } = supabase.auth.onAuthStateChange((_event, session) => {
      setSession(session);
      setUser(session?.user ?? null);
    });

    return () => subscription.unsubscribe();
  }, []);

  const signIn = (email: string, password: string) =>
    supabase.auth.signInWithPassword({ email, password });

  const signUp = (email: string, password: string) =>
    supabase.auth.signUp({ email, password });

  const signOut = () => supabase.auth.signOut();

  const signInWithGitHub = () =>
    supabase.auth.signInWithOAuth({ provider: 'github' });

  return { user, session, loading, signIn, signUp, signOut, signInWithGitHub };
}
```

### `src/pages/LoginPage.tsx`

```tsx
import { useState } from 'react';
import { useNavigate, useLocation } from 'react-router-dom';
import { useAuth } from '@/hooks/useAuth';
import { cn } from '@/lib/cn';

type Tab = 'login' | 'register';

export function LoginPage() {
  const { signIn, signUp } = useAuth();
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
        {/* 标题 */}
        <h1 className="text-3xl font-bold text-black mb-8 text-center">
          {tab === 'login' ? '欢迎回来' : '创建账号'}
        </h1>

        {/* Tab 切换 */}
        <div className="flex border border-black rounded-full p-1 mb-8">
          {(['login', 'register'] as Tab[]).map((t) => (
            <button
              key={t}
              onClick={() => { setTab(t); setError(''); }}
              className={cn(
                'flex-1 py-2 text-sm font-medium rounded-full transition-all duration-200',
                tab === t ? 'bg-black text-white' : 'text-black/50 hover:text-black'
              )}
            >
              {t === 'login' ? '登录' : '注册'}
            </button>
          ))}
        </div>

        {/* 表单 */}
        <form onSubmit={handleSubmit} className="space-y-4">
          <div>
            <label className="block text-sm font-medium text-black mb-1.5">邮箱</label>
            <input
              type="email"
              required
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              placeholder="you@example.com"
              className="w-full border border-black rounded-xl px-4 py-3 text-sm focus:outline-none focus:ring-2 focus:ring-black/20"
            />
          </div>

          <div>
            <label className="block text-sm font-medium text-black mb-1.5">密码</label>
            <input
              type="password"
              required
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              placeholder="••••••••"
              className="w-full border border-black rounded-xl px-4 py-3 text-sm focus:outline-none focus:ring-2 focus:ring-black/20"
            />
          </div>

          {tab === 'register' && (
            <div>
              <label className="block text-sm font-medium text-black mb-1.5">确认密码</label>
              <input
                type="password"
                required
                value={confirm}
                onChange={(e) => setConfirm(e.target.value)}
                placeholder="••••••••"
                className={cn(
                  'w-full border rounded-xl px-4 py-3 text-sm focus:outline-none focus:ring-2',
                  confirm && password !== confirm
                    ? 'border-red-500 focus:ring-red-200'
                    : 'border-black focus:ring-black/20'
                )}
              />
            </div>
          )}

          {error && (
            <p className={cn(
              'text-sm px-3 py-2 rounded-lg',
              error.includes('成功') ? 'bg-green-50 text-green-700' : 'bg-red-50 text-red-600'
            )}>
              {error}
            </p>
          )}

          <button
            type="submit"
            disabled={loading}
            className="w-full py-3 bg-black text-white font-medium rounded-full hover:bg-black/80 transition-colors disabled:opacity-60"
          >
            {loading ? '处理中...' : tab === 'login' ? '登录' : '创建账号'}
          </button>
        </form>

        {/* 分隔线 */}
        <div className="flex items-center gap-3 my-6">
          <div className="flex-1 h-px bg-black/10" />
          <span className="text-xs text-black/30">或</span>
          <div className="flex-1 h-px bg-black/10" />
        </div>

        {/* GitHub OAuth */}
        <button
          onClick={() => useAuth().signInWithGitHub()}
          className="w-full py-3 border border-black rounded-full text-sm font-medium text-black hover:bg-black hover:text-white transition-colors flex items-center justify-center gap-2"
        >
          <span>GitHub 登录</span>
        </button>
      </div>
    </div>
  );
}
```

### 路由守卫 `src/components/ProtectedRoute.tsx`

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
import { NavBar } from '@/components/layout/NavBar';
import { Footer } from '@/components/layout/Footer';
import { HomePage } from '@/pages/HomePage';
import { LoginPage } from '@/pages/LoginPage';
import { ShopPage } from '@/pages/ShopPage';
import { CartPage } from '@/pages/CartPage';
import { OrdersPage } from '@/pages/OrdersPage';
import { CheckoutSuccessPage } from '@/pages/CheckoutSuccessPage';
import { CheckoutCancelPage } from '@/pages/CheckoutCancelPage';
import { ProtectedRoute } from '@/components/ProtectedRoute';

export function App() {
  return (
    <BrowserRouter>
      <NavBar />
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route path="/login" element={<LoginPage />} />
        <Route path="/shop" element={<ShopPage />} />
        <Route path="/cart" element={<CartPage />} />
        <Route path="/checkout/success" element={<CheckoutSuccessPage />} />
        <Route path="/checkout/cancel" element={<CheckoutCancelPage />} />
        <Route path="/orders" element={
          <ProtectedRoute><OrdersPage /></ProtectedRoute>
        } />
      </Routes>
      <Footer />
    </BrowserRouter>
  );
}
```

---

## 常见错误处理

| 错误 | 原因 | 解决 |
|------|------|------|
| `Invalid API key` | anon key 填写错误 | 检查 `.env.local` 中的 `VITE_SUPABASE_ANON_KEY` |
| `Email not confirmed` | 邮件未验证就登录 | 告知用户先验证邮件，或在 Supabase 控制台关闭邮件验证（仅开发环境） |
| `User already registered` | 邮箱已注册 | 提示切换到登录 Tab |
| CORS 错误 | Supabase URL 配置问题 | 检查 `VITE_SUPABASE_URL` 是否正确（不要有结尾 `/`） |
