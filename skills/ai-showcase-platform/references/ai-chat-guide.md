# AI 助手对话页面规范

本文档详细描述 AI 助手对话页面的实现规范。

---

## 1. 页面概述

**路径**：`/ai-chat`

**功能**：多模型 AI 对话界面，用户可以选择不同的 AI 模型进行对话。

**是否需要登录**：否（降低使用门槛，无需登录即可体验）

**数据存储**：内存中，刷新页面丢失（Mock 实现）

---

## 2. 路由配置

```tsx
// src/App.tsx
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import AIChatPage from './pages/AIChatPage';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        {/* ... 其他路由 */}
        <Route path="/ai-chat" element={<AIChatPage />} />
        {/* ... 其他路由 */}
      </Routes>
    </BrowserRouter>
  );
}
```

---

## 3. 页面组件结构

```tsx
// src/pages/AIChatPage.tsx
import { useState, useRef, useEffect } from 'react';
import NavBar from '../components/NavBar';

interface Message {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  model?: string;
  timestamp: Date;
}

interface Model {
  id: string;
  name: string;
  icon: string;
  color: string;
  description: string;
}

const AIChatPage = () => {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');
  const [selectedModel, setSelectedModel] = useState('gpt-4');
  const [isLoading, setIsLoading] = useState(false);
  const messagesEndRef = useRef<HTMLDivElement>(null);

  // 自动滚动到底部
  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  // 初始欢迎消息
  useEffect(() => {
    if (messages.length === 0) {
      setMessages([
        {
          id: 'welcome',
          role: 'assistant',
          content: '你好！我是 AI 助手。请选择上方的模型，然后开始对话吧！',
          model: selectedModel,
          timestamp: new Date(),
        },
      ]);
    }
  }, []);

  return (
    <div className="min-h-screen bg-white">
      <NavBar />
      <div className="max-w-4xl mx-auto px-4 py-8">
        {/* 页面标题 */}
        <div className="mb-6">
          <h1 className="text-3xl font-bold text-black">AI 助手</h1>
          <p className="text-gray-600 mt-2">选择模型，开始智能对话</p>
        </div>

        {/* 模型选择器 */}
        <ModelSelector
          selected={selectedModel}
          onChange={setSelectedModel}
        />

        {/* 聊天区域 */}
        <div className="bg-gray-50 rounded-2xl p-4 mb-4 min-h-[400px] max-h-[60vh] overflow-y-auto">
          {messages.map((msg) => (
            <MessageBubble key={msg.id} message={msg} />
          ))}
          {isLoading && <LoadingIndicator />}
          <div ref={messagesEndRef} />
        </div>

        {/* 输入区域 */}
        <ChatInput
          value={input}
          onChange={setInput}
          onSend={handleSend}
          disabled={isLoading}
        />
      </div>
    </div>
  );
};
```

---

## 4. 模型选择器组件

```tsx
// src/components/ModelSelector.tsx
interface ModelSelectorProps {
  selected: string;
  onChange: (modelId: string) => void;
}

const models: Model[] = [
  {
    id: 'gpt-4',
    name: 'GPT-4',
    icon: 'ri-robot-line',
    color: '#10a37f',
    description: 'OpenAI 最强模型',
  },
  {
    id: 'claude-3',
    name: 'Claude 3',
    icon: 'ri-brain-line',
    color: '#d4a574',
    description: 'Anthropic 推理助手',
  },
  {
    id: 'gemini',
    name: 'Gemini',
    icon: 'ri-sparkling-line',
    color: '#8e44ad',
    description: 'Google 多模态 AI',
  },
  {
    id: 'llama-3',
    name: 'Llama 3',
    icon: 'ri-bear-smile-line',
    color: '#f39c12',
    description: 'Meta 开源模型',
  },
];

const ModelSelector = ({ selected, onChange }: ModelSelectorProps) => {
  const [isOpen, setIsOpen] = useState(false);
  const selectedModel = models.find((m) => m.id === selected) || models[0];

  return (
    <div className="relative mb-6">
      <button
        onClick={() => setIsOpen(!isOpen)}
        className="flex items-center gap-3 px-4 py-3 bg-white border-2 border-black rounded-xl hover:bg-gray-50 transition-colors"
      >
        <div
          className="w-10 h-10 rounded-full flex items-center justify-center"
          style={{ backgroundColor: `${selectedModel.color}20` }}
        >
          <i
            className={`${selectedModel.icon} text-xl`}
            style={{ color: selectedModel.color }}
          />
        </div>
        <div className="text-left">
          <div className="font-semibold text-black">{selectedModel.name}</div>
          <div className="text-sm text-gray-500">{selectedModel.description}</div>
        </div>
        <i className={`ri-arrow-${isOpen ? 'up' : 'down'}-s-line ml-auto`} />
      </button>

      {isOpen && (
        <div className="absolute top-full left-0 mt-2 w-full bg-white border-2 border-black rounded-xl overflow-hidden z-10">
          {models.map((model) => (
            <button
              key={model.id}
              onClick={() => {
                onChange(model.id);
                setIsOpen(false);
              }}
              className={`w-full flex items-center gap-3 px-4 py-3 hover:bg-gray-50 transition-colors ${
                model.id === selected ? 'bg-gray-100' : ''
              }`}
            >
              <div
                className="w-10 h-10 rounded-full flex items-center justify-center"
                style={{ backgroundColor: `${model.color}20` }}
              >
                <i className={model.icon} style={{ color: model.color }} />
              </div>
              <div className="text-left flex-1">
                <div className="font-semibold text-black">{model.name}</div>
                <div className="text-sm text-gray-500">{model.description}</div>
              </div>
              {model.id === selected && (
                <i className="ri-check-line text-green-600" />
              )}
            </button>
          ))}
        </div>
      )}
    </div>
  );
};
```

---

## 5. 消息气泡组件

```tsx
// src/components/MessageBubble.tsx
interface MessageBubbleProps {
  message: Message;
}

const MessageBubble = ({ message }: MessageBubbleProps) => {
  const isUser = message.role === 'user';

  return (
    <div className={`flex items-start gap-3 mb-4 ${isUser ? 'flex-row-reverse' : ''}`}>
      {/* 头像 */}
      <div
        className={`w-10 h-10 rounded-full flex items-center justify-center flex-shrink-0 ${
          isUser ? 'bg-black' : 'bg-gray-200'
        }`}
      >
        <i
          className={`${isUser ? 'ri-user-line' : 'ri-robot-line'} text-lg ${
            isUser ? 'text-white' : 'text-gray-600'
          }`}
        />
      </div>

      {/* 消息内容 */}
      <div
        className={`max-w-[70%] px-4 py-3 rounded-2xl ${
          isUser
            ? 'bg-gradient-to-r from-[#00ff88] to-[#6600ff] text-white rounded-tr-none'
            : 'bg-white border border-gray-200 text-black rounded-tl-none'
        }`}
      >
        <p className="whitespace-pre-wrap">{message.content}</p>
        <span
          className={`text-xs mt-1 block ${isUser ? 'text-white/70' : 'text-gray-400'}`}
        >
          {message.model && !isUser
            ? `via ${message.model.toUpperCase()}`
            : ''}
        </span>
      </div>
    </div>
  );
};
```

---

## 6. 聊天输入组件

```tsx
// src/components/ChatInput.tsx
interface ChatInputProps {
  value: string;
  onChange: (value: string) => void;
  onSend: (message: string) => void;
  disabled?: boolean;
}

const ChatInput = ({ value, onChange, onSend, disabled }: ChatInputProps) => {
  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      if (value.trim() && !disabled) {
        onSend(value);
      }
    }
  };

  return (
    <div className="flex gap-3">
      <textarea
        value={value}
        onChange={(e) => onChange(e.target.value)}
        onKeyDown={handleKeyDown}
        placeholder="输入你的问题..."
        rows={1}
        disabled={disabled}
        className="flex-1 px-4 py-3 border-2 border-black rounded-xl resize-none focus:outline-none focus:border-[#6600ff] transition-colors disabled:bg-gray-100"
      />
      <button
        onClick={() => value.trim() && onSend(value)}
        disabled={!value.trim() || disabled}
        className="px-6 py-3 bg-gradient-to-r from-[#00ff88] to-[#6600ff] text-white font-semibold rounded-xl hover:opacity-90 transition-opacity disabled:opacity-50 disabled:cursor-not-allowed"
      >
        <i className="ri-send-plane-fill text-lg" />
      </button>
    </div>
  );
};
```

---

## 7. 加载指示器

```tsx
// src/components/LoadingIndicator.tsx
const LoadingIndicator = () => {
  return (
    <div className="flex items-start gap-3 mb-4">
      <div className="w-10 h-10 rounded-full bg-gray-200 flex items-center justify-center">
        <i className="ri-robot-line text-lg text-gray-600" />
      </div>
      <div className="bg-white border border-gray-200 px-4 py-3 rounded-2xl rounded-tl-none">
        <div className="flex gap-1">
          <span className="w-2 h-2 bg-gray-400 rounded-full animate-bounce" />
          <span className="w-2 h-2 bg-gray-400 rounded-full animate-bounce" style={{ animationDelay: '0.1s' }} />
          <span className="w-2 h-2 bg-gray-400 rounded-full animate-bounce" style={{ animationDelay: '0.2s' }} />
        </div>
      </div>
    </div>
  );
};
```

---

## 8. Mock 响应逻辑

```tsx
// src/lib/mockResponses.ts

interface AIModel {
  id: string;
  name: string;
}

const mockResponses: Record<string, string[]> = {
  'gpt-4': [
    '你好！我是 GPT-4，很高兴为你服务。作为 OpenAI 最强大的语言模型，我可以帮你解答问题、写作、编程、分析数据等。有什么我可以帮助你的吗？',
    '这是一个很好的问题。让我从多个角度来分析... GPT-4 在推理能力和上下文理解方面表现出色，可以处理复杂的对话场景。',
    '让我帮你分析一下这个问题。根据我的知识库，我可以提供以下几个方向的建议... 你觉得哪个方向更适合你的需求？',
    '作为 GPT-4，我会尽力提供准确、有帮助的信息。如果你有更具体的问题，欢迎继续提问！',
  ],
  'claude-3': [
    '你好！我是 Claude 3。相较于其他模型，我更擅长深度思考、创意写作和复杂的代码分析。有什么我可以帮助你的吗？',
    '这个问题很有趣！让我从不同的角度来思考... Claude 3 在长文本处理和上下文连贯性方面有很好的表现。',
    '作为 Anthropic 的 AI 助手，我会尽量提供深思熟虑的回答。如果你需要创意写作或复杂推理，我很乐意帮忙！',
    '让我来帮你分析这个情况... 我会考虑多种可能性，并给出平衡的观点。',
  ],
  'gemini': [
    '你好！我是 Google Gemini，一个多模态 AI 模型。我可以处理文本、图像、音频等多种类型的数据。有什么我可以帮助你的吗？',
    '有趣的问题！作为 Google 的 AI，Gemini 在信息检索和多模态理解方面有独特优势。让我来帮你分析...',
    'Gemini 结合了 Google 在搜索和 AI 领域的最新技术。我可以帮你查找信息、解答问题、生成内容等！',
    '让我来处理你的请求... 作为 Google 的多模态 AI，Gemini 可以处理各种复杂任务。',
  ],
  'llama-3': [
    '你好！我是 Llama 3，Meta 的开源大语言模型。虽然我是开源模型，但在许多任务上表现出色。有什么我可以帮助你的吗？',
    '作为开源模型的代表，Llama 3 在社区驱动的优化下不断进步。让我来回答你的问题...',
    'Llama 3 是一个开源项目，这意味着任何人都可以使用和改进它。让我来帮你...',
    '开源 AI 的力量！作为 Llama 3，我会提供清晰、实用的回答。有什么具体问题吗？',
  ],
};

export function getMockResponse(modelId: string, userMessage: string): string {
  const responses = mockResponses[modelId] || mockResponses['gpt-4'];
  // 根据用户消息的长度选择不同长度的回复
  const index = Math.min(
    Math.floor(userMessage.length / 50),
    responses.length - 1
  );
  return responses[Math.floor(Math.random() * responses.length)];
}

export function getContextualResponse(modelId: string, userMessage: string): string {
  const lowerMessage = userMessage.toLowerCase();

  // 代码相关问题
  if (lowerMessage.includes('代码') || lowerMessage.includes('code') || lowerMessage.includes('编程')) {
    const codeResponses: Record<string, string> = {
      'gpt-4': '```javascript\n// 示例代码\nconst greeting = "Hello, World!";\nconsole.log(greeting);\n```\n这是一段简单的 JavaScript 代码示例。如果你需要特定功能的代码，请告诉我更多细节！',
      'claude-3': '当然可以！以下是满足你需求的代码结构：\n\n```python\ndef main():\n    print("Hello, World!")\n\nif __name__ == "__main__":\n    main()\n```\nClaude 3 在代码解释和重构方面特别擅长。',
      'gemini': '我可以帮你生成代码！以下是 Google 推荐的最佳实践... 如果需要联网搜索相关资料，我也可以帮忙。',
      'llama-3': '开源社区有丰富的代码资源！以下是基于 Llama 3 最佳实践的代码示例...',
    };
    return codeResponses[modelId] || codeResponses['gpt-4'];
  }

  // 写作相关
  if (lowerMessage.includes('写') || lowerMessage.includes('创作') || lowerMessage.includes('文章')) {
    return '好的，让我来帮你完成这个写作任务。我会注意文章结构、语言表达和逻辑连贯性。请告诉我更多关于主题和风格的细节！';
  }

  // 默认响应
  return getMockResponse(modelId, userMessage);
}
```

---

## 9. 发送消息处理

```tsx
// src/pages/AIChatPage.tsx 中的 handleSend 函数

const handleSend = async (message: string) => {
  if (!message.trim()) return;

  const userMessage: Message = {
    id: `user-${Date.now()}`,
    role: 'user',
    content: message.trim(),
    timestamp: new Date(),
  };

  setMessages((prev) => [...prev, userMessage]);
  setInput('');
  setIsLoading(true);

  // 模拟 AI 响应延迟（1-2秒）
  const delay = 1000 + Math.random() * 1000;

  setTimeout(() => {
    const response = getContextualResponse(selectedModel, message);

    const aiMessage: Message = {
      id: `ai-${Date.now()}`,
      role: 'assistant',
      content: response,
      model: selectedModel,
      timestamp: new Date(),
    };

    setMessages((prev) => [...prev, aiMessage]);
    setIsLoading(false);
  }, delay);
};
```

---

## 10. 样式补充

```css
/* src/index.css 或 Tailwind 配置中添加 */

@layer components {
  .chat-container {
    @apply max-w-4xl mx-auto px-4;
  }

  .message-bubble-user {
    @apply bg-gradient-to-r from-[#00ff88] to-[#6600ff] text-white;
    background-size: 200% 200%;
    animation: gradient-shift 3s ease infinite;
  }

  .message-bubble-ai {
    @apply bg-white border border-gray-200 text-black;
  }

  .model-btn {
    @apply transition-all duration-200;
  }

  .model-btn:hover {
    @apply shadow-lg;
    transform: translateY(-2px);
  }

  .send-btn {
    @apply transition-all duration-200;
  }

  .send-btn:hover:not(:disabled) {
    @apply shadow-lg;
    transform: scale(1.05);
  }
}

@keyframes gradient-shift {
  0%, 100% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
}
```

---

## 11. 可选：扩展真实 API

如需对接真实 AI API，可在 `.env.local` 中配置：

```bash
VITE_OPENAI_API_KEY=sk-xxx
VITE_ANTHROPIC_API_KEY=sk-ant-xxx
VITE_GOOGLE_API_KEY=xxx
```

然后修改 `getContextualResponse` 函数：

```tsx
export async function getAIResponse(
  modelId: string,
  message: string,
  apiKeys: { openai?: string; anthropic?: string; google?: string }
): Promise<string> {
  // 优先使用 Mock
  if (!apiKeys.openai && !apiKeys.anthropic && !apiKeys.google) {
    return getContextualResponse(modelId, message);
  }

  switch (modelId) {
    case 'gpt-4':
      if (!apiKeys.openai) return getContextualResponse(modelId, message);
      return await callOpenAI(apiKeys.openai, message);
    case 'claude-3':
      if (!apiKeys.anthropic) return getContextualResponse(modelId, message);
      return await callAnthropic(apiKeys.anthropic, message);
    case 'gemini':
      if (!apiKeys.google) return getContextualResponse(modelId, message);
      return await callGemini(apiKeys.google, message);
    default:
      return getContextualResponse(modelId, message);
  }
}
```

---

## 12. 移动端适配

```tsx
// AIChatPage 移动端适配
const AIChatPage = () => {
  // ... 现有代码

  return (
    <div className="min-h-screen bg-white flex flex-col">
      <NavBar />
      {/* 移动端：固定高度的聊天区域 */}
      <div className="flex-1 overflow-y-auto p-4">
        {/* 消息列表 */}
      </div>
      {/* 固定底部的输入框 */}
      <div className="sticky bottom-0 bg-white p-4 border-t">
        <ChatInput ... />
      </div>
    </div>
  );
};
```

```css
/* 移动端样式 */
@media (max-width: 640px) {
  .chat-messages {
    max-height: calc(100vh - 140px);
  }

  .message-bubble {
    max-width: 85%;
  }
}
```

---

## 13. 错误处理

```tsx
const handleSend = async (message: string) => {
  try {
    // ... 发送逻辑
  } catch (error) {
    const errorMessage: Message = {
      id: `error-${Date.now()}`,
      role: 'assistant',
      content: '抱歉，发生了错误。请稍后重试。',
      timestamp: new Date(),
    };
    setMessages((prev) => [...prev, errorMessage]);
    setIsLoading(false);
  }
};
```

---

## 14. 文件结构

```
src/
├── pages/
│   └── AIChatPage.tsx        # AI 助手主页面
├── components/
│   ├── ModelSelector.tsx     # 模型选择下拉
│   ├── MessageBubble.tsx      # 消息气泡
│   ├── ChatInput.tsx         # 聊天输入框
│   └── LoadingIndicator.tsx # 加载指示器
└── lib/
    └── mockResponses.ts       # Mock 响应逻辑
```

---

## 15. 验证清单

- [ ] 页面正确渲染，NavBar 正常显示
- [ ] 模型选择器下拉正常，可切换不同模型
- [ ] 发送消息后正确添加用户消息气泡
- [ ] AI 响应延迟效果正常（1-2秒）
- [ ] 不同模型返回不同风格的回复
- [ ] 消息气泡对齐正确（用户右，AI 左）
- [ ] 用户消息有渐变背景
- [ ] 输入框支持 Enter 发送
- [ ] 空消息不能发送
- [ ] 移动端布局正常
- [ ] 滚动到底部功能正常
