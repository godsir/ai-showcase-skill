# Edge Functions + KV Storage 接入指南

本文档说明如何在 AI 聚合展示平台中集成 EdgeOne Edge Functions 和 KV 持久化存储。

---

## 架构说明

```
浏览器 → EdgeOne CDN → Edge Function (V8 运行时)
                              ↕
                       KV Storage（全球边缘持久化）
```

**为什么用 Edge Functions？**
- V8 运行时，冷启动 <1ms，无需服务器
- 与 KV Storage 原生集成，天然支持边缘计算
- 全球部署，访问统计数据在边缘节点实时聚合

---

## 项目目录

```
project-root/
└── functions/
    └── api/
        └── stats.js    # 访问量统计接口
```

> EdgeOne Pages 会自动将 `functions/` 下的文件映射为 `/api/*` 路径的边缘函数。

---

## `functions/api/stats.js` 完整实现

```javascript
/**
 * EdgeOne Edge Function — 页面访问量统计
 * 路由：GET/POST /api/stats
 * 
 * GET  → 返回当前 pageviews 数量
 * POST → 访问量 +1，返回更新后的数量
 * 
 * KV 命名空间：AI_SHOWCASE_KV（在控制台创建后绑定）
 */
export async function onRequest({ request, env }) {
  const kv = env.AI_SHOWCASE_KV;

  // CORS 响应头
  const corsHeaders = {
    'Content-Type': 'application/json',
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
    'Cache-Control': 'no-store',
  };

  // OPTIONS 预检
  if (request.method === 'OPTIONS') {
    return new Response(null, { status: 204, headers: corsHeaders });
  }

  try {
    if (request.method === 'POST') {
      // 原子递增访问量（读取 → +1 → 写回）
      const raw = await kv.get('pageviews');
      const count = raw ? parseInt(raw, 10) + 1 : 1;
      await kv.put('pageviews', String(count));

      return new Response(
        JSON.stringify({ pageviews: count, updated: true }),
        { status: 200, headers: corsHeaders }
      );
    }

    if (request.method === 'GET') {
      const raw = await kv.get('pageviews');
      const count = raw ? parseInt(raw, 10) : 0;

      return new Response(
        JSON.stringify({ pageviews: count }),
        { status: 200, headers: corsHeaders }
      );
    }

    return new Response(
      JSON.stringify({ error: 'Method Not Allowed' }),
      { status: 405, headers: corsHeaders }
    );

  } catch (err) {
    // KV 不可用时（如本地开发未关联项目），返回降级数据
    return new Response(
      JSON.stringify({ pageviews: 0, fallback: true }),
      { status: 200, headers: corsHeaders }
    );
  }
}
```

---

## 前端调用（`src/hooks/useStats.ts`）

```typescript
import { useEffect, useState } from 'react';

export function useStats() {
  const [pageviews, setPageviews] = useState<number | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // 上报访问 + 获取最新统计
    fetch('/api/stats', { method: 'POST' })
      .then((res) => {
        if (!res.ok) throw new Error('stats fetch failed');
        return res.json();
      })
      .then((data) => {
        setPageviews(data.pageviews);
      })
      .catch(() => {
        // 失败时降级为只读
        fetch('/api/stats')
          .then((r) => r.json())
          .then((d) => setPageviews(d.pageviews))
          .catch(() => setPageviews(null));
      })
      .finally(() => setLoading(false));
  }, []);

  return { pageviews, loading };
}
```

---

## KV Storage 配置步骤

### Step 1 — 在控制台创建 KV 命名空间

1. 登录 [EdgeOne Pages 控制台](https://console.cloud.tencent.com/edgeone/pages)
2. 进入项目 → **存储** → **KV 存储**
3. 点击 **新建 KV 命名空间**
4. 名称填写：`AI_SHOWCASE_KV`
5. 点击确认创建

### Step 2 — 绑定到项目

```bash
# 关联本地项目与线上项目
edgeone pages link

# 拉取环境变量（包含 KV 绑定信息）到本地
edgeone pages env pull
```

执行后，本地 `.env` 文件会自动包含 KV 相关绑定，`edgeone pages dev` 时可直接调用。

### Step 3 — 本地验证

```bash
# 启动本地完整开发环境（包含 Edge Functions）
edgeone pages dev
# → http://localhost:8088

# 测试 KV 读写
curl -X POST http://localhost:8088/api/stats
# 预期：{"pageviews":1,"updated":true}

curl http://localhost:8088/api/stats
# 预期：{"pageviews":1}
```

> ⚠️ **注意**：如果未执行 `edgeone pages link`，`env.AI_SHOWCASE_KV` 将为 `undefined`，
> 此时 `stats.js` 中的 `catch` 分支会自动返回 `{ pageviews: 0, fallback: true }`，
> 不影响前端渲染，只是统计数字不实时。

---

## 运行时限制说明（Edge Functions）

| 限制项 | 值 | 备注 |
|--------|-----|------|
| 运行时 | V8（非 Node.js） | 不可用 `fs`, `path`, `crypto` 等 Node 内置模块 |
| 最大代码体积 | 5 MB | |
| 最大请求体 | 1 MB | |
| 最大 CPU 时间 | 200ms | KV 操作耗时计入 |
| KV 单次读写 | 512 KB | value 最大 512KB |
| KV 每日写入 | 无限额（免费账户有上限） | |

---

## 扩展能力（可选接入）

除页面统计外，可扩展以下 KV 用途：

```javascript
// 功能点赞数统计
await kv.put(`likes:${moduleId}`, String(count));
const likes = await kv.get(`likes:image-gen`);

// 最近访问时间戳记录（用于显示"X 秒前有人访问"）
await kv.put('last_visit', new Date().toISOString());

// A/B 测试流量分配标记
await kv.put(`ab:hero_variant`, 'variant_b');
```
