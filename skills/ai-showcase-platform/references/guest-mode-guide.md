# 游客体验模式规范（Guest Mode）

## 设计理念

游客模式让访客无需注册、无需配置 Supabase，即可体验平台全部功能（浏览、购物车、结算、个人中心）。目的是**预览网站功能完整性**，不产生真实交易。

---

## 架构

```
点击「游客模式体验」
  ↓
guestSignIn() → 创建 mock User → 存入 localStorage
  ↓
AuthContext setUser(mockUser) + setIsGuest(true)
  ↓
NavBar 显示用户头像 + 购物车图标
  ↓
所有页面（购物车 / 结算 / 个人中心）正常可用
  ↓
signOut() → 清除 localStorage + setUser(null) → 回到未登录状态
```

---

## Mock User 数据结构

```typescript
interface GuestUser {
  id: 'guest-demo-001';
  email: 'demo@mindforge.ai';
  aud: 'authenticated';
  role: 'authenticated';
  app_metadata: {};
  user_metadata: { name: '体验用户' };
  created_at: string; // 创建时间
  identities: [];
}
```

存储 key：`mindforge_guest_user`，值为 `{ created_at, email }` 的 JSON。

---

## 登录页「游客模式体验」按钮

位置：在登录/注册表单下方，用分隔线隔开。

```tsx
{/* 分隔线 */}
<div className="relative my-6">
  <div className="absolute inset-0 flex items-center">
    <div className="w-full border-t border-black/10" />
  </div>
  <div className="relative flex justify-center text-xs">
    <span className="bg-white px-3 text-black/30">或者</span>
  </div>
</div>

{/* 游客体验按钮 */}
<button
  type="button"
  onClick={() => { guestSignIn(); navigate(from, { replace: true }); }}
  className="w-full py-3 border border-black/20 text-black font-medium rounded-full
    hover:border-black/50 hover:bg-black/[0.02] transition-colors
    flex items-center justify-center gap-2"
>
  <Sparkles size={16} />
  游客模式体验
</button>
```

**设计要点**：
- 样式用 `border border-black/20`（非实心），与主登录按钮视觉区分
- 图标用 `Sparkles`（lucide-react），传达"快速体验"的感觉
- 点击后直接跳转首页，无需任何验证

---

## NavBar 适配

### 登录状态判断（关键！）

```tsx
// ✅ 正确写法：只判断 user
const { user } = useAuth();

// 渲染逻辑
{user ? (
  <UserAvatar />   // 头像 + 下拉菜单
) : (
  <LoginButton />  // 登录/注册按钮
)}
```

```tsx
// ❌ 错误写法：isConfigured && user
// 游客 isConfigured=false，永远不会显示头像！
{isConfigured && user ? <UserAvatar /> : <LoginButton />}
```

### 退出按钮文案

```tsx
<button onClick={handleSignOut}>
  {isGuest ? '退出体验' : '退出登录'}
</button>
```

---

## 购物车页面适配

- 未登录 → 显示"请先登录"引导
- 游客/正式用户 → 正常显示购物车
- 游客结算时底部提示：`💡 这是体验模式，结算不会产生真实扣费`

---

## 结算页面适配

### 支付模拟流程

```
点击「确认支付」
  ↓
processing = true → 显示 Loader 动画
  ↓
Mock 延迟 2 秒（setTimeout）
  ↓
checkout() → 创建 Mock 订单（status='active'）
  ↓
显示「订阅成功！」页面
  ↓
提供「查看我的订阅」和「返回首页」两个入口
```

### 游客特有提示

在支付按钮下方显示：
```
💡 体验模式：点击确认支付将模拟完成订阅，不会产生真实扣费
```

---

## 个人中心适配（ProfilePage）

### 三 Tab 布局

| Tab | 图标 | 游客行为 |
|-----|------|---------|
| 账号信息 | User | 显示 mock 信息，隐藏"修改密码" |
| 我的订单 | Package | 显示 Mock 订单（或空状态） |
| 使用量 | BarChart3 | 显示 Mock 用量数据 |

### 用户信息头部

- 用户名旁显示紫色渐变「体验账号」标签：

```tsx
{isGuest && (
  <span className="inline-flex items-center gap-1 px-2.5 py-0.5 text-xs font-medium
    bg-gradient-to-r from-[#00ff88]/15 to-[#6600ff]/15 text-[#6600ff]
    rounded-full border border-[#6600ff]/20">
    <Sparkles size={10} />
    体验账号
  </span>
)}
```

- 渐变背景 + 头像的布局：

```tsx
{/* ⚠️ 不要用 overflow-hidden！ */}
<div className="border border-black rounded-2xl">
  <div className="h-28 bg-gradient-to-r from-[#00ff88] via-[#6600ff] to-[#ff00aa] opacity-80 rounded-t-2xl" />
  <div className="px-8 pb-6 -mt-12 relative z-10">
    <div className="w-24 h-24 rounded-full bg-black border-4 border-white flex items-center justify-center shadow-lg">
      <span className="text-3xl font-bold text-white">{user?.email?.[0]?.toUpperCase()}</span>
    </div>
    {/* 用户名 + 标签 + 邮箱 */}
  </div>
</div>
```

### 修改密码区域

游客模式下**完全隐藏**修改密码功能：

```tsx
{!isGuest && (
  <div>
    <button onClick={() => setShowPasswordSection(!showPasswordSection)}>修改密码</button>
    {showPasswordSection && <form>...</form>}
  </div>
)}
```

### Mock 使用量数据

```typescript
const usageData = [
  { icon: ImageIcon, label: '图片生成', used: 23, limit: 100, color: '#00ff88' },
  { icon: MessageSquare, label: '大模型对话', used: 156, limit: 1000, color: '#ffee00' },
  { icon: Code2, label: '代码生成', used: 8, limit: 50, color: '#ff6600' },
];
```

---

## 移除「认证服务未配置」阻断页

旧版 ProfilePage 在 `!isConfigured` 时显示阻断页，阻止游客访问个人中心。

**必须移除此阻断**。游客模式下 `isConfigured=false`，但 `user` 有值，个人中心应正常渲染。

```tsx
// ❌ 旧版：未配置直接阻断
if (!loading && !isConfigured) {
  return <div>认证服务未配置</div>;
}

// ✅ 新版：只判断 user
if (!loading && !user) {
  return <div>请先登录</div>;
}
```

---

## signOut 必须做的操作

```typescript
const signOut = async () => {
  // 1. 清除游客 localStorage
  localStorage.removeItem(GUEST_STORAGE_KEY);
  // 2. 重置游客标记
  setIsGuest(false);
  // 3. ⚠️ 清除 user（必须！否则 NavBar 不刷新）
  setUser(null);
  // 4. 清除 session
  setSession(null);
  // 5. 如果 Supabase 已配置，也登出正式用户
  if (isSupabaseConfigured) {
    await supabase.auth.signOut();
  }
};
```

> ⚠️ **最常见的 bug**：忘记 `setUser(null)`，导致退出后 NavBar 仍显示用户头像。
