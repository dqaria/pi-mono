# Pi Mono 项目架构全景

> 本文档旨在帮助开发者快速了解 `pi-mono` 单仓库（monorepo）的整体架构、核心包职责、依赖关系及关键技术选型。

---

## 一、项目概览

`pi-mono` 是一个 **AI 编程智能体开发平台**，采用 npm workspaces 管理的 TypeScript 单仓库。项目核心提供统一的大模型（LLM）接口层、可扩展的 Agent 运行时，并在此基础上构建了 CLI 编程助手、Web 聊天界面、Slack 机器人等多种交互终端。

| 特性 | 说明 |
|------|------|
| **语言** | 100% TypeScript（ES Modules） |
| **Node 版本** | ≥ 20.0.0 |
| **包管理** | npm workspaces |
| **构建工具** | tsgo（TypeScript 编译） |
| **代码质量** | Biome（格式化 + Lint） |
| **测试框架** | Vitest（主要）/ Node 原生测试（tui） |
| **包作用域** | `@mariozechner/*` |
| **许可证** | MIT |

---

## 二、包架构总览图（ASCII）

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         pi-mono (Monorepo)                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    第 4 层 · 业务编排层                          │    │
│  │                                                                 │    │
│  │   ┌────────────────┐                                            │    │
│  │   │   pi-mom        │  Slack 机器人，将消息委派给编程 Agent      │    │
│  │   │   packages/mom  │                                           │    │
│  │   └───┬──┬──┬───────┘                                           │    │
│  │       │  │  │                                                   │    │
│  └───────┼──┼──┼───────────────────────────────────────────────────┘    │
│          │  │  │                                                        │
│  ┌───────┼──┼──┼───────────────────────────────────────────────────┐    │
│  │       │  │  │      第 3 层 · 应用层                              │    │
│  │       │  │  │                                                   │    │
│  │       │  │  ▼                                                   │    │
│  │       │  │ ┌──────────────────┐    ┌──────────────────┐         │    │
│  │       │  │ │ pi-coding-agent  │    │    pi-web-ui     │         │    │
│  │       │  │ │ packages/        │    │    packages/     │         │    │
│  │       │  │ │   coding-agent   │    │      web-ui      │         │    │
│  │       │  │ │                  │    │                  │         │    │
│  │       │  │ │ CLI 编程助手     │    │ Web 聊天组件库   │         │    │
│  │       │  │ │ (主入口 `pi`)    │    │ (Lit Web组件)    │         │    │
│  │       │  │ └──┬──┬──┬────────┘    └──┬──┬────────────┘         │    │
│  │       │  │    │  │  │               │  │                       │    │
│  │       │  │    │  │  │     ┌─────────┘  │                       │    │
│  │       │  │    │  │  │     │            │                       │    │
│  └───────┼──┼────┼──┼──┼─────┼────────────┼───────────────────────┘    │
│          │  │    │  │  │     │            │                             │
│  ┌───────┼──┼────┼──┼──┼─────┼────────────┼───────────────────────┐    │
│  │       │  │    │  │  │     │            │  第 2 层 · Agent 核心  │    │
│  │       │  │    │  │  │     │            │                       │    │
│  │       ▼  │    ▼  │  │     │            │                       │    │
│  │  ┌───────────────────┐    │            │   ┌────────────────┐  │    │
│  │  │  pi-agent-core    │    │            │   │    pi-pods     │  │    │
│  │  │  packages/agent   │    │            │   │  packages/pods │  │    │
│  │  │                   │    │            │   │                │  │    │
│  │  │  Agent 运行时     │    │            │   │ GPU Pod 管理   │  │    │
│  │  │  (工具调用/状态)  │    │            │   │   CLI 工具     │  │    │
│  │  └──────┬────────────┘    │            │   └──────┬─────────┘  │    │
│  │         │                 │            │          │            │    │
│  └─────────┼─────────────────┼────────────┼──────────┼────────────┘    │
│            │                 │            │          │                  │
│  ┌─────────┼─────────────────┼────────────┼──────────┼────────────┐    │
│  │         │                 │            │          │             │    │
│  │         ▼                 ▼            ▼          │             │    │
│  │  ┌─────────────────┐  ┌──────────────────┐       │             │    │
│  │  │     pi-ai       │  │     pi-tui       │       │             │    │
│  │  │  packages/ai    │  │   packages/tui   │       │             │    │
│  │  │                 │  │                  │       │             │    │
│  │  │ 统一 LLM API    │  │ 终端 UI 渲染库  │       │             │    │
│  │  │ (20+ 模型提供商)│  │ (差分渲染引擎)  │       │             │    │
│  │  └─────────────────┘  └──────────────────┘       │             │    │
│  │                                                   │             │    │
│  │              第 1 层 · 基础设施层（无内部依赖）    │             │    │
│  └───────────────────────────────────────────────────┘             │    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 三、各包详细说明

### 3.1 pi-ai（统一 LLM API 层）

| 属性 | 值 |
|------|---|
| **路径** | `packages/ai/` |
| **包名** | `@mariozechner/pi-ai` |
| **角色** | 基础设施层 — 无内部依赖 |

**核心能力：**
- 统一封装 20+ LLM 提供商（Anthropic、OpenAI、Google Gemini、AWS Bedrock、Mistral 等）
- 自动模型发现与提供商配置
- 统一的消息类型（`Message`、`Content`）和流式响应（`streamSimple`）
- 工具定义 Schema（基于 JSON Schema / Zod）

**关键外部依赖：**
- `@anthropic-ai/sdk` — Anthropic Claude API
- `openai` — OpenAI API
- `@google/genai` — Google Gemini API
- `@aws-sdk/client-bedrock-runtime` — AWS Bedrock
- `@mistralai/mistralai` — Mistral API
- `@sinclair/typebox` — JSON Schema 类型定义

---

### 3.2 pi-tui（终端 UI 渲染库）

| 属性 | 值 |
|------|---|
| **路径** | `packages/tui/` |
| **包名** | `@mariozechner/pi-tui` |
| **角色** | 基础设施层 — 无内部依赖 |

**核心能力：**
- 差分渲染引擎（仅更新终端变化区域，高效刷新）
- Markdown 渲染（基于 `marked`）
- 终端宽度计算（含 CJK 字符支持）
- 编辑器组件、进程终端、颜色主题

**关键外部依赖：**
- `chalk` — 终端颜色
- `marked` — Markdown 解析
- `get-east-asian-width` — 东亚字符宽度

---

### 3.3 pi-agent-core（Agent 运行时核心）

| 属性 | 值 |
|------|---|
| **路径** | `packages/agent/` |
| **包名** | `@mariozechner/pi-agent-core` |
| **角色** | 核心层 — 依赖 pi-ai |

**核心能力：**
- Agent 循环（消息 → LLM → 工具调用 → 结果 → 循环）
- 传输层抽象（支持本地/远程执行）
- 状态管理与订阅模式（`Agent.subscribe()`）
- 工具注册与执行框架（`AgentTool` 接口）
- 附件支持（文件、图片等上下文注入）

**内部依赖：**
```
pi-agent-core → pi-ai
```

---

### 3.4 pi-coding-agent（编程 Agent CLI — 主入口）

| 属性 | 值 |
|------|---|
| **路径** | `packages/coding-agent/` |
| **包名** | `@mariozechner/pi-coding-agent` |
| **CLI 命令** | `pi` |
| **角色** | 应用层 — 项目主产品 |

**核心能力：**
- 交互式 CLI 编程助手（类似 Claude Code）
- 内置工具集：文件读写（Read/Write/Edit）、Bash 执行、Glob 搜索、Grep 搜索
- 会话管理（创建/恢复/导出 HTML）
- RPC 模式（stdin/stdout JSON Lines 协议，支持无头运行）
- 扩展系统（`pi.extensions` 支持自定义工具和提供商）
- Hook 系统（会话启动、工具调用前后等生命周期钩子）
- 主题支持（亮色/暗色 JSON 主题文件）

**内部依赖：**
```
pi-coding-agent → pi-agent-core
                → pi-ai
                → pi-tui
```

**关键外部依赖：**
- `glob` — 文件模式匹配
- `ignore` — .gitignore 规则处理
- `diff` — 差异对比
- `marked` — Markdown 渲染
- `cli-highlight` — 代码语法高亮
- `proper-lockfile` — 文件锁

---

### 3.5 pi-web-ui（Web 聊天组件库）

| 属性 | 值 |
|------|---|
| **路径** | `packages/web-ui/` |
| **包名** | `@mariozechner/pi-web-ui` |
| **角色** | 应用层 — Web 前端 |

**核心能力：**
- 基于 Lit 3.x 的 Web Components 聊天 UI
- 多模型提供商支持（浏览器内直连或代理）
- 文件预览（PDF、DOCX、XLSX）
- IndexedDB 本地存储（会话/设置/密钥）
- 自定义提供商配置
- 本地模型支持（Ollama、LM Studio）

**内部依赖：**
```
pi-web-ui → pi-ai
           → pi-tui
```

**关键外部依赖：**
- `lit` / `@mariozechner/mini-lit` — Web 组件框架
- `pdfjs-dist` — PDF 渲染
- `docx-preview` — DOCX 预览
- `xlsx` — Excel 处理
- `lucide` — 图标库
- `ollama` / `@lmstudio/sdk` — 本地模型接入

**样式方案：**
- Tailwind CSS v4（CLI 编译）
- CSS 变量主题系统

---

### 3.6 pi-mom（Slack 机器人）

| 属性 | 值 |
|------|---|
| **路径** | `packages/mom/` |
| **包名** | `@mariozechner/pi-mom` |
| **CLI 命令** | `mom` |
| **角色** | 编排层 — Slack 集成 |

**核心能力：**
- Slack Socket Mode 实时消息接收
- 将 Slack 消息委派给 pi-coding-agent 处理
- 支持 Anthropic Sandbox 运行时（隔离执行）
- 定时任务（基于 `croner`）
- TypeBox Schema 校验

**内部依赖：**
```
pi-mom → pi-agent-core
       → pi-ai
       → pi-coding-agent
```

**关键外部依赖：**
- `@slack/socket-mode` — Slack 实时连接
- `@slack/web-api` — Slack HTTP API
- `@anthropic-ai/sandbox-runtime` — 沙箱执行环境
- `croner` — Cron 定时调度

---

### 3.7 pi-pods（GPU Pod 管理工具）

| 属性 | 值 |
|------|---|
| **路径** | `packages/pods/` |
| **包名** | `@mariozechner/pi` |
| **CLI 命令** | `pi-pods` |
| **角色** | 核心层 — 基础设施运维 |

**核心能力：**
- vLLM 部署管理（GPU Pod 编排）
- SSH 远程执行
- 模型配置管理（`models.json`）

**内部依赖：**
```
pi-pods → pi-agent-core
```

---

## 四、依赖关系矩阵

### 4.1 依赖方向图

```
                    ┌──────────┐
                    │  pi-mom  │
                    └──┬─┬─┬──┘
                       │ │ │
          ┌────────────┘ │ └────────────────┐
          │              │                  │
          ▼              ▼                  ▼
   ┌──────────────┐  ┌────────┐  ┌──────────────────┐
   │ pi-agent-core│  │ pi-ai  │  │ pi-coding-agent  │
   └──────┬───────┘  └────────┘  └──┬───┬───┬───────┘
          │                         │   │   │
          │              ┌──────────┘   │   │
          ▼              ▼              │   │
       ┌────────┐  ┌──────────────┐    │   │
       │ pi-ai  │  │ pi-agent-core│    │   │
       └────────┘  └──────────────┘    │   │
                                       ▼   ▼
                              ┌────────┐ ┌────────┐
                              │ pi-ai  │ │ pi-tui │
                              └────────┘ └────────┘

   ┌────────────┐        ┌──────────────┐
   │  pi-pods   │        │  pi-web-ui   │
   └─────┬──────┘        └───┬─────┬────┘
         │                    │     │
         ▼                    ▼     ▼
   ┌──────────────┐    ┌────────┐ ┌────────┐
   │ pi-agent-core│    │ pi-ai  │ │ pi-tui │
   └──────────────┘    └────────┘ └────────┘
```

### 4.2 依赖矩阵表

| 包 ↓ 依赖于 → | pi-ai | pi-tui | pi-agent-core | pi-coding-agent |
|:---|:---:|:---:|:---:|:---:|
| **pi-ai** | — | | | |
| **pi-tui** | | — | | |
| **pi-agent-core** | ✅ | | — | |
| **pi-pods** | | | ✅ | |
| **pi-coding-agent** | ✅ | ✅ | ✅ | — |
| **pi-web-ui** | ✅ | ✅ | | |
| **pi-mom** | ✅ | | ✅ | ✅ |

### 4.3 分层架构

```
┌─────────────────────────────────────────────────────┐
│         第 4 层 · 编排层（Orchestration）             │
│                                                     │
│   pi-mom（Slack → Agent 委派与调度）                  │
├─────────────────────────────────────────────────────┤
│         第 3 层 · 应用层（Application）               │
│                                                     │
│   pi-coding-agent（CLI 编程助手）                     │
│   pi-web-ui（Web 聊天组件库）                         │
├─────────────────────────────────────────────────────┤
│         第 2 层 · 核心层（Core）                      │
│                                                     │
│   pi-agent-core（Agent 运行时 / 工具调用框架）        │
│   pi-pods（GPU Pod 管理）                             │
├─────────────────────────────────────────────────────┤
│         第 1 层 · 基础设施层（Foundation）             │
│                                                     │
│   pi-ai（统一 LLM API / 20+ 提供商）                 │
│   pi-tui（终端 UI 差分渲染引擎）                      │
└─────────────────────────────────────────────────────┘
```

---

## 五、核心技术选型

### 5.1 LLM 提供商支持

| 提供商 | SDK | 说明 |
|--------|-----|------|
| Anthropic Claude | `@anthropic-ai/sdk` | 主要 AI 提供商 |
| OpenAI / GPT | `openai` | ChatGPT 系列模型 |
| Google Gemini | `@google/genai` | Gemini Pro/Ultra |
| AWS Bedrock | `@aws-sdk/client-bedrock-runtime` | 云端多模型 |
| Mistral | `@mistralai/mistralai` | 开源大模型 |
| Ollama | `ollama` | 本地模型运行 |
| LM Studio | `@lmstudio/sdk` | 本地模型 GUI |
| 自定义提供商 | 扩展系统 | 支持 GitLab Duo、Qwen 等 |

### 5.2 关键技术决策

| 领域 | 选型 | 说明 |
|------|------|------|
| **前端框架** | Lit 3.x + Web Components | 轻量级，无 React/Vue |
| **终端 UI** | 自研 pi-tui | 差分渲染，高性能 |
| **状态管理** | 自研 Observable 模式 | 事件驱动，无 Redux/Zustand |
| **API 模式** | RPC (stdin/stdout JSON Lines) | 支持无头模式和 SDK 调用 |
| **数据存储** | IndexedDB (Web) / 文件系统 (CLI) | 无传统数据库 |
| **样式方案** | Tailwind CSS v4 | CLI 编译，CSS 变量主题 |
| **构建工具** | tsgo | TypeScript 原生编译 |
| **代码质量** | Biome 2.x | 替代 ESLint + Prettier |
| **测试** | Vitest | 快速，兼容 Jest API |
| **二进制打包** | Bun compile | 单文件可执行程序 |
| **CI/CD** | GitHub Actions | 构建/检查/测试/发布二进制 |

### 5.3 通信协议

```
                    ┌─────────────────┐
                    │     用户交互     │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
         ┌────▼────┐   ┌────▼────┐   ┌─────▼────┐
         │ CLI/TUI │   │  Web UI │   │  Slack   │
         │ (终端)  │   │ (浏览器)│   │  (Bot)   │
         └────┬────┘   └────┬────┘   └─────┬────┘
              │              │              │
              │         HTTP/直连      Socket Mode
              │              │              │
         ┌────▼──────────────▼──────────────▼────┐
         │          Agent 运行时                   │
         │     (pi-agent-core)                    │
         │                                        │
         │  ┌──────────────────────────────────┐  │
         │  │  工具调用循环:                     │  │
         │  │  消息 → LLM → 工具调用 → 结果     │  │
         │  └──────────────────────────────────┘  │
         └────────────────┬───────────────────────┘
                          │
                    ┌─────▼─────┐
                    │   pi-ai   │
                    │ 统一 LLM  │
                    │   API     │
                    └─────┬─────┘
                          │
           ┌──────────────┼──────────────┐
           │              │              │
      ┌────▼────┐   ┌────▼────┐   ┌─────▼────┐
      │Anthropic│   │ OpenAI  │   │ Google   │  ...
      └─────────┘   └─────────┘   └──────────┘
```

---

## 六、扩展系统深度解析

`pi-coding-agent` 的扩展系统是本项目最核心的差异化设计之一。它采用 **TypeScript 原生扩展** 模式，允许开发者通过简单的函数导出即可扩展 Agent 的工具、提供商、命令、快捷键和全生命周期行为。

---

### 6.1 扩展系统架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                      扩展系统架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────┐      │
│  │                  扩展发现与加载                         │      │
│  │                                                       │      │
│  │  ~/.pi/agent/extensions/  (全局扩展)                   │      │
│  │  .pi/extensions/          (项目扩展)                   │      │
│  │  settings.json paths      (显式配置)                   │      │
│  │  pi install npm:xxx       (Pi 包管理)                  │      │
│  │                                                       │      │
│  │         ┌──────────┐                                  │      │
│  │         │   Jiti   │  TypeScript 运行时加载器          │      │
│  │         └────┬─────┘  (无需预编译，直接执行 .ts)       │      │
│  │              │                                        │      │
│  └──────────────┼────────────────────────────────────────┘      │
│                 ▼                                                │
│  ┌───────────────────────────────────────────────────────┐      │
│  │              ExtensionAPI (注册阶段)                    │      │
│  │                                                       │      │
│  │  ┌─────────────┐ ┌──────────────┐ ┌───────────────┐  │      │
│  │  │registerTool │ │registerProv. │ │ registerCmd   │  │      │
│  │  │  自定义工具  │ │ 自定义提供商  │ │  斜杠命令     │  │      │
│  │  └─────────────┘ └──────────────┘ └───────────────┘  │      │
│  │  ┌─────────────┐ ┌──────────────┐ ┌───────────────┐  │      │
│  │  │registerFlag │ │registerShort │ │ pi.on(event)  │  │      │
│  │  │  CLI 标志    │ │  快捷键      │ │  事件钩子     │  │      │
│  │  └─────────────┘ └──────────────┘ └───────────────┘  │      │
│  └───────────────────────────────────────────────────────┘      │
│                 │                                                │
│                 ▼ bindCore()                                    │
│  ┌───────────────────────────────────────────────────────┐      │
│  │            ExtensionRuntime (运行阶段)                  │      │
│  │                                                       │      │
│  │  sendMessage()  appendEntry()  setActiveTools()       │      │
│  │  sendUserMessage()  exec()  compact()  abort()        │      │
│  │  setModel()  getThinkingLevel()  shutdown()           │      │
│  └───────────────────────────────────────────────────────┘      │
│                 │                                                │
│                 ▼                                                │
│  ┌───────────────────────────────────────────────────────┐      │
│  │            ExtensionContext (事件回调上下文)             │      │
│  │                                                       │      │
│  │  ctx.ui ─── 终端 UI（select/confirm/input/notify）     │      │
│  │  ctx.sessionManager ─── 会话历史读取                    │      │
│  │  ctx.modelRegistry ─── 模型与密钥管理                   │      │
│  │  ctx.cwd ─── 当前工作目录                               │      │
│  │  ctx.model ─── 当前模型                                 │      │
│  │  ctx.abort() / ctx.compact() / ctx.shutdown()          │      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### 6.2 扩展生命周期

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   发现阶段    │────▶│   加载阶段    │────▶│   绑定阶段    │
│  Discovery   │     │   Loading    │     │   Binding    │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
      ┌───────────────────────────────────────────┘
      ▼
┌──────────────┐     ┌──────────────┐
│   运行阶段    │────▶│   卸载阶段    │
│   Runtime    │     │  Shutdown    │
└──────────────┘     └──────────────┘
```

| 阶段 | 说明 |
|------|------|
| **发现** | 从全局/项目/配置路径扫描扩展目录，解析 `package.json` 中 `"pi"` 字段 |
| **加载** | 通过 Jiti（TypeScript 运行时加载器）动态导入 `.ts` 文件，调用 `factory(api)` |
| **绑定** | 调用 `runner.bindCore()` 将真实实现注入 API（加载阶段 action 方法为抛异常桩） |
| **运行** | 事件驱动：接收 session/tool/agent/input 等事件，执行注册的 handler |
| **卸载** | 触发 `session_shutdown` 事件，扩展清理资源 |

---

### 6.3 事件钩子体系（Hook System）

这是扩展系统的核心能力。扩展通过 `pi.on(event, handler)` 订阅事件：

```
┌─────────────────────────────────────────────────────────────────┐
│                        事件钩子全景图                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  会话生命周期                    Agent 循环                      │
│  ┌─────────────────────┐       ┌─────────────────────────┐     │
│  │ session_start       │       │ before_agent_start      │     │
│  │ session_before_switch│      │   → 注入系统提示/消息    │     │
│  │ session_switch      │       │ agent_start             │     │
│  │ session_before_fork │       │ turn_start              │     │
│  │ session_fork        │       │ turn_end                │     │
│  │ session_before_compact│     │ agent_end               │     │
│  │ session_compact     │       │ context                 │     │
│  │ session_before_tree │       │   → 过滤/修改消息       │     │
│  │ session_tree        │       └─────────────────────────┘     │
│  │ session_shutdown    │                                       │
│  └─────────────────────┘       工具执行                         │
│                                ┌─────────────────────────┐     │
│  输入处理                       │ tool_call               │     │
│  ┌─────────────────────┐       │   → 拦截/阻止工具调用   │     │
│  │ input               │       │ tool_result             │     │
│  │   → 变换用户输入    │       │   → 修改工具返回结果    │     │
│  │ user_bash           │       └─────────────────────────┘     │
│  │   → 自定义 ! 命令   │                                       │
│  └─────────────────────┘       模型 & 资源                     │
│                                ┌─────────────────────────┐     │
│                                │ model_select            │     │
│                                │ resources_discover      │     │
│                                │   → 提供技能/提示/主题  │     │
│                                └─────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

**关键事件的返回值能力：**

| 事件 | 可以做什么 |
|------|-----------|
| `tool_call` | 返回 `{ block: true, reason }` 阻止工具执行 |
| `tool_result` | 修改 `content`、`details`、`isError` |
| `input` | 变换文本 (`transform`) 或标记为已处理 (`handled`) |
| `context` | 修改发送给 LLM 的消息数组 |
| `before_agent_start` | 注入额外消息、修改系统提示词 |
| `session_before_compact` | 提供自定义压缩逻辑 |
| `session_before_switch/fork` | 返回取消标记阻止操作 |
| `resources_discover` | 返回技能/提示/主题文件路径 |

---

### 6.4 核心注册 API

#### 自定义工具（registerTool）

```typescript
import { ExtensionFactory } from "@mariozechner/pi-coding-agent"
import { Type } from "@sinclair/typebox"

const extension: ExtensionFactory = (pi) => {
  pi.registerTool({
    name: "hello",                              // LLM 调用名称
    label: "Hello",                             // UI 显示标签
    description: "向用户打招呼",                  // LLM 描述
    parameters: Type.Object({                   // TypeBox Schema
      name: Type.String({ description: "姓名" }),
    }),

    // 核心执行逻辑
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      return {
        content: [{ type: "text", text: `你好，${params.name}！` }],
        details: { greeted: params.name },       // 持久化数据
      }
    },

    // 可选：自定义渲染
    renderCall(args, theme) { /* TUI 组件 */ },
    renderResult(result, options, theme) { /* TUI 组件 */ },
  })
}

export default extension
```

#### 自定义提供商（registerProvider）

```typescript
pi.registerProvider("my-llm", {
  baseUrl: "https://api.my-llm.com",
  apiKey: "MY_LLM_API_KEY",                    // 环境变量名
  api: "openai-responses",                      // 使用 OpenAI 兼容协议
  models: [{
    id: "my-model-v1",
    name: "My Model v1",
    reasoning: true,
    input: ["text", "image"],
    cost: { input: 0.01, output: 0.03, cacheRead: 0.001, cacheWrite: 0.003 },
    contextWindow: 200000,
    maxTokens: 8192,
  }],
  // 可选：OAuth 认证
  oauth: {
    name: "My LLM",
    login: async (callbacks) => { /* PKCE/设备码流程 */ },
    refreshToken: async (creds) => { /* 刷新令牌 */ },
    getApiKey: (creds) => creds.access,
  },
  // 可选：完全自定义流式调用
  streamSimple: (model, context, options) => { /* EventStream */ },
})
```

#### 斜杠命令（registerCommand）

```typescript
pi.registerCommand("mycommand", {
  description: "我的自定义命令",
  getArgumentCompletions: (prefix) => [
    { value: "arg1", label: "参数一" }
  ],
  handler: async (args, ctx) => {
    // ctx 包含 session 控制方法
    await ctx.ui.notify("命令已执行", "success")
  },
})
```

#### 快捷键（registerShortcut）和 CLI 标志（registerFlag）

```typescript
pi.registerShortcut("ctrl+shift+x", {
  description: "执行自定义操作",
  handler: async (ctx) => { /* ... */ },
})

pi.registerFlag("my-feature", {
  type: "boolean",
  default: true,
  description: "启用我的特性",
})
```

---

### 6.5 扩展配置方式

#### 方式一：项目目录自动发现

```
.pi/extensions/
├── my-tool.ts                    # 单文件扩展
├── my-package/
│   ├── package.json              # { "pi": { "extensions": ["./index.ts"] } }
│   ├── index.ts
│   └── node_modules/             # 扩展自身依赖
```

#### 方式二：全局目录

```
~/.pi/agent/extensions/
├── global-tool.ts
└── global-package/
```

#### 方式三：Pi 包管理器安装

```bash
pi install npm:@foo/pi-mcp-adapter      # 从 npm 安装
pi install git:github.com/user/repo      # 从 Git 仓库安装
```

#### 方式四：CLI 参数加载

```bash
pi --extension ./path/to/extension.ts    # 指定扩展路径
pi -e ./path/to/extension                # 简写
```

#### 方式五：package.json 声明

```json
{
  "pi": {
    "extensions": ["./src/main.ts"],
    "themes": ["./themes/"],
    "skills": ["./skills/"],
    "prompts": ["./prompts/"]
  }
}
```

---

### 6.6 内置示例扩展分析

项目包含 **70+ 个示例扩展**，按功能分为以下类别：

#### 目录级扩展（多文件 + package.json）

| 扩展 | 代码量 | 功能 | 使用的 API |
|------|--------|------|-----------|
| **sandbox** | ~318 行 | OS 级沙箱隔离 Bash 命令执行 | `registerFlag`, `registerTool`, `registerCommand`, `on("user_bash")`, `on("session_start/shutdown")` |
| **plan-mode** | ~509 行 | 只读探索模式 + 计划提取 + 进度追踪 | `registerFlag`, `registerCommand`, `registerShortcut`, `on("tool_call")`, `on("context")`, `on("before_agent_start")`, `on("agent_end")`, `appendEntry`, `setActiveTools` |
| **subagent** | ~963 行 | 将任务委派给子进程 Agent | `registerTool`, 自定义 `renderCall/renderResult`, 子进程管理, 并发控制 |
| **custom-provider-anthropic** | ~604 行 | 带 OAuth 的自定义 Anthropic 提供商 | `registerProvider`, OAuth PKCE, 自定义 `streamSimple` |
| **custom-provider-gitlab-duo** | ~349 行 | GitLab Duo AI Gateway 提供商 | `registerProvider`, 委托内置 `streamSimpleAnthropic/OpenAI` |
| **custom-provider-qwen-cli** | ~345 行 | 通义千问设备码 OAuth 提供商 | `registerProvider`, 设备码 OAuth (RFC 8628) |
| **with-deps** | ~36 行 | 演示 npm 依赖解析 | `registerTool` + 外部依赖 `ms` |
| **doom-overlay** | ~74 行 | 终端内可玩 DOOM 游戏 | `registerCommand`, `ctx.ui.custom()` (WASM 渲染) |
| **dynamic-resources** | ~15 行 | 动态提供技能/提示/主题 | `on("resources_discover")` |

#### 单文件扩展按类别分布

| 类别 | 示例数 | 代表扩展 |
|------|--------|---------|
| **安全防护** | 5+ | `permission-gate.ts`（确认危险命令）, `protected-paths.ts`（阻止写敏感路径）, `dirty-repo-guard.ts`（脏仓库保护）|
| **自定义工具** | 10+ | `todo.ts`（待办列表 + 状态持久化）, `ssh.ts`（SSH 远程工具）, `tool-override.ts`（覆盖内置工具）|
| **命令与 UI** | 20+ | `preset.ts`（模型/思考预设）, `tools.ts`（工具管理）, `snake.ts`（贪吃蛇游戏）, `modal-editor.ts`（Vim 模态编辑器）|
| **Git 集成** | 2 | `git-checkpoint.ts`（每轮 stash）, `auto-commit-on-exit.ts`（退出自动提交）|
| **系统提示词** | 4 | `pirate.ts`（动态修改提示词）, `claude-rules.ts`（加载规则文件）, `custom-compaction.ts`（自定义压缩）|
| **事件钩子** | 10+ | `bash-spawn-hook.ts`（Bash 进程钩子）, `input-transform.ts`（输入变换）, `file-trigger.ts`（文件触发器）|
| **会话管理** | 2 | `session-name.ts`（会话命名）, `bookmark.ts`（书签标记）|
| **消息渲染** | 2 | `message-renderer.ts`（自定义渲染）, `event-bus.ts`（扩展间通信）|

---

### 6.7 热门社区扩展与生态

Pi 已拥有 **活跃的社区扩展生态**，以下是按热度和实用性排序的知名扩展：

#### 官方资源

| 资源 | 说明 |
|------|------|
| [buildwithpi.ai/packages](https://buildwithpi.ai/packages) | 官方扩展包浏览器 |
| [qualisero/awesome-pi-agent](https://github.com/qualisero/awesome-pi-agent) | 社区精选列表（Awesome List） |
| [badlogic/pi-skills](https://github.com/badlogic/pi-skills) | 官方技能仓库 |

#### 热门第三方扩展

```
┌─────────────────────────────────────────────────────────────────────┐
│                    热门社区扩展生态图                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  MCP 协议                            多 Agent 协作                  │
│  ┌──────────────────────────┐       ┌──────────────────────────┐   │
│  │ pi-mcp-adapter           │       │ pi-messenger             │   │
│  │ ─────────────────────    │       │ ─────────────────────    │   │
│  │ Token 高效 MCP 代理      │       │ 多 Agent 终端间通信      │   │
│  │ 懒加载+空闲超时          │       │ 文件协调+任务领取        │   │
│  │ 交互式 /mcp 配置覆层    │       │ Crew Mode 工作流        │   │
│  │ by nicobailon            │       │ by nicobailon            │   │
│  └──────────────────────────┘       └──────────────────────────┘   │
│                                                                     │
│  交互式 Shell                        子 Agent 委派                  │
│  ┌──────────────────────────┐       ┌──────────────────────────┐   │
│  │ pi-interactive-shell     │       │ pi-subagents             │   │
│  │ ─────────────────────    │       │ ─────────────────────    │   │
│  │ 自主运行交互式 CLI       │       │ 异步子 Agent 委派        │   │
│  │ 完整 PTY 模拟            │       │ 输出截断+产物管理        │   │
│  │ vim/htop/psql/ssh 等    │       │ 会话共享                 │   │
│  │ by nicobailon            │       │ by nicobailon            │   │
│  └──────────────────────────┘       └──────────────────────────┘   │
│                                                                     │
│  安全与成本                           UI 与游戏                     │
│  ┌──────────────────────────┐       ┌──────────────────────────┐   │
│  │ filter-output            │       │ tmustier/pi-extensions   │   │
│  │ ─────────────────────    │       │ ─────────────────────    │   │
│  │ 脱敏 API 密钥/令牌/密码  │       │ 终端文件浏览器           │   │
│  │ 防止 LLM 看到敏感数据    │       │ 迷你游戏（太空侵略者、   │   │
│  │                          │       │ 吃豆人、俄罗斯方块等）   │   │
│  │ cost-tracker             │       │ 等待测试时消磨时间       │   │
│  │ ─────────────────────    │       │ by tmustier              │   │
│  │ 会话开支分析             │       └──────────────────────────┘   │
│  └──────────────────────────┘                                       │
│                                                                     │
│  前端集成                            其他                           │
│  ┌──────────────────────────┐       ┌──────────────────────────┐   │
│  │ dnouri/pi-coding-agent   │       │ oracle                   │   │
│  │ ─────────────────────    │       │  → 第二模型意见           │   │
│  │ Emacs 完整前端           │       │ memory-mode              │   │
│  │ 基于 RPC 模式            │       │  → AI 辅助写入 AGENTS.md │   │
│  │ Markdown+语法高亮        │       │ background-notify        │   │
│  │ Vim 键绑定               │       │  → 任务完成通知          │   │
│  │ by dnouri                │       │ session-emoji            │   │
│  └──────────────────────────┘       │  → 会话表情符号          │   │
│                                     └──────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

#### 热门扩展详细分析

| 扩展 | 作者 | 核心能力 | 技术亮点 |
|------|------|---------|---------|
| **pi-mcp-adapter** | nicobailon | 将 MCP 协议服务器桥接为 Pi 工具 | 单个 ~200 token 代理工具替代数百个冗余定义；懒加载服务器；空闲超时释放资源 |
| **pi-messenger** | nicobailon | 多 Agent 终端间通信 | 无 daemon 文件协调；消息队列；文件锁定；Crew Mode（计划→执行→审查） |
| **pi-interactive-shell** | nicobailon | 让 Pi 自主操作交互式 CLI | 完整 PTY 模拟；用户可随时接管；支持 vim/htop/psql/docker |
| **pi-subagents** | nicobailon | 异步子 Agent 委派 | 上下文截断；产物持久化；会话状态共享 |
| **filter-output** | 社区 | 敏感数据脱敏 | 在 tool_result 事件中正则匹配并替换 API 密钥、令牌、密码 |
| **cost-tracker** | 社区 | 使用量和费用追踪 | 解析日志计算 token 开支；按会话/提供商统计 |
| **oracle** | 社区 | 获取备选模型的意见 | 不切换上下文即可咨询其他 LLM；适合代码审查场景 |

#### 知名下游产品

| 项目 | 说明 |
|------|------|
| **OpenClaw** | 使用 Pi 作为 SDK 引擎的 AI 助手，首周 GitHub 145,000+ 星 |
| **Emacs 前端** | 完整的 Emacs 编辑器集成，基于 Pi 的 RPC 模式 |

---

### 6.8 扩展开发最佳实践

#### 状态持久化模式

```typescript
// 保存状态
pi.appendEntry("my-state", { todos: [...] })

// 会话恢复时重建
pi.on("session_start", async (_event, ctx) => {
  for (const entry of ctx.sessionManager.getEntries()) {
    if (entry.type === "custom" && entry.customType === "my-state") {
      state = entry.data  // 重建内存状态
    }
  }
})
```

#### 工具拦截模式（安全防护）

```typescript
pi.on("tool_call", async (event, ctx) => {
  if (event.tool === "bash" && /rm\s+-rf/.test(event.args.command)) {
    const ok = await ctx.ui.confirm("危险操作", "检测到 rm -rf，确认执行？")
    if (!ok) return { block: true, reason: "用户取消了危险操作" }
  }
})
```

#### 上下文注入模式

```typescript
pi.on("before_agent_start", async (_event, ctx) => {
  return {
    systemPrompt: ctx.getSystemPrompt() + "\n\n## 额外规则\n请始终使用中文回复。",
    message: { role: "user", content: "当前 Git 分支: main" }
  }
})
```

#### TypeBox Schema 注意事项

```typescript
// 枚举参数：Google API 兼容需使用 StringEnum 而非 Union
import { StringEnum } from "@mariozechner/pi-coding-agent"
parameters: Type.Object({
  action: StringEnum(["list", "add", "remove"] as const),
})
```

---

### 6.9 关键源码文件索引

| 文件 | 职责 |
|------|------|
| `coding-agent/src/core/extensions/types.ts` | 全部类型定义（ExtensionAPI, 事件类型, 上下文接口） |
| `coding-agent/src/core/extensions/loader.ts` | 扩展发现与 Jiti 加载 |
| `coding-agent/src/core/extensions/runner.ts` | 事件分发、生命周期管理、bindCore |
| `coding-agent/src/core/extensions/wrapper.ts` | 内置工具的扩展事件包装 |
| `coding-agent/src/core/extensions/index.ts` | 公开导出 |
| `coding-agent/docs/extensions.md` | 官方扩展开发文档 |
| `coding-agent/docs/providers.md` | 提供商注册文档 |
| `coding-agent/docs/skills.md` | 技能格式文档 |
| `coding-agent/examples/extensions/` | 70+ 示例扩展 |

---

## 七、构建与开发流程

### 7.1 构建顺序（依赖拓扑排序）

```bash
# 根目录 build 脚本按依赖顺序构建：
tui → ai → agent → coding-agent → mom → web-ui → pods
```

### 7.2 开发模式

```bash
npm run dev    # 并发启动所有包的 watch 模式
npm run check  # Biome lint + TypeScript 类型检查
npm run test   # 所有包的测试
```

### 7.3 CI/CD 流水线

```
Push/PR to main
       │
       ├──▶ npm ci（安装依赖）
       ├──▶ npm run build（全量构建）
       ├──▶ npm run check（Biome + tsc）
       └──▶ npm run test（Vitest）

Release
       │
       ├──▶ scripts/sync-versions.js（版本同步）
       ├──▶ scripts/release.mjs（发布管理）
       └──▶ scripts/build-binaries.sh（二进制打包）
```

---

## 八、快速导航

| 想了解... | 查看 |
|-----------|------|
| LLM 如何统一调用 | `packages/ai/src/` |
| Agent 循环如何运作 | `packages/agent/src/` |
| CLI 入口和工具实现 | `packages/coding-agent/src/` |
| Web UI 组件 | `packages/web-ui/src/` |
| Slack 集成 | `packages/mom/src/` |
| 终端渲染原理 | `packages/tui/src/` |
| GPU Pod 管理 | `packages/pods/src/` |
| 扩展开发 | `packages/coding-agent/examples/extensions/` |
| 构建配置 | `tsconfig.base.json` / `biome.json` |
| CI/CD | `.github/workflows/` |
