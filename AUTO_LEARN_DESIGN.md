# Auto-Learning System Design - 自动学习系统设计文档

> 记录 Novi Agent 自动学习系统的设计与实现历程

---

## 1. 背景与问题

### 1.1 原始问题
Agent 在 Windows 环境运行，但尝试使用 Linux 命令（如 `date`, `ls`, `cat`），导致重复失败。

### 1.2 期望目标
- Agent 自动从错误中学习，避免重复犯错
- 无需人工手动添加知识
- 真正的"自动学习"而非静态文档

---

## 2. 实现历程

### 2.1 第一版：硬编码提示（已废弃）

**文件**: `example_use/demo_agent.py`

**方式**: 在 `load_workspace_context()` 中硬编码 Windows 命令提示

**问题**:
- 需要人工维护命令映射
- 不是真正的"学习"，是预设知识
- 无法适应新的错误场景

### 2.2 第二版：动态 OS 检测 + 原始日志提取

**改动**:
- 使用 `platform.system()` 动态检测操作系统
- 添加 `import platform` 并替换硬编码 Windows 提示

**模块**: `novi_skill_framework/skill_system/framework/auto_learn.py`

**功能**:
- 从 `debug.jsonl` 提取失败记录
- 解析 `tool_call_result` 事件中 `ok=false` 的命令
- 自动注入到 system prompt

**问题**:
- 传入多少条记录是硬编码
- 传入的是原始日志信息，不够精炼
- LLM 可能会忽略或遗漏

### 2.3 第三版：智能日志分析与文档沉淀

**新增模块**:
- `log_analyzer.py` - 用 LLM 从日志中提炼经验
- `debug_lookup.py` - 封装 grep 实现快速查询

**工作流程**:
```
原始日志 → auto_learn → (可选) log_analyzer(LLM) → debug_insights.md
                                        ↓
                              debug_lookup(查询工具) → agent 查询解决方案
```

**问题**:
- 本质仍是静态文档，不是实时学习
- Agent 需要主动查询，不是自动应用
- 无法验证是否真的"学会"

### 2.4 第四版（当前）：闭环自愈系统

**目标**: 实现真正的自动学习，包括：
1. 执行前预防性检查
2. 失败后自动尝试替代方案
3. 闭环验证量化学习效果

---

## 3. 架构设计

### 3.1 核心模块

```
auto_learn_system/
├── core/
│   ├── __init__.py
│   ├── self_healer.py      # 执行前检查 + 失败自愈
│   ├── failure_recorder.py # 失败记录与模式提取
│   ├── solution_store.py  # 向量化解决方案存储
│   └── verifier.py         # 闭环验证与效果量化
├── tools/
│   ├── __init__.py
│   └── learning_tools.py   # LangGraph 可用的 tools
└── utils/
    ├── __init__.py
    └── cli_tools.py        # CLI 辅助工具
```

### 3.2 数据流

```
Agent 执行命令
       ↓
[SelfHealer] 执行前检查
       ↓
  ┌───┴───┐
  ↓       ↓
已知失败  未知命令
  ↓       ↓
返回警告/拒绝  执行命令
       ↓
  ┌───┴───┐
  ↓       ↓
 成功    失败
  ↓       ↓
记录成功经验  [SolutionStore] 查找替代方案
                    ↓
              ┌─────┴─────┐
              ↓           ↓
         有替代方案    无替代方案
              ↓           ↓
          尝试替代      记录失败
              ↓           ↓
         执行替代      [FailureRecorder]
              ↓
         成功/失败 → [Verifier] 验证学习效果
```

### 3.3 核心接口

```python
class SelfHealingAgent:
    """自愈 Agent 包装器"""
    
    def execute_with_learning(self, tool_name: str, tool_args: dict) -> dict:
        """执行命令，失败后自动尝试替代方案"""
        
    def should_allow_execution(self, tool_name: str, tool_args: dict) -> tuple[bool, str]:
        """执行前检查，返回 (是否允许, 原因)"""
        
    def verify_learning_effectiveness(self) -> dict:
        """验证学习效果，返回量化指标"""


class LearningMetrics:
    """学习效果量化指标"""
    
    total_attempts: int           # 总尝试次数
    successful_fixes: int        # 成功自愈次数
    prevented_failures: int      # 预防失败次数
    regression_count: int        # 重蹈覆辙次数
    improvement_rate: float       # 改进率 = (prevented + fixed) / total
```

---

## 4. 集成方式

### 4.1 快速接入

```python
from auto_learn_system import SelfHealingAgent
from auto_learn_system.tools import build_learning_tools

# 包装现有 agent
healing_agent = SelfHealingAgent(
    original_agent=my_agent,
    log_path="path/to/debug.jsonl",
    solution_store_path="path/to/solutions",
)

# 在 LangGraph 节点中使用
def node_with_learning(state):
    result = healing_agent.execute_with_learning("run_shell_command", {"command": "date"})
    return {"result": result}

# 或者使用 tools
tools = build_learning_tools(healing_agent)
```

### 4.2 配置项

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `log_path` | 必填 | 调试日志路径 |
| `solution_store_path` | `./learning_data/solutions` | 向量存储路径 |
| `max_retries` | 2 | 最大替代方案尝试次数 |
| `prevention_mode` | true | 是否启用执行前预防 |
| `auto_heal` | true | 是否启用失败后自愈 |

---

## 5. 验证指标

### 5.1 学习效果指标

```python
{
    "improvement_rate": 0.85,        # 改进率 (0-1)
    "total_attempts": 100,          # 总尝试
    "prevented_failures": 50,       # 预防失败
    "successful_fixes": 35,         # 成功自愈
    "regression_count": 15,         # 重蹈覆辙
    "learning_streak": 5,           # 连续学习成功次数
}
```

### 5.2 验证方法

1. **预防验证**: 记录被预防的失败次数
2. **自愈验证**: 记录通过替代方案成功的次数
3. **回归验证**: 记录同类型错误重复发生的次数
4. **长期追踪**: 按时间窗口统计改进率趋势

---

## 6. 文件清单

| 文件 | 功能 |
|------|------|
| `auto_learn.py` | 原始日志提取（第二版） |
| `log_analyzer.py` | LLM 日志分析与文档沉淀（第三版） |
| `debug_lookup.py` | 快速查询工具（第三版） |
| `self_healer.py` | 执行前检查 + 失败自愈（第四版核心） |
| `failure_recorder.py` | 失败记录与模式提取 |
| `solution_store.py` | 向量化解决方案存储 |
| `verifier.py` | 闭环验证与效果量化 |
| `learning_tools.py` | LangGraph tools 封装 |

---

## 7. 待实现

- [ ] self_healer.py - 核心自愈逻辑
- [ ] failure_recorder.py - 失败记录
- [ ] solution_store.py - 向量存储
- [ ] verifier.py - 闭环验证
- [ ] learning_tools.py - LangGraph 集成
- [ ] CLI 工具
- [ ] 单元测试

---

## 8. 经验教训

1. **不要假设 OS**: 用 `platform.system()` 动态检测
2. **不要硬编码知识**: 从日志中自动提取
3. **静态文档不够**: 需要实时闭环
4. **验证很重要**: 量化学习效果才能持续优化

---

## 9. 第四版实现：闭环自愈系统

### 9.1 模块结构

```
novi_skill_framework/skill_system/framework/auto_learn_system/
├── __init__.py          # 导出入口
├── self_healer.py       # 核心自愈逻辑
├── failure_recorder.py  # 失败记录
├── solution_store.py    # 解决方案存储
├── verifier.py          # 学习效果验证
├── learning_tools.py    # LangGraph 工具封装
└── cli.py               # CLI 工具
```

### 9.2 快速接入

```python
from novi_skill_framework.skill_system.framework.auto_learn_system import SelfHealingAgent

# 定义原始执行函数
def my_execute(tool_name, tool_args):
    return execute_tool(tool_name, tool_args)

# 创建自愈 Agent
healing_agent = SelfHealingAgent(
    original_agent=my_execute,
    log_path="path/to/debug.jsonl",
    solution_store_path="path/to/solutions",
    prevention_mode=True,  # 执行前检查
    auto_heal=True,        # 失败后自愈
    max_retries=2,
)

# 使用
result = healing_agent.execute_with_learning(
    "run_shell_command", 
    {"command": "date"}
)

# 查看学习效果
metrics = healing_agent.verify_learning_effectiveness()
print(metrics)
# {'improvement_rate': 0.85, 'total_attempts': 100, ...}
```

### 9.3 使用 LangGraph Tools

```python
from novi_skill_framework.skill_system.framework.auto_learn_system import build_learning_tools

tools = build_learning_tools(healing_agent)
# 返回 5 个 tool:
# - learning_query_failures: 查询已知失败
# - learning_query_solutions: 查询解决方案
# - learning_check_safety: 执行前检查安全
# - learning_get_metrics: 获取学习指标
# - learning_rebuild: 重建学习数据
```

### 9.4 CLI 使用

```bash
# 分析日志
python -m auto_learn_system.cli analyze path/to/debug.jsonl

# 查看学习指标
python -m auto_learn_system.cli metrics --store ./solutions

# 检查命令安全性
python -m auto_learn_system.cli check "date" --store ./solutions

# 重建学习数据
python -m auto_learn_system.cli rebuild path/to/debug.jsonl --store ./solutions
```

## 10. 经验提炼机制（第五版）

### 10.1 背景问题
- 原始数据占用上下文过多
- 信息冗余导致噪声增加
- Agent 难以从原始日志中提取有价值信息

### 10.2 解决方案
新增 `ExperienceLearner` 模块，从原始记录中**提炼结构化经验**：

```
原始记录 (successes/failures)
        ↓
ExperienceLearner.learn_from_records()
        ↓
提炼成结构化经验 (summary, lesson, action)
        ↓
存储到 learned_experiences.json
        ↓
注入到 Agent 的 system prompt
```

### 10.3 ExperienceLearner 模块

```python
from auto_learn_system import ExperienceLearner

learner = ExperienceLearner(store_path="./learning_data/experiences")

# 从记录中提炼经验（无 LLM，使用规则引擎）
experiences = learner.learn_from_records(successes, failures)

# 从记录中提炼经验（使用 LLM）
experiences = learner.learn_from_records(
    successes, failures,
    llm_provider=lambda p: llm.invoke(p).content
)

# 获取用于 prompt 的经验上下文
context = learner.format_experiences_for_prompt(max_count=10)
```

### 10.4 经验格式

```json
{
  "command": "date",
  "type": "failure",
  "summary": "命令 date 失败 2 次",
  "lesson": "Command 'date' timed out after 30 seconds",
  "action": "使用内置替代命令",
  "learned_at": "2026-04-03T12:00:00Z"
}
```

### 10.5 集成到 Agent Prompt

```python
# demo_agent.py 中的用法
healing = _create_healing_agent()
learned_context = healing.learn_and_get_context()
# 返回格式化的经验文本，注入到 system prompt
```

### 10.6 记录成功经验

FailureRecorder 新增成功记录功能：

```python
# 成功执行时记录
recorder.record_success(tool_name, tool_args, output)

# 获取所有成功/失败记录
successes = recorder.get_all_successes()
failures = recorder.get_all_failures()
```

---

## 11. 数据流完整版

```
┌─────────────────────────────────────────────────────────────────┐
│                        Agent 执行命令                            │
└─────────────────────────────┬───────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    SelfHealingAgent                              │
│  ┌─────────────────┐    ┌─────────────────┐                   │
│  │ 执行前检查       │    │ 执行并记录结果   │                   │
│  │ should_allow    │    │ execute_with_   │                   │
│  │ _execution()    │    │ learning()      │                   │
│  └────────┬────────┘    └────────┬────────┘                   │
│           ↓                      ↓                            │
│  ┌─────────────────┐    ┌─────────────────┐                   │
│  │ 已知失败?      │    │ 成功/失败记录   │                   │
│  │ 阻止/警告      │    │ FailureRecorder │                   │
│  └─────────────────┘    └────────┬────────┘                   │
│                                  ↓                             │
│                          ┌─────────────────┐                   │
│                          │ 经验提炼        │                   │
│                          │ Experience     │                   │
│                          │ Learner        │                   │
│                          └────────┬────────┘                   │
│                                   ↓                            │
│                          ┌─────────────────┐                   │
│                          │ 注入 prompt    │                   │
│                          │ 下次推理使用    │                   │
│                          └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 12. 文件清单（完整版）

| 文件 | 功能 | 版本 |
|------|------|------|
| `auto_learn.py` | 原始日志提取 | v2 |
| `log_analyzer.py` | LLM 日志分析与文档沉淀 | v3 |
| `debug_lookup.py` | 快速查询工具 | v3 |
| `self_healer.py` | 执行前检查 + 失败自愈 | v4 |
| `failure_recorder.py` | 成功/失败记录与模式提取 | v4 |
| `solution_store.py` | 解决方案存储 + 内置 OS 映射 | v4 |
| `verifier.py` | 闭环验证与效果量化 | v4 |
| `experience_learner.py` | 经验提炼（核心） | v5 |
| `learning_tools.py` | LangGraph tools 封装 | v4 |
| `cli.py` | 命令行工具 | v4 |

---

## 13. 已完成功能

- [x] self_healer.py - 核心自愈逻辑
- [x] failure_recorder.py - 成功/失败记录
- [x] solution_store.py - 内置 OS 解决方案 + 可扩展
- [x] verifier.py - 闭环验证
- [x] learning_tools.py - LangGraph 集成
- [x] cli.py - 命令行工具
- [x] experience_learner.py - 经验提炼
- [x] 集成到 demo_agent.py - 每次推理自动学习

---

## 14. 核心经验教训

1. **不要假设 OS**: 用 `platform.system()` 动态检测
2. **不要硬编码知识**: 从日志中自动提取
3. **不要用原始数据**: 提炼成经验后再注入
4. **静态文档不够**: 需要实时闭环
5. **验证很重要**: 量化学习效果才能持续优化
6. **信息要精简**: 原始日志是噪声，提炼后才是信号

---

## 15. Agentic 模式（第六版）

### 15.1 新增特性

- **LLM 判断函数**: 可自定义结果判断逻辑
- **用户校正反馈**: 记录用户提供的正确做法
- **自我反思**: 执行后自动判断是否需要用户关注
- **泛化支持**: 支持所有工具类型，不只是 shell 命令

### 15.2 LLM 判断函数

```python
# 自定义判断逻辑
def my_llm_judger(tool_name, tool_args, result):
    # 使用 LLM 判断执行结果
    return llm.invoke(f"判断以下结果是否成功: {result}")

healing_agent = SelfHealingAgent(
    original_agent=my_execute,
    llm_judger=my_llm_judger,  # 可选
)
```

### 15.3 用户校正反馈

当用户指出 LLM 的错误并提供正确做法时记录：

```python
# 用户允许执行越权命令后记录
healing_agent.record_user_correction(
    tool_name="run_shell_command",
    tool_args={"command": "ls /root", "cwd": "/root"},
    user_correction="用户批准访问受限目录: /root",
    original_result="拒绝访问"
)
```

### 15.4 自我反思模式

```python
# 带有自我反思的执行
result = healing_agent.execute_with_reflection(
    "skill_analyze_data",
    {"file": "data.csv"}
)

# 返回:
# {
#     "ok": bool,
#     "result": {...},
#     "reflection": "执行失败，错误: ...",  # 反思内容
#     "needs_user_attention": True/False   # 是否需要用户关注
# }
```

### 15.5 泛化支持

现在支持所有工具类型，不只是 `run_shell_command`：

```python
# 自动为所有工具启用学习
result = healing_agent.execute_with_learning("skill_analyze_data", {"file": "x.csv"})
result = healing_agent.execute_with_learning("memory_search", {"query": "..."})
result = healing_agent.execute_with_learning("submit_cluster_job", {"command": "..."})
```

### 15.6 集成到 demo_agent.py

```python
# 1. 创建自愈 Agent
healing_agent = SelfHealingAgent(
    original_agent=original_execute,
    log_path=log_path,
    solution_store_path=solution_path,
    prevention_mode=True,
    auto_heal=True,
    max_retries=2,
)

# 2. 在 LLM 推理前注入学习上下文
learned_context = healing_agent.learn_and_get_context()

# 3. 所有工具执行都经过自愈系统
result = healing_agent.execute_with_learning(tool_name, tool_args)

# 4. HITL allow 时记录用户校正
healing_agent.record_user_correction(
    tool_name=tname,
    tool_args=targs,
    user_correction="用户批准...",
    original_result=original_resp
)
```

### 15.7 获取所有经验（包括用户校正）

```python
# 获取格式化的经验上下文
context = healing_agent.experience_learner.format_all_for_prompt(max_count=10)

# 输出格式:
# ## 已学习经验
# ### ❌ 命令失败: xxx
# - 总结: ...
# - 经验: ...
#
# ## 用户校正经验
# ### 工具: xxx
# - 用户校正: ...
# - 原始错误输出: ...
```

---

## 16. Agentic 工作流

```
┌─────────────────────────────────────────────────────────────────┐
│                     Agent 执行工具                               │
└─────────────────────────────┬───────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  SelfHealingAgent                               │
│  ┌─────────────────┐    ┌─────────────────┐                   │
│  │ 执行前检查       │    │ 执行工具         │                   │
│  │ should_allow    │    │ original_agent  │                   │
│  └────────┬────────┘    └────────┬────────┘                   │
│           ↓                      ↓                             │
│  ┌─────────────────┐    ┌─────────────────┐                   │
│  │ 已知失败?      │    │ LLM 判断结果    │                   │
│  │ 阻止/警告      │    │ _llm_judge()    │                   │
│  └─────────────────┘    └────────┬────────┘                   │
│                                  ↓                             │
│                          ┌─────────────────┐                   │
│                          │ 记录结果        │                   │
│                          │ _record_       │                   │
│                          │ execution_     │                   │
│                          │ result()       │                   │
│                          └────────┬────────┘                   │
│                                   ↓                            │
│                          ┌─────────────────┐                   │
│                          │ 自我反思        │                   │
│                          │ execute_with_  │                   │
│                          │ reflection()   │                   │
│                          └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│              用户校正反馈回路 (HITL)                             │
│                                                                  │
│  用户 allow → record_user_correction() → 学习用户意图          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 17. 配置项（完整版）

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `original_agent` | 必填 | 原始执行函数 |
| `log_path` | 必填 | 调试日志路径 |
| `solution_store_path` | `./learning_data/solutions` | 解决方案存储 |
| `max_retries` | 2 | 最大替代方案尝试次数 |
| `prevention_mode` | true | 执行前预防检查 |
| `auto_heal` | true | 失败后自动尝试替代 |
| `llm_judger` | None | 自定义 LLM 判断函数 |

---

## 18. 文件清单（第六版）

| 文件 | 功能 | 版本 |
|------|------|------|
| `self_healer.py` | Agentic 核心（LLM判断、用户校正、自我反思） | v6 |
| `failure_recorder.py` | 泛化 key 生成，支持所有工具 | v6 |
| `solution_store.py` | 解决方案存储 | v4 |
| `verifier.py` | 闭环验证与效果量化 | v4 |
| `experience_learner.py` | 用户校正记录 + 经验格式化 | v6 |
| `learning_tools.py` | LangGraph tools 封装 | v4 |

---

## 19. 已完成功能

- [x] LLM 判断函数 (llm_judger)
- [x] 用户校正反馈记录 (record_user_correction)
- [x] 自我反思模式 (execute_with_reflection)
- [x] 泛化到所有工具类型
- [x] 集成到 demo_agent.py
- [x] HITL allow 时记录校正

---

*本文档更新于 2026-04-03*
