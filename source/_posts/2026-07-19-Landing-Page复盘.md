---
title: 从 Prompt 到产品级 Landing Page — 一份写给设计师的前端工程复盘
date: 2026-07-19 20:00:00
categories: 前端
tags:
  - React
  - Tailwind CSS
  - Framer Motion
  - 产品设计
  - 网页设计
  - 交互设计
---

# 从 Prompt 到产品级 Landing Page — 一份写给设计师的前端工程复盘

> 用一份 17KB 的 Prompt，让 AI 从零搭建了一个产品级官网。  
> 这篇文章记录整个过程：设计决策、技术选型、踩坑修复、以及最终交付。

---

## 一、项目背景

为投递**月之暗面（Moonshot AI）网页美学评论实习生**岗位，需要做一个作品集级别的 Landing Page。

核心要求：基于 Notion 2013 年的 "Democratize Software" 理念，重新演绎为一个 2026 年 AI 时代的新产品。

**线上地址**：[nonoxs111.github.io/notion-landing](https://nonoxs111.github.io/notion-landing/)  
**GitHub 仓库**：[Nonoxs111/notion-landing](https://github.com/Nonoxs111/notion-landing)

---

## 二、产品设定

产品定位：一个融合 AI Agent、Visual Editor、Modular Builder、Workflow Automation 的软件创造空间。

核心理念：**Software should be created, not coded.**

无需编程，只需要描述需求。例如"帮我创建一个个人项目管理工具"→ AI 自动生成页面、数据结构、工作流、自动化能力。

---

## 三、设计系统

### 3.1 色彩

| 角色 | 颜色 | 用途 |
|------|------|------|
| 主色 | `#FF4B4A` (Coral Red) | CTA 按钮、重点强调、交互反馈 |
| 背景 | `#FAFAF8` (Warm White) | 主页面背景 |
| 文字 | `#1A1A1A` / `#6B6B6B` | 标题和正文 |
| 边框 | `#E8E8E4` | 卡片和分割线 |

**比例**：80% 白色空间 / 15% 黑灰文字 / 5% 红色强调

关键技巧：背景不用纯白 `#FFFFFF`，而是 `#FAFAF8`——多了一点点暖色，页面立刻有了"高级感"。

### 3.2 字体

| 字体 | 用途 | 感觉 |
|------|------|------|
| Playfair Display (serif) | Hero 标题、Vision 宣言 | 编辑感、权威、人文 |
| Inter (sans-serif) | 正文、按钮、标签 | 清晰、现代、高可读性 |

为什么用 Playfair Display？大部分 AI 产品官网用几何无衬线字体（如 SF Pro），但我们走编辑杂志风格——这让页面在同类产品中立刻不同。

### 3.3 叙事结构

页面不是功能列表，而是一个**叙事弧线**：

```
Hero       →  "这是未来"        （情感共鸣）
Problem    →  "现在有问题"      （痛点确认）
Solution   →  "我们有方案"      （价值主张）
Features   →  "具体能做什么"    （能力证明）
Vision     →  "这是信仰"        （升华、深色背景）
CTA        →  "开始吧"          （行动转化）
```

Vision 安排在 Features 之后，而且是全页面唯一的深色背景区域。因为浏览到页面下半部用户注意力下降，深色切换能重新抓住注意力。

---

## 四、技术栈

| 层 | 选择 | 理由 |
|---|------|------|
| 框架 | React 18 | 组件化，状态管理灵活 |
| 构建 | Vite | 比 CRA 快 10 倍 |
| 样式 | Tailwind CSS 3 | 设计 tokens 落地到代码 |
| 动画 | Framer Motion | React 生态最好的动画库 |
| 图标 | Lucide React | 轻量、现代 |
| 部署 | GitHub Pages | 免费、无需注册 |

---

## 五、核心组件实现

### 5.1 Product Demo — 整个页面的灵魂

Demo 经历了三次重写：

**V1**：假输入框 → 打字动画 → 5 张彩色卡片弹出  
**问题**：用户输入是假的（总是生成同样的东西）；卡片展开时把整排卡片挤飞；看不出产品到底能干什么

**V2**：Block 调色板 + 画布，点击添加模块  
**问题**：JSON 序列化丢失函数导致拖拽失效；模块预览看不懂（灰杠杠）；画布是网格不是页面

**V3（最终版）**：竖排页面布局的 Block 构建器

```
┌─────────────┬──────────────────────────────────┐
│  Blocks     │                                  │
│             │  📊 Table — 任务数据表格         │
│  📊 Table   │  ┌──────────────────────────┐   │
│  📋 Board   │  │ Task name │ Status │ ... │   │
│  📅 Cal     │  │ Design   │ Done   │ ... │   │
│  📝 Doc     │  └──────────────────────────┘   │
│  📊 Timeline│                                  │
│             │  📋 Board — 看板视图              │
│  ─────────  │  [To Do] [In Progress] [Done]   │
│  ✨ AI Fill │                                  │
│             │  📅 Calendar — 七月日历          │
│             │  📝 Document — 富文本文档        │
└─────────────┴──────────────────────────────────┘
```

**交互方式**：

- **拖拽**：从左侧拖 Block 到右侧画布（HTML5 Drag & Drop API）
- **点击添加**：点击左侧 Block 直接添加
- **拖拽排序**：画布内 Block 可上下重排（Framer Motion Reorder）
- **展开详情**：点击 Block → 浮层放大（fixed overlay，不影响布局）
- **删除**：hover 出现 × 按钮
- **AI Fill**：一键填充完整预设（项目管理/知识库）

**关键 Bug 修复**：拖拽使用 `JSON.stringify` 传递数据时会丢失函数（`preview`），导致 Block 拖过去是空壳。修复方式是只传 `blockId`，drop 时从原数组按 id 查找完整定义。

### 5.2 滚动动画

每个 Section 使用 Framer Motion 的 `useInView`：

```jsx
const ref = useRef(null);
const isInView = useInView(ref, { once: true, margin: '-80px' });

<motion.div
  ref={ref}
  initial={{ opacity: 0, y: 24 }}
  animate={isInView ? { opacity: 1, y: 0 } : {}}
/>
```

- `once: true` — 动画只触发一次
- `margin: '-80px'` — 元素进入视口前 80px 就触发，用户看到时动画已完成

### 5.3 按钮滚动到 Demo

三个"Start Building"按钮点击后平滑滚动到 Demo 区域。

**踩坑**：CSS `scroll-behavior: smooth` + JS `scrollIntoView({ behavior: 'smooth' })` 叠加会导致浏览器滚动动画期间鼠标滚轮被吞掉，用户无法中断滚动回到顶部。

**解决方案**：移除 CSS `scroll-behavior: smooth`，用 `requestAnimationFrame` 手写缓动动画，监听 `wheel` 事件随时取消：

```javascript
export function scrollToElement(elementId, offset = 96) {
  const el = document.getElementById(elementId);
  const target = el.getBoundingClientRect().top + window.scrollY - offset;
  // easeOutCubic + requestAnimationFrame
  // 监听 wheel 事件，用户转动滚轮立即取消
}
```

---

## 六、项目结构

```
notion-landing/
├── index.html
├── tailwind.config.js          ← 设计系统（颜色、字体、间距 tokens）
├── .github/workflows/deploy.yml ← GitHub Pages 自动部署
└── src/
    ├── main.jsx
    ├── App.jsx                  ← 根组件，7 个 Section 纵向排列
    ├── index.css                ← Tailwind + 全局质感样式
    ├── utils/
    │   └── scroll.js            ← 可中断的平滑滚动工具
    ├── components/
    │   ├── Navbar.jsx           ← 固定导航（滚动透明→毛玻璃）
    │   ├── Button.jsx           ← 动画按钮（Framer Motion）
    │   ├── SectionBadge.jsx     ← 区块标签
    │   ├── Footer.jsx
    │   └── demo/
    │       └── ProductDemo.jsx  ← ★ 核心交互 Demo
    └── sections/
        ├── Hero.jsx             ← 大标题 + Demo
        ├── Problem.jsx          ← 左文案右卡片
        ├── Solution.jsx         ← 5 支柱网格
        ├── Features.jsx         ← 2×2 功能卡片
        ├── Vision.jsx           ← 深色宣言（唯一深色区块）
        └── CTA.jsx              ← 行动号召 + Footer
```

---

## 七、为什么它看起来"像官网"

| 要素 | 做法 |
|------|------|
| **留白** | 每个 Section 间距 120px+，留白是设计元素不是空着的地方 |
| **色彩克制** | 红色只出现在 CTA 按钮和极少数强调处 |
| **字体分工** | Playfair Display（情感）+ Inter（信息），两套字体各司其职 |
| **动画有原因** | 卡片弹簧动画 = 实体感，打字机 = 自然感，而不是炫技 |
| **叙事有结构** | 不是罗列功能，是说一个完整的故事 |
| **双语不打架** | 英文是视觉主语言，中文小字辅助 |
| **深色只出现一次** | Vision 区块制造页面"高潮"，用完就回到浅色 |

---

## 八、问题与解决汇总

| # | 问题 | 原因 | 解决方案 |
|---|------|------|---------|
| 1 | Demo 卡片展开挤飞布局 | 内联膨胀改变文档流 | 改为 fixed overlay 浮层 |
| 2 | 拖拽 Block 到画布后消失 | `JSON.stringify` 丢失 `preview` 函数 | 传 `blockId`，drop 时从原数组还原 |
| 3 | 彩色模块 AI 味重 | 蓝绿紫琥珀粉五种颜色 | 统一珊瑚红 + 黑白灰 |
| 4 | 预设按钮点击无反应 | `applyPreset` 中 blocks 数组引用错误 | 重写为按 id 查找定义 |
| 5 | Document Block 看不懂 | 灰色占位条太抽象 | 改为带标题、段落、列表、关联数据库的真实文档 |
| 6 | Table Block 有 emoji | `✅ Done` `🔄 In progress` | 纯文字状态标签 |
| 7 | 点击按钮后滚轮失效 | CSS `scroll-behavior: smooth` + JS `scrollIntoView` 冲突 | 移除 CSS smooth，手写可中断动画 |
| 8 | 三个按钮点不动 | 按钮没有 onClick | 统一跳转到 `#demo` 锚点 |
| 9 | npm ci 跨平台失败 | Windows 生成的 lock 文件缺 Linux 平台依赖 | 改用 `npm install`，lock 文件不提交 |

---

## 九、部署

```bash
# 本地开发
cd notion-landing
npm install
npm run dev        # → http://localhost:5173

# 生产构建
npm run build      # → dist/

# 推送即部署（GitHub Actions 自动构建到 GitHub Pages）
git push
```

部署后在仓库 Settings → Pages 选择 GitHub Actions 源，线上地址：  
`https://nonoxs111.github.io/notion-landing/`

---

## 十、总结

这个项目最核心的收获：

1. **Prompt 即设计文档**：一份好的 Prompt 里包含了视觉规范、信息架构、交互预期的全部信息。关键在于把它们翻译成代码决策。

2. **动画是功能不是装饰**：每个动画都要有原因——弹簧 = 实体感、打字机 = 自然感、fade up = 引导阅读节奏。

3. **Demo 是信任的桥梁**：一个真实可交互的 Demo 比 1000 字的产品描述更有说服力。但前提是它必须是真的，不能是假的。

4. **色彩统一 > 色彩丰富**：AI 产品最容易犯的错误就是"每个模块一个颜色"。统一的色调才有品牌感。

5. **先修 Bug 再加功能**：拖拽失效、按钮没反应、滚动锁定——这些看似小问题，对用户体验是致命的。

---

> 从一份 Prompt 到一个产品级 Landing Page。  
> 写好 Prompt，AI 可以是一个很好的前端搭档。
