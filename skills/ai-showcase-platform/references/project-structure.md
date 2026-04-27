# 项目目录结构规范

## 完整目录树

```
<platform-name>/
├── public/
│   └── favicon.svg
├── src/
│   ├── components/
│   │   ├── layout/
│   │   │   ├── NavBar.tsx         # 顶部导航栏
│   │   │   └── Footer.tsx         # 底部面板
│   │   ├── sections/
│   │   │   ├── HeroSection.tsx    # 主视觉区
│   │   │   ├── ProductsSection.tsx # 产品卡片区
│   │   │   ├── DemoSection.tsx    # 交互演示区
│   │   │   └── StatsSection.tsx   # 数据统计区
│   │   ├── cards/
│   │   │   └── ProductCard.tsx    # 产品卡片组件
│   │   ├── demo/
│   │   │   ├── ImageGenDemo.tsx   # 生图 Mock 界面
│   │   │   ├── ModelGenDemo.tsx   # 3D 模型 Mock 界面
│   │   │   ├── LLMChatDemo.tsx    # 大模型对话 Mock 界面
│   │   │   └── VideoGenDemo.tsx   # 视频生成 Mock 界面
│   │   └── ui/
│   │       ├── Button.tsx         # 通用按钮（solid / ghost / pill）
│   │       ├── GradientOrb.tsx    # 渐变光晕装饰组件
│   │       └── AnimatedCounter.tsx # 数字滚动动画
│   ├── hooks/
│   │   ├── useStats.ts            # 获取 /api/stats 数据
│   │   └── useScrollSpy.ts        # 导航高亮滚动检测
│   ├── lib/
│   │   ├── cn.ts                  # clsx + tailwind-merge 工具
│   │   └── modules.ts             # AI 模块元数据配置
│   ├── styles/
│   │   └── globals.css            # Tailwind 入口 + CSS 变量
│   ├── App.tsx
│   └── main.tsx
├── functions/
│   └── api/
│       └── stats.js               # EdgeOne Edge Function
├── edgeone.json                   # EdgeOne Pages 项目配置（部署后自动生成）
├── tailwind.config.ts
├── vite.config.ts
├── tsconfig.json
└── package.json
```

## 关键文件说明

### `src/lib/modules.ts` — AI 模块元数据

```typescript
export interface AIModuleConfig {
  id: string;
  name: string;            // 显示名称（中文）
  nameEn: string;          // 英文名
  description: string;     // 一句话描述
  themeColor: string;      // CSS 变量名，如 var(--theme-image-gen)
  themeColorHex: string;   // 直接颜色值
  icon: string;            // Lucide 图标名
  demoComponent: string;   // 对应 Demo 组件名
}

export const AI_MODULES: AIModuleConfig[] = [
  {
    id: 'image-gen',
    name: '智能生图',
    nameEn: 'Image Generation',
    description: '输入文字描述，秒级生成高质量图像',
    themeColor: 'var(--theme-image-gen)',
    themeColorHex: '#00ff88',
    icon: 'ImageIcon',
    demoComponent: 'ImageGenDemo',
  },
  {
    id: '3d-model',
    name: '3D 模型制作',
    nameEn: '3D Model Generation',
    description: '从文字或图片一键生成可编辑三维模型',
    themeColor: 'var(--theme-3d-model)',
    themeColorHex: '#6600ff',
    icon: 'Box',
    demoComponent: 'ModelGenDemo',
  },
  {
    id: 'llm-chat',
    name: '大模型交互',
    nameEn: 'LLM Chat',
    description: '与多款顶级大模型实时对话，解锁无限可能',
    themeColor: 'var(--theme-llm-chat)',
    themeColorHex: '#ffee00',
    icon: 'MessageSquare',
    demoComponent: 'LLMChatDemo',
  },
  {
    id: 'video-gen',
    name: '视频生成',
    nameEn: 'Video Generation',
    description: '文字或图片转高清视频，创作从未如此简单',
    themeColor: 'var(--theme-video-gen)',
    themeColorHex: '#ff00aa',
    icon: 'Video',
    demoComponent: 'VideoGenDemo',
  },
];
```

### `tailwind.config.ts` — 扩展配置

```typescript
import type { Config } from 'tailwindcss'

export default {
  content: ['./src/**/*.{ts,tsx}'],
  theme: {
    extend: {
      fontFamily: {
        sans: ['"figmaSans"', '"figmaSans Fallback"', '"SF Pro Display"', 'system-ui', 'helvetica', 'sans-serif'],
      },
      colors: {
        'theme-image':  '#00ff88',
        'theme-3d':     '#6600ff',
        'theme-llm':    '#ffee00',
        'theme-video':  '#ff00aa',
        'theme-audio':  '#00ccff',
      },
      animation: {
        'float': 'float 6s ease-in-out infinite',
        'spin-slow': 'spin 20s linear infinite',
      },
      keyframes: {
        float: {
          '0%, 100%': { transform: 'translateY(0px)' },
          '50%': { transform: 'translateY(-12px)' },
        },
      },
    },
  },
  plugins: [],
} satisfies Config
```
