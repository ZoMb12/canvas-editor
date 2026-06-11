# Canvas Editor Skill 🎨

> Turn any content module into a free-form drag/resize canvas editor — with alignment guides, preview sync, and scroll animations.

[English](#english) | [中文](#中文) | [For AI Agents](#for-ai-agents)

---

## English

### What is this?

This is a **Claude Code Skill** — a blueprint that teaches AI coding agents how to build a canvas editor. It's not a library you install with npm. Instead, you send this GitHub link to your AI agent, and it builds the canvas system directly into your project.

### What you get

After your AI agent follows this skill, your website will have:

- **🖱️ Drag & Drop** — freely rearrange cards anywhere on a canvas
- **📐 Resize Handles** — corner grips to scale cards up/down
- **📏 Snap Alignment** — automatic pink guide lines when edges align (like PowerPoint)
- **👁️ Preview Sync** — what you arrange in edit mode is exactly what visitors see
- **✨ Scroll Animation** — each card slides in individually as users scroll
- **💾 Manual Save — positions save only when you click save; full control

### How to use

**Step 1:** Copy this repo's URL:

```
https://github.com/ZoMb12/canvas-editor
```

**Step 2:** Paste it into Claude Code (or any AI coding tool that supports skills):

```
/install-skill https://github.com/ZoMb12/canvas-editor
```

**Step 3:** Tell your AI what module to convert:

> *"Use the canvas-editor skill to turn the Featured Products section into a free canvas editor."*

The AI will read the skill, analyze your project's tech stack, and build
everything — choosing the right storage backend, adapting to your CSS
framework, and matching your code conventions.

### What you need

- A web project (React, Vue, Next.js, plain HTML — anything works)
- Claude Code or another AI coding tool with skill support
- That's it. No dependencies, no API keys.

### How it works (30 seconds)

```
Your project
  ├── A static product grid (boring)
  │
  ▼ AI reads canvas-editor skill
  │
  ├── Edit mode: drag, resize, snap, arrange freely
  ├── Save button → positions stored
  ├── Preview mode: same layout, animated entrance
  │
  ▼ Visitors see your beautiful layout
```

The skill doesn't force a specific database or styling. Your AI figures out
what fits your project — Supabase, localStorage, REST API, Tailwind, CSS
Modules, whatever you already use.

### Project Structure

```
canvas-editor/
├── SKILL.md                          ← Main guide (AI reads this)
└── references/
    ├── architecture.md               ← Component tree & data flow
    ├── interactions.md               ← Drag/resize/snap algorithms
    └── preview-sync.md               ← Edit ↔ preview coordinate math
```

---

## 中文

### 这是什么？

这是一个 **Claude Code 技能** —— 一份教 AI 编程助手如何搭建画布编辑器的蓝图。它不是用 npm 安装的库。你只需要把这个 GitHub 链接丢给你的 AI 助手，它就会直接把画布系统构建到你的项目里。

### 你能得到什么

AI 助手按照这个技能操作后，你的网站将拥有：

- **🖱️ 自由拖拽** — 在画布上任意拖动卡片排列
- **📐 四角缩放** — 拖拽卡片四角调整大小
- **📏 吸附对齐** — 贴近时自动出现粉色对齐线（像 PPT 一样）
- **👁️ 预览同步** — 编辑模式排好的布局，预览模式一模一样
- **✨ 滚动动画** — 每张卡片在用户滚动时独立滑入
- **💾 手动保存 — 点击保存按钮才写入；完全控制

### 怎么用

**第一步：** 复制这个仓库的地址：

```
https://github.com/ZoMb12/canvas-editor
```

**第二步：** 在 Claude Code（或任何支持 skill 的 AI 编程工具）中输入：

```
/install-skill https://github.com/ZoMb12/canvas-editor
```

**第三步：** 告诉 AI 你要改造哪个模块：

> *"使用 canvas-editor 技能，把精选推荐模块改成自由画布编辑模式。"*

AI 会读取技能指南，分析你项目的技术栈，然后自己搞定一切——
选择合适的存储方案、适配你的 CSS 框架、遵循你的代码风格。

### 你需要什么

- 一个网页项目（React、Vue、Next.js、原生 HTML 都行）
- Claude Code 或其他支持 skill 的 AI 编程工具
- 就这些。不需要额外依赖，不需要 API 密钥。

### 一分钟理解原理

```
你的项目
  ├── 一个静态的卡片网格（无聊）
  │
  ▼ AI 读取 canvas-editor 技能
  │
  ├── 编辑模式：拖拽、缩放、对齐、自由排列
  ├── 保存按钮 → 位置数据持久化
  ├── 预览模式：相同布局、动画入场
  │
  ▼ 访客看到你设计的精美布局
```

技能不会强制你用特定数据库或样式方案。你的 AI 会自动判断什么适合你的项目——
Supabase、localStorage、REST API、Tailwind、CSS Modules，你用什么它就适配什么。

---

## For AI Agents

> **You are an AI coding agent. Read this section to quickly understand this skill.**

### TL;DR

This skill teaches you to build a **free-form canvas editor** in a web
project. Users drag/resize cards on a canvas, snap to alignment guides, and
the layout syncs to a preview mode. It's framework-agnostic and
storage-agnostic — you choose the right implementation for the project.

### What to do

1. Read `SKILL.md` first — it has the complete implementation guide with 7 phases
2. Reference files give you algorithms and architecture details when needed:
   - `references/architecture.md` — component tree, state shape, event flow
   - `references/interactions.md` — drag/resize/snap/text-drag pseudocode
   - `references/preview-sync.md` — coordinate math for edit↔preview sync
3. Adapt everything to the project's actual tech stack (React/Vue/etc,
   Tailwind/CSS/etc, Supabase/API/localStorage)
4. Follow the implementation checklist at the bottom of SKILL.md

### Key things NOT to do

- ❌ Do NOT copy-paste code literally — it's pseudocode for illustration
- ❌ Do NOT hardcode a storage backend — the project already has one
- ❌ Do NOT use `setPointerCapture` for drag — it breaks everything (read why in interactions.md)
- ❌ Do NOT put resize handles inside `overflow-hidden rounded` containers
- ❌ Do NOT use a single IntersectionObserver for the entire preview

### Quick architecture

```
State: positions[] (5 cards), textPos (label + title)
Edit:  absolute px positioning + pointer events on canvas
Preview: percentage positioning with unified coordinate basis
Storage: abstract interface → you implement with project's DB/API
```

### File map

| File | Read when |
|------|-----------|
| `SKILL.md` | Always — start here |
| `references/architecture.md` | Need component structure or event flow clarity |
| `references/interactions.md` | Implementing drag/resize/snap handlers |
| `references/preview-sync.md` | Edit↔preview layout doesn't match (debugging) |
