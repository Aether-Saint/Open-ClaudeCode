# Open-ClaudeCode 深入架构解析：意图识别与 AI 智能体编排

本文基于对 Open-ClaudeCode 项目核心代码（主要位于 `src/QueryEngine.ts`、`src/utils/processUserInput/` 目录以及 `src/tools/AgentTool/` 目录）的深度挖掘，为您详细解密其用户意图识别机制与 Agent 编排管理系统。

---

## 1. 意图识别系统 (Intent Recognition)

Open-ClaudeCode 的意图识别并非仅靠大模型猜测，而是在发送给 LLM 之前，存在一套极其复杂的 **前置处理管线 (`src/utils/processUserInput/processUserInput.ts`)**。

它的意图识别按以下层级递进：

### 1.1 输入预处理与附件嗅探
当用户在终端按下回车后，系统首先进行多模态输入拆分：
*   **图像归一化 (Image Resize & Downsample)**：将粘贴的过大图片在本地降采样并放入本地暂存区，生成供模型读取的基础 `ContentBlock`，并同时抽取出图像尺寸等 Meta 描述文字一并存入。
*   **@提及 (Agent Mention)**：提取输入中的 `@agent-xxx` 语法，将其转化为系统认知层面的 `agent_mention_attachment`。系统会对“仅含有@agent”或“以@agent开头”的输入进行记录与特殊路由，让用户明确指定由哪个特化模型/Agent接管任务。
*   **文件提及嗅探**：将文本中的路径、文件进行补齐。

### 1.2 快捷命令与硬意图 (Hard Intents) 拦截
*   **Slash Command (`/`) 解析**：任何以 `/` 开头的输入直接由系统本身截获。这些意图划分为：
    *   `local` / `local-jsx`：纯本地系统命令（如 `/clear`, `/cost`, `/theme`）。系统会在前端拦截，绝不调用大模型API，节省 Token 与时间。
    *   `skill` / `prompt`：需要调用大模型的“模板化技能”（如 `/code-review`）。系统会在用户输入背后拼接大量的隐式 Prompt 与上下文再交给 `QueryEngine`。
*   **安全与远程网关过滤**：当判断请求是通过 `Remote Control`（比如移动端小程序或网页版）传来时，它会跳过执行任何具有本地破坏性或涉及 UI 挂载的 Slash Command，仅放行处于白名单（`BRIDGE_SAFE_COMMANDS`）中的命令。

### 1.3 隐式指令触发 (Implicit Keyword Triggers)
系统会监听特定的模式来做自动流转。例如：
*   **Ultraplan 拦截器**：内置了特定关键字（如计划模式等词汇）监听器 `hasUltraplanKeyword`。一旦检测到用户在进行战略性规划或探索，即使没有加 `/`，系统也会在后台自动将其重写为 `/ultraplan` 命令，调用高度系统级沙盒以预防危险行为。

### 1.4 Post-Hook 阻塞与执行
在决定放行给 `QueryEngine` 之前，会经历一把 `executeUserPromptSubmitHooks` (用户 Prompt 提交流程钩子)。在这一步，可以通过注册的插件或脚本进一步补充上下文（如动态读取外部文档）或者阻拦当前任务。

---

## 2. 智能体协调与编排管理 (Agent Orchestration)

Open-ClaudeCode 的 Agent 系统极为庞大，采用**树状/蜂群架构 (Swarm Architecture)**，主循环大模型具备决定何时、以何种方式下发子任务的自主权。核心实现在 `src/tools/AgentTool/`。

### 2.1 Agent 基础定义
每个 Agent 通过 `AgentDefinition` 描述自身特性。内置的包括：
*   `GENERAL_PURPOSE_AGENT` (通用)
*   `Explore` (只读的安全搜索环境)
*   `Plan` (规划)
*   也可以自定义 Agent，每个 Agent 可以配置：
    *   独立的模型覆写 (`model: "haiku" | "sonnet"`)，控制成本。
    *   独立的权限沙箱模式 (`permissionMode`)。
    *   甚至自带 **私有的 MCP Servers (上下文协议服务)**。

### 2.2 派发子代理 (AgentTool 机制)
当主模型意识到一个任务过于庞大、容易踩坑或者需要较长的前置环境准备时，它会调用 `AgentTool` (这是它暴露给大模型最强大的 Tool 之一)。

#### 2.2.1 派发模式
系统提供了三种 Agent 生成机制：
1.  **同步执行 (Sync Subagent)**：主模型进入等待，终端当前显示 Subagent 正在思考和执行。子 Agent 任务结束返回后，主模型继续。
2.  **异步执行 (Async Background Agent)**：由参数 `run_in_background` 触发。它被注册为 `LocalAgentTask`，会在后台默默跑，生成一个单独的输出文件。主 Agent 立刻将控制权交还给用户。
3.  **Tmux 蜂群 (Agent Swarms & Teammates)**：当调用参数带有 `team_name` 和 `name` 属性时，系统会在真实的 CLI Tmux 里**水平切割窗口 (Splitpane)**，生成一个新的 Agent 进程！主 Agent 与 Teammates 协同推进。

#### 2.2.2 高级物理隔离 (Isolation Modes)
这可能是这个项目最惊艳的设计之一：
*   **Worktree 隔离 (`isolation: "worktree"`)**: 如果子 Agent 被指派去“尝试并验证重构一段代码库”，系统会利用 git `worktree` 在磁盘缓存目录下极其快速地**Copy 出一份当前代码库分身**。子 Agent 在分身里增删改查、运行报错乃至搞乱代码，一切发生在这个无痕环境中！结束后，主模型只需要查看这颗抛弃型 Worktree 的变动（或Diff），绝不会破坏用户的当前主战场！
*   **Remote 隔离**：针对企业级，自动将任务派发至云端计算节点。

### 2.3 信息交互机制与 `runAgent.ts`
当一个 Agent 启动时，`runAgent` 函数将负责装载它的物理执行容器：

1.  **上下文裁剪 (Context Truncation)**
    主系统默认抓取了超过 3万 token 的上下文 (系统指令、Git Status等)。但当派发类似 `Explore` 这类不负责写代码的 Agent 时，系统会故意裁剪掉 `Claude.md` 解析与 `Git 历史` 注入，借此节约约 15% 的 Token 成本并防止子 Agent 注意力涣散。
2.  **动态 MCP 服务挂载 (`initializeAgentMcpServers`)**
    如果 Agent 声明了特定的工具集合，系统会实时启动这些子级 MCP 服务，分配对应的 Tools。**子 Agent 运行结束时，挂载的临时 MCP 服务自动清理销毁**，从源头避免了工具污染。
3.  **Prompt 缓存复制 (Fork Subagent 实验性功能)**
    当启用 `isForkPath` 时，子 Agent 不重新生成全新的 System Prompt，而是**字节级 (Byte-identical) 克隆父对话的 System Prompt 及工具链序列**，伪装出跟父节点完全相同的前缀，这使得它向 Anthropic 的 API 请求时，能达到近乎 **100% 的 Prompt Cache Hit**！
4.  **独立的 File State Cache**
    每个 Agent 会被分配独立的读取记忆（File Read Cache），多个并发后台 Agent 各看各的文件，互不篡改。

### 总结
整体流程就是：**预处理与静态分析完成初步的安全网拦截与工具路由 -> 主 QueryEngine 基于对话内容与工具系统维持运转 -> 遇到复杂树枝状任务，主 LLM 通过 AgentTool，带上精确的物理/工作区沙盒指令，分裂出一个个轻量/或带有环境备份（worktree）的特化型 Agent。**
