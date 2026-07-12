---
title: Edge插件AI阅读助手开发记录
date: 2026-07-12 20:00:00
categories: 技术
tags:
  - AI
  - Edge扩展
  - React
  - TypeScript
  - 面试
  - WXT
---

> 一款基于 React + WXT + 智谱 GLM-4 的 Edge 浏览器扩展，把 AI 阅读和面试备考嵌入到每一次划词操作里。  
> 项目地址：[github.com/Nonoxs111/ai-reader](https://github.com/Nonoxs111/ai-reader)

---

## 缘起：阅读技术文章时的痛点

准备前端面试的时候，我每天会大量阅读技术文章。过程中有几个很让人烦躁的瞬间：

1. **遇到不懂的概念** → 选中 → 右键搜索 → 打开新标签页 → 读搜索结果 → 切回来……一套操作下来，刚才读到哪了？
2. **想翻译某段英文** → 复制 → 打开翻译工具 → 粘贴 → 等结果 → 切回来……又一次打断。
3. **想判断某个知识点面试考不考** → 这得去牛客/掘金搜一圈，流程更长。
4. **读到重点想标记** → 浏览器没有内置高亮功能，只能开个笔记软件记。

每次切换都打断阅读心流，累积下来效率很低。

于是有了这个项目：**AI 阅读助手**。选中文字，AI 在鼠标旁边弹出，原地给你解释、翻译、分析面试考点，还能高亮标记重点——全程不离开当前页面。

---

## 技术选型

| 层级 | 选型 | 理由 |
|---|---|---|
| 前端框架 | React 19 + TypeScript | 类型安全，组件化开发浮动菜单和 Popup |
| 扩展框架 | [WXT](https://wxt.dev/) | 统一管理 content/background/popup，自带 HMR，一键构建 Chrome/Firefox/Edge |
| 大模型 | 智谱 GLM-4-Flash | 免费额度够用，SSE 流式响应速度快 |
| 正文提取 | Mozilla Readability | 从任意网页提取纯净正文，一键摘要 |
| 测试 | Vitest | Vite 生态，速度快 |
| CI/CD | GitHub Actions | push 自动构建打 zip 包、创建 Release |

选择 WXT 是个好决定。它屏蔽了 Chrome 扩展开发中很多琐碎细节（manifest 配置、content script 注入、跨环境通信），开发体验接近写普通 React 应用。

---

## 架构设计：四层 Agent 模型

设计之初，我参考 AI Agent 的经典范式，把浏览器阅读过程抽象成四个层次：

```
┌─────────────────────────────────────┐
│            感知层 Perception          │
│   监听划词事件、获取选区位置与上下文    │
├─────────────────────────────────────┤
│            记忆层 Memory              │
│   chrome.storage 持久化对话/高亮/历史  │
├─────────────────────────────────────┤
│            决策层 Decision            │
│   识别用户意图，匹配 Prompt 模板       │
├─────────────────────────────────────┤
│            执行层 Execution           │
│   调用 GLM-4 API，SSE 流式渲染结果     │
└─────────────────────────────────────┘
```

这个分层让每个模块职责清晰——感知层只管"用户选中了什么"，决策层只管"用户想干嘛"，执行层只管"调 AI 并展示"。后续要换模型、加功能，改动范围都锁在对应层里。

---

## 核心功能

### 1. 划词浮动菜单

选中页面上的任意文本，鼠标上方立刻弹出操作菜单：

- **解释**：通俗解读技术概念，200 字以内，配合生活化类比
- **翻译**：专业中英互译，保留代码、变量名和 URL
- **面试**：判断是否为前端高频考点，自动生成 3-5 道面试题和回答要点
- **高亮**：五色高亮标记，支持改色和取消

菜单位置动态计算——根据选区坐标和视口边界 clamp，确保不会超出屏幕边缘。

### 2. 面试考点智能分析（核心场景）

这是整个项目最贴合面试备考的功能。选中一个技术术语后点「面试」，AI 会：

1. **先下结论**：是高频考点还是低频考点，附难度星级（⭐ ~ ⭐⭐⭐⭐⭐）
2. **如果是高频**：列出 3-5 道典型面试题
3. **每道题附带回答要点**：不是给完整答案，而是给关键点，逼自己组织语言

ResultCard 组件会解析 AI 返回的结构化文本，渲染成卡片式布局：

```
┌──────────────────────────────────┐
│  🔥 高频考点 · ⭐⭐⭐⭐           │
│                                  │
│  面试题 1：请解释闭包的原理...    │
│  ├ 回答要点：                    │
│  │  • 函数 + 词法作用域          │
│  │  • 垃圾回收与内存泄漏         │
│  │  • 实际应用场景               │
│                                  │
│  面试题 2：闭包在实际项目中的...  │
│  └ ...                          │
└──────────────────────────────────┘
```

这样在读文章的同时就能积累面试素材，不用单独花时间去找面经。

### 3. SSE 流式打字机效果

不是等 AI 全部生成完再显示，而是逐字输出。核心实现是手动解析 SSE（Server-Sent Events）协议：

```typescript
const reader = response.body!.getReader();
const decoder = new TextDecoder();
let buffer = '';

while (true) {
  const { done, value } = await reader.read();
  if (done) break;

  buffer += decoder.decode(value, { stream: true });
  const lines = buffer.split('\n');
  buffer = lines.pop() || '';

  for (const line of lines) {
    if (line.startsWith('data: ')) {
      const data = JSON.parse(line.slice(6));
      appendToStream(data.choices[0].delta.content);
    }
  }
}
```

配合 `AbortController`，用户可以随时关闭卡片取消请求。

### 4. 五色高亮系统

高亮不只是改个背景色。需要解决三个问题：

- **持久化**：页面刷新后高亮还在
- **去重**：同一段文字不能被重复高亮
- **跨页面**：不同标签页各自独立

解决方案是用 **XPath + 文本偏移量** 精确定位，而非依赖 DOM 引用：

```json
{
  "id": "uuid",
  "url": "https://example.com/article",
  "text": "被高亮的文字",
  "color": "yellow",
  "startXPath": "/html/body/div[2]/p[3]/text()",
  "startOffset": 42,
  "endXPath": "/html/body/div[2]/p[3]/text()",
  "endOffset": 58,
  "timestamp": 1720000000000
}
```

页面加载时自动恢复当前 URL 的所有高亮。点击已有高亮还能弹出迷你菜单改颜色或取消。

### 5. Popup 对话窗口 + 一键摘要

点击扩展图标弹出 360×520 的小窗口：

- **一键摘要**：获取当前页面正文，生成三句话总结 + 关键要点列表
- **查询历史**：所有划词操作的历史记录，点击可回溯到当时上下文
- **自由对话**：输入问题直接和 AI 聊

对话历史持久化在 `chrome.storage` 里，关闭浏览器也不丢。

### 6. 追问链

在任意 AI 结果下方都有一个追问输入框。比如 AI 解释了「闭包」，你可以继续问「那在实际项目里什么时候用它？有什么坑？」——形成递进式的理解链条。

---

## 实现中遇到的坑

### Readability 兼容性

某些网站（如掘金）的 `document.cloneNode(true)` 会直接抛异常。做了降级处理：

```typescript
let documentClone = document.cloneNode(true) as Document;
if (!documentClone.querySelector('body')) {
  // 降级：直接从 body.innerText 构造最小可用 DOM
  const text = document.body.innerText;
  ...
}
```

### Markdown 渲染不引入第三方库

AI 返回的是 Markdown，但我不想为这点功能引入 `marked` 之类的库。手写了一个轻量解析器，支持行内代码、加粗、无序列表、有序列表和段落。减少包体积，渲染也更可控。

### XPath 跨浏览器一致性

Edge 和 Chrome 的 XPath 实现有细微差异，尤其是处理 `text()` 节点编号时。花了不少时间调试，最终通过兼容两种编号规则解决。

---

## CI/CD：推送即发布

每次 push 到 main 分支，GitHub Actions 自动：

1. 安装依赖并构建扩展
2. 打包 zip
3. 创建带时间戳的 GitHub Release

不需要手动打包上传，开发流程很顺畅。

---

## 项目结构

```
ai-reader/
├── api/
│   ├── glm.ts                # GLM-4 流式调用
│   ├── prompts.ts            # 解释/翻译/追问/摘要 Prompt 模板
│   └── prompts/interview.ts  # 面试考点专用 Prompt
├── components/
│   ├── TextSelectionMenu/    # 划词浮动菜单
│   ├── ResultCard/           # AI 结果卡片（含面试解析）
│   └── LoadingDots/          # 三点弹跳加载动画
├── entrypoints/
│   ├── content.ts            # Content Script 核心（568 行）
│   ├── background.ts         # Service Worker
│   └── popup/                # 扩展弹窗 UI
└── .github/workflows/        # 自动构建发布
```

---

## 待改进

诚实地列一下当前的技术债务：

1. **API Key 硬编码**：目前直接写在 `glm.ts` 里，生产环境应该做服务端代理或让用户配置
2. **缺少 E2E 测试**：扩展交互流程还没有自动化测试覆盖
3. **Prompt 调优空间**：面试分析的准确率还可以用更精细的 Prompt Engineering 提升

---

## 总结

这个项目不只是一个浏览器扩展，更是一次对「AI 如何融入日常阅读和备考体验」的探索。从四层 Agent 架构设计，到 SSE 流式渲染，到 XPath 高亮持久化，每个功能背后都是一次「用户到底需要什么」的追问。

如果你也在准备前端面试，或者经常阅读技术文章时需要查东西——欢迎试试这个插件，或者直接看[源码](https://github.com/Nonoxs111/ai-reader)自己改。

---

> **技术栈**：React 19 · TypeScript · WXT · 智谱 GLM-4 · Vite · GitHub Actions
>
> **项目地址**：[github.com/Nonoxs111/ai-reader](https://github.com/Nonoxs111/ai-reader)
