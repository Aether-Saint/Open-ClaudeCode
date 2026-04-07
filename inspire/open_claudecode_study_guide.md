# Open-ClaudeCode 源码学习指南

本项目 (Open-ClaudeCode) 是基于 Anthropic 官方发布的 Claude Code npm 包源码重建的开源版本，采用 TypeScript 编写，包含完整的终端交互界面 (TUI)、代理执行系统 (Agent System)、工具系统 (Tool System) 和插件机制。

以下指南是对该代码库结构的深入剖析，旨在帮助你系统地学习和二次开发。

---

## 1. 项目整体结构概览

项目的核心入口位于 `src/`，编译后的 CLI 位于 `package/`。除了 `inspire` 文件夹外，主要包含以下内容：

*   **`src/`**: 核心源代码（TypeScript）。
*   **`plugins/`**: Anthropic 官方提供的多个插件（例如 `code-review`, `commit-commands`, `feature-dev` 等），展示了如何通过插件扩展 Claude 的能力。
*   **`package/`**: 编译后的可执行文件 (`cli.js`) 和相关依赖项。
*   **`examples/`**: 设置示例，如 `settings-strict.json`，展示了权限管理的配置方法。

---

## 2. 核心架构与模块拆解

### 2.1 启动入口 (`src/main.tsx`)
这是 CLI 启动的核心枢纽。
*   **初始化流程**: 检查 Node.js 环境、解析 CLI 参数、加载本地预设 (`settings.json`)、进行身份验证 (OAuth/API Key)。
*   **安全与合规**: 阻止默认 cwd 漏洞，检查是否处于调试模式（处于调试模式可能会由于安全性直接退出）。
*   **状态与上下文**: 在安全的环境下预读取系统上下文（包括 Git 状态）。
*   **渲染挂载**: 最终调用 Ink 框架渲染界面的主组件 (`REPL.tsx` 或类似命令入口)。

### 2.2 核心状态管理 (`src/state/AppStateStore.ts`)
项目使用统一的全局状态来管理整个终端 UI 及会话状态：
*   管理 `toolPermissionContext`（工具权限上下文：批准/拒绝/始终允许）。
*   管理会话状态，如 `messages` (聊天记录)、`tasks`、`mcp`（MCP连接状态）。
*   管理与用户的当前交互状态（正在执行的任务、对话提示建议、费用计算追踪）。

### 2.3 查询引擎 (`src/QueryEngine.ts`)
**这是项目最核心的大脑。** `QueryEngine` 类负责管理对话的生命周期和执行流：
*   **`submitMessage` 流程**: 接收用户输入后的入口。
*   **上下文构建**: 获取当前的系统 Prompt (`fetchSystemPromptParts`)，收集附加工作区目录、Git状态等。
*   **循环流 (`queryLoop`)**: 这是一个异步生成器 (`async generator`)，包含了 AI 执行的核心逻辑：
    1.  **自动压缩 (Auto Compact)** & **上下文折叠 (Context Collapse)**：当对话历史超过 Token 限制时，调用 `autocompact` 或者 `snip` 截断历史，生成摘要并保持上下文窗口有效。
    2.  **API 调用 (`deps.callModel`)**: 将合并好的消息发送给 Claude 的大模型。采用流式响应。
    3.  **大模型 Streaming 处理**: 实时接收 `message_start`、`message_delta`，记录 token 消耗。
    4.  **工具调用拦截**: 当模型返回 `tool_use` 块时，停止文本流，进入工具执行系统 (`StreamingToolExecutor`)。
    5.  它通过 `yield` 将中间进度抛出给 UI 渲染。

---

## 3. 工具系统 (Tool System)

工具系统让 Claude 能够与本地系统进行交互。

### 3.1 基础定义 (`src/Tool.ts`)
定义了 `Tool` 接口模板：
*   `inputSchema`: 基于 Zod 定义的工具输入结构。
*   `call`: 实际执行业务逻辑的方法。
*   `checkPermissions`: 在执行前校验用户权限和沙箱限制。
*   `isSearchOrReadCommand` / `isDestructive`: 元数据标识，用于 UI 如何呈现以及判定危险程度。
*   `renderToolUseMessage` / `renderToolResultMessage`: 决定工具在 Ink 终端中如何展示其执行过程和最终结果。

### 3.2 注册与分发 (`src/tools.ts`)
统一收集和导出所有工具。
*   提供了如 `BashTool`, `FileEditTool`, `FileReadTool`, `AgentTool` 等工具。
*   根据模式 (`isReplModeEnabled`, `COORDINATOR_MODE`) 动态筛选可暴露给模型的工具集。
*   结合 `mcpTools` (Model Context Protocol 动态注册的工具) 一起进行权限过滤 (`filterToolsByDenyRules`) 和组合。

### 3.3 BashTool 剖析 (`src/tools/BashTool/`)
这是目前最核心也最复杂的工具之一。
*   实现了安全的 `BashTool` 调用。
*   **权限校验**: 拦截了破坏性操作，禁止自动放入后台执行某些命令（如 `sleep` 单独命令会引导使用 `MonitorTool`）。
*   **结果捕获**: 将 stdout/stderr 返回，对于大型输出具有持久化能力 (`maxResultSizeChars: 30000`)，当超出长度时截断并生成一个本地链接，防止撑爆大模型的上下文。

---

## 4. 命令与技能系统 (Commands & Skills)

用户在终端输入 `/` 触发的内容。位于 `src/commands/` 和 `src/commands.ts`。

*   **命令类型 (`Command` interface)**:
    1.  **内置 UI 操作 (`local` / `local-jsx`)**: 纯前端行为，例如 `/clear` (清屏), `/cost` (显示计费), `/theme` (切换主题), `/exit`。
    2.  **向模型发出的功能性 Prompt (`prompt` / Skills)**: 直接作为包装好的 Prompt 发送给模型执行。比如 `/code-review`、`/commit` 等。
*   **命令装载机制**: `getCommands` 时，不仅加载内部的内置命令，还会动态读取 `plugins/` 目录以及自定义路径下的工作流配置 (`WorkflowTool`)。

---

## 5. 项目亮点与特殊机制

### 5.1 上下文压缩机制 (Context Compression)
当对话变长时，项目采用了业界领先的做法：
*   **Microcompact**: 特针对长期驻留的 Tool Output 进行微型摘要替换，减少单个长输出的负担。
*   **Auto Compact**: 当总 tokens 接近上限，系统会触发自动压缩算法，它把之前的详细问答交由模型进行历史摘要提取，然后替换掉原来的长消息列表，这是长线程任务不崩溃的精髓。

### 5.2 权限与沙箱防线 (Permissions & Sandbox)
*   **拦截策略**: 基于 Glob 的白名单/黑名单模式（如配置始终允许 `Bash(ls *)`，阻止 `WebSearch` 等）。
*   **自动拒绝模式**: 对于后台执行的 Agent，为了防止阻塞，会开启 `shouldAvoidPermissionPrompts`，直接返回权限拒绝给模型，让其尝试其他思路。

### 5.3 花费追踪 (`src/cost-tracker.ts`)
*   内部使用 `ModelUsage` 同步记录 `inputTokens`, `outputTokens`, `cacheReadInputTokens` 等数据。
*   在缓存构建方面（Anthropic Prompt Caching），极其精细地记录了 `cacheCreationInputTokens`，实时换算为美元 ($) 呈现在终端上。

### 5.4 插件与 Agents (Agent Swarms)
*   项目不仅限于单线聊天，还具备 Subagent (子代理) 处理的逻辑 (如 `AgentTool` 引发的 `TeamCreateTool`)，允许主模型将特定的耗时、复杂的子任务派发给新的实例。
*   `plugins/code-review` 示例展示了由多个评估器组成的复杂流水线，每个独立审核、打分，最终过滤虚假报警合并输出，是非常好的生产级代理开发学习材料。

---

## 6. 推荐学习路径

为了快速掌握这个庞大系统，建议按照以下先后顺序阅读源码：

1.  **入口挂载流**: 
    *   阅读 `src/main.tsx`，了解从进程唤醒到配置加载的过程。
2.  **核心执行流**: 
    *   阅读 `src/QueryEngine.ts`。重点看 `queryLoop()` 方法，搞懂一段文本如何变成 API 请求，并在流中途执行工具的。
3.  **工具是如何被 AI 调用的**:
    *   看 `src/Tool.ts` 掌握基类，然后看 `src/tools/FileReadTool` (相对简单，适合作为范例)。
    *   接着看 `src/tools/BashTool.tsx` 了解复杂工具、命令拆解和沙箱。
4.  **UI 是如何渲染的 (Ink)**:
    *   搜索并查看项目中的 `.tsx` UI 文件，体会如何用 React 构建终端 UI 和实时刷新进度日志。
5.  **系统高级功能扩展**:
    *   **上下文处理**: 研究 `src/utils/messages.js` 和 `src/services/compact/`。
    *   **插件编写**: 研究 `plugins/code-review/README.md` 和其对应的命令源码。

---
*提示：本项目源码恢复自主，非常适合做为基于 AI 驱动开发平台 (LLM Agent OS) 的终极参照模板。*
