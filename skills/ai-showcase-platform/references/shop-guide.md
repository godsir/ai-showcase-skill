# 购物车 + 结算 + 订单 + 个人中心规范

## 架构概览

使用 **React Context**（非 Zustand）管理购物车和订单状态，数据持久化到 localStorage。

```
CartProvider（React Context）
  ├─ items: CartItem[]          → 购物车商品
  ├─ orders: Order[]            → 历史订单
  ├─ activePlan: Order | null   → 当前生效套餐
  ├─ addToCart / removeFromCart / clearCart
  ├─ cartTotal / cartCount
  ├─ checkout() → Order         → Mock 结算
  └─ cancelOrder(orderId)
```

> ⚠️ **为什么不用 Zustand**：React Context + localStorage 对本项目（纯前端 Mock、无复杂异步操作）足够，减少依赖。如需接入真实后端，可切换到 Zustand。

---

## 商品数据模型（前端静态配置）

```typescript
// src/lib/products.ts
export interface Product {
  id: string;
  name: string;
  nameEn: string;
  description: string;
  features: string[];
  price: number;
  priceDisplay: string;
  themeColor: string;
  popular: boolean;
  badge?: string;
}

export const PRODUCTS: Product[] = [
  {
    id: 'starter',
    name: '入门套餐',
    nameEn: 'Starter',
    description: '适合个人探索，体验核心 AI 能力',
    features: ['生图：每月 100 张', '大模型对话：每月 1,000 次', '标准响应速度', '邮件支持'],
    price: 9.9,
    priceDisplay: '$9.9/mo',
    themeColor: '#00ff88',
    popular: false,
  },
  {
    id: 'pro',
    name: '专业套餐',
    nameEn: 'Pro',
    description: '适合创作者和开发者，解锁全部模块',
    features: ['生图：每月 1,000 张', '3D 模型生成：每月 50 个', '视频生成：每月 20 段',
      '大模型对话：无限次', '优先响应速度', '7×24 在线支持'],
    price: 29.9,
    priceDisplay: '$29.9/mo',
    themeColor: '#6600ff',
    popular: true,
    badge: '最受欢迎',
  },
  {
    id: 'enterprise',
    name: '企业套餐',
    nameEn: 'Enterprise',
    description: '面向团队和企业，无限额度 + 专属服务',
    features: ['全模块无限使用', 'API 访问权限', '专属客户经理',
      'SLA 99.9% 保障', '私有化部署咨询', '团队协作工作台'],
    price: 99.9,
    priceDisplay: '$99.9/mo',
    themeColor: '#ff00aa',
    popular: false,
  },
];
```

---

## CartContext（`src/hooks/useCart.tsx`）

```typescript
import { createContext, useContext, useState, useCallback, useEffect, type ReactNode } from 'react';

export interface CartItem {
  planId: string;
  name: string;
  nameEn: string;
  price: number;
  themeColor: string;
  quantity: number;
}

export interface Order {
  id: string;
  items: CartItem[];
  total: number;
  status: 'pending' | 'paid' | 'active' | 'cancelled';
  createdAt: string;
  email: string;
}

interface CartContextType {
  items: CartItem[];
  orders: Order[];
  activePlan: Order | null;
  addToCart: (item: Omit<CartItem, 'quantity'>) => void;
  removeFromCart: (planId: string) => void;
  clearCart: () => void;
  cartTotal: number;
  cartCount: number;
  checkout: () => Order;
  cancelOrder: (orderId: string) => void;
}

const CART_KEY = 'mindforge_cart';
const ORDERS_KEY = 'mindforge_orders';

const CartContext = createContext<CartContextType | null>(null);

export function CartProvider({ children }: { children: ReactNode }) {
  const [items, setItems] = useState<CartItem[]>(() => {
    try { const raw = localStorage.getItem(CART_KEY); return raw ? JSON.parse(raw) : []; }
    catch { return []; }
  });

  const [orders, setOrders] = useState<Order[]>(() => {
    try { const raw = localStorage.getItem(ORDERS_KEY); return raw ? JSON.parse(raw) : []; }
    catch { return []; }
  });

  const activePlan = orders.find(o => o.status === 'active') ?? null;

  useEffect(() => { localStorage.setItem(CART_KEY, JSON.stringify(items)); }, [items]);
  useEffect(() => { localStorage.setItem(ORDERS_KEY, JSON.stringify(orders)); }, [orders]);

  const addToCart = useCallback((item: Omit<CartItem, 'quantity'>) => {
    setItems(prev => {
      if (prev.find(i => i.planId === item.planId)) return prev; // 套餐不重复添加
      return [...prev, { ...item, quantity: 1 }];
    });
  }, []);

  const removeFromCart = useCallback((planId: string) => {
    setItems(prev => prev.filter(i => i.planId !== planId));
  }, []);

  const clearCart = useCallback(() => setItems([]), []);

  const cartTotal = items.reduce((sum, i) => sum + i.price, 0);
  const cartCount = items.length;

  const checkout = useCallback((): Order => {
    if (items.length === 0) throw new Error('购物车为空');
    const order: Order = {
      id: `ORD-${Date.now().toString(36).toUpperCase()}`,
      items: [...items],
      total: cartTotal,
      status: 'active',
      createdAt: new Date().toISOString(),
      email: 'demo@mindforge.ai',
    };
    setOrders(prev => [
      ...prev.map(o => o.status === 'active' ? { ...o, status: 'cancelled' as const } : o),
      order,
    ]);
    setItems([]);
    return order;
  }, [items, cartTotal]);

  const cancelOrder = useCallback((orderId: string) => {
    setOrders(prev => prev.map(o =>
      o.id === orderId ? { ...o, status: 'cancelled' as const } : o
    ));
  }, []);

  return (
    <CartContext.Provider value={{
      items, orders, activePlan,
      addToCart, removeFromCart, clearCart,
      cartTotal, cartCount, checkout, cancelOrder,
    }}>
      {children}
    </CartContext.Provider>
  );
}

export function useCart() {
  const ctx = useContext(CartContext);
  if (!ctx) throw new Error('useCart must be used within CartProvider');
  return ctx;
}
```

---

## PricingSection 组件（首页定价区）

### 关键改动

- **「立即购买」→「加入购物车」**，带状态反馈（已添加 / 已在购物车）
- 未登录点击自动跳转 `/login`（保存来源路径）

```tsx
export function PricingSection() {
  const { user, isGuest } = useAuth();
  const { addToCart, items } = useCart();
  const navigate = useNavigate();
  const [addedId, setAddedId] = useState<string | null>(null);

  const handleAddToCart = (product: Product) => {
    if (!user) {
      navigate('/login', { state: { from: window.location.pathname } });
      return;
    }
    addToCart({
      planId: product.id,
      name: product.name,
      nameEn: product.nameEn,
      price: product.price,
      themeColor: product.themeColor,
    });
    setAddedId(product.id);
    setTimeout(() => setAddedId(null), 2000);
  };

  // 按钮渲染逻辑
  const isInCart = items.some(i => i.planId === product.id);
  const isAdded = addedId === product.id;

  <button onClick={() => handleAddToCart(product)}>
    {isAdded ? '✓ 已添加' : isInCart ? '已在购物车' : '加入购物车'}
  </button>
}
```

---

## CartPage（`/cart`）

### 功能要点

- 未登录 → 显示"请先登录"引导页
- 空购物车 → 显示空状态（图标 + 文案 + "浏览套餐"按钮）
- 有商品 → 列表 + 底部汇总

### 空状态设计

```tsx
<div className="text-center py-20">
  <div className="w-20 h-20 mx-auto mb-6 rounded-full bg-black/[0.03] flex items-center justify-center">
    <ShoppingBag size={36} className="text-black/20" />
  </div>
  <h3 className="text-xl font-bold text-black mb-2">购物车是空的</h3>
  <p className="text-black/40 text-sm mb-8">去挑选一个适合你的 AI 套餐吧</p>
  <Link to="/#pricing" className="...">浏览套餐</Link>
</div>
```

### 商品列表项

每个商品卡片带色条标识、名称、价格、删除按钮：

```tsx
<div key={item.planId} className="flex items-center gap-4 p-5 border border-black/10 rounded-2xl">
  <div className="w-1.5 h-14 rounded-full flex-shrink-0" style={{ background: item.themeColor }} />
  <div className="flex-1">
    <h3 className="text-base font-bold text-black">{item.name}</h3>
    <p className="text-xs text-black/40">{item.nameEn}</p>
  </div>
  <div className="text-right">
    <div className="text-lg font-bold">${item.price}</div>
    <div className="text-xs text-black/40">/月</div>
  </div>
  <button onClick={() => removeFromCart(item.planId)}>
    <Trash2 size={16} />
  </button>
</div>
```

### 底部汇总区

```tsx
<div className="border border-black rounded-2xl p-6">
  <div className="flex justify-between mb-6">
    <span className="text-black/50">小计</span>
    <span className="text-xl font-bold">${cartTotal.toFixed(2)}</span>
  </div>
  <button onClick={() => navigate('/checkout')} className="w-full py-3.5 bg-black text-white font-medium rounded-full">
    立即结算
  </button>
  {isGuest && <p className="text-xs text-center text-black/30 mt-4">💡 这是体验模式，结算不会产生真实扣费</p>}
</div>
```

---

## CheckoutPage（`/checkout`）

### 页面流程

```
进入结算页
  ├─ 未登录 → 跳转 /login
  ├─ 购物车空 → 跳转 /cart
  └─ 正常 → 显示订单确认
       ↓
  点击「确认支付」
       ↓
  processing = true → Loader 动画（2 秒）
       ↓
  checkout() → 创建 Mock 订单
       ↓
  显示「订阅成功！」
       ↓
  提供「查看我的订阅」和「返回首页」
```

### 订单确认区域

```tsx
<div className="border border-black rounded-2xl overflow-hidden mb-6">
  <div className="px-6 py-4 bg-black/[0.03] border-b border-black/10">
    <h3 className="text-sm font-medium text-black/50">订阅详情</h3>
  </div>
  {items.map(item => (
    <div key={item.planId} className="flex items-center gap-4 px-6 py-4 border-b border-black/5 last:border-b-0">
      <div className="w-1.5 h-10 rounded-full" style={{ background: item.themeColor }} />
      <div className="flex-1">
        <p className="font-medium text-sm">{item.name}</p>
        <p className="text-xs text-black/40">{item.nameEn} · 按月订阅</p>
      </div>
      <p className="font-bold">${item.price}<span className="text-xs text-black/40">/月</span></p>
    </div>
  ))}
</div>
```

### 优惠码（占位 UI）

```tsx
<div className="flex gap-3 mb-6">
  <input placeholder="输入优惠码" className="flex-1 border border-black/15 rounded-xl px-4 py-3 text-sm" />
  <button className="px-5 py-3 border border-black text-sm font-medium rounded-xl">应用</button>
</div>
```

### 支付方式选择

```tsx
<div className="space-y-3">
  {['微信支付', '支付宝', '信用卡 / 借记卡'].map((method, i) => (
    <label key={method} className="flex items-center gap-3 p-3 border border-black/10 rounded-xl cursor-pointer">
      <input type="radio" name="payment" defaultChecked={i === 0} className="accent-black" />
      <span className="text-sm">{method}</span>
      {i === 0 && <span className="ml-auto text-xs text-green-600 font-medium">推荐</span>}
    </label>
  ))}
</div>
```

### 成功页面

```tsx
<div className="text-center">
  <div className="w-20 h-20 mx-auto mb-6 rounded-full bg-green-50 flex items-center justify-center">
    <Check size={40} className="text-green-600" />
  </div>
  <h1 className="text-3xl font-bold mb-3">订阅成功！</h1>
  <p className="text-black/50 text-sm mb-2">订单号：{orderId}</p>
  <p className="text-black/40 text-sm mb-8">
    {isGuest ? '这是体验模式的模拟订单' : '你的 AI 套餐已激活'}
  </p>
  <div className="flex gap-3 justify-center">
    <Link to="/profile">查看我的订阅</Link>
    <Link to="/">返回首页</Link>
  </div>
</div>
```

---

## ProfilePage（`/profile`）— 个人中心

### 三 Tab 布局

```tsx
const tabs: { id: Tab; label: string; icon: typeof User }[] = [
  { id: 'info', label: '账号信息', icon: User },
  { id: 'orders', label: '我的订单', icon: Package },
  { id: 'usage', label: '使用量', icon: BarChart3 },
];
```

### Tab 1：账号信息

- 用户头部（渐变背景 + 头像 + 名称 + 标签 + 邮箱）
- 信息卡片网格（邮箱、注册时间、上次登录）
- 修改密码（游客隐藏）
- 退出按钮（文案区分游客/正式用户）

### Tab 2：我的订单

```tsx
const STATUS_MAP = {
  active: { label: '生效中', color: 'bg-green-50 text-green-700' },
  paid: { label: '已完成', color: 'bg-green-50 text-green-700' },
  pending: { label: '待支付', color: 'bg-yellow-50 text-yellow-700' },
  cancelled: { label: '已取消', color: 'bg-red-50 text-red-600' },
};

{orders.length === 0 ? (
  <div className="text-center py-16">
    <Package size={40} className="mx-auto text-black/20 mb-4" />
    <p className="text-black/40">暂无订单记录</p>
    <Link to="/#pricing">浏览套餐</Link>
  </div>
) : (
  orders.map(order => (
    <div key={order.id} className="border border-black/10 rounded-2xl p-5">
      {/* 订单号 + 状态 */}
      {/* 商品列表 */}
      {/* 总价 + 日期 + 取消订阅按钮（active 订单） */}
    </div>
  ))
)}
```

### Tab 3：使用量

```tsx
const usageData = [
  { icon: ImageIcon, label: '图片生成', used: 67, limit: 1000, color: '#00ff88' },
  { icon: MessageSquare, label: '大模型对话', used: 432, limit: 99999, color: '#ffee00', unlimited: true },
  { icon: Code2, label: '代码生成', used: 29, limit: 99999, color: '#ff6600', unlimited: true },
];

// 进度条组件
<div className="space-y-6">
  {usageData.map(item => (
    <div key={item.label}>
      <div className="flex justify-between text-sm mb-2">
        <span className="flex items-center gap-2">
          <item.icon size={14} style={{ color: item.color }} />
          {item.label}
        </span>
        <span className="text-black/50">
          {item.unlimited ? '无限' : `${item.used} / ${item.limit}`}
        </span>
      </div>
      <div className="h-2 bg-black/5 rounded-full overflow-hidden">
        <div
          className="h-full rounded-full transition-all duration-500"
          style={{
            width: `${item.unlimited ? 15 : Math.min((item.used / item.limit) * 100, 100)}%`,
            background: item.color,
          }}
        />
      </div>
    </div>
  ))}
</div>
```

---

## NavBar 购物车图标

### 图标 + 数量角标

```tsx
import { ShoppingCart } from 'lucide-react';
import { useCart } from '@/hooks/useCart';

const { cartCount } = useCart();

<Link to="/cart" className="relative p-2 text-black/60 hover:text-black transition-colors">
  <ShoppingCart size={20} />
  {cartCount > 0 && (
    <span className="absolute -top-0.5 -right-0.5 w-4.5 h-4.5 bg-black text-white text-[10px] font-bold rounded-full flex items-center justify-center">
      {cartCount}
    </span>
  )}
</Link>
```

### 下拉菜单（已登录）

```tsx
{/* 用户下拉菜单 */}
{user ? (
  <div className="relative group">
    <button className="w-9 h-9 rounded-full bg-black text-white text-sm font-bold flex items-center justify-center">
      {user.email?.[0]?.toUpperCase()}
    </button>
    <div className="absolute right-0 mt-2 w-48 bg-white border border-black/10 rounded-xl shadow-lg opacity-0 invisible group-hover:opacity-100 group-hover:visible transition-all">
      <Link to="/profile" className="block px-4 py-3 text-sm hover:bg-black/[0.03]">个人中心</Link>
      <Link to="/cart" className="block px-4 py-3 text-sm hover:bg-black/[0.03]">购物车</Link>
      <button onClick={handleSignOut} className="block w-full text-left px-4 py-3 text-sm text-red-500 hover:bg-red-50">
        {isGuest ? '退出体验' : '退出登录'}
      </button>
    </div>
  </div>
) : (
  <Link to="/login" className="px-5 py-2 bg-black text-white text-sm font-medium rounded-full">
    登录 / 注册
  </Link>
)}
```

---

## 更新后的项目目录

```
src/
  pages/
    HomePage.tsx          # 首页
    LoginPage.tsx         # 登录/注册 + 游客体验入口
    ProfilePage.tsx       # 个人中心（三 Tab）
    CartPage.tsx          # 购物车
    CheckoutPage.tsx      # 结算页
    ShopPage.tsx          # 商城（可选）
    AIChatPage.tsx        # AI 助手（可选）
  hooks/
    useAuth.tsx           # AuthContext + 游客模式
    useCart.tsx           # CartContext + OrderContext
  components/
    ProtectedRoute.tsx    # 路由守卫
    sections/
      PricingSection.tsx  # 定价区（加入购物车）
  lib/
    products.ts           # 商品静态配置
    cn.ts                 # clsx + tailwind-merge
    supabase.ts           # Supabase 客户端
```
