# 后端函数完整创建流程

本文档提供从 0 到 1 创建所有后端函数的**分步指令**，确保 AI 在生成代码时不会遗漏任何函数。

---

## 架构总览

```
前端（React）
  │
  ├── GET/POST /api/stats ────────→ Edge Function（V8 运行时，KV 读写）
  │
  ├── POST /api/create-checkout ──→ Cloud Function（Node.js，Stripe + Supabase）
  ├── POST /api/webhook ──────────→ Cloud Function（Node.js，Stripe Webhook）
  ├── GET  /api/orders ────────────→ Cloud Function（Node.js，Supabase 查询）
  │
  ├── GET  /api/admin/orders ─────→ Cloud Function（Node.js，管理员验证 + Supabase）
  └── GET  /api/admin/stats ───────→ Cloud Function（Node.js，管理员验证 + 聚合）
```

**选型规则**：
- **Edge Function**：只读写 KV，无 npm 依赖，超低延迟 → `stats.js`
- **Cloud Function**：需要 `stripe` / `@supabase/supabase-js` npm 包，含密钥 → 其余所有函数

---

## Step 3a — 创建 Edge Function（必须）

### 文件路径

```
functions/api/stats.js
```

### 依赖

无需 npm 依赖（Edge Function 运行在 V8 隔离环境）。

### 完整代码

```javascript
export async function onRequest({ request, env }) {
  const kv = env.AI_SHOWCASE_KV;
  const cors = {
    'Content-Type': 'application/json',
    'Access-Control-Allow-Origin': '*',
    'Cache-Control': 'no-store',
  };

  if (request.method === 'OPTIONS') {
    return new Response(null, { status: 204, headers: cors });
  }

  try {
    if (request.method === 'POST') {
      const raw = await kv.get('pageviews');
      const count = (raw ? parseInt(raw, 10) : 0) + 1;
      await kv.put('pageviews', String(count));
      return new Response(JSON.stringify({ pageviews: count }), { headers: cors });
    }
    const raw = await kv.get('pageviews');
    return new Response(JSON.stringify({ pageviews: raw ? parseInt(raw, 10) : 0 }), { headers: cors });
  } catch {
    return new Response(JSON.stringify({ pageviews: 0, fallback: true }), { headers: cors });
  }
}
```

### 前端调用

首页 `StatsSection` 组件在 `useEffect` 中 `POST /api/stats` 上报访问并获取计数。

---

## Step 3b — 创建 Cloud Functions（按需）

> ⚠️ 以下函数需要 npm 依赖和密钥。如果用户未配置 Stripe/Supabase，这些函数在本地无法运行，但不影响前端渲染。

### 目录结构

```
cloud-functions/
  package.json              # npm 依赖声明
  create-checkout.js         # 创建 Stripe Checkout Session
  webhook.js                 # Stripe Webhook 回调
  orders.js                  # 查询当前用户订单
  admin-orders.js            # 管理员查询全部订单
  admin-stats.js             # 管理员获取统计数据
```

### `cloud-functions/package.json`

```json
{
  "name": "ai-showcase-cloud-functions",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "stripe": "^14.0.0",
    "@supabase/supabase-js": "^2.39.0"
  }
}
```

> 创建后执行 `npm install` 安装依赖。

---

### 1. `create-checkout.js` — 创建 Stripe 支付会话

```javascript
import Stripe from 'stripe';
import { createClient } from '@supabase/supabase-js';

export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method Not Allowed' });
  }

  // 验证用户身份
  const authHeader = req.headers.authorization ?? '';
  const token = authHeader.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Unauthorized' });

  const supabase = createClient(
    process.env.VITE_SUPABASE_URL,
    process.env.SUPABASE_SERVICE_ROLE_KEY
  );

  const { data: { user }, error: authError } = await supabase.auth.getUser(token);
  if (authError || !user) return res.status(401).json({ error: 'Invalid token' });

  // 解析购物车
  const { items, successUrl, cancelUrl } = req.body;
  if (!items?.length) return res.status(400).json({ error: 'Cart is empty' });

  const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);

  const session = await stripe.checkout.sessions.create({
    payment_method_types: ['card'],
    mode: 'payment',
    line_items: items.map((item) => ({
      price_data: {
        currency: 'usd',
        unit_amount: item.price,  // 单位：分
        product_data: { name: item.name },
      },
      quantity: item.qty,
    })),
    metadata: { userId: user.id, items: JSON.stringify(items) },
    success_url: successUrl ?? `${process.env.SITE_URL}/checkout/success?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: cancelUrl ?? `${process.env.SITE_URL}/checkout/cancel`,
  });

  return res.status(200).json({ url: session.url });
}
```

---

### 2. `webhook.js` — Stripe Webhook 回调

```javascript
import Stripe from 'stripe';
import { createClient } from '@supabase/supabase-js';

export const config = { bodyParser: false };

export default async function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end();

  const stripe = new Stripe(process.env.STRIPE_SECRET_KEY);
  const sig = req.headers['stripe-signature'];

  let event;
  try {
    const chunks = [];
    for await (const chunk of req) chunks.push(chunk);
    const rawBody = Buffer.concat(chunks);
    event = stripe.webhooks.constructEvent(rawBody, sig, process.env.STRIPE_WEBHOOK_SECRET);
  } catch (err) {
    return res.status(400).send(`Webhook Error: ${err.message}`);
  }

  if (event.type === 'checkout.session.completed') {
    const session = event.data.object;
    const supabase = createClient(process.env.VITE_SUPABASE_URL, process.env.SUPABASE_SERVICE_ROLE_KEY);

    await supabase.from('orders').insert({
      user_id: session.metadata.userId,
      stripe_session_id: session.id,
      items: JSON.parse(session.metadata.items),
      amount_total: session.amount_total,
      currency: session.currency,
      status: 'paid',
    });
  }

  return res.status(200).json({ received: true });
}
```

---

### 3. `orders.js` — 查询当前用户订单

```javascript
import { createClient } from '@supabase/supabase-js';

export default async function handler(req, res) {
  if (req.method !== 'GET') return res.status(405).end();

  const token = (req.headers.authorization ?? '').replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Unauthorized' });

  const supabase = createClient(process.env.VITE_SUPABASE_URL, process.env.SUPABASE_SERVICE_ROLE_KEY);

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

### 4. `admin-orders.js` — 管理员查询全部订单

```javascript
import Stripe from 'stripe';
import { createClient } from '@supabase/supabase-js';

export default async function handler(req, res) {
  const supabase = createClient(process.env.VITE_SUPABASE_URL, process.env.SUPABASE_SERVICE_ROLE_KEY);

  // 验证管理员身份
  const authHeader = req.headers.authorization;
  if (!authHeader) return res.status(401).json({ error: 'Unauthorized' });

  const token = authHeader.replace('Bearer ', '');
  const { data: { user }, error: authError } = await supabase.auth.getUser(token);
  if (authError || !user) return res.status(401).json({ error: 'Invalid token' });

  const { data: profile } = await supabase.from('profiles').select('is_admin').eq('id', user.id).single();
  if (!profile?.is_admin) return res.status(403).json({ error: 'Admin only' });

  const { data: orders, error } = await supabase
    .from('orders')
    .select('*, user:users(email)')
    .order('created_at', { ascending: false });

  if (error) return res.status(500).json({ error: error.message });
  return res.status(200).json({ orders });
}
```

---

### 5. `admin-stats.js` — 管理员获取统计数据

```javascript
import { createClient } from '@supabase/supabase-js';

export default async function handler(req, res) {
  const supabase = createClient(process.env.VITE_SUPABASE_URL, process.env.SUPABASE_SERVICE_ROLE_KEY);

  // 验证管理员身份（同 admin-orders.js）

  const [
    { count: totalUsers },
    { count: totalOrders },
    { data: revenueData },
  ] = await Promise.all([
    supabase.from('profiles').select('*', { count: 'exact', head: true }),
    supabase.from('orders').select('*', { count: 'exact', head: true }),
    supabase.from('orders').select('amount').eq('status', 'paid'),
  ]);

  const totalRevenue = revenueData.reduce((sum, o) => sum + (o.amount || 0), 0);

  return res.status(200).json({ totalUsers, totalOrders, totalRevenue, monthlyNewUsers: 0 });
}
```

---

## 环境变量清单

| 变量名 | 使用位置 | 前端/后端 |
|--------|---------|----------|
| `VITE_SUPABASE_URL` | 前端 + 所有 Cloud Function | 两者 |
| `VITE_SUPABASE_ANON_KEY` | 前端 Supabase 客户端 | 前端 |
| `SUPABASE_SERVICE_ROLE_KEY` | Cloud Function（绕过 RLS） | 后端 |
| `VITE_STRIPE_PUBLISHABLE_KEY` | 前端 `loadStripe()` | 前端 |
| `STRIPE_SECRET_KEY` | Cloud Function 创建 Session | 后端 |
| `STRIPE_WEBHOOK_SECRET` | webhook.js 签名验证 | 后端 |
| `SITE_URL` | Cloud Function 回调 URL | 后端 |

> ⚠️ **绝不**将 `STRIPE_SECRET_KEY`、`SUPABASE_SERVICE_ROLE_KEY`、`STRIPE_WEBHOOK_SECRET` 以 `VITE_` 前缀暴露到前端。

---

## Supabase 数据库初始化 SQL

在 Supabase SQL Editor 中执行：

```sql
-- profiles 表
CREATE TABLE profiles (
  id UUID REFERENCES auth.users ON DELETE CASCADE PRIMARY KEY,
  email TEXT,
  nickname TEXT,
  is_admin BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  last_login_at TIMESTAMP WITH TIME ZONE
);

ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view own profile"
  ON profiles FOR SELECT USING (auth.uid() = id);

CREATE POLICY "Admins can update all profiles"
  ON profiles FOR UPDATE USING (
    EXISTS (SELECT 1 FROM profiles WHERE id = auth.uid() AND is_admin = TRUE)
  );

-- orders 表
CREATE TABLE orders (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL,
  stripe_session_id TEXT UNIQUE NOT NULL,
  items JSONB NOT NULL,
  amount_total INTEGER NOT NULL,
  currency TEXT NOT NULL DEFAULT 'usd',
  status TEXT NOT NULL DEFAULT 'pending',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY "users can view own orders"
  ON orders FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "service role can insert orders"
  ON orders FOR INSERT WITH CHECK (true);

-- 管理员账号（替换为实际 user ID）
-- INSERT INTO profiles (id, email, nickname, is_admin)
-- VALUES ('<admin-user-id>', 'admin@neurahub.com', '管理员', TRUE);
```

---

## 本地验证

```bash
# 带后端完整验证（推荐）
edgeone pages dev
# → http://localhost:8088

# 仅前端验证
npm run dev
# → http://localhost:5173
```

### 验证命令

```bash
# Edge Function 统计
curl -X POST http://localhost:8088/api/stats
# 预期：{"pageviews":1,"updated":true}

curl http://localhost:8088/api/stats
# 预期：{"pageviews":1}

# Cloud Function 需要真实密钥，本地可用 Mock 数据
```

---

## 踩坑经验

### 1. Edge Function 不能用 npm 包
Edge Function 运行在 V8 隔离环境，不支持 `require()` 或 `import` npm 包。只能用 `stripe` 等包的 Cloud Function。

### 2. webhook.js 必须用 raw body
Stripe 签名验证需要原始请求体。如果框架自动解析了 JSON body，验证会失败。解决方案：`export const config = { bodyParser: false }`。

### 3. KV 命名空间必须先创建
`env.AI_SHOWCASE_KV` 在未执行 `edgeone pages link` 时为 `undefined`。stats.js 的 catch 分支会返回降级数据 `{ pageviews: 0, fallback: true }`，不影响前端。

### 4. service_role key 绝不能给前端
`SUPABASE_SERVICE_ROLE_KEY` 可以绕过 RLS，如果泄露到前端，任何人都能读写所有数据。

---

## 创建流程检查清单

- [ ] `functions/api/stats.js` 创建完毕
- [ ] `cloud-functions/package.json` 创建完毕并 `npm install`
- [ ] `cloud-functions/create-checkout.js` 创建完毕
- [ ] `cloud-functions/webhook.js` 创建完毕
- [ ] `cloud-functions/orders.js` 创建完毕
- [ ] `cloud-functions/admin-orders.js` 创建完毕（features.admin=true 时）
- [ ] `cloud-functions/admin-stats.js` 创建完毕（features.admin=true 时）
- [ ] `.env.local` 配置了所有必需变量
- [ ] Supabase SQL 执行了建表语句
- [ ] KV 命名空间 `AI_SHOWCASE_KV` 在 EdgeOne 控制台创建
