# 购物车 + 商城 + 订单页面规范

## 商品数据模型（前端静态配置）

```typescript
// src/lib/products.ts
export interface Product {
  id: string;
  name: string;
  nameEn: string;
  description: string;
  features: string[];     // 功能列表（用于 PricingSection 的 ✓ 逐条展示）
  price: number;          // 单位：美元，如 9.9
  priceDisplay: string;   // 显示文本，如 "$9.9/mo"
  themeColor: string;     // accent 颜色
  popular: boolean;       // 是否为推荐套餐（边框加粗）
  badge?: string;         // 可选角标文字，如 "最受欢迎"
}

export const PRODUCTS: Product[] = [
  {
    id: 'starter',
    name: '入门套餐',
    nameEn: 'Starter',
    description: '适合个人探索，体验核心 AI 能力',
    features: [
      '生图：每月 100 张',
      '大模型对话：每月 1,000 次',
      '标准响应速度',
      '邮件支持',
    ],
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
    features: [
      '生图：每月 1,000 张',
      '3D 模型生成：每月 50 个',
      '视频生成：每月 20 段',
      '大模型对话：无限次',
      '优先响应速度',
      '7×24 在线支持',
    ],
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
    features: [
      '全模块无限使用',
      'API 访问权限',
      '专属客户经理',
      'SLA 99.9% 保障',
      '私有化部署咨询',
      '团队协作工作台',
    ],
    price: 99.9,
    priceDisplay: '$99.9/mo',
    themeColor: '#ff00aa',
    popular: false,
  },
];
```

---

## 全局购物车状态（Zustand）

```typescript
// src/lib/store.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import type { Product } from './products';

export interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface CartStore {
  items: CartItem[];
  addItem: (product: Product) => void;
  removeItem: (id: string) => void;
  updateQty: (id: string, qty: number) => void;
  clearCart: () => void;
  total: () => number;
  count: () => number;
}

export const useCartStore = create<CartStore>()(
  persist(
    (set, get) => ({
      items: [],

      addItem: (product) => set((state) => {
        const existing = state.items.find((i) => i.id === product.id);
        if (existing) {
          return {
            items: state.items.map((i) =>
              i.id === product.id ? { ...i, quantity: i.quantity + 1 } : i
            ),
          };
        }
        return {
          items: [...state.items, {
            id: product.id,
            name: product.name,
            price: product.price,
            quantity: 1,
          }],
        };
      }),

      removeItem: (id) => set((state) => ({
        items: state.items.filter((i) => i.id !== id),
      })),

      updateQty: (id, qty) => set((state) => ({
        items: qty <= 0
          ? state.items.filter((i) => i.id !== id)
          : state.items.map((i) => i.id === id ? { ...i, quantity: qty } : i),
      })),

      clearCart: () => set({ items: [] }),

      total: () => get().items.reduce((sum, i) => sum + i.price * i.quantity, 0),

      count: () => get().items.reduce((sum, i) => sum + i.quantity, 0),
    }),
    { name: 'ai-showcase-cart' }
  )
);
```

---

## PricingSection 组件（首页定价区）

```tsx
// src/components/sections/PricingSection.tsx
import { motion } from 'framer-motion';
import { Check } from 'lucide-react';
import { PRODUCTS } from '@/lib/products';
import { useCartStore } from '@/lib/store';
import { useAuth } from '@/hooks/useAuth';
import { useNavigate } from 'react-router-dom';
import { cn } from '@/lib/cn';

export function PricingSection() {
  const { user } = useAuth();
  const addItem = useCartStore((s) => s.addItem);
  const navigate = useNavigate();

  const handleBuy = (product: typeof PRODUCTS[0]) => {
    if (!user) {
      navigate('/login', { state: { from: '/shop' } });
      return;
    }
    addItem(product);
    navigate('/cart');
  };

  return (
    <section id="pricing" className="py-20 px-6 bg-white">
      <div className="max-w-6xl mx-auto">
        <motion.div
          initial={{ opacity: 0, y: 20 }}
          whileInView={{ opacity: 1, y: 0 }}
          viewport={{ once: true }}
          className="text-center mb-14"
        >
          <h2 className="text-4xl font-bold text-black mb-3">选择适合你的套餐</h2>
          <p className="text-black/50 text-lg">按月订阅，随时取消</p>
        </motion.div>

        <div className="grid grid-cols-1 md:grid-cols-3 gap-8">
          {PRODUCTS.map((product, i) => (
            <motion.div
              key={product.id}
              initial={{ opacity: 0, y: 30 }}
              whileInView={{ opacity: 1, y: 0 }}
              viewport={{ once: true }}
              transition={{ delay: i * 0.1 }}
              className={cn(
                'relative bg-white rounded-2xl p-8 flex flex-col',
                product.popular
                  ? 'border-2 border-black shadow-lg'
                  : 'border border-black/20'
              )}
            >
              {/* 推荐角标 */}
              {product.badge && (
                <div
                  className="absolute -top-3.5 left-1/2 -translate-x-1/2 px-4 py-1 text-xs font-bold rounded-full text-black"
                  style={{ background: product.themeColor }}
                >
                  {product.badge}
                </div>
              )}

              {/* Accent bar */}
              <div
                className="h-1 w-12 rounded-full mb-6"
                style={{ background: product.themeColor }}
              />

              {/* 套餐名 */}
              <h3 className="text-xl font-bold text-black mb-1">{product.name}</h3>
              <p className="text-sm text-black/40 mb-6">{product.description}</p>

              {/* 价格 */}
              <div className="mb-8">
                <span className="text-5xl font-bold text-black">${product.price}</span>
                <span className="text-black/40 text-sm ml-1">/月</span>
              </div>

              {/* 功能列表 */}
              <ul className="space-y-3 mb-8 flex-1">
                {product.features.map((feat) => (
                  <li key={feat} className="flex items-start gap-2.5 text-sm text-black/70">
                    <Check size={14} className="mt-0.5 flex-shrink-0 text-black" />
                    {feat}
                  </li>
                ))}
              </ul>

              {/* 购买按钮 */}
              <button
                onClick={() => handleBuy(product)}
                className={cn(
                  'w-full py-3 font-medium rounded-full transition-all duration-200',
                  product.popular
                    ? 'bg-black text-white hover:bg-black/80'
                    : 'border border-black text-black hover:bg-black hover:text-white'
                )}
              >
                {user ? '加入购物车' : '立即购买'}
              </button>
            </motion.div>
          ))}
        </div>
      </div>
    </section>
  );
}
```

---

## ShopPage（`/shop`）

```tsx
// src/pages/ShopPage.tsx
import { motion } from 'framer-motion';
import { ShoppingCart } from 'lucide-react';
import { PRODUCTS } from '@/lib/products';
import { useCartStore } from '@/lib/store';
import { useAuth } from '@/hooks/useAuth';
import { useNavigate } from 'react-router-dom';

export function ShopPage() {
  const { user } = useAuth();
  const addItem = useCartStore((s) => s.addItem);
  const navigate = useNavigate();

  const handleAdd = (product: typeof PRODUCTS[0]) => {
    if (!user) { navigate('/login', { state: { from: '/shop' } }); return; }
    addItem(product);
  };

  return (
    <div className="min-h-screen bg-white pt-20 pb-16 px-6">
      <div className="max-w-6xl mx-auto">
        <h1 className="text-4xl font-bold text-black mb-2">AI 能力商城</h1>
        <p className="text-black/50 mb-12">选购适合你的 AI 能力套餐，即买即用</p>

        <div className="grid grid-cols-1 md:grid-cols-3 gap-8">
          {PRODUCTS.map((p, i) => (
            <motion.div
              key={p.id}
              initial={{ opacity: 0, y: 24 }}
              animate={{ opacity: 1, y: 0 }}
              transition={{ delay: i * 0.1 }}
              className="border border-black rounded-2xl overflow-hidden"
            >
              {/* 封面占位 */}
              <div
                className="h-40 flex items-center justify-center text-4xl"
                style={{ background: p.themeColor + '18' }}
              >
                {i === 0 ? '🌱' : i === 1 ? '⚡' : '🏢'}
              </div>

              <div className="p-6">
                <div className="flex items-start justify-between mb-3">
                  <div>
                    <h3 className="font-bold text-black text-lg">{p.name}</h3>
                    <p className="text-sm text-black/40">{p.nameEn}</p>
                  </div>
                  <span className="text-xl font-bold text-black">${p.price}</span>
                </div>
                <p className="text-sm text-black/60 mb-5">{p.description}</p>
                <button
                  onClick={() => handleAdd(p)}
                  className="w-full py-2.5 flex items-center justify-center gap-2 border border-black rounded-full text-sm font-medium hover:bg-black hover:text-white transition-colors"
                >
                  <ShoppingCart size={14} />
                  加入购物车
                </button>
              </div>
            </motion.div>
          ))}
        </div>
      </div>
    </div>
  );
}
```

---

## CartPage（`/cart`）

```tsx
// src/pages/CartPage.tsx
import { Minus, Plus, Trash2 } from 'lucide-react';
import { useCartStore } from '@/lib/store';
import { useCheckout } from '@/hooks/useCheckout';
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';

const TAX_RATE = 0.1;

export function CartPage() {
  const { items, removeItem, updateQty, total } = useCartStore();
  const { startCheckout } = useCheckout();
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState('');
  const navigate = useNavigate();

  const subtotal = total();
  const tax = subtotal * TAX_RATE;
  const grandTotal = subtotal + tax;

  const handleCheckout = async () => {
    setError('');
    setLoading(true);
    try {
      await startCheckout(items);
    } catch (err: unknown) {
      setError(err instanceof Error ? err.message : '结算失败，请重试');
    } finally {
      setLoading(false);
    }
  };

  if (items.length === 0) {
    return (
      <div className="min-h-screen bg-white pt-20 flex items-center justify-center">
        <div className="text-center">
          <div className="text-5xl mb-4">🛒</div>
          <p className="text-black/50 text-lg mb-6">购物车是空的</p>
          <button
            onClick={() => navigate('/shop')}
            className="px-6 py-3 bg-black text-white rounded-full font-medium hover:bg-black/80 transition-colors"
          >
            去选购
          </button>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-white pt-20 pb-16 px-6">
      <div className="max-w-4xl mx-auto">
        <h1 className="text-3xl font-bold text-black mb-8">购物车</h1>

        <div className="flex flex-col lg:flex-row gap-8">
          {/* 商品列表 */}
          <div className="flex-1 space-y-4">
            {items.map((item) => (
              <div
                key={item.id}
                className="flex items-center gap-4 border border-black/10 rounded-xl p-4"
              >
                <div className="flex-1">
                  <div className="font-medium text-black">{item.name}</div>
                  <div className="text-sm text-black/40">${item.price.toFixed(2)} / 月</div>
                </div>

                {/* 数量控制 */}
                <div className="flex items-center gap-2">
                  <button
                    onClick={() => updateQty(item.id, item.quantity - 1)}
                    className="w-7 h-7 rounded-full border border-black/20 flex items-center justify-center hover:bg-black hover:text-white hover:border-black transition-colors"
                  >
                    <Minus size={12} />
                  </button>
                  <span className="w-6 text-center text-sm font-medium">{item.quantity}</span>
                  <button
                    onClick={() => updateQty(item.id, item.quantity + 1)}
                    className="w-7 h-7 rounded-full border border-black/20 flex items-center justify-center hover:bg-black hover:text-white hover:border-black transition-colors"
                  >
                    <Plus size={12} />
                  </button>
                </div>

                {/* 小计 */}
                <div className="w-20 text-right font-medium">
                  ${(item.price * item.quantity).toFixed(2)}
                </div>

                {/* 删除 */}
                <button
                  onClick={() => removeItem(item.id)}
                  className="text-black/30 hover:text-black transition-colors"
                >
                  <Trash2 size={16} />
                </button>
              </div>
            ))}
          </div>

          {/* 汇总区 */}
          <div className="lg:w-72">
            <div className="border border-black rounded-2xl p-6 sticky top-20">
              <h2 className="font-bold text-black text-lg mb-4">订单摘要</h2>

              <div className="space-y-3 mb-4 text-sm">
                <div className="flex justify-between text-black/60">
                  <span>小计</span>
                  <span>${subtotal.toFixed(2)}</span>
                </div>
                <div className="flex justify-between text-black/60">
                  <span>税费（10%）</span>
                  <span>${tax.toFixed(2)}</span>
                </div>
                <div className="h-px bg-black/10" />
                <div className="flex justify-between font-bold text-black text-base">
                  <span>合计</span>
                  <span>${grandTotal.toFixed(2)}</span>
                </div>
              </div>

              {error && (
                <p className="text-sm text-red-600 bg-red-50 rounded-lg px-3 py-2 mb-4">{error}</p>
              )}

              <button
                onClick={handleCheckout}
                disabled={loading}
                className="w-full py-3 bg-black text-white font-medium rounded-full hover:bg-black/80 transition-colors disabled:opacity-60"
              >
                {loading ? '跳转支付中...' : '去结算 →'}
              </button>

              <p className="text-xs text-black/30 text-center mt-3">
                由 Stripe 安全支付保障
              </p>
            </div>
          </div>
        </div>
      </div>
    </div>
  );
}
```

---

## OrdersPage（`/orders`，需登录）

```tsx
// src/pages/OrdersPage.tsx
import { useEffect, useState } from 'react';
import { supabase } from '@/lib/supabase';

interface Order {
  id: string;
  stripe_session_id: string;
  items: { productId: string; name: string; price: number; qty: number }[];
  amount_total: number;
  currency: string;
  status: 'pending' | 'paid' | 'cancelled';
  created_at: string;
}

const STATUS_LABEL: Record<Order['status'], string> = {
  pending: '待支付',
  paid: '已完成',
  cancelled: '已取消',
};

const STATUS_COLOR: Record<Order['status'], string> = {
  pending: 'bg-yellow-50 text-yellow-700',
  paid: 'bg-green-50 text-green-700',
  cancelled: 'bg-red-50 text-red-600',
};

export function OrdersPage() {
  const [orders, setOrders] = useState<Order[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchOrders = async () => {
      const { data: { session } } = await supabase.auth.getSession();
      if (!session) return;

      const res = await fetch('/api/orders', {
        headers: { Authorization: `Bearer ${session.access_token}` },
      });
      if (res.ok) {
        const { orders } = await res.json();
        setOrders(orders);
      }
      setLoading(false);
    };
    fetchOrders();
  }, []);

  return (
    <div className="min-h-screen bg-white pt-20 pb-16 px-6">
      <div className="max-w-4xl mx-auto">
        <h1 className="text-3xl font-bold text-black mb-8">我的订单</h1>

        {loading ? (
          <div className="text-black/40 text-center py-20">加载中...</div>
        ) : orders.length === 0 ? (
          <div className="text-center py-20">
            <div className="text-4xl mb-4">📭</div>
            <p className="text-black/50">暂无订单记录</p>
          </div>
        ) : (
          <div className="space-y-4">
            {orders.map((order) => (
              <div key={order.id} className="border border-black/10 rounded-2xl p-6">
                <div className="flex items-start justify-between mb-4">
                  <div>
                    <div className="text-xs text-black/30 mb-1">订单号</div>
                    <div className="text-sm font-mono text-black/70 truncate max-w-xs">
                      {order.stripe_session_id}
                    </div>
                  </div>
                  <span className={`px-3 py-1 rounded-full text-xs font-medium ${STATUS_COLOR[order.status]}`}>
                    {STATUS_LABEL[order.status]}
                  </span>
                </div>

                {/* 商品 */}
                <div className="space-y-1.5 mb-4">
                  {order.items.map((item) => (
                    <div key={item.productId} className="flex justify-between text-sm">
                      <span className="text-black/70">{item.name} × {item.qty}</span>
                      <span className="text-black">${(item.price * item.qty).toFixed(2)}</span>
                    </div>
                  ))}
                </div>

                <div className="flex items-center justify-between pt-3 border-t border-black/10">
                  <span className="text-xs text-black/30">
                    {new Date(order.created_at).toLocaleString('zh-CN')}
                  </span>
                  <span className="font-bold text-black">
                    合计 ${(order.amount_total / 100).toFixed(2)} {order.currency.toUpperCase()}
                  </span>
                </div>
              </div>
            ))}
          </div>
        )}
      </div>
    </div>
  );
}
```

---

## 更新后的项目目录（新增文件）

```
src/
  pages/
    HomePage.tsx          # 首页（原 App.tsx 内容拆出）
    LoginPage.tsx         # 登录/注册（见 auth-guide.md）
    ShopPage.tsx          # 商城（本文件）
    CartPage.tsx          # 购物车（本文件）
    OrdersPage.tsx        # 订单（本文件，ProtectedRoute 保护）
    CheckoutSuccessPage.tsx
    CheckoutCancelPage.tsx
  components/
    ProtectedRoute.tsx    # 路由守卫（见 auth-guide.md）
    sections/
      PricingSection.tsx  # 定价区（本文件）
  lib/
    products.ts           # 商品静态配置
    store.ts              # Zustand 购物车 store
  hooks/
    useCheckout.ts        # Stripe 结算（见 payment-guide.md）
cloud-functions/
  create-checkout.js      # 创建 Stripe Session
  webhook.js              # Webhook 写订单
  orders.js               # 查询订单
```
