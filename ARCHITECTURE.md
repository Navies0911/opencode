# OpenCode 项目架构文档

## 目录

- [项目概览](#项目概览)
- [架构图](#架构图)
- [Monorepo 结构](#monorepo-结构)
- [核心模块详解](#核心模块详解)
- [工作流分析](#工作流分析)
- [第三方依赖总览](#第三方依赖总览)

---

## 项目概览

OpenCode 是一个 **AI 驱动的编程助手**，支持 CLI、Web、桌面端（Tauri）等多种交互方式。它通过集成多种大语言模型（LLM）提供商，为开发者提供代码编写、搜索、编辑、调试等智能化开发能力。

**技术栈**: TypeScript · Bun · Solid.js · Hono · Tauri · Vite · Turborepo

---

## 架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         用户交互层 (Frontend)                         │
│                                                                     │
│   ┌──────────┐   ┌──────────────┐   ┌───────────┐   ┌───────────┐ │
│   │  CLI      │   │  Web App     │   │  Desktop  │   │  VS Code  │ │
│   │ (yargs)  │   │ (Solid.js)   │   │  (Tauri)  │   │ Extension │ │
│   └────┬─────┘   └──────┬───────┘   └─────┬─────┘   └─────┬─────┘ │
│        │                │                  │               │       │
└────────┼────────────────┼──────────────────┼───────────────┼───────┘
         │                │                  │               │
         ▼                ▼                  ▼               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     API / SDK 层 (Communication)                     │
│                                                                     │
│   ┌──────────────────────┐   ┌────────────────────────────────┐    │
│   │  Hono HTTP Server    │   │  @opencode-ai/sdk (TypeScript) │    │
│   │  (REST + OpenAPI)    │   │  (自动生成的客户端 SDK)           │    │
│   └──────────┬───────────┘   └────────────────────────────────┘    │
│              │                                                      │
└──────────────┼──────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      核心引擎层 (Core Engine)                        │
│                                                                     │
│  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  Agent   │  │  Session  │  │ Provider │  │   Permission     │  │
│  │  System  │◄─┤ Processor ├─►│  Layer   │  │   System         │  │
│  └────┬─────┘  └─────┬─────┘  └────┬─────┘  └──────────────────┘  │
│       │              │              │                               │
│       ▼              ▼              ▼                               │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Tool Execution Framework                   │  │
│  │                                                              │  │
│  │  read · write · edit · bash · grep · glob · webfetch ·      │  │
│  │  websearch · lsp · task · plan · todo · question · batch    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌───────────┐ ┌───────┐ ┌───────────┐ ┌──────────┐ ┌──────────┐ │
│  │  MCP      │ │  LSP  │ │  Snapshot │ │ Worktree │ │ Scheduler│ │
│  │  Client   │ │Client │ │  Manager  │ │ Manager  │ │          │ │
│  └───────────┘ └───────┘ └───────────┘ └──────────┘ └──────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       基础设施层 (Infrastructure)                     │
│                                                                     │
│  ┌───────────┐ ┌───────────┐ ┌──────────┐ ┌──────────┐            │
│  │  Storage  │ │  Config   │ │  Bus     │ │  Shell   │            │
│  │  (文件系统) │ │  (JSONC)  │ │(事件总线) │ │ (进程管理) │            │
│  └───────────┘ └───────────┘ └──────────┘ └──────────┘            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      外部服务层 (External Services)                   │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    LLM Providers                             │  │
│  │  OpenAI · Anthropic · Google · Azure · Bedrock · Mistral    │  │
│  │  Groq · Cohere · DeepInfra · Together · XAI · Copilot ...  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────┐ ┌───────────────┐ ┌─────────────────────────┐   │
│  │  MCP Servers │ │  LSP Servers  │ │  GitHub / GitLab API    │   │
│  └──────────────┘ └───────────────┘ └─────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Monorepo 结构

项目使用 **Bun Workspaces + Turborepo** 管理多包，整体结构如下：

```
opencode/
├── packages/
│   ├── opencode/        # 核心引擎 (CLI + Agent + Server)
│   ├── app/             # Web 应用 (Solid.js + Vite)
│   ├── desktop/         # 桌面应用 (Tauri + Solid.js)
│   ├── ui/              # 共享 UI 组件库
│   ├── sdk/js/          # TypeScript SDK (自动生成)
│   ├── web/             # 官网 (Astro)
│   ├── plugin/          # 插件框架
│   ├── util/            # 公共工具函数
│   ├── enterprise/      # 企业版功能
│   ├── script/          # 构建脚本
│   ├── slack/           # Slack 集成
│   └── console/         # 管理控制台
│       ├── app/         #   控制台前端 (SolidStart)
│       ├── core/        #   控制台后端 (Drizzle + PlanetScale)
│       ├── function/    #   Serverless 函数
│       ├── mail/        #   邮件服务
│       └── resource/    #   资源管理 (Cloudflare)
├── sdks/
│   └── vscode/          # VS Code 扩展
├── infra/               # SST 基础设施代码
└── themes/              # 主题文件
```

### 包间依赖关系

```
                    ┌───────────────┐
                    │  opencode     │ (核心)
                    │  (CLI+Engine) │
                    └──────┬────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │   sdk    │ │  plugin  │ │   util   │
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │            │            │
     ┌───────┼──────┬─────┘            │
     ▼       ▼      ▼                  ▼
┌────────┐ ┌────┐ ┌───────┐     ┌────────────┐
│  app   │ │ ui │ │ slack │     │ enterprise │
│ (Web)  │ │    │ │       │     │            │
└────┬───┘ └──┬─┘ └───────┘     └────────────┘
     │        │
     ▼        ▼
┌─────────┐  │
│ desktop │──┘
│ (Tauri) │
└─────────┘
```

---

## 核心模块详解

### 1. Agent System (`src/agent/`)

| 角色 | 说明 |
|------|------|
| **build** | 主 Agent，默认使用，拥有全部工具权限 |
| **plan** | 只读模式 Agent，仅允许读取和搜索操作 |
| **general** | 通用子 Agent，用于并行处理多步骤任务 |
| **explore** | 探索子 Agent，专注于代码库搜索和理解 |
| **compaction** | 隐藏 Agent，用于对话压缩 |
| **title / summary** | 隐藏 Agent，用于自动生成标题和摘要 |

每个 Agent 定义了：权限范围、可用工具集、系统提示词、模型参数（temperature/topP）。

### 2. Session Processor (`src/session/`)

**会话处理器**是整个系统的核心调度中心：

- 管理用户与 Agent 之间的对话状态
- 执行 LLM 流式推理循环
- 解析并执行工具调用
- 包含 **Doom Loop 检测**（同一工具连续调用 3 次则中断）
- 支持会话持久化和恢复

### 3. Provider Layer (`src/provider/`)

统一的 LLM 提供商抽象层，支持 **20+ 提供商**：

| 类别 | 提供商 |
|------|--------|
| **主流** | OpenAI, Anthropic, Google (Gemini) |
| **云厂商** | Azure OpenAI, Amazon Bedrock, Google Vertex |
| **开源/推理** | Groq, Together, DeepInfra, Cerebras, Mistral |
| **聚合** | OpenRouter, Copilot, GitLab |
| **其他** | Cohere, Perplexity, X.AI, Vercel |

关键功能：
- 模型列表从 `models.dev` API 动态获取
- 支持 OAuth 和 API Key 认证
- 基于 Vercel AI SDK 的统一接口

### 4. Tool Framework (`src/tool/`)

Agent 可动态调用的工具集：

| 类别 | 工具 | 说明 |
|------|------|------|
| **文件操作** | `read`, `write`, `edit`, `multiedit`, `apply_patch` | 读写编辑文件 |
| **搜索发现** | `grep`, `glob`, `codesearch`, `ls` | 搜索代码和文件 |
| **执行** | `bash`, `task`, `batch` | 运行命令和并发任务 |
| **Web** | `webfetch`, `websearch` | 抓取网页和搜索互联网 |
| **IDE 集成** | `lsp` | Language Server Protocol 操作 |
| **规划** | `plan`, `todo` (todoread/todowrite) | 任务规划和跟踪 |
| **交互** | `question` | 向用户提问获取更多信息 |

### 5. MCP Client (`src/mcp/`)

**Model Context Protocol** 客户端，连接外部工具服务：
- 支持 **stdio / HTTP / SSE** 三种传输方式
- 支持 OAuth 认证流程
- 动态发现和注册外部工具

### 6. LSP Client (`src/lsp/`)

**Language Server Protocol** 客户端：
- 管理多语言 LSP 服务端的生命周期
- 提供代码补全、跳转定义、符号查找等 IDE 功能
- 作为 Agent 的 `lsp` 工具暴露

### 7. Server (`src/server/`)

基于 **Hono** 框架的 HTTP API 服务端：
- 提供 REST API + OpenAPI 文档
- 暴露项目管理、会话、文件、配置等接口
- 作为 Web/Desktop 前端的后端服务

### 8. Storage (`src/storage/`)

文件系统级持久化存储：
- 会话、消息、配置的持久化
- 支持数据迁移和版本管理
- 基于文件锁保证并发安全

### 9. Event Bus (`src/bus/`)

发布/订阅事件系统：
- 跨模块解耦通信
- 使用 Zod Schema 定义类型安全的事件
- 支持多实例间的事件分发

### 10. Permission System (`src/permission/`)

细粒度权限控制：
- 定义哪些 Agent 可以使用哪些工具
- 支持通配符模式匹配
- 执行工具前进行权限检查

### 11. Config (`src/config/`)

配置管理系统：
- 读取 **JSONC** 格式配置文件
- 支持项目级和全局级配置
- 通过 Zod Schema 验证配置结构

### 12. 其他模块

| 模块 | 说明 |
|------|------|
| **snapshot** | Git 快照管理，定期清理旧的 worktree 快照 |
| **worktree** | Git Worktree 生命周期管理 |
| **shell** | 子进程管理，跨平台进程终止 |
| **pty** | 伪终端支持 |
| **patch** | 代码补丁应用 |
| **format** | 代码格式化 |
| **share** | 会话分享功能 |
| **ide** | IDE 集成检测 |
| **skill** | 用户自定义 Skill（SKILL.md） |
| **scheduler** | 基于 setInterval 的定时任务调度 |
| **acp** | Agent Communication Protocol |

---

## 工作流分析

### ✅ 动态工作流 (Dynamic / Agentic Workflow)

OpenCode 采用的是**动态 Agent 工作流**，而非静态工作流。核心区别：

| 特征 | 静态工作流 | **OpenCode (动态工作流)** |
|------|-----------|-------------------------|
| 执行顺序 | 预定义的固定步骤 | **LLM 实时决策** |
| 工具调用 | 按序编排 | **Agent 自主选择** |
| 分支逻辑 | 预设的 if/else | **基于上下文推理** |
| 终止条件 | 步骤执行完毕 | **Agent 判断任务完成** |

### 执行流程

```
用户输入
  │
  ▼
┌─────────────────────────────────────────────────────┐
│              Session Processor (循环)                 │
│                                                     │
│  1. 构造系统提示词 + 工具定义 + 对话历史              │
│  2. 调用 LLM 流式推理                                │
│  3. 解析 LLM 输出：                                  │
│     ├─ text-delta → 推理文本输出                     │
│     ├─ tool-call → 执行工具 (权限检查)               │
│     └─ tool-result → 结果反馈给 LLM                 │
│  4. 循环检测 (Doom Loop: 同一工具调用 ≥3 次)         │
│  5. LLM 决定是否继续 → 是: 回到步骤 2               │
│                       → 否: 结束                     │
│                                                     │
└─────────────────────────────────────────────────────┘
  │
  ▼
输出结果
```

### 子 Agent 委托

主 Agent（build）可通过 `task` 工具委托子 Agent（general / explore）执行子任务，形成**多层 Agent 协作**：

```
Build Agent (主)
  ├── task → General Agent (子) → 独立完成多步骤任务
  ├── task → Explore Agent (子) → 代码库搜索分析
  └── 直接工具调用 → read, write, bash, grep ...
```

### 调度器

`scheduler` 模块提供**简单的定时任务**（非工作流编排），用于：
- 定期清理 Git 快照
- 后台维护任务
- 基于 `setInterval` 实现，支持 instance/global 作用域

---

## 第三方依赖总览

### 核心引擎 (`opencode`)

| 依赖 | 用途 |
|------|------|
| `ai` (Vercel AI SDK) | 统一的 LLM 调用接口 |
| `@ai-sdk/openai` | OpenAI 提供商适配 |
| `@ai-sdk/anthropic` | Anthropic 提供商适配 |
| `@ai-sdk/google` | Google AI 提供商适配 |
| `@ai-sdk/azure` | Azure OpenAI 提供商适配 |
| `@ai-sdk/amazon-bedrock` | AWS Bedrock 提供商适配 |
| `@ai-sdk/mistral` | Mistral 提供商适配 |
| `@ai-sdk/google-vertex` | Google Vertex AI 适配 |
| `@ai-sdk/cohere` | Cohere 提供商适配 |
| `@ai-sdk/groq` | Groq 提供商适配 |
| `@ai-sdk/deepinfra` | DeepInfra 提供商适配 |
| `@ai-sdk/togetherai` | Together AI 提供商适配 |
| `@ai-sdk/cerebras` | Cerebras 提供商适配 |
| `@ai-sdk/xai` | X.AI 提供商适配 |
| `@ai-sdk/perplexity` | Perplexity 提供商适配 |
| `@modelcontextprotocol/sdk` | MCP 协议 SDK |
| `hono` | HTTP 服务端框架 |
| `yargs` | CLI 参数解析 |
| `zod` | 运行时类型验证 |
| `tree-sitter` / `tree-sitter-wasms` | 代码 AST 解析 |
| `@octokit/rest` / `@octokit/graphql` | GitHub API 集成 |
| `diff` | 文件差异比较 |
| `xterm-headless` | 终端模拟 |
| `openauth` | OAuth 认证 |
| `@clack/prompts` | CLI 交互提示 |

### Web 应用 (`app`)

| 依赖 | 用途 |
|------|------|
| `solid-js` | 响应式 UI 框架 |
| `@solidjs/router` | 路由管理 |
| `vite` / `vite-plugin-solid` | 构建工具 |
| `@kobalte/core` | Solid.js 无障碍组件库 |
| `shiki` | 代码语法高亮 |
| `marked` | Markdown 渲染 |
| `tailwindcss` | 原子化 CSS |

### 桌面应用 (`desktop`)

| 依赖 | 用途 |
|------|------|
| `@tauri-apps/api` | Tauri 桌面端 API |
| `@tauri-apps/plugin-*` | 系统级插件 (shell, dialog 等) |
| `solid-js` | UI 框架 (复用 Web 应用) |

### UI 组件库 (`ui`)

| 依赖 | 用途 |
|------|------|
| `@kobalte/core` | 无障碍 UI 基础组件 |
| `shiki` | 代码高亮 |
| `marked` / `dompurify` | Markdown 渲染与安全过滤 |
| `solid-js` | 组件框架 |

### 控制台 (`console`)

| 依赖 | 用途 |
|------|------|
| `@solidjs/start` | 全栈 Solid 框架 |
| `drizzle-orm` | ORM 数据库操作 |
| `@planetscale/database` | PlanetScale 数据库驱动 |
| `stripe` | 支付集成 |
| `chart.js` | 图表可视化 |
| `jsx-email` | 邮件模板 |

### SDK 和集成

| 包 | 依赖 | 用途 |
|----|------|------|
| `@opencode-ai/sdk` | 无运行时依赖 | 自动生成的 TypeScript 客户端 |
| `@opencode-ai/plugin` | `zod` | 插件类型定义 |
| `@opencode-ai/slack` | `@slack/bolt` | Slack Bot 集成 |
| `@opencode-ai/util` | `zod` | 公共工具函数 |
| VS Code Extension | `@types/vscode` | VS Code 插件 |

### 构建工具链

| 工具 | 用途 |
|------|------|
| **Bun** | 包管理器 + 运行时 |
| **Turborepo** | Monorepo 构建编排 |
| **Vite** | 前端构建 |
| **TypeScript** | 类型系统 |
| **Tailwind CSS** | 样式方案 |
| **Playwright** | E2E 测试 |
| **SST** | 基础设施即代码 (IaC) |

---

> 本文档基于项目源码梳理编写，对应版本 v1.1.53。
