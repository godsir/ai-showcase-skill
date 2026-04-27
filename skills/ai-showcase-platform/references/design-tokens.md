# Design Token 完整 CSS 变量表

将以下内容写入 `src/styles/globals.css`：

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

/* ============================================
   AI Showcase Platform — Design Token System
   ============================================ */

@layer base {
  :root {
    /* ─── 基础色彩 ─── */
    --color-bg:              #ffffff;
    --color-surface:         #ffffff;
    --color-text-primary:    #000000;
    --color-text-secondary:  rgba(0, 0, 0, 0.6);
    --color-text-on-dark:    #ffffff;
    --color-border:          #000000;
    --color-border-subtle:   rgba(0, 0, 0, 0.12);

    /* ─── 按钮 ─── */
    --color-btn-solid-bg:    #000000;
    --color-btn-solid-text:  #ffffff;
    --color-btn-ghost-text:  #000000;

    /* ─── 覆盖层（微妙玻璃感）─── */
    --overlay-card-subtle:   rgba(0, 0, 0, 0.08);
    --overlay-dark-glass:    rgba(255, 255, 255, 0.16);

    /* ─── 渐变系统 ─── */
    /* 主渐变：电绿 → 亮黄 → 深紫 → 艳粉 */
    --gradient-hero: linear-gradient(135deg,
      #00ff88 0%,
      #ffee00 25%,
      #6600ff 60%,
      #ff00aa 100%
    );
    /* 渐变文字效果（用于大标题点缀词） */
    --gradient-text: linear-gradient(90deg, #00ff88, #6600ff, #ff00aa);

    /* ─── 产品域主题色 ─── */
    --theme-image-gen:  #00ff88;  /* 电绿   — 生图 */
    --theme-3d-model:   #6600ff;  /* 深紫   — 3D 模型 */
    --theme-llm-chat:   #ffee00;  /* 亮黄   — 大模型交互 */
    --theme-video-gen:  #ff00aa;  /* 艳粉   — 视频生成 */
    --theme-audio-gen:  #00ccff;  /* 电蓝   — 音频合成 */
    --theme-code-gen:   #ff6600;  /* 橙红   — 代码生成 */

    /* ─── 圆角 ─── */
    --radius-card:     16px;
    --radius-btn-pill: 9999px;
    --radius-btn-sq:   8px;

    /* ─── 字体 ─── */
    --font-sans: 'figmaSans', 'figmaSans Fallback', 'SF Pro Display', system-ui, helvetica, sans-serif;

    /* ─── 间距 ─── */
    --spacing-section:  80px;
    --spacing-card-gap: 24px;

    /* ─── 动效时间 ─── */
    --duration-fast:   150ms;
    --duration-normal: 300ms;
    --duration-slow:   600ms;
    --ease-out-expo: cubic-bezier(0.16, 1, 0.3, 1);
  }

  /* 基础重置 */
  *, *::before, *::after {
    box-sizing: border-box;
  }

  html {
    scroll-behavior: smooth;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
  }

  body {
    font-family: var(--font-sans);
    background-color: var(--color-bg);
    color: var(--color-text-primary);
    line-height: 1.6;
  }
}

/* ─── 渐变文字工具类 ─── */
@layer utilities {
  .text-gradient {
    background: var(--gradient-text);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
  }

  .gradient-orb {
    background: var(--gradient-hero);
    filter: blur(80px);
    opacity: 0.15;
    border-radius: 50%;
  }

  .glass-dark {
    background: var(--overlay-dark-glass);
    backdrop-filter: blur(12px);
    -webkit-backdrop-filter: blur(12px);
  }

  .border-subtle {
    border: 1px solid var(--color-border-subtle);
  }
}
```

## 使用原则

| 场景 | 使用的变量 |
|------|-----------|
| 页面背景 | `--color-bg` (#fff) |
| 卡片背景 | `--color-surface` (#fff) |
| 普通文字 | `--color-text-primary` (#000) |
| 深色背景上的文字 | `--color-text-on-dark` (#fff) |
| 实心按钮 | bg: `--color-btn-solid-bg`, text: `--color-btn-solid-text` |
| 幽灵按钮 | border: `--color-border`, text: `--color-btn-ghost-text` |
| 卡片悬停遮罩 | `--overlay-card-subtle` |
| 深色面板上的按钮 | `--overlay-dark-glass` |
| 生图模块 accent | `--theme-image-gen` |
| 3D模型模块 accent | `--theme-3d-model` |
| 大模型模块 accent | `--theme-llm-chat` |
| 视频生成模块 accent | `--theme-video-gen` |
