---
name: ai-showcase-platform
version: 3.1.0
author: workbuddy-contest
description: >
  一句话触发，帮助用户在 EdgeOne Pages 上搭建并部署一个酷炫科技风的 AI 聚合全栈平台。
  平台集成生图、3D模型制作、大模型交互、视频生成等 AI 能力展示，
  同时具备完整的用户登录注册（Supabase Auth）、AI 助手多模型对话（features.aiChat）、
  AI 能力购买（Stripe 支付）、购物车与订单管理等全栈电商功能，
  以及可选的管理后台（features.admin）。
  全站白底黑字科技渐变风格，EdgeOne Edge Functions + Cloud Functions + KV Storage 驱动后端。

triggers:
  - "帮我做一个 AI 展示平台"
  - "AI 聚合平台"
  - "AI showcase"
  - "AI portal"
  - "做一个展示 AI 能力的网站"
  - "AI 产品展示页"
  - "生图展示网站"
  - "3D 模型展示平台"
  - "大模型交互展示"
  - "视频生成平台"
  - "AI 工具合集网站"
  - "build an AI showcase"
  - "create AI platform website"
  - "AI landing page"
  - "带登录的 AI 平台"
  - "带支付的 AI 网站"
  - "AI 能力购买平台"
  - "AI SaaS 全栈网站"
  - "带购物功能的 AI 网站"
  - "AI 会员平台"

not_triggers:
  - 用户只是想问 AI 怎么用（咨询类）
  - 用户已有项目只想部署 → 使用 edgeone-pages-deploy
  - 用户只想做纯静态博客 / 企业官网
---

# AI 聚合展示平台 Skill（v3.1 全栈增强版）

你是一个专注于构建**酷炫科技风 AI 聚合全栈平台**的建站专家。  
你的目标是帮助用户从 0 到 1，在 EdgeOne Pages 上生成并部署一个视觉惊艳、功能完整的 AI 平台：
- 🎨 **展示层**：4 大 AI 模块展示卡片 + Mock 交互 Demo
- 🤖 **AI 助手**：多模型对话界面（GPT-4 / Claude / Gemini / Llama）
- 🔐 **认证层**：用户注册 / 登录 / 社交登录 / 退出（Supabase Auth）+ 游客体验模式
- 🛒 **购物层**：AI 能力套餐商品列表 + 购物车 + 结算
- 💳 **支付层**：Stripe Checkout 支付 + 订单状态管理
- ⚡ **边缘层**：EdgeOne Edge Functions（KV 访问统计）+ Cloud Functions（订单 API）
- 📊 **管理后台**：订单管理、用户管理、套餐管理、数据分析（可选）

---

## ⛔ 关键规则（永不跳过）

1. **先问用户配置，再动手写代码** — 通过 Plan 阶段分步收集完所有配置后再开始生成
2. **技术栈固定**：React + Vite + TypeScript + Tailwind CSS，不接受更换
3. **UI 设计系统固定**（见下文 Design Token），所有组件必须严格遵守，不得自由发挥颜色
4. **AI 功能使用 Mock** — AI 生图/3D/视频/对话均为 Mock 展示，AI 助手为前端 Mock（可扩展真实 API）
5. **认证必须用 Supabase Auth** — 不自己实现 JWT，不用 Firebase，统一 Supabase，支持 GitHub OAuth
6. **支付必须用 Stripe** — Checkout Session 模式，密钥只在 Cloud Function 中使用，绝不暴露到前端
7. **Cloud Functions 处理所有敏感操作**（支付、订单写入、管理员查询），Edge Functions 处理轻量统计
8. **环境变量不得硬编码进源文件** — 全部从 `.env.local` 读取，`.env.local` 加入 `.gitignore`
9. **部署交给 edgeone-pages-deploy Skill**，本 Skill 只负责生成代码和本地验证
10. **全站响应式**，移动端优先
11. **管理后台需管理员权限** — 通过 Supabase `profiles.is_admin` 字段控制，非管理员自动跳转

---

## 🔔 Plan 阶段 — 分布式配置引导（必做）

**每次触发 Skill 时，正式开始生成前必须通过 `ask_followup_question` 逐步收集配置。**

本阶段分 4 轮提问，每轮只问一类问题，用户可跳过（使用默认值）。全部收集完毕后写入完整 Spec，再进入代码生成。

---

### 第一轮 — 平台基础信息

```
标题：第 1 / 4 步：平台基础信息

问题：
1. 平台名称（必填）：网站名称，例如 `NeuraHub`、`AI Universe`，将显示在导航栏和 Footer Logo。留空默认 `NeuraHub`。
2. 主 Slogan（必填）：一句话描述平台，例如 `探索 AI 的无限可能`。留空默认 `探索 AI 的无限可能，一站体验未来`。
3. 展示的 AI 产品模块（多选）：默认全选 4 个
   - ✅ 生图（Image Generation）
   - ✅ 3D 模型生成
   - ✅ 大模型对话（LLM Chat）
   - ✅ 视频生成
   - ⬜ 音频合成（Audio Generation）
   - ⬜ 代码生成（Code Generation）

如果用户说"默认"，使用：NeuraHub + Slogan 留空默认 + 全选 4 个模块。
```

---

### 第二轮 — 接入的 API 服务（Supabase + Stripe）

```
标题：第 2 / 4 步：API 服务配置

说明：平台需要接入 Supabase（用户登录/数据库）和 Stripe（在线支付），请提供以下信息：

问题：
1. Supabase Project URL（必填）：在 supabase.com → 项目 → Settings → API 中复制。留空则使用示例值（无法真实登录）。
2. Supabase Anon/Public Key（必填）：同上页面。留空则使用示例值。
3. Supabase Auth 邮箱确认（必填）：是否开启邮箱验证注册？默认 `关闭`（演示模式更方便）。
4. Stripe Publishable Key（必填）：在 stripe.com → Developers → API keys 复制 pk_live_... 或 pk_test_...。留空则使用示例值（无法真实支付）。
5. Stripe Secret Key（必填，仅 Cloud Function 用）：在 stripe.com → Developers → API keys 复制 sk_live_... 或 sk_test_...。留空则使用示例值。

如果用户跳过，平台以演示模式运行（可体验 UI，但无法真实登录/支付）。
```

---

### 第三轮 — 定价与套餐策略

```
标题：第 3 / 4 步：定价与套餐配置

问题：
1. 货币单位（必填）：默认 `USD`（支持 USD / CNY / EUR）。
2. 套餐 A — 入门套餐（必填）：套餐名称，默认 `入门套餐 Starter`。价格（数字），默认 `9.9`。包含内容（简短描述），默认 `生图 100 张/月，大模型 1,000 次/月`。
3. 套餐 B — 专业套餐（必填）：套餐名称，默认 `专业套餐 Pro`。价格（数字），默认 `29.9`。包含内容（简短描述），默认 `全模块，解锁 3D + 视频，无限对话`。
4. 套餐 C — 企业套餐（必填）：套餐名称，默认 `企业套餐 Enterprise`。价格（数字），默认 `99.9`。包含内容（简短描述），默认 `全模块无限，SLA 保障，私有化部署`。
5. 是否在首页展示价格套餐区？默认 `是`。
6. 是否启用 Stripe Checkout？默认 `是`（关闭则仅展示购物车，不跳转支付）。

如果用户跳过，全部使用默认值。
```

---

### 第四轮 — 版权与最终确认

```
标题：第 4 / 5 步：版权与发布信息

问题：
1. 底部版权声明（必填）：例如 `© 2025 苏逸`。留空默认 `© 2025 苏逸`。
2. Footer 技术栈展示文字（可选）：默认 `Powered by EdgeOne Pages`，可改为自己的技术栈描述。
3. 是否开启演示账号体验功能？（默认 `是`，开启后在登录页显示一键演示按钮）。
4. 演示账号邮箱：默认 `demo@neurahub.com`。
5. 演示账号密码：默认 `demo123456`。

如果用户跳过，全部使用默认值。
```

---

### 第五轮（新增）— 功能模块开关

```
标题：第 5 / 5 步：功能模块选择

说明：根据需要启用以下高级功能，无需的可跳过使用默认值。

问题：
1. 是否启用 AI 助手对话功能？（默认 `是`，开启后在导航栏显示 AI 助手入口）
2. 是否启用管理后台？（默认 `否`，管理员专用，需额外配置 admin 账号）
3. 管理员演示账号邮箱（启用管理后台时必填）：默认 `admin@neurahub.com`。
4. 管理员演示账号密码：默认 `admin123456`。
5. 是否启用 GitHub 社交登录？（默认 `否`，需在 Supabase 配置 GitHub OAuth）
6. 是否启用订单邮件通知？（默认 `否`，Stripe Webhook 触发后发送邮件）

如果用户跳过，全部使用默认值。
```

---

### Spec 汇总

收集完毕后，将所有配置汇总为以下 Spec 对象，写入内存并向用户展示确认：

```typescript
interface PlatformSpec {
  // 基础信息
  platformName: string;        // Q1
  slogan: string;              // Q2
  modules: AIModule[];         // Q3
  copyright: string;           // Q4-1
  poweredBy: string;           // Q4-2

  // API 配置
  supabase: {
    url: string;
    anonKey: string;
    emailConfirm: boolean;
  };
  stripe: {
    publishableKey: string;
    secretKey: string;        // 仅 Cloud Function 使用
    enabled: boolean;          // Q3-6
  };

  // 功能开关
  features: {
    auth: boolean;             // 始终为 true（Supabase Auth）
    shop: boolean;             // 始终为 true
    payment: boolean;          // Q3-6
    orders: boolean;           // 始终为 true
    demoAccount: boolean;      // Q4-3
    aiChat: boolean;           // Q5-1：AI 助手对话功能
    admin: boolean;            // Q5-2：管理后台
    githubOAuth: boolean;      // Q5-5：GitHub 社交登录
    emailNotification: boolean; // Q5-6：订单邮件通知
  };
  demoAccount: {
    email: string;             // Q4-4
    password: string;         // Q4-5
  };
  adminAccount: {
    email: string;             // Q5-3
    password: string;         // Q5-4
  };

  // 套餐
  currency: string;           // Q3-1
  priceTiers: {
    id: string;
    name: string;
    nameEn: string;
    description: string;
    price: number;
    features: string[];
    themeColor: string;       // 自动分配
    coverIcon: string;        // Remix Icon 类名
    popular: boolean;
    badge?: string;
  }[];

  // 展示开关
  showPricing: boolean;       // Q3-5
  friendlyLinks: { label: string; url: string }[];
}
```

向用户确认 Spec 正确后，再进入 Step 1 开始生成代码。

> **注意**：若用户中途说"默认"或"直接开始"，立即使用上述所有默认值填充 Spec，进入代码生成。

---

## 🎨 Design Token（设计规范，必须严格遵守）

### 色彩系统

```css
/* 基础色 */
--color-bg: #ffffff;
--color-surface: #ffffff;
--color-text-primary: #000000;
--color-text-on-dark: #ffffff;
--color-border: #000000;

/* 按钮 */
--color-btn-solid-bg: #000000;
--color-btn-solid-text: #ffffff;
--color-btn-ghost-text: #000000;

/* 微妙覆盖层 */
--overlay-card-subtle: rgba(0, 0, 0, 0.08);       /* 次级圆形按钮 / 玻璃效果暗色覆盖 */
--overlay-dark-glass: rgba(255, 255, 255, 0.16);  /* 深色/彩色表面按钮磨砂玻璃覆盖 */

/* 渐变系统 — 充满活力的多色渐变 */
--gradient-hero: linear-gradient(135deg, #00ff88 0%, #ffee00 25%, #6600ff 60%, #ff00aa 100%);
/* 电绿 → 亮黄 → 深紫 → 艳粉 */

/* 各产品域主题色（用于卡片 accent） */
--theme-image-gen:   #00ff88;   /* 电绿 — 生图 */
--theme-3d-model:    #6600ff;   /* 深紫 — 3D 模型 */
--theme-llm-chat:    #ffee00;   /* 亮黄 — 大模型交互 */
--theme-video-gen:   #ff00aa;   /* 艳粉 — 视频生成 */
--theme-audio-gen:   #00ccff;   /* 电蓝 — 音频合成（可选扩展） */
```

### 字体系统

```css
--font-family: 'figmaSans', 'figmaSans Fallback', 'SF Pro Display', system-ui, helvetica, sans-serif;
--font-weight-regular: 400;
--font-weight-medium: 500;
--font-weight-bold: 700;
```

### 圆角 & 间距

```css
--radius-card: 16px;
--radius-btn-pill: 9999px;
--radius-btn-square: 8px;
--spacing-section: 80px;
--spacing-card-gap: 24px;
```

---

## 图标使用规范（Remix Icon）

**强制使用 Remix Icon 图标库，禁止使用 Emoji 作为功能图标。**

### CDN 引入

```html
<!-- 在 <head> 中引入 Remix Icon -->
<link href="https://cdn.jsdelivr.net/npm/remixicon@4.5.0/fonts/remixicon.min.css" rel="stylesheet" />
```

### 图标渲染方式

所有图标均使用 `<i class="ri-{icon-name}"></i>` 标签渲染，例如：

```html
<i class="ri-shopping-cart-line"></i> 购物车
<i class="ri-file-list-3-line"></i> 我的订单
<i class="ri-star-fill"></i> 生图
<i class="ri-cube-line"></i> 3D 建模
<i class="ri-message-3-line"></i> 大模型对话
<i class="ri-movie-2-line"></i> 视频生成
<i class="ri-play-fill"></i> 开始生成
<i class="ri-restart-line"></i> 重置
<i class="ri-settings-3-line"></i> 处理中
<i class="ri-send-plane-fill"></i> 发送
<i class="ri-delete-bin-line"></i> 删除
<i class="ri-shield-check-line"></i> 安全支付
<i class="ri-gift-line"></i> 成功
<i class="ri-emotion-unhappy-line"></i> 取消
<i class="ri-inbox-archive-line"></i> 无订单
<i class="ri-image-2-line"></i> 图片生成图标
<i class="ri-planet-line"></i> 量子花园
<i class="ri-rocket-line"></i> 星际旅人
<i class="ri-computer-line"></i> 流光代码
<i class="ri-menu-line"></i> 汉堡菜单
<i class="ri-seedling-line"></i> 入门套餐封面
<i class="ri-flashlight-line"></i> 专业套餐封面
<i class="ri-government-line"></i> 企业套餐封面
<i class="ri-film-line"></i> 视频生成
<i class="ri-pause-fill"></i> 暂停
```

### 图标大小与颜色

- 图标默认继承父元素 `color`，可通过 `style="font-size: Npx"` 和 `style="color: #hex"` 单独设置
- 图标可搭配 `ri-fw` 类实现固定宽度居中对齐
- 产品模块卡片图标建议尺寸：20–24px（搭配 48px 图标容器）
- 按钮/标签内图标建议尺寸：14–18px
- 展示/空状态图标建议尺寸：40–60px

### 通用映射表

| 场景 | 推荐图标 | 类名 |
|------|---------|------|
| 购物车按钮 | 购物车 | `ri-shopping-cart-line` / `ri-shopping-cart-2-line` |
| 添加购物车 | 购物车+ | `ri-shopping-cart-line` |
| 删除 | 垃圾桶 | `ri-delete-bin-line` |
| 发送 | 发送飞机 | `ri-send-plane-fill` |
| 播放 | 播放 | `ri-play-fill` |
| 暂停 | 暂停 | `ri-pause-fill` |
| 重置 | 重启 | `ri-restart-line` |
| 成功 | 礼物 | `ri-gift-line` |
| 失败 | 悲伤 | `ri-emotion-unhappy-line` |
| 加载中 | 设置 | `ri-settings-3-line` |
| 菜单 | 汉堡 | `ri-menu-line` |
| 订单 | 文件列表 | `ri-file-list-3-line` |
| 安全 | 盾牌 | `ri-shield-check-line` |
| 无内容 | 收件箱 | `ri-inbox-archive-line` |

---

## 站点配置（SITE_CONFIG）

网站核心配置（版权、名称等）统一声明在 JS 文件顶部的 `SITE_CONFIG` 对象中：

```javascript
const SITE_CONFIG = {
  name: 'NeuraHub',
  copyright: '© 2025 苏逸',
};
```

页脚 HTML 中用 `<span id="footerCopy"></span>` 占位，在 `init()` 中通过 `document.getElementById('footerCopy').textContent = SITE_CONFIG.copyright` 注入。

**重要**：版权信息必须通过此方式配置，不得硬编码在 HTML 中。

---

## 演示账号规范（Demo Account）

### 演示账号凭据


```
邮箱：demo@neurahub.com
密码：demo123456
```

### 登录页演示入口

在登录表单按钮下方、OAuth 分隔符上方放置「演示账号体验」按钮：

```html
<button class="form-submit" onclick="doLogin()" id="loginBtn">登录</button>
<button class="demo-btn" onclick="fillDemo()">
  <i class="ri-rocket-line"></i> 演示账号体验
</button>
<div class="or-divider">或</div>
```

**CSS 样式**（`demo-btn` 使用渐变背景 `var(--gradient-hero)`）

### `fillDemo()` 实现

```javascript
function fillDemo() {
  document.getElementById('loginEmail').value = 'demo@neurahub.com';
  document.getElementById('loginPwd').value = 'demo123456';
  doLogin();
}
```

### 个人中心页面规范（Personal Center / Profile）

个人中心页面（`page-profile`）必须包含以下 4 个区块，全部使用 **Mock 数据**，不依赖真实 API：

#### 区块 1：用户信息头部

- 渐变头像（显示用户名首字母缩写）
- 用户名 + 邮箱
- 账号标签（演示账号 / 正式会员）
- 注册时间（Mock）

#### 区块 2：AI 能力使用统计（全部 4 个模块）

所有模块均展示 Mock 数据：

| 模块 | Mock 用量 | 上限 |
|------|-----------|------|
| 生图 Image Generation | 847 | 1,000 次 |
| 3D 模型生成 | 42 | 50 次 |
| 大模型对话 LLM Chat | 1,247 | 无限次 |
| 视频生成 Video Generation | 18 | 20 次 |

每个统计卡包含：进度条（百分比填充） + 已用量/上限数字

#### 区块 3：快速体验全部 AI 能力（4 个可点击卡片）

点击任意卡片 → 导航到对应 Demo 页面（`switchDemo(moduleId)`）：

- 生图 Image Generation → `switchDemo('image-gen')`
- 3D 建模 → `switchDemo('3d-model')`
- 大模型对话 → `switchDemo('llm-chat')`
- 视频生成 → `switchDemo('video-gen')`

#### 区块 4：账号设置（纯 UI）

- 个人信息
- 修改密码
- 通知设置
- 安全与隐私
- 退出登录（点击触发 `doLogout()`）

所有功能均标注「演示模式」，无真实数据操作。

#### `renderProfile()` 实现要点

```javascript
function renderProfile() {
  // 1. 渲染用户头部（currentUser || demo 默认值）
  // 2. 渲染 4 个模块的使用统计（Mock 数据）
  // 3. 渲染 4 个可点击的 Demo 卡片
  // 4. 渲染最近 3 条订单（取 orders 数组末尾）
  // 5. 账号设置为纯 UI，无需动态数据
}
```

#### 导航栏集成

在用户下拉菜单中加入「个人中心」入口：

```html
<a href="#" onclick="showPage('profile');closeDropdown()">
  <i class="ri-user-line"></i> 个人中心
</a>
```

---

## Step 1 — 初始化项目（Spec 已在 Plan 阶段完成）

```bash
# 创建 Vite + React + TypeScript 项目
npm create vite@latest <platformName-kebab-case> -- --template react-ts
cd <platformName-kebab-case>

# 基础依赖
npm install
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
npm install framer-motion lucide-react clsx tailwind-merge

# 全栈依赖
npm install @supabase/supabase-js        # 认证 + 数据库
npm install @stripe/stripe-js            # Stripe 前端 SDK（仅用于跳转 Checkout）
npm install zustand                      # 购物车全局状态
npm install react-router-dom             # 路由（登录/订单/商城页面）
```

初始化后，按以下结构组织代码（见 `references/project-structure.md`）。

> ⚠️ 复制 `.env.example` → `.env.local`，填写以下变量（见 `references/auth-guide.md` 和 `references/payment-guide.md`）：
> ```
> VITE_SUPABASE_URL=
> VITE_SUPABASE_ANON_KEY=
> VITE_STRIPE_PUBLISHABLE_KEY=
> # 以下变量只用于 Cloud Function，不要以 VITE_ 前缀暴露到前端
> STRIPE_SECRET_KEY=
> SUPABASE_SERVICE_ROLE_KEY=
> ```

---

## Step 2 — 生成完整网站代码

严格按照 Design Token 和 UI 规范，**按以下子步骤依次生成**所有组件和页面。

> ⚠️ **每个子步骤都是独立的，必须逐一完成，不得跳过任何步骤。**
> 详细代码规范见对应 references 文件。

### 路由结构（React Router）

```
/                     → 首页（展示 + 商城入口）
/login                → 登录 / 注册页 + 游客模式体验入口
/profile              → 个人中心（三 Tab：账号信息 / 我的订单 / 使用量）
/cart                 → 购物车
/checkout             → 结算页（Mock 支付）
/shop                 → AI 能力套餐商城（可选）
/ai-chat              → AI 助手对话页（features.aiChat=true 时启用）
/admin                → 管理后台首页（features.admin=true 且 is_admin=true 时可访问）
/admin/orders         → 订单管理
/admin/users         → 用户管理
/admin/products      → 套餐管理
/admin/analytics     → 数据分析
```

> ⚠️ **游客模式**：未配置 Supabase 时，用户可通过登录页的「游客模式体验」按钮以 Mock 身份访问所有页面（/cart、/checkout、/profile），数据存 localStorage。详见 `references/guest-mode-guide.md`。

---

### Step 2a — 基础架构（必须）

**创建顺序**：

1. **`src/styles/globals.css`** — Design Token + CSS 变量 → `references/design-tokens.md`
2. **`src/lib/cn.ts`** — clsx + tailwind-merge 工具函数 → `references/component-guide.md`
3. **`src/lib/supabase.ts`** — Supabase 客户端初始化（isSupabaseConfigured 导出）→ `references/auth-guide.md`
4. **`src/lib/modules.ts`** — AI 模块元数据配置 → `references/project-structure.md`
5. **`src/lib/products.ts`** — 商品静态配置 → `references/shop-guide.md`
6. **`src/hooks/useAuth.tsx`** — AuthContext + 游客模式（含 guestSignIn / signOut）→ `references/auth-guide.md`
7. **`src/hooks/useCart.tsx`** — CartContext + OrderContext → `references/shop-guide.md`
8. **`src/components/ProtectedRoute.tsx`** — 路由守卫 → `references/auth-guide.md`
9. **`src/App.tsx`** — BrowserRouter + AuthProvider + CartProvider + Routes → `references/auth-guide.md`
10. **`src/main.tsx`** — 入口文件

---

### Step 2b — 展示层组件（必须）

**首页结构**：

```
<HomePage>
  <NavBar />                    // 顶部导航：Logo + 导航链接 + 登录状态 + 购物车图标
  <HeroSection />               // 主视觉区：大标题 + Slogan + 渐变光晕
  <ProductsSection />           // AI 能力卡片区（4 大模块 + "立即体验"入口）
  <DemoSection />               // Mock 交互演示区（黑底）
  <PricingSection />            // 套餐定价区（3 档卡片 + 加入购物车按钮）
  <StatsSection />              // 数据统计（KV 实时访问量）
  <Footer />                    // 底部版权 + 友链
</HomePage>
```

**创建顺序**：

1. **`src/components/layout/NavBar.tsx`** — 顶部导航栏 + 购物车图标 + 用户下拉菜单 → `references/component-guide.md` + `references/shop-guide.md`
2. **`src/components/layout/Footer.tsx`** — 底部面板 → `references/component-guide.md`
3. **`src/components/sections/HeroSection.tsx`** — 主视觉区 → `references/component-guide.md`
4. **`src/components/cards/ProductCard.tsx`** — 产品卡片组件 → `references/component-guide.md`
5. **`src/components/sections/ProductsSection.tsx`** — AI 能力卡片区 → `references/component-guide.md`
6. **`src/components/demo/ImageGenDemo.tsx`** — 生图 Mock → `references/mock-interactions.md`
7. **`src/components/demo/ModelGenDemo.tsx`** — 3D 模型 Mock（Canvas 线框立方体）→ `references/mock-interactions.md`
8. **`src/components/demo/LLMChatDemo.tsx`** — 大模型对话 Mock → `references/mock-interactions.md`
9. **`src/components/demo/VideoGenDemo.tsx`** — 视频生成 Mock（进度条动画）→ `references/mock-interactions.md`
10. **`src/components/sections/DemoSection.tsx`** — Demo 容器 + Tab 切换 → `references/mock-interactions.md`
11. **`src/components/sections/PricingSection.tsx`** — 定价区 + 加入购物车按钮 → `references/shop-guide.md`
12. **`src/components/sections/StatsSection.tsx`** — KV 访问量统计 → `references/component-guide.md`
13. **`src/pages/HomePage.tsx`** — 首页组合

---

### Step 2c — 认证与个人中心页面（必须）

1. **`src/pages/LoginPage.tsx`** — 登录/注册 + 游客体验入口 → `references/auth-guide.md`
   - Tab 切换：登录 / 注册
   - 表单：邮箱 + 密码（注册多确认密码）
   - 未配置 Supabase 提示
   - 「游客模式体验」按钮
   - GitHub OAuth（可选）
2. **`src/pages/ProfilePage.tsx`** — 个人中心三 Tab → `references/shop-guide.md` + `references/guest-mode-guide.md`
   - 账号信息：渐变背景 + 头像 + 用户名 + 「体验账号」标签（游客）
   - 我的订单：Mock 订单列表 + 状态标签
   - 使用量：Mock 进度条数据
   - **⚠️ 不要用 overflow-hidden**（头像被裁切）
   - **⚠️ 游客隐藏修改密码，不阻断页面渲染**

---

### Step 2d — 购物与结算页面（必须）

1. **`src/pages/CartPage.tsx`** — 购物车 → `references/shop-guide.md`
   - 未登录 → 「请先登录」引导
   - 空购物车 → 空状态（图标 + 文案）
   - 商品列表：色条 + 名称 + 价格 + 删除
   - 底部汇总 + 结算按钮 + 游客提示
2. **`src/pages/CheckoutPage.tsx`** — 结算页 → `references/shop-guide.md`
   - 订单确认区
   - 优惠码（占位 UI）
   - 支付方式选择（微信/支付宝/信用卡）
   - Mock 支付流程（2 秒延迟 → 成功页）

---

### Step 2e — AI 助手对话页面（features.aiChat=true 时）

> ⚠️ **此步骤仅在 Plan 阶段用户选择启用 AI 助手时执行。不可跳过。**

**创建顺序**：

1. **`src/lib/mockResponses.ts`** — Mock 响应逻辑（4 模型不同风格）→ `references/ai-chat-guide.md`
2. **`src/components/ModelSelector.tsx`** — 模型选择下拉（GPT-4 / Claude 3 / Gemini / Llama 3）→ `references/ai-chat-guide.md`
3. **`src/components/MessageBubble.tsx`** — 消息气泡（用户右对齐渐变 / AI 左对齐灰底）→ `references/ai-chat-guide.md`
4. **`src/components/ChatInput.tsx`** — 聊天输入框（Enter 发送，Shift+Enter 换行）→ `references/ai-chat-guide.md`
5. **`src/components/LoadingIndicator.tsx`** — 加载指示器（三点跳动动画）→ `references/ai-chat-guide.md`
6. **`src/pages/AIChatPage.tsx`** — AI 助手主页面 → `references/ai-chat-guide.md`

**NavBar 升级**：在导航项中加入「AI 助手」链接（仅 features.aiChat=true 时显示）。

**首页升级**：在 DemoSection 和 PricingSection 之间插入 `<AIPreviewSection />`（AI 助手预览卡片）。

---

### Step 2f — 管理后台（features.admin=true 时）

> ⚠️ **此步骤仅在 Plan 阶段用户选择启用管理后台时执行。不可跳过。**

**创建顺序**：

1. **`src/layouts/AdminLayout.tsx`** — 管理后台布局（侧边栏 + Outlet）→ `references/admin-guide.md`
   - 侧边栏：渐变 Logo + 5 个导航项 + 退出按钮
   - 顶部栏：汉堡菜单 + 用户信息
   - 可折叠侧边栏
2. **`src/pages/admin/Dashboard.tsx`** — 数据概览 → `references/admin-guide.md`
   - 4 张统计卡片（用户数、订单数、收入、月新增）
   - 快捷操作入口
3. **`src/pages/admin/Orders.tsx`** — 订单管理 → `references/admin-guide.md`
   - 状态筛选 + 搜索框
   - 订单表格（ID、用户、商品、金额、状态、时间）
4. **`src/pages/admin/Users.tsx`** — 用户管理 → `references/admin-guide.md`
   - 搜索框
   - 用户表格（邮箱、昵称、角色、注册时间）
   - 「设为/移除管理员」操作
5. **`src/pages/admin/Products.tsx`** — 套餐管理 → `references/admin-guide.md`
   - 套餐卡片列表 + 编辑表单
   - Mock 数据（实际应从数据库读取）
6. **`src/pages/admin/Analytics.tsx`** — 数据分析 → `references/admin-guide.md`
   - 渐变总收入卡片
   - 月度收入趋势条形图
   - 套餐销量分布

**路由配置**：

```tsx
<Route path="/admin" element={<ProtectedRoute requireAdmin><AdminLayout /></ProtectedRoute>}>
  <Route index element={<Dashboard />} />
  <Route path="orders" element={<AdminOrders />} />
  <Route path="users" element={<AdminUsers />} />
  <Route path="products" element={<AdminProducts />} />
  <Route path="analytics" element={<AdminAnalytics />} />
</Route>
```

**Supabase 数据库**：需执行 `references/admin-guide.md` 中的 SQL（profiles 表 + RLS + set_admin 函数）。

**登录页**：添加管理员演示入口按钮（仅 features.admin=true 时显示）。

---

## Step 3 — 接入后端函数

> ⚠️ **每个子步骤都是独立的，必须逐一完成。**
> 完整代码和创建流程见 `references/backend-guide.md`。

### Step 3a — Edge Function（必须）

**文件**：`functions/api/stats.js`

创建 KV 访问量统计接口。EdgeOne Pages 自动将 `functions/` 映射为 `/api/*`。

无需 npm 依赖（V8 运行时）。详见 `references/backend-guide.md`。

### Step 3b — Cloud Functions（按需）

**目录**：`cloud-functions/`

需要 `stripe` 和 `@supabase/supabase-js` npm 包。

**创建顺序**：

1. **`package.json`** — 声明依赖并 `npm install`
2. **`create-checkout.js`** — 创建 Stripe Checkout Session（需 STRIPE_SECRET_KEY）
3. **`webhook.js`** — Stripe Webhook 回调写入 orders 表（需 STRIPE_WEBHOOK_SECRET）
4. **`orders.js`** — 查询当前用户订单
5. **`admin-orders.js`** — 管理员查询全部订单（features.admin=true 时，需 SUPABASE_SERVICE_ROLE_KEY）
6. **`admin-stats.js`** — 管理员统计数据（features.admin=true 时）

**环境变量**：详见 `references/backend-guide.md` 环境变量清单。

**数据库初始化**：在 Supabase SQL Editor 执行建表语句（profiles + orders + RLS）。详见 `references/backend-guide.md`。

**创建流程检查清单**：
- [ ] `functions/api/stats.js` 创建完毕
- [ ] `cloud-functions/package.json` 创建并 `npm install`
- [ ] `cloud-functions/create-checkout.js` 创建完毕
- [ ] `cloud-functions/webhook.js` 创建完毕
- [ ] `cloud-functions/orders.js` 创建完毕
- [ ] `cloud-functions/admin-orders.js` 创建完毕（features.admin=true 时）
- [ ] `cloud-functions/admin-stats.js` 创建完毕（features.admin=true 时）
- [ ] `.env.local` 配置完毕
- [ ] Supabase SQL 执行完毕

---

## Step 4 — 本地验证

```bash
# 带 Edge Functions + Cloud Functions 完整验证（推荐）
edgeone pages dev
# → 访问 http://localhost:8088

# 仅前端验证（无法测试后端函数）
npm run dev
# → 访问 http://localhost:5173
```

验证清单：

**基础展示**
- [ ] 顶部导航显示正确，响应式汉堡菜单正常
- [ ] Hero 区渐变光晕动效正常
- [ ] 产品卡片 4 张全部显示，Hover 动效正确
- [ ] Demo 区切换 Tab，Mock 交互正常
- [ ] 定价套餐 3 张卡片显示，中间卡片有推荐标识
- [ ] `/api/stats` 返回 `{ pageviews: N }`

**认证流程**
- [ ] 访问 `/login` 页面正常渲染
- [ ] 注册新账号（需 Supabase 配置），收到验证邮件
- [ ] 登录后 NavBar 显示用户信息 + 购物车图标
- [ ] 登录后访问 `/profile` 正常，未登录时自动跳转 `/login`

**⚠️ 游客模式（重点验证）**
- [ ] 登录页显示「游客模式体验」按钮，点击后跳转首页
- [ ] 游客登录后 NavBar 显示用户头像（不能显示登录/注册按钮）
- [ ] 游客可正常访问 `/cart`、`/checkout`、`/profile`
- [ ] 购物车底部显示「体验模式」提示
- [ ] 结算页模拟支付成功后显示「订阅成功」页
- [ ] 个人中心显示「体验账号」标签，隐藏修改密码
- [ ] 个人中心头像不被渐变背景遮挡
- [ ] **点击「退出体验」后 NavBar 立即刷新为未登录状态**
- [ ] 未配置 Supabase 时不白屏，游客模式正常可用

**购物 & 支付流程**
- [ ] 定价区「加入购物车」按钮有状态反馈（已添加 / 已在购物车）
- [ ] 未登录点击加入购物车 → 跳转 `/login` 并保存来源路径
- [ ] NavBar 购物车图标显示数量角标
- [ ] 访问 `/cart` 显示已选商品，可删除，空购物车显示引导
- [ ] 结算页显示订单详情、优惠码输入、支付方式选择
- [ ] Mock 支付流程正常（2 秒延迟 → 成功页）

**个人中心**
- [ ] 三 Tab 切换正常（账号信息 / 我的订单 / 使用量）
- [ ] 账号信息显示基本信息 + 修改密码
- [ ] 我的订单显示历史订单，可取消订阅
- [ ] 使用量显示 Mock 进度条数据

**移动端**
- [ ] 375px 宽度下所有页面布局正常

**AI 助手（features.aiChat=true 时）**
- [ ] `/ai-chat` 页面正常渲染
- [ ] 模型选择下拉正常切换
- [ ] 发送消息后 AI 正确回复
- [ ] 不同模型显示不同回复风格
- [ ] 移动端输入框和聊天区域适配正常

**管理后台（features.admin=true 时）**
- [ ] 管理员演示账号登录成功
- [ ] `/admin` 侧边栏导航正常
- [ ] `/admin/orders` 订单列表显示
- [ ] `/admin/users` 用户列表显示
- [ ] `/admin/products` 套餐管理显示
- [ ] `/admin/analytics` 数据概览显示
- [ ] 非管理员访问 `/admin` 自动跳转首页

---

## Step 5 — 交接部署

检查用户是否已安装 `edgeone-pages-deploy` Skill：

**已安装** → 告知用户说：「**部署到 EdgeOne Pages**」即可触发部署流程

**未安装** → 提示用户安装，或给出手动部署命令：
```bash
npm install -g edgeone@latest
edgeone pages deploy -n <platform-name>
```

部署完成后：
1. 展示完整的 `EDGEONE_DEPLOY_URL`（含查询参数，不得截断）
2. 展示 EdgeOne Pages 控制台链接
3. 提示：如需长期稳定访问，建议在控制台绑定已备案的自定义域名

---

## 参考文档索引

| 内容 | 文件 |
|------|------|
| 项目目录结构规范 | `references/project-structure.md` |
| 各展示组件完整代码 | `references/component-guide.md` |
| Design Token 完整 CSS 变量表 | `references/design-tokens.md` |
| Mock 数据与交互动效规范 | `references/mock-interactions.md` |
| **用户认证方案（Supabase Auth + 游客模式）** | `references/auth-guide.md` |
| **游客体验模式完整规范** | `references/guest-mode-guide.md` |
| **支付方案（Stripe + 订单流程）** | `references/payment-guide.md` |
| **购物车 + 结算 + 订单 + 个人中心规范** | `references/shop-guide.md` |
| **AI 助手对话页面规范** | `references/ai-chat-guide.md` |
| **管理后台规范** | `references/admin-guide.md` |
| **后端函数完整创建流程（Edge Functions + Cloud Functions）** | `references/backend-guide.md` |
| Edge Functions + KV 接入指南 | `references/edge-functions-kv.md` |

---

## ⚠️ 常见踩坑清单（生成代码时必须规避）

> 以下是实际项目中遇到的 bug 和修复方案，生成代码时**务必避免**这些错误。

### 1. signOut 必须调用 setUser(null) + setSession(null)
只清 localStorage 不清 React state，NavBar 不会重新渲染，头像仍然显示。
→ 详见 `references/auth-guide.md` 踩坑经验 #1

### 2. NavBar 用户判断不能加 isConfigured 条件
游客 `isConfigured = false`，加此条件会导致游客永远看不到头像。
→ 详见 `references/auth-guide.md` 踩坑经验 #2

### 3. 个人中心卡片不要 overflow-hidden
头像用 -mt-12 上浮到渐变区域，overflow-hidden 会裁掉头像。
→ 详见 `references/auth-guide.md` 踩坑经验 #3

### 4. supabase 未配置时不能 throw Error
应用优雅降级让游客模式正常工作，不是直接白屏。
→ 详见 `references/auth-guide.md` 踩坑经验 #4

### 5. JSX 条件渲染内不能有多个根元素
`{condition && (<A/><B/>)}` 会导致 PARSE_ERROR，需要包一个 `<div>`。
→ 详见 `references/auth-guide.md` 踩坑经验 #5

### 6. 个人中心不能因 isConfigured=false 阻断
游客模式下 isConfigured=false 但 user 有值，个人中心应正常渲染。
→ 详见 `references/guest-mode-guide.md`
