# 从经验中学习：Claude Code 与 Novi Agent 自动学习机制的深度解析与对比

在开发复杂的 Agentic 系统时，如何让 Agent **“不重蹈覆辙”**并且**“越用越懂用户”**是核心痛点。本文将系统性解密 Claude Code 的长期记忆机制，并将其与您所设计的 Novi Agent `AUTO_LEARN_DESIGN.md` 进行深度横向对比。

---

## 1. Claude Code 机制解密：基于索引的自动记忆系统 (Auto Memory)

Claude Code 的长期记忆并没有采用复杂的向量数据库（Vector DB），而是完全依赖于**物理文件系统和聪明的大模型心智设定**。其核心代码位于 `src/memdir/`。

### 1.1 核心架构与载体
*   **物理存储**：所有记忆均存储于工作区的特定隐藏目录中（如 `.claude/projects/{项目名}/memory/`）。
*   **结构树 (Index & Topic Files)**：
    *   **快速索引：`MEMORY.md`**。相当于 Agent 的“记忆脑图”，里面只有一行行的超链接，每行是对一个特定记忆点的简短描述。
    *   **详情文件**：具体的细节被要求以标准的 Markdown (甚至要求附带 Frontmatter，如 name、type) 独立存储，如 `user_role.md`, `feedback_linting.md`。

### 1.2 何时触发学习与写入？
*   **常规模式 (交互式)**：系统在初始化对话时，给予模型极其严厉且精细的系统提示词（`buildMemoryLines`）。如果人类纠正了它，或者它自己发现了一个不易察觉的架构约定，它被授权**立即自行调用文件写入工具**，生成一个具体的话题文件，并在 `MEMORY.md` 加上链接。
*   **KAIROS 模式 (异步记录与夜间做梦)**：在长期常驻后台的助手模式下，随时修改索引池容易弄坏上下文，因此系统让它只能进行单纯的“流水账追加”——将其学到的东西**Append**到一个按日期命名的 `YYYY-MM-DD.md` 日志中。然后通过一个名为 `/dream` 的后台特化技能，在“夜间（闲置时）”定期将这些杂乱的日志蒸馏和提炼到原本分类良好的记忆文件中。

### 1.3 它是如何利用记忆的？
每次发起全新的对话进程，甚至每次分裂出一个平行的 Subagent，**系统都会强制将 `MEMORY.md` 以及里面的关键索引条目无条件塞入 System Prompt 中**。结合上下文，大模型如果认为某条索引十分关键（比如 `[环境配置坑点](env_bugs.md)`），它会主动自己去调用阅读工具调取详情。

---

## 2. 机制对比：Claude Code 🆚 Novi Agent

当你查看 Novi Agent 的 `AUTO_LEARN_DESIGN.md` 时，你会发现这两者虽然目标一致（从错误中学习），但由于面临的侧重点不同，它们走向了两条截然不同的架构设计道路。

### 2.1 学习对象与侧重点

| 特度 | Claude Code (Auto Memory) | Novi Agent (Self Healing) |
| :--- | :--- | :--- |
| **核心目标** | **性格对齐与项目级抽象经验沉淀** | **工具执行过程的自愈与防故障** |
| **侧重点** | 偏向人类偏好（"不要给看全部diff"）、架构事实（"生产数据库不能碰"）等难以从代码中获取的信息。 | 偏向程序执行事实（"Windows下不该运行ls"，"这个命令会time out"）。 |
| **存储介质** | Markdown 文件池 + 全局索引文件 `MEMORY.md` | 向量数据库 (`solution_store`) + 结构化 JSON (`learned_experiences`) |

### 2.2 学习闭环的触发方式 (Workflow)

*   **Claude Code —— LLM 完全自主驱动 (Agentic Autonomy)**：
    没有任何 Python 级的中间件去“强迫”它学习。系统仅仅是给它定下了规则：“嘿，遇到有价值的东西，请把它写到你的 memory 文件夹里”。这是大模型的纯自发行为，属于**声明式的宏观学习**。
*   **Novi Agent —— 框架中间件接管 (Middleware Interception)**：
    采用的是极其严密的**工程拦截器**与装饰器模式。利用 `SelfHealer` 拦截了每一层工具调用：执行前（防御） -> 执行后（捕获 JSON 日志） -> 通过特定的学习模块 `ExperienceLearner` 将原始报错提炼为严谨的经验卡片（`Summary, Lesson, Action`）。这属于**过程式的微观学习**。

### 2.3 经验的注入与生效机制

*   **Claude Code 的上下文注入**：“内存索引”作为一个整体环境铺底，它防范风险的方式是希望大模型“凭借记忆自己绕开坑”。
*   **Novi Agent 的事前预防与阻断**：不仅将提取的 `learned_context` 注入提示词，最亮眼的是它的**预防模式 (Prevention Mode) 与自动修复 (Auto Heal)**。当 Agent 准备发起一次明知必败的操作时，框架层甚至可以基于历史向量匹配，**在发往系统之前直接物理阻断执行，并抛回之前总结的替代方案**。

---

## 3. 架构碰撞与灵感归纳

如果将 Claude Code 的思想引入 Novi 的架构，或是反之，都会擦出极好的火花：

1.  **Novi 的 "Nightly Dream" 升级**：
    Novi 提到了日志可能很冗长，目前是实时进行处理。可以参考 Claude 的 `KAIROS` 机制，在 Novi 引入“延迟批量反思”。让 `ExperienceLearner` 作为后台常驻进程，在每天系统闲置时批量处理 `debug.jsonl`，定期自动合并相似的 `learned_experiences`。
2.  **给 Claude 注入阻断机制**：
    Claude 目前的记忆机制过于依赖大模型的“听话程度”（大模型如果一时走神没看 `MEMORY.md` 中“不要运行rm -rf”的警告，依然会发车）。在实际生产环境中，Novi 的 `SelfHealer.should_allow_execution()` 这种**物理防御层**是非常必要且具备更强安全保障的。
3.  **Human-in-the-Loop 的双向学习**：
    Novi 设计了非常出色的 `record_user_correction()`，用于捕获人类用户的直接介入与校正。在高度自动化的 Agent 时代，人类拦截不仅是安全保障，更是最精华的高信噪比训练数据源。

**总结**：Claude Code 构建了一个极具文科特质的**自主知识管理库**，而 Novi Agent 构建了一个极具理科特质的**带有反馈控制环的自动化执行拦截器**。两者完美呈现了在解决系统泛化与学习能力时，"Prompt Engineering" 和 "Software Engineering" 的不同美感。
