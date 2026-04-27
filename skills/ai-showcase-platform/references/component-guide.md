# 各组件完整代码规范

本文档提供所有核心组件的完整实现代码，AI 应严格按照此规范生成，不得随意更改颜色、字体、间距。

---

## 1. `src/lib/cn.ts`

```typescript
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

---

## 2. `src/components/layout/NavBar.tsx`

```tsx
import { useState, useEffect } from 'react';
import { Menu, X } from 'lucide-react';
import { cn } from '@/lib/cn';

interface NavBarProps {
  platformName: string;
}

const NAV_LINKS = [
  { label: 'Products', href: '#products' },
  { label: 'Demo', href: '#demo' },
  { label: 'About', href: '#stats' },
];

export function NavBar({ platformName }: NavBarProps) {
  const [menuOpen, setMenuOpen] = useState(false);
  const [scrolled, setScrolled] = useState(false);

  useEffect(() => {
    const handler = () => setScrolled(window.scrollY > 8);
    window.addEventListener('scroll', handler, { passive: true });
    return () => window.removeEventListener('scroll', handler);
  }, []);

  return (
    <header
      className={cn(
        'fixed top-0 left-0 right-0 z-50 bg-white transition-shadow duration-300',
        scrolled ? 'shadow-[0_1px_0_0_#000]' : 'border-b border-black'
      )}
    >
      <nav className="max-w-7xl mx-auto px-6 h-16 flex items-center justify-between">
        {/* Logo */}
        <a href="#" className="text-xl font-bold tracking-tight text-black">
          {platformName}
        </a>

        {/* Desktop Nav */}
        <ul className="hidden md:flex items-center gap-8">
          {NAV_LINKS.map((link) => (
            <li key={link.href}>
              <a
                href={link.href}
                className="text-sm font-medium text-black/70 hover:text-black transition-colors"
              >
                {link.label}
              </a>
            </li>
          ))}
        </ul>

        {/* CTA */}
        <div className="hidden md:flex items-center gap-3">
          <a
            href="#demo"
            className="px-5 py-2 bg-black text-white text-sm font-medium rounded-full hover:bg-black/80 transition-colors"
          >
            开始体验
          </a>
        </div>

        {/* Mobile Toggle */}
        <button
          className="md:hidden p-2 text-black"
          onClick={() => setMenuOpen(!menuOpen)}
          aria-label="Toggle menu"
        >
          {menuOpen ? <X size={20} /> : <Menu size={20} />}
        </button>
      </nav>

      {/* Mobile Menu */}
      {menuOpen && (
        <div className="md:hidden bg-white border-t border-black">
          <ul className="flex flex-col px-6 py-4 gap-4">
            {NAV_LINKS.map((link) => (
              <li key={link.href}>
                <a
                  href={link.href}
                  className="text-base font-medium text-black"
                  onClick={() => setMenuOpen(false)}
                >
                  {link.label}
                </a>
              </li>
            ))}
            <li>
              <a
                href="#demo"
                className="inline-block px-5 py-2 bg-black text-white text-sm font-medium rounded-full"
                onClick={() => setMenuOpen(false)}
              >
                开始体验
              </a>
            </li>
          </ul>
        </div>
      )}
    </header>
  );
}
```

---

## 3. `src/components/sections/HeroSection.tsx`

```tsx
import { motion } from 'framer-motion';

interface HeroSectionProps {
  platformName: string;
  slogan: string;
}

export function HeroSection({ platformName, slogan }: HeroSectionProps) {
  return (
    <section className="relative min-h-screen flex items-center justify-center overflow-hidden bg-white pt-16">
      {/* 渐变光晕装饰 */}
      <div
        className="absolute bottom-[-10%] right-[-5%] w-[600px] h-[600px] rounded-full pointer-events-none"
        style={{
          background: 'linear-gradient(135deg, #00ff88 0%, #ffee00 25%, #6600ff 60%, #ff00aa 100%)',
          filter: 'blur(120px)',
          opacity: 0.12,
        }}
      />
      <div
        className="absolute top-[10%] left-[-8%] w-[400px] h-[400px] rounded-full pointer-events-none"
        style={{
          background: 'linear-gradient(135deg, #6600ff 0%, #ff00aa 100%)',
          filter: 'blur(100px)',
          opacity: 0.08,
        }}
      />

      {/* 内容 */}
      <div className="relative z-10 max-w-5xl mx-auto px-6 text-center">
        <motion.div
          initial={{ opacity: 0, y: 30 }}
          animate={{ opacity: 1, y: 0 }}
          transition={{ duration: 0.8, ease: [0.16, 1, 0.3, 1] }}
        >
          {/* 主标题 */}
          <h1
            className="font-bold text-black leading-none mb-6"
            style={{ fontSize: 'clamp(48px, 8vw, 96px)' }}
          >
            Welcome to{' '}
            <span
              style={{
                background: 'linear-gradient(90deg, #00ff88, #6600ff, #ff00aa)',
                WebkitBackgroundClip: 'text',
                WebkitTextFillColor: 'transparent',
                backgroundClip: 'text',
              }}
            >
              {platformName}
            </span>
          </h1>

          {/* Slogan */}
          <p className="text-xl text-black/60 max-w-2xl mx-auto mb-10 leading-relaxed">
            {slogan}
          </p>

          {/* CTA 按钮组 */}
          <div className="flex flex-wrap gap-4 justify-center">
            <a
              href="#products"
              className="px-8 py-3.5 bg-black text-white font-medium rounded-full hover:bg-black/80 transition-all duration-200 hover:-translate-y-0.5"
            >
              探索 AI 能力
            </a>
            <a
              href="#demo"
              className="px-8 py-3.5 border border-black text-black font-medium rounded-full hover:bg-black hover:text-white transition-all duration-200 hover:-translate-y-0.5"
            >
              立即体验 Demo →
            </a>
          </div>
        </motion.div>

        {/* 模块 Tag 列表 */}
        <motion.div
          initial={{ opacity: 0, y: 20 }}
          animate={{ opacity: 1, y: 0 }}
          transition={{ duration: 0.8, delay: 0.3, ease: [0.16, 1, 0.3, 1] }}
          className="mt-16 flex flex-wrap gap-3 justify-center"
        >
          {[
            { label: '生图', color: '#00ff88' },
            { label: '3D 建模', color: '#6600ff' },
            { label: '大模型对话', color: '#ffee00' },
            { label: '视频生成', color: '#ff00aa' },
          ].map((tag) => (
            <span
              key={tag.label}
              className="px-4 py-1.5 text-sm font-medium border border-black/20 rounded-full"
              style={{
                background: tag.color + '14',
                borderColor: tag.color + '40',
                color: '#000',
              }}
            >
              {tag.label}
            </span>
          ))}
        </motion.div>
      </div>
    </section>
  );
}
```

---

## 4. `src/components/cards/ProductCard.tsx`

```tsx
import { motion } from 'framer-motion';
import * as LucideIcons from 'lucide-react';
import type { AIModuleConfig } from '@/lib/modules';
import { cn } from '@/lib/cn';

interface ProductCardProps {
  module: AIModuleConfig;
  index: number;
  onSelect?: (id: string) => void;
}

export function ProductCard({ module, index, onSelect }: ProductCardProps) {
  const IconComponent = (LucideIcons as Record<string, React.ComponentType<{ size?: number }>>)[module.icon];

  return (
    <motion.div
      initial={{ opacity: 0, y: 24 }}
      whileInView={{ opacity: 1, y: 0 }}
      viewport={{ once: true }}
      transition={{ duration: 0.6, delay: index * 0.1, ease: [0.16, 1, 0.3, 1] }}
      whileHover={{ y: -4 }}
      className="group relative bg-white border border-black rounded-2xl overflow-hidden cursor-pointer"
      onClick={() => onSelect?.(module.id)}
    >
      {/* Accent bar — 顶部彩色条 */}
      <div
        className="h-1 w-full transition-all duration-300 group-hover:h-1.5"
        style={{ background: module.themeColorHex }}
      />

      <div className="p-6">
        {/* Icon */}
        <div
          className="w-12 h-12 rounded-xl flex items-center justify-center mb-4"
          style={{ background: module.themeColorHex + '18' }}
        >
          {IconComponent && (
            <IconComponent size={22} />
          )}
        </div>

        {/* 名称 */}
        <h3 className="text-lg font-bold text-black mb-1">{module.name}</h3>
        <p className="text-xs font-medium text-black/40 mb-3 uppercase tracking-wider">{module.nameEn}</p>

        {/* 描述 */}
        <p className="text-sm text-black/60 leading-relaxed mb-5">{module.description}</p>

        {/* 体验按钮 */}
        <button
          className={cn(
            'w-full py-2 text-sm font-medium border border-black rounded-full',
            'text-black transition-all duration-200',
            'group-hover:text-white group-hover:border-transparent'
          )}
          style={{
            background: 'transparent',
          }}
          onMouseEnter={(e) => {
            (e.currentTarget as HTMLElement).style.background = module.themeColorHex;
            (e.currentTarget as HTMLElement).style.borderColor = module.themeColorHex;
            (e.currentTarget as HTMLElement).style.color = '#000';
          }}
          onMouseLeave={(e) => {
            (e.currentTarget as HTMLElement).style.background = 'transparent';
            (e.currentTarget as HTMLElement).style.borderColor = '#000';
            (e.currentTarget as HTMLElement).style.color = '#000';
          }}
        >
          体验 →
        </button>
      </div>
    </motion.div>
  );
}
```

---

## 5. `src/components/sections/ProductsSection.tsx`

```tsx
import { useState } from 'react';
import { motion } from 'framer-motion';
import { ProductCard } from '@/components/cards/ProductCard';
import { AI_MODULES } from '@/lib/modules';

interface ProductsSectionProps {
  onModuleSelect?: (id: string) => void;
}

export function ProductsSection({ onModuleSelect }: ProductsSectionProps) {
  return (
    <section id="products" className="py-20 px-6 bg-white">
      <div className="max-w-7xl mx-auto">
        {/* Section 标题 */}
        <motion.div
          initial={{ opacity: 0, x: -20 }}
          whileInView={{ opacity: 1, x: 0 }}
          viewport={{ once: true }}
          transition={{ duration: 0.6 }}
          className="mb-12"
        >
          <h2 className="text-4xl font-bold text-black mb-3">AI 能力矩阵</h2>
          <p className="text-black/50 text-lg">四大核心模块，覆盖 AI 创作全场景</p>
        </motion.div>

        {/* 卡片网格 */}
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
          {AI_MODULES.map((module, index) => (
            <ProductCard
              key={module.id}
              module={module}
              index={index}
              onSelect={(id) => {
                onModuleSelect?.(id);
                document.getElementById('demo')?.scrollIntoView({ behavior: 'smooth' });
              }}
            />
          ))}
        </div>
      </div>
    </section>
  );
}
```

---

## 6. `src/components/sections/StatsSection.tsx`

```tsx
import { useEffect, useState } from 'react';
import { motion } from 'framer-motion';

function AnimatedCounter({ value, suffix = '' }: { value: number; suffix?: string }) {
  const [display, setDisplay] = useState(0);

  useEffect(() => {
    const duration = 1500;
    const start = Date.now();
    const tick = () => {
      const elapsed = Date.now() - start;
      const progress = Math.min(elapsed / duration, 1);
      const eased = 1 - Math.pow(1 - progress, 3); // ease-out-cubic
      setDisplay(Math.floor(eased * value));
      if (progress < 1) requestAnimationFrame(tick);
    };
    requestAnimationFrame(tick);
  }, [value]);

  return <span>{display.toLocaleString()}{suffix}</span>;
}

export function StatsSection() {
  const [pageviews, setPageviews] = useState<number | null>(null);

  useEffect(() => {
    // 上报访问 + 获取统计
    fetch('/api/stats', { method: 'POST' })
      .then((r) => r.json())
      .then((data) => setPageviews(data.pageviews))
      .catch(() => setPageviews(null));
  }, []);

  const stats = [
    {
      value: pageviews ?? 0,
      label: '累计访问量',
      suffix: '+',
      note: 'EdgeOne KV 实时统计',
    },
    { value: 4, label: '支持 AI 模块', suffix: '' },
    { value: 100, label: '毫秒级响应', suffix: 'ms', note: 'EdgeOne 全球加速' },
  ];

  return (
    <section id="stats" className="py-20 px-6 bg-black text-white">
      <div className="max-w-7xl mx-auto">
        <motion.h2
          initial={{ opacity: 0, y: 20 }}
          whileInView={{ opacity: 1, y: 0 }}
          viewport={{ once: true }}
          className="text-4xl font-bold mb-16 text-center"
        >
          平台数据
        </motion.h2>

        <div className="grid grid-cols-1 md:grid-cols-3 gap-12 text-center">
          {stats.map((stat, i) => (
            <motion.div
              key={i}
              initial={{ opacity: 0, y: 24 }}
              whileInView={{ opacity: 1, y: 0 }}
              viewport={{ once: true }}
              transition={{ delay: i * 0.15 }}
            >
              <div className="text-7xl font-bold leading-none mb-3">
                <AnimatedCounter value={stat.value} suffix={stat.suffix} />
              </div>
              <div className="text-white/60 text-lg">{stat.label}</div>
              {stat.note && (
                <div className="mt-2 text-xs text-white/30">{stat.note}</div>
              )}
            </motion.div>
          ))}
        </div>
      </div>
    </section>
  );
}
```

---

## 7. `src/components/layout/Footer.tsx`

```tsx
interface FooterProps {
  platformName: string;
  copyright: string;
  friendlyLinks?: { label: string; url: string }[];
}

export function Footer({ platformName, copyright, friendlyLinks = [] }: FooterProps) {
  return (
    <footer className="bg-black text-white pt-16 pb-8 px-6">
      <div className="max-w-7xl mx-auto">
        {/* 上方三列 */}
        <div className="grid grid-cols-1 md:grid-cols-3 gap-12 mb-12 pb-12 border-b border-white/10">
          {/* 左：Logo + 简介 */}
          <div>
            <div className="text-2xl font-bold mb-4">{platformName}</div>
            <p className="text-white/50 text-sm leading-relaxed">
              一站式 AI 能力聚合平台，连接生图、3D、大模型、视频等前沿 AI 技术。
              由 EdgeOne Pages 全球加速。
            </p>
          </div>

          {/* 中：友情链接 */}
          <div>
            <h4 className="text-sm font-semibold text-white/30 uppercase tracking-widest mb-4">
              友情链接
            </h4>
            <ul className="space-y-2">
              <li>
                <a href="https://edgeone.ai" target="_blank" rel="noopener noreferrer"
                  className="text-white/60 text-sm hover:text-white transition-colors">
                  EdgeOne Pages
                </a>
              </li>
              <li>
                <a href="https://cloud.tencent.com" target="_blank" rel="noopener noreferrer"
                  className="text-white/60 text-sm hover:text-white transition-colors">
                  腾讯云
                </a>
              </li>
              {friendlyLinks.map((link) => (
                <li key={link.url}>
                  <a href={link.url} target="_blank" rel="noopener noreferrer"
                    className="text-white/60 text-sm hover:text-white transition-colors">
                    {link.label}
                  </a>
                </li>
              ))}
            </ul>
          </div>

          {/* 右：技术栈 */}
          <div>
            <h4 className="text-sm font-semibold text-white/30 uppercase tracking-widest mb-4">
              技术栈
            </h4>
            <ul className="space-y-2 text-white/60 text-sm">
              <li>React + Vite + TypeScript</li>
              <li>Tailwind CSS + Framer Motion</li>
              <li>EdgeOne Edge Functions</li>
              <li>EdgeOne KV Storage</li>
              <li>EdgeOne Global CDN</li>
            </ul>
          </div>
        </div>

        {/* 底部版权 */}
        <div className="flex flex-col md:flex-row items-center justify-between gap-4">
          <p className="text-white/30 text-xs">{copyright}</p>
          <div className="flex items-center gap-2 text-white/20 text-xs">
            <span>Powered by</span>
            <a href="https://edgeone.ai" target="_blank" rel="noopener noreferrer"
              className="text-white/40 hover:text-white/60 transition-colors font-medium">
              EdgeOne Pages
            </a>
          </div>
        </div>
      </div>
    </footer>
  );
}
```
