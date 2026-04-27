# 支付方案（Stripe Checkout + 订单流程）

## 整体支付流程

```
用户点击「去结算」
  ↓
前端 POST /api/create-checkout（携带购物车数据 + JWT）
  ↓
Cloud Function（Node.js）验证 JWT → 调用 Stripe API 创建 Checkout Session
  ↓
返回 { url: "https://checkout.stripe.com/..." }
  ↓
前端跳转到 Stripe 托管支付页
  ↓ 用户输入卡号完成支付
Stripe 回调 /checkout/success?session_id=xxx（浏览器跳转）
Stripe Webhook POST /api/webhook（服务端异步）
  ↓
Cloud Function 验证 Webhook 签名
  ↓ checkout.session.completed 事件
写入 orders 表（Supabase，status: 'paid'）
  ↓
订单写入完成
```

---

## Stripe 初始化

1. 注册 [https://stripe.com](https://stripe.com)，进入 **Dashboard → Developers → API Keys**
2. 复制：
   - `VITE_STRIPE_PUBLISHABLE_KEY`（前端可见，`pk_test_...`）
   - `STRIPE_SECRET_KEY`（仅服务端，`sk_test_...`，**绝不暴露前端**）
3. 创建 Webhook：Dashboard → Webhooks → Add endpoint
   - URL：`https://<your-domain>/api/webhook`
   - 监听事件：`checkout.session.completed`
   - 复制 **Webhook Secret**（`whsec_...`）→ 环境变量 `STRIPE_WEBHOOK_SECRET`

---

## Cloud Function 实现

### `cloud-functions/create-checkout.js`

```javascript
import Stripe from 'stripe';
import { createClient } from '@supabase/supabase-js';

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method Not Allowed' });
  }

  // ─── 验证用户身份 ───
  const authHeader = req.headers.authorization ?? '';
  const token = authHeader.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Unauthorized' });

  const supabase = createClient(
    process.env.VITE_SUPABASE_URL,
    process.env.SUPABASE_SERVICE_ROLE_KEY
  );

  const { data: { user }, error: authError } = await supabase.auth.getUser(token);
  if (authError || !user) return res.status(401).json({ error: 'Invalid token' });

  // ─── 解析购物车 ───
  const { items, successUrl, cancelUrl } = req.body;
  // items: [{ productId, name, price, qty }]  price 单位：分（cents）

  if (!items?.length) return res.status(400).json({ error: 'Cart is empty' });

  const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);

  // ─── 创建 Stripe Checkout Session ───
  const session = await stripe.checkout.sessions.create({
    payment_method_types: ['card'],
    mode: 'payment',
    line_items: items.map((item) => ({
      price_data: {
        currency: 'usd',
        unit_amount: item.price,      // 单位：分
        product_data: { name: item.name },
      },
      quantity: item.qty,
    })),
    metadata: {
      userId: user.id,
      items: JSON.stringify(items),
    },
    success_url: successUrl ?? `${process.env.SITE_URL}/checkout/success?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: cancelUrl ?? `${process.env.SITE_URL}/checkout/cancel`,
  });

  return res.status(200).json({ url: session.url });
}
```

### `cloud-functions/webhook.js`

```javascript
import Stripe from 'stripe';
import { createClient } from '@supabase/supabase-js';

// ⚠️ 关键：必须以 raw body（Buffer）接收，否则签名验证失败
export const config = { bodyParser: false };

export default async function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end();

  const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);
  const sig = req.headers['stripe-signature'];

  let event;
  try {
    // 收集原始 body
    const chunks = [];
    for await (const chunk of req) chunks.push(chunk);
    const rawBody = Buffer.concat(chunks);

    event = stripe.webhooks.constructEvent(
      rawBody,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET
    );
  } catch (err) {
    console.error('Webhook signature verification failed:', err.message);
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  // ─── 处理支付完成事件 ───
  if (event.type === 'checkout.session.completed') {
    const session = event.data.object;

    const supabase = createClient(
      process.env.VITE_SUPABASE_URL,
      process.env.SUPABASE_SERVICE_ROLE_KEY
    );

    const { error } = await supabase.from('orders').insert({
      user_id: session.metadata.userId,
      stripe_session_id: session.id,
      items: JSON.parse(session.metadata.items),
      amount_total: session.amount_total,   // 单位：分
      currency: session.currency,
      status: 'paid',
    });

    if (error) {
      console.error('Failed to insert order:', error);
      return res.status(500).json({ error: 'DB insert failed' });
    }
  }

  return res.status(200).json({ received: true });
}
```

### `cloud-functions/orders.js`

```javascript
import { createClient } from '@supabase/supabase-js';

export default async function handler(req, res) {
  if (req.method !== 'GET') return res.status(405).end();

  const token = (req.headers.authorization ?? '').replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Unauthorized' });

  const supabase = createClient(
    process.env.VITE_SUPABASE_URL,
    process.env.SUPABASE_SERVICE_ROLE_KEY
  );

  const { data: { user }, error: authErr } = await supabase.auth.getUser(token);
  if (authErr || !user) return res.status(401).json({ error: 'Invalid token' });

  const { data: orders, error } = await supabase
    .from('orders')
    .select('*')
    .eq('user_id', user.id)
    .order('created_at', { ascending: false });

  if (error) return res.status(500).json({ error: error.message });

  return res.status(200).json({ orders });
}
```

---

## 前端支付调用

### `src/hooks/useCheckout.ts`

```typescript
import { loadStripe } from '@stripe/stripe-js';
import { supabase } from '@/lib/supabase';
import type { CartItem } from '@/lib/store';

const stripePromise = loadStripe(import.meta.env.VITE_STRIPE_PUBLISHABLE_KEY);

export function useCheckout() {
  const startCheckout = async (items: CartItem[]) => {
    // 获取当前用户 JWT
    const { data: { session } } = await supabase.auth.getSession();
    if (!session) throw new Error('请先登录');

    const payload = items.map((item) => ({
      productId: item.id,
      name: item.name,
      price: Math.round(item.price * 100), // 转成分
      qty: item.quantity,
    }));

    const res = await fetch('/api/create-checkout', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${session.access_token}`,
      },
      body: JSON.stringify({ items: payload }),
    });

    if (!res.ok) {
      const { error } = await res.json();
      throw new Error(error ?? '支付服务异常');
    }

    const { url } = await res.json();
    window.location.href = url;  // 跳转到 Stripe 支付页
  };

  return { startCheckout };
}
```

---

## 支付成功/取消页面

### `src/pages/CheckoutSuccessPage.tsx`

```tsx
import { useEffect } from 'react';
import { Link } from 'react-router-dom';
import { useCartStore } from '@/lib/store';

export function CheckoutSuccessPage() {
  const clearCart = useCartStore((s) => s.clearCart);

  useEffect(() => {
    clearCart(); // 支付成功后清空购物车
  }, [clearCart]);

  return (
    <div className="min-h-screen bg-white flex items-center justify-center pt-16">
      <div className="text-center max-w-md px-6">
        <div className="text-6xl mb-6">✅</div>
        <h1 className="text-3xl font-bold text-black mb-3">支付成功！</h1>
        <p className="text-black/60 mb-8">
          你的 AI 能力套餐已激活，感谢你的购买。订单记录将在片刻后出现在订单页。
        </p>
        <div className="flex gap-4 justify-center">
          <Link
            to="/orders"
            className="px-6 py-3 bg-black text-white font-medium rounded-full hover:bg-black/80 transition-colors"
          >
            查看订单
          </Link>
          <Link
            to="/"
            className="px-6 py-3 border border-black text-black font-medium rounded-full hover:bg-black hover:text-white transition-colors"
          >
            返回首页
          </Link>
        </div>
      </div>
    </div>
  );
}
```

---

## 环境变量清单

| 变量名 | 使用位置 | 说明 |
|--------|---------|------|
| `VITE_STRIPE_PUBLISHABLE_KEY` | 前端 | Stripe 可公开密钥（`pk_test_...`） |
| `STRIPE_SECRET_KEY` | Cloud Function | Stripe 私密密钥（**仅服务端**） |
| `STRIPE_WEBHOOK_SECRET` | Cloud Function | Webhook 验签密钥（`whsec_...`） |
| `SITE_URL` | Cloud Function | 部署后域名，用于拼接回调 URL |
| `VITE_SUPABASE_URL` | 前端 + Cloud Function | Supabase 项目 URL |
| `VITE_SUPABASE_ANON_KEY` | 前端 | Supabase 匿名密钥 |
| `SUPABASE_SERVICE_ROLE_KEY` | Cloud Function | Supabase 服务密钥（**仅服务端**） |

> ⚠️ `STRIPE_SECRET_KEY`、`STRIPE_WEBHOOK_SECRET`、`SUPABASE_SERVICE_ROLE_KEY` **绝不能**以 `VITE_` 前缀暴露，否则会打包进客户端代码

---

## Stripe 测试卡号

| 场景 | 卡号 |
|------|------|
| 支付成功 | `4242 4242 4242 4242` |
| 支付失败 | `4000 0000 0000 0002` |
| 需要 3DS 验证 | `4000 0025 0000 3155` |

有效期：任意未来日期；CVV：任意 3 位；邮编：任意 5 位
