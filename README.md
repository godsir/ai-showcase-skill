# AI 聚合展示平台 Skill（v2.0 全栈版）

> 🏆 WorkBuddy × Tencent EdgeOne AI Prompts × Skills 挑战赛参赛作品

## 一句话描述

一句话触发，帮你在 **EdgeOne Pages** 上搭建并部署一个**酷炫科技风 AI 聚合全栈平台** ——  
集成 AI 展示 + 用户登录注册 + 在线商城 + Stripe 支付 + 订单管理，真实接入 EdgeOne Edge Functions + Cloud Functions + KV Storage。

---

## 效果预览

```
┌─────────────────────────────────────────────────┐
│  NavBar：Logo + Products/Shop/Demo + 登录状态    │
│           + 购物车图标（数量徽章）               │
├─────────────────────────────────────────────────┤
│  HeroSection：超大渐变标题 + Slogan + 光晕装饰   │
├─────────────────────────────────────────────────┤
│  ProductsSection：4 大 AI 模块卡片               │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐          │
│  │ 生图 │ │ 3D  │ │ LLM │ │视频 │          │
│  └──────┘ └──────┘ └──────┘ └──────┘          │
├─────────────────────────────────────────────────┤
│  DemoSection（黑底）：Tab 切换 Mock 交互          │
├─────────────────────────────────────────────────┤
│  PricingSection：3 档套餐定价卡片 + 购买按钮     │
│  ┌──────────┐ ┌══════════╗ ┌──────────┐        │
│  │  $9.9   │ ║  $29.9  ║ │  $99.9  │        │
│  │ 入门版  │ ║  专业版  ║ │  企业版  │        │
│  └──────────┘ └══════════╝ └──────────┘        │
├─────────────────────────────────────────────────┤
│  StatsSection（黑底）：KV 实时访问量统计          │
├─────────────────────────────────────────────────┤
│  Footer（黑底）：版权 + 友链 + EdgeOne 标注      │
└─────────────────────────────────────────────────┘

独立路由页面：
  /login         登录/注册（Supabase Auth）
  /shop          AI 能力商城
  /cart          购物车 + 结算
  /orders        我的订单（需登录）
  /checkout/success  支付成功
```

---

## 安装方式

```bash
npx skills add TencentEdgeOne/awesome-website-prompts-and-skills
```

触发词示例：
- **"帮我做一个带登录和支付的 AI 展示平台"**
- **"AI 聚合平台，要能买套餐、有订单管理"**
- **"build an AI showcase with shop and payment"**

---

## 技术栈

| 层 | 技术 |
|----|------|
| 前端框架 | React 18 + Vite + TypeScript |
| 样式 / 动效 | Tailwind CSS + Framer Motion |
| 路由 | React Router DOM |
| 状态管理 | Zustand（购物车持久化） |
| 用户认证 | Supabase Auth（邮箱/密码 + GitHub OAuth） |
| 数据库 | Supabase PostgreSQL（orders 表 + RLS） |
| 在线支付 | Stripe Checkout + Webhook |
| 边缘计算 | EdgeOne Edge Functions（V8，KV 访问统计） |
| 服务端函数 | EdgeOne Cloud Functions（Node.js，支付 + 订单 API） |
| 部署 | EdgeOne Pages（全球 CDN + 自动 HTTPS） |

---

## 文件结构

```
skills/ai-showcase-platform/
├── SKILL.md                          # 主入口
└── references/
    ├── project-structure.md          # 目录树 + 关键文件规范
    ├── design-tokens.md              # 完整 CSS 变量表
    ├── component-guide.md            # 展示层组件完整代码
    ├── mock-interactions.md          # Demo 区 Mock 交互实现
    ├── edge-functions-kv.md          # EdgeOne Edge Functions + KV
    ├── auth-guide.md                 # Supabase Auth + 登录页面
    ├── payment-guide.md              # Stripe 支付 + Cloud Functions
    └── shop-guide.md                 # 商城 + 购物车 + 订单页面
```

---

## 前置依赖

- Node.js ≥ 16
- EdgeOne Pages 账号（[注册](https://edgeone.ai)）
- Supabase 账号（[注册](https://supabase.com)，免费）
- Stripe 账号（[注册](https://stripe.com)，测试模式免费）
- WorkBuddy + `edgeone-pages-deploy` Skill（用于部署）

---

## License

MIT © 2025
