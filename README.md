# AI 聚合展示平台 Skill（v3.1 全栈增强版）

> 🏆 WorkBuddy × Tencent EdgeOne AI Prompts × Skills 挑战赛参赛作品

## 一句话描述

一句话触发，帮你在 **EdgeOne Pages** 上搭建并部署一个**酷炫科技风 AI 聚合全栈平台** ——  
集成 AI 展示 + 用户登录注册（Supabase Auth + 游客模式）+ AI 助手多模型对话 + 在线商城 + Stripe 支付 + 订单管理 + 管理后台，真实接入 EdgeOne Edge Functions + Cloud Functions + KV Storage。

---

## 效果预览

```
┌─────────────────────────────────────────────────┐
│  NavBar：Logo + Products/AI助手/Shop/Demo      │
│           + 登录状态 + 购物车图标（数量徽章）   │
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
│  AIPreviewSection：AI 助手预览卡片（可选）       │
├─────────────────────────────────────────────────┤
│  PricingSection：3 档套餐定价卡片 + 加入购物车   │
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
  /login         登录/注册（Supabase Auth + 游客模式）
  /profile       个人中心（账号/订单/使用量 三 Tab）
  /shop          AI 能力商城
  /cart          购物车
  /checkout      结算页（Mock 支付）
  /ai-chat       AI 助手对话（GPT-4/Claude/Gemini/Llama）

管理后台（可选，需 is_admin）：
  /admin              数据概览仪表盘
  /admin/orders       订单管理
  /admin/users        用户管理
  /admin/products     套餐管理
  /admin/analytics    数据分析
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
- **"AI 平台加个 AI 助手和管理后台"**

---

## 核心特性

### 🎨 展示层
- 4 大 AI 模块展示卡片（生图 / 3D 模型 / 大模型对话 / 视频生成）
- Mock 交互 Demo（生图输入框 / Canvas 线框 / 聊天气泡 / 进度条动画）
- 白底黑字科技渐变风格，Framer Motion 动效

### 🤖 AI 助手（可选）
- 多模型选择：GPT-4 / Claude 3 / Gemini / Llama 3
- 聊天对话界面（渐变气泡 + 三点跳动加载动画）
- Mock 响应逻辑，可对接真实 AI API

### 🔐 认证系统
- Supabase Auth（邮箱/密码 + GitHub OAuth）
- **游客体验模式**：未配置 Supabase 时以 Mock 身份访问所有页面
- ProtectedRoute 路由守卫 + Admin 权限控制

### 🛒 电商系统
- React Context 购物车 + 订单管理
- 3 档套餐定价 + Stripe Checkout 支付
- 个人中心（账号信息 / 我的订单 / 使用量）

### 📊 管理后台（可选）
- 5 个子页面：仪表盘 / 订单 / 用户 / 套餐 / 数据分析
- 侧边栏布局 + 权限控制 + 管理员演示入口

### ⚡ 后端函数
- Edge Function（V8）：KV 访问量统计，超低延迟
- Cloud Functions（Node.js）：Stripe 支付 + 订单 API + 管理员统计

---

## 技术栈

| 层 | 技术 |
|----|------|
| 前端框架 | React 18 + Vite + TypeScript |
| 样式 / 动效 | Tailwind CSS v4 + Framer Motion |
| 路由 | React Router DOM |
| 状态管理 | React Context（购物车 + 认证） |
| 用户认证 | Supabase Auth + 游客模式（localStorage） |
| 数据库 | Supabase PostgreSQL（profiles + orders 表 + RLS） |
| 在线支付 | Stripe Checkout + Webhook |
| 边缘计算 | EdgeOne Edge Functions（V8，KV 访问统计） |
| 服务端函数 | EdgeOne Cloud Functions（Node.js，支付 + 订单 + 管理员 API） |
| 部署 | EdgeOne Pages（全球 CDN + 自动 HTTPS） |

---

## 文件结构

```
skills/ai-showcase-platform/
├── SKILL.md                          # 主入口（v3.1，6 步创建流程）
└── references/
    ├── project-structure.md          # 目录树 + 关键文件规范
    ├── design-tokens.md              # 完整 CSS 变量表
    ├── component-guide.md            # 展示层组件完整代码
    ├── mock-interactions.md          # Demo 区 Mock 交互实现
    ├── edge-functions-kv.md          # EdgeOne Edge Functions + KV
    ├── auth-guide.md                 # Supabase Auth + 游客模式 + 登录页面
    ├── guest-mode-guide.md           # 游客体验模式完整规范
    ├── payment-guide.md              # Stripe 支付 + Cloud Functions
    ├── shop-guide.md                 # 商城 + 购物车 + 结算 + 个人中心
    ├── ai-chat-guide.md              # AI 助手对话页面规范
    ├── admin-guide.md                # 管理后台规范（5 个子页面）
    └── backend-guide.md              # 后端函数完整创建流程
```

---

## 创建流程（Step 2 拆分）

Skill 的 Step 2 代码生成阶段被拆为 **6 个独立子步骤**，确保每个模块不被遗漏：

| 步骤 | 内容 | 必须 |
|------|------|------|
| 2a | 基础架构（Context / Hooks / Routes） | ✅ |
| 2b | 展示层组件（NavBar / Hero / Demo / Pricing / Stats） | ✅ |
| 2c | 认证与个人中心（Login / Profile） | ✅ |
| 2d | 购物与结算（Cart / Checkout） | ✅ |
| 2e | AI 助手对话页面（features.aiChat=true） | 按需 |
| 2f | 管理后台 5 个子页面（features.admin=true） | 按需 |

Step 3 后端函数同样拆为 3a（Edge Function）+ 3b（Cloud Functions），含创建检查清单。

---

## 前置依赖

- Node.js ≥ 18
- EdgeOne Pages 账号（[注册](https://edgeone.ai)）
- Supabase 账号（[注册](https://supabase.com)，免费）— 未配置时自动降级为游客模式
- Stripe 账号（[注册](https://stripe.com)，测试模式免费）
- WorkBuddy + `edgeone-pages-deploy` Skill（用于部署）

---

## 版本历史

| 版本 | 更新 |
|------|------|
| v3.1 | Step 2/3 拆分子步骤 + backend-guide + guest-mode-guide + admin 完整流程 |
| v3.0 | AI 助手 + 管理后台 + 游客模式 + 购物车 React Context 重构 |
| v2.0 | 全栈版：Supabase Auth + Stripe 支付 + Cloud Functions |
| v1.0 | 初始版：展示层 + Edge Functions + KV |

---

## License

MIT © 2025
