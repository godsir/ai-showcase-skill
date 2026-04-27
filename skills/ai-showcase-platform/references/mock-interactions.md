# Mock 交互动效规范

本文档定义 Demo 区各 AI 模块的 Mock 交互界面实现规范。
Demo 区整体背景为 `#000`（黑色），文字为 `#fff`，体现科技感。

---

## Demo 区容器结构

```tsx
// src/components/sections/DemoSection.tsx
import { useState } from 'react';
import { motion, AnimatePresence } from 'framer-motion';
import { AI_MODULES } from '@/lib/modules';
import { ImageGenDemo } from '@/components/demo/ImageGenDemo';
import { ModelGenDemo } from '@/components/demo/ModelGenDemo';
import { LLMChatDemo } from '@/components/demo/LLMChatDemo';
import { VideoGenDemo } from '@/components/demo/VideoGenDemo';

const DEMO_MAP: Record<string, React.ComponentType> = {
  'image-gen': ImageGenDemo,
  '3d-model': ModelGenDemo,
  'llm-chat': LLMChatDemo,
  'video-gen': VideoGenDemo,
};

export function DemoSection() {
  const [activeId, setActiveId] = useState('image-gen');
  const ActiveDemo = DEMO_MAP[activeId] ?? ImageGenDemo;
  const activeModule = AI_MODULES.find((m) => m.id === activeId)!;

  return (
    <section id="demo" className="py-20 bg-black text-white">
      <div className="max-w-7xl mx-auto px-6">
        <h2 className="text-4xl font-bold mb-12">交互 Demo</h2>

        <div className="flex flex-col lg:flex-row gap-8">
          {/* 左侧 Tab 选择 */}
          <div className="lg:w-56 flex lg:flex-col gap-2 overflow-x-auto lg:overflow-visible">
            {AI_MODULES.map((m) => (
              <button
                key={m.id}
                onClick={() => setActiveId(m.id)}
                className="flex-shrink-0 px-4 py-3 rounded-xl text-left transition-all duration-200"
                style={{
                  background: activeId === m.id
                    ? m.themeColorHex + '20'
                    : 'transparent',
                  borderLeft: activeId === m.id
                    ? `3px solid ${m.themeColorHex}`
                    : '3px solid transparent',
                  color: activeId === m.id ? '#fff' : 'rgba(255,255,255,0.4)',
                }}
              >
                <div className="font-medium text-sm">{m.name}</div>
                <div className="text-xs opacity-60 mt-0.5">{m.nameEn}</div>
              </button>
            ))}
          </div>

          {/* 右侧 Demo 面板 */}
          <div className="flex-1 min-h-[400px] rounded-2xl overflow-hidden border border-white/10">
            <AnimatePresence mode="wait">
              <motion.div
                key={activeId}
                initial={{ opacity: 0, x: 20 }}
                animate={{ opacity: 1, x: 0 }}
                exit={{ opacity: 0, x: -20 }}
                transition={{ duration: 0.25 }}
                className="h-full"
              >
                <ActiveDemo />
              </motion.div>
            </AnimatePresence>
          </div>
        </div>
      </div>
    </section>
  );
}
```

---

## 1. 生图 Demo — `ImageGenDemo.tsx`

```tsx
import { useState } from 'react';
import { Sparkles, Loader2 } from 'lucide-react';

const MOCK_PROMPTS = [
  'A futuristic city at night with neon lights and flying cars',
  '赛博朋克风格的东京街头，霓虹灯倒映在雨水中',
  'Abstract digital art with flowing particles and electric colors',
];

const MOCK_IMAGES = [
  { label: '赛博都市', desc: '霓虹色调，未来感十足的城市夜景' },
  { label: '量子花园', desc: '数字粒子构成的超现实花卉场景' },
  { label: '星际旅人', desc: '宇宙背景下的孤独行者剪影' },
  { label: '流光代码', desc: '代码雨与都市轮廓的双重曝光' },
];

export function ImageGenDemo() {
  const [prompt, setPrompt] = useState('');
  const [loading, setLoading] = useState(false);
  const [generated, setGenerated] = useState(false);

  const handleGenerate = () => {
    if (!prompt.trim()) return;
    setLoading(true);
    setGenerated(false);
    setTimeout(() => {
      setLoading(false);
      setGenerated(true);
    }, 2000);
  };

  return (
    <div className="p-6 bg-zinc-900 h-full flex flex-col gap-6">
      <div className="flex items-center gap-2 text-[#00ff88]">
        <Sparkles size={16} />
        <span className="text-sm font-medium">Image Generation</span>
      </div>

      {/* 输入区 */}
      <div className="flex gap-3">
        <input
          value={prompt}
          onChange={(e) => setPrompt(e.target.value)}
          placeholder="描述你想生成的图像..."
          className="flex-1 bg-white/5 border border-white/10 rounded-xl px-4 py-3 text-white text-sm placeholder-white/30 focus:outline-none focus:border-[#00ff88]/50"
          onKeyDown={(e) => e.key === 'Enter' && handleGenerate()}
        />
        <button
          onClick={handleGenerate}
          disabled={loading}
          className="px-5 py-3 bg-[#00ff88] text-black font-medium rounded-xl text-sm flex items-center gap-2 hover:bg-[#00ff88]/80 transition-colors disabled:opacity-60"
        >
          {loading ? <Loader2 size={16} className="animate-spin" /> : '生成'}
        </button>
      </div>

      {/* 快捷 Prompt */}
      <div className="flex flex-wrap gap-2">
        {MOCK_PROMPTS.map((p) => (
          <button
            key={p}
            onClick={() => setPrompt(p)}
            className="px-3 py-1 bg-white/5 border border-white/10 rounded-full text-xs text-white/50 hover:text-white hover:border-white/30 transition-colors"
          >
            {p.substring(0, 20)}...
          </button>
        ))}
      </div>

      {/* 生成结果 */}
      {generated && (
        <div className="grid grid-cols-2 gap-3">
          {MOCK_IMAGES.map((img) => (
            <div
              key={img.label}
              className="aspect-square rounded-xl border border-white/10 flex flex-col items-center justify-center gap-2"
              style={{
                background: 'linear-gradient(135deg, rgba(0,255,136,0.08), rgba(102,0,255,0.12))',
              }}
            >
              <div className="text-2xl">🖼️</div>
              <div className="text-white text-xs font-medium">{img.label}</div>
              <div className="text-white/30 text-xs text-center px-3">{img.desc}</div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
```

---

## 2. 3D 模型 Demo — `ModelGenDemo.tsx`

```tsx
import { useEffect, useRef } from 'react';

export function ModelGenDemo() {
  const canvasRef = useRef<HTMLCanvasElement>(null);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    let angle = 0;
    let raf: number;

    const draw = () => {
      const W = canvas.width;
      const H = canvas.height;
      ctx.clearRect(0, 0, W, H);

      // 旋转线框立方体（纯 CSS 3D 线框投影）
      const size = Math.min(W, H) * 0.3;
      const cx = W / 2;
      const cy = H / 2;

      const vertices = [
        [-1, -1, -1], [1, -1, -1], [1, 1, -1], [-1, 1, -1],
        [-1, -1,  1], [1, -1,  1], [1, 1,  1], [-1, 1,  1],
      ];

      const edges = [
        [0,1],[1,2],[2,3],[3,0],
        [4,5],[5,6],[6,7],[7,4],
        [0,4],[1,5],[2,6],[3,7],
      ];

      const project = ([x, y, z]: number[]) => {
        const cosA = Math.cos(angle);
        const sinA = Math.sin(angle);
        const cosB = Math.cos(angle * 0.6);
        const sinB = Math.sin(angle * 0.6);
        // Rotate Y
        const x1 = x * cosA - z * sinA;
        const z1 = x * sinA + z * cosA;
        // Rotate X
        const y2 = y * cosB - z1 * sinB;
        const z2 = y * sinB + z1 * cosB;
        const fov = 4;
        const scale = size / (z2 + fov);
        return [cx + x1 * scale, cy + y2 * scale];
      };

      // 渐变描边
      const grad = ctx.createLinearGradient(cx - size, cy, cx + size, cy);
      grad.addColorStop(0, '#6600ff');
      grad.addColorStop(0.5, '#00ff88');
      grad.addColorStop(1, '#ff00aa');

      ctx.strokeStyle = grad;
      ctx.lineWidth = 1.5;

      edges.forEach(([a, b]) => {
        const [ax, ay] = project(vertices[a]);
        const [bx, by] = project(vertices[b]);
        ctx.beginPath();
        ctx.moveTo(ax, ay);
        ctx.lineTo(bx, by);
        ctx.stroke();
      });

      // 顶点
      ctx.fillStyle = '#00ff88';
      vertices.forEach((v) => {
        const [px, py] = project(v);
        ctx.beginPath();
        ctx.arc(px, py, 2.5, 0, Math.PI * 2);
        ctx.fill();
      });

      angle += 0.012;
      raf = requestAnimationFrame(draw);
    };

    draw();
    return () => cancelAnimationFrame(raf);
  }, []);

  return (
    <div className="p-6 bg-zinc-900 h-full flex flex-col gap-4">
      <div className="flex items-center gap-2 text-[#6600ff]">
        <span className="text-sm font-medium">3D Model Generation</span>
      </div>
      <p className="text-white/40 text-sm">实时渲染线框模型预览 — 由 EdgeOne 边缘加速</p>
      <canvas
        ref={canvasRef}
        width={500}
        height={300}
        className="w-full rounded-xl bg-black/50"
        style={{ maxHeight: '320px' }}
      />
      <div className="flex gap-3 mt-auto">
        {['立方体', '球体', '圆柱'].map((shape) => (
          <button
            key={shape}
            className="flex-1 py-2 text-xs font-medium border border-white/10 rounded-lg text-white/50 hover:border-[#6600ff]/60 hover:text-white transition-colors"
          >
            {shape}
          </button>
        ))}
      </div>
    </div>
  );
}
```

---

## 3. 大模型交互 Demo — `LLMChatDemo.tsx`

```tsx
import { useState } from 'react';
import { Send } from 'lucide-react';

const MOCK_CONVERSATIONS = [
  { role: 'user', text: '帮我写一首关于 AI 未来的短诗' },
  { role: 'assistant', text: '代码如诗，流转于光速之间，\n硅基梦境，编织万千星辰。\n人机共舞，跨越时空边界，\n智能曙光，照亮无尽可能。' },
  { role: 'user', text: '帮我解释量子计算的基本原理' },
  { role: 'assistant', text: '量子计算利用量子比特（qubit）的叠加态和纠缠特性，能够同时处理多种状态，比传统计算机指数级地提升特定问题的求解速度……' },
];

export function LLMChatDemo() {
  const [messages, setMessages] = useState(MOCK_CONVERSATIONS);
  const [input, setInput] = useState('');

  const sendMessage = () => {
    if (!input.trim()) return;
    const userMsg = { role: 'user', text: input };
    const aiMsg = { role: 'assistant', text: '这是一个 Mock 响应示例。在实际集成后，此处将由真实大模型 API 返回内容。当前仅用于展示交互界面效果 ✨' };
    setMessages((prev) => [...prev, userMsg, aiMsg]);
    setInput('');
  };

  return (
    <div className="flex flex-col h-full min-h-[380px] bg-zinc-900">
      <div className="flex items-center gap-2 px-4 py-3 border-b border-white/10 text-[#ffee00]">
        <span className="text-sm font-medium">LLM Chat Interface</span>
        <span className="ml-auto text-xs text-white/30">Mock Mode</span>
      </div>

      {/* 对话列表 */}
      <div className="flex-1 overflow-y-auto p-4 space-y-3">
        {messages.map((msg, i) => (
          <div key={i} className={`flex ${msg.role === 'user' ? 'justify-end' : 'justify-start'}`}>
            <div
              className={`max-w-[80%] px-4 py-2.5 rounded-2xl text-sm whitespace-pre-wrap ${
                msg.role === 'user'
                  ? 'bg-[#ffee00] text-black rounded-br-sm'
                  : 'bg-white/8 text-white/80 rounded-bl-sm border border-white/10'
              }`}
            >
              {msg.text}
            </div>
          </div>
        ))}
      </div>

      {/* 输入区 */}
      <div className="p-4 border-t border-white/10 flex gap-3">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === 'Enter' && sendMessage()}
          placeholder="输入消息..."
          className="flex-1 bg-white/5 border border-white/10 rounded-xl px-4 py-2.5 text-white text-sm placeholder-white/30 focus:outline-none focus:border-[#ffee00]/50"
        />
        <button
          onClick={sendMessage}
          className="p-2.5 bg-[#ffee00] text-black rounded-xl hover:bg-[#ffee00]/80 transition-colors"
        >
          <Send size={16} />
        </button>
      </div>
    </div>
  );
}
```

---

## 4. 视频生成 Demo — `VideoGenDemo.tsx`

```tsx
import { useState, useEffect, useRef } from 'react';
import { Play, Pause, RotateCcw } from 'lucide-react';

export function VideoGenDemo() {
  const [progress, setProgress] = useState(0);
  const [running, setRunning] = useState(false);
  const [done, setDone] = useState(false);
  const intervalRef = useRef<ReturnType<typeof setInterval> | null>(null);

  const STAGES = [
    { threshold: 15, label: '解析场景描述...' },
    { threshold: 35, label: '生成关键帧...' },
    { threshold: 60, label: '帧间插值渲染...' },
    { threshold: 80, label: '音频合成...' },
    { threshold: 100, label: '视频编码输出...' },
  ];

  const currentStage = STAGES.find((s) => progress < s.threshold)?.label ?? '完成！';

  const start = () => {
    setRunning(true);
    setDone(false);
    intervalRef.current = setInterval(() => {
      setProgress((p) => {
        if (p >= 100) {
          clearInterval(intervalRef.current!);
          setRunning(false);
          setDone(true);
          return 100;
        }
        return p + 1.2;
      });
    }, 60);
  };

  const reset = () => {
    clearInterval(intervalRef.current!);
    setProgress(0);
    setRunning(false);
    setDone(false);
  };

  useEffect(() => () => clearInterval(intervalRef.current!), []);

  return (
    <div className="p-6 bg-zinc-900 h-full flex flex-col gap-6">
      <div className="flex items-center gap-2 text-[#ff00aa]">
        <span className="text-sm font-medium">Video Generation</span>
      </div>

      {/* 视频预览占位 */}
      <div
        className="rounded-xl aspect-video flex items-center justify-center relative overflow-hidden"
        style={{
          background: 'linear-gradient(135deg, rgba(255,0,170,0.08), rgba(102,0,255,0.15))',
          border: '1px solid rgba(255,0,170,0.2)',
        }}
      >
        {done ? (
          <div className="text-center">
            <div className="text-4xl mb-2">🎬</div>
            <div className="text-white text-sm font-medium">视频已生成完成</div>
            <div className="text-white/40 text-xs mt-1">1920×1080 · 10s · H.264</div>
          </div>
        ) : running ? (
          <div className="text-center">
            <div className="text-3xl mb-3 animate-pulse">⚙️</div>
            <div className="text-white/60 text-sm">{currentStage}</div>
          </div>
        ) : (
          <div className="text-white/20 text-sm">点击"开始生成"查看模拟进度</div>
        )}
      </div>

      {/* 进度条 */}
      <div>
        <div className="flex justify-between text-xs text-white/40 mb-1.5">
          <span>{running ? currentStage : done ? '生成完成' : '等待开始'}</span>
          <span>{Math.floor(progress)}%</span>
        </div>
        <div className="h-1.5 bg-white/10 rounded-full overflow-hidden">
          <div
            className="h-full rounded-full transition-all duration-100"
            style={{
              width: `${progress}%`,
              background: 'linear-gradient(90deg, #ff00aa, #6600ff)',
            }}
          />
        </div>
      </div>

      {/* 控制按钮 */}
      <div className="flex gap-3 mt-auto">
        <button
          onClick={running ? reset : start}
          disabled={done && !running}
          className="flex-1 py-2.5 flex items-center justify-center gap-2 rounded-xl text-sm font-medium border transition-colors"
          style={{
            background: running ? 'transparent' : '#ff00aa',
            borderColor: '#ff00aa',
            color: running ? '#ff00aa' : '#fff',
          }}
        >
          {running ? <Pause size={14} /> : <Play size={14} />}
          {running ? '停止' : '开始生成'}
        </button>
        <button
          onClick={reset}
          className="px-4 py-2.5 rounded-xl border border-white/10 text-white/40 hover:text-white hover:border-white/30 transition-colors"
        >
          <RotateCcw size={14} />
        </button>
      </div>
    </div>
  );
}
```
