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

## 六、扩展系统

`pi-coding-agent` 提供了灵活的扩展机制：

```
package.json 中声明:
{
  "pi": {
    "extensions": ["./index.ts"]
  }
}
```

**扩展能力：**
- 自定义工具（新增 Agent 可调用的工具）
- 自定义 LLM 提供商（接入企业私有模型）
- 生命周期 Hook（会话启动、工具调用前后）

**示例扩展：**

| 示例 | 路径 | 说明 |
|------|------|------|
| with-deps | `coding-agent/examples/extensions/with-deps` | 带外部依赖的扩展 |
| custom-provider-anthropic | `coding-agent/examples/extensions/custom-provider-anthropic` | 自定义 Anthropic 提供商 |
| custom-provider-gitlab-duo | `coding-agent/examples/extensions/custom-provider-gitlab-duo` | GitLab Duo 集成 |
| custom-provider-qwen-cli | `coding-agent/examples/extensions/custom-provider-qwen-cli` | 通义千问 CLI 集成 |
| sandbox | `coding-agent/examples/extensions/sandbox` | 沙箱运行时扩展 |

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
