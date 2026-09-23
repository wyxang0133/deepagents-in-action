# 【DeepAgents 实战】Task4：任务规划与分解 —— 实践笔记

> 📅 学习日期：2026-09-23
> 📖 章节地址：https://datawhalechina.github.io/deepagents-in-action/chapters/ch04-task-planning/
> 🗓️ 学习周期：Day 4-6（3 天，第四章）

---

## 一、本章内容归纳

### 1. 为什么 Agent 需要"规划"能力？

简单任务（如查天气）一步即可完成；复杂任务（调研 → 对比 → 写报告）需要多步执行。没有规划的 Agent 容易出现四类问题：

- **遗漏关键步骤**：直接写报告，忘了先搜索竞品
- **重复劳动**：同一个关键词搜了三次
- **半途而废**：上下文太长后失去整体进度把控
- **质量不稳定**：有时好有时跳过环节

规划能力的核心是"**先思考再行动**"：拆解任务 → 逐步执行 → 追踪进度 → 动态调整。

### 2. v0.7 的重要变化：TodoListMiddleware 需显式启用

```python
from deepagents import create_deep_agent
from langchain.agents.middleware import TodoListMiddleware

agent = create_deep_agent(
    model=model,
    middleware=[TodoListMiddleware()],
)
```

是否启用应按任务特征决定：

| 场景 | 建议 |
| --- | --- |
| 单步问答、短工具调用 | 保持关闭，避免"计划比任务本身还长" |
| 长程、多阶段、易漏步骤的任务 | 启用，并用真实任务验证 |
| 能力较弱、易失去主线的模型 | 先做 A/B 评测 |
| UI 需要展示计划/进度 | 启用，`todos` 即产品状态协议 |

### 3. write_todos 工具

- **数据结构**：`{"content": "...", "status": "pending|in_progress|completed"}`
- **典型行为三阶段**：制定计划 → 逐步执行 → 动态调整（发现新需求可追加任务）
- **关键提醒**：`completed` 只是 Agent 写入的进度标记，应用仍需检查产物质量（报告是否生成、引用能否核实）

### 4. 任务清单的持久化规则

- 清单保存在 Agent State 的 `todos` 字段，与消息历史分开管理
- 默认对话总结**不会删除** `todos` 字段 —— 这正是任务清单的"锚定作用"
- 跨次 `invoke()` 接续需要 **Checkpointer + 同一个 thread_id**；只加 TodoListMiddleware 不会自动接续
- `subagents=[...]` 声明的子 Agent 有独立 Middleware 栈，需要在**自己的 spec 中**单独启用 Todo

### 5. 揭开引擎盖：LangChain 中间件

- **两类 Hook**：Node-style（before_agent / before_model / after_model / after_agent，编译为图节点）vs Wrap-style（wrap_model_call / wrap_tool_call，包裹调用）。自定义人工中断应优先放在 Node-style Hook
- **create_deep_agent() 的组装逻辑**：
  - 默认层：FilesystemMiddleware、SummarizationMiddleware、PatchToolCallsMiddleware
  - 参数激活层：SubAgentMiddleware（subagents=）、SkillsMiddleware（skills=）、MemoryMiddleware（memory=）、HumanInTheLoopMiddleware（interrupt_on=）
  - middleware=[...] 层：TodoListMiddleware 等可选策略；**同名实例是原位置完整替换，不做字段合并**
- **PatchToolCallsMiddleware**：只补齐历史中缺失的工具响应（如运行被取消留下的缺口），不重试、不校验结果正确性
- **同名陷阱**：LangChain 和 Deep Agents 都有 `SummarizationMiddleware`，执行位置（before_model 节点 vs model 节点内部）和消息处理方式不同；且两者手动实例化时 `trigger` 默认都是 None，**不会主动摘要**，必须手动设置

### 6. 规划 × 上下文管理的协同

长任务执行中，大结果卸载到文件系统 + 对话超阈值自动总结后，`todos` 字段仍完整保留，帮助 Agent 在"失忆"后仍能回答：总共有哪些步骤、哪些已完成、下一步做什么。

---

## 二、代码运行记录

### 环境

```bash
pip install deepagents langchain-openai tavily-python
export SILICONFLOW_API_KEY=xxx
export TAVILY_API_KEY=xxx
```

### 实验 1：LangChain 层手动组装（write_todos 的真身）

```python
import os
from langchain_openai import ChatOpenAI
from langchain.agents import create_agent
from langchain.agents.middleware import TodoListMiddleware
from deepagents.middleware import FilesystemMiddleware

model = ChatOpenAI(
    model="zai-org/GLM-5.2",
    api_key=os.environ["SILICONFLOW_API_KEY"],
    base_url="https://api.siliconflow.cn/v1",
)

agent = create_agent(
    model=model,
    tools=[],
    middleware=[
        TodoListMiddleware(),
        FilesystemMiddleware(),
    ],
)

result = agent.invoke({"messages": [{"role": "user", "content": "帮我规划并完成：调研 Deep Agents 的任务规划能力并写一份小结"}]})
print(result["todos"])   # 查看最终任务清单
```

**观察点**：
- Agent 在收到复杂任务时是否先调用 `write_todos` 制定计划
- 任务清单状态流转是否为 pending → in_progress → completed
- 简单任务下 Agent 是否会"跳过"规划（这是合理行为）

### 实验 2：完整实战（Tavily 搜索 + 规划 + 写文件）

```python
import os
from langchain_openai import ChatOpenAI
from tavily import TavilyClient
from deepagents import create_deep_agent
from langchain.agents.middleware import TodoListMiddleware

model = ChatOpenAI(
    model="zai-org/GLM-5.2",
    api_key=os.environ["SILICONFLOW_API_KEY"],
    base_url="https://api.siliconflow.cn/v1",
)

tavily_client = TavilyClient(api_key=os.environ["TAVILY_API_KEY"])

def internet_search(query: str, max_results: int = 5) -> dict:
    """搜索互联网获取最新信息。"""
    return tavily_client.search(query, max_results=max_results)

agent = create_deep_agent(
    model=model,
    tools=[internet_search],
    middleware=[TodoListMiddleware()],
    system_prompt="""你是一位专业的技术研究员。
面对复杂研究任务时，你会：
1. 先用 write_todos 制定研究计划
2. 逐步执行每个步骤，及时更新进度
3. 将搜索结果写入文件系统整理
4. 最终输出完整的研究报告
""",
)

result = agent.invoke({
    "messages": [{
        "role": "user",
        "content": "请调研 Agent 开发领域的三大 Harness 框架（Deep Agents、Claude Agent SDK、Codex SDK），对比它们的核心能力差异，写一份简要分析报告。"
    }]
})

print(result["messages"][-1].content)
print(result.get("todos", []))  # 检查清单与产物是否一致
```

### 运行结果

> 【此处粘贴你的运行截图 / 输出。建议展示：Agent 调用 write_todos 的工具调用记录 → 中间搜索与写文件过程 → 最终报告 → `result["todos"]` 的完成状态。可用 LangSmith 或 `agent.get_graph().draw_ascii()` 辅助展示图结构。】

### 观察与结论

1. 强模型在提示词引导下能稳定执行"先规划后执行"的流程；
2. 任务数与规划粒度受任务描述详细程度影响，任务描述越具体，计划越合理；
3. 最终报告质量仍需人工检查——清单全部 completed ≠ 内容正确（例如对比维度的合理性）。

---

## 三、踩坑填坑记录

| # | 坑 | 现象 | 解法 |
| --- | --- | --- | --- |
| 1 | v0.7 默认不启用 TodoListMiddleware | 代码与旧教程一致，但 Agent 从不调用 write_todos | 显式传入 `middleware=[TodoListMiddleware()]` |
| 2 | 跨次调用清单"丢失" | 第二次 `invoke()` 时 todos 为空 | 配置 Checkpointer + 复用同一 thread_id；InMemorySaver 仅进程内有效 |
| 3 | 手动实例化 SummarizationMiddleware 后从不摘要 | 以为是配置问题 | 两版同名类 `trigger` 默认都是 None，需手动设置，如 `trigger=("tokens", 4000), keep=("messages", 20)` |
| 4 | 子 Agent 不规划 | subagent 从不写 todo | `subagents=[...]` 声明的子 Agent 有独立 Middleware 栈，需在自己的 spec 中启用 Todo |
| 5 | 同名中间件替换困惑 | 替换后行为不符合预期 | 原位置完整替换，不逐字段合并；替换后需重新验证 |
| 6 | 把 completed 当质量保证 | 清单全绿但报告没写/写错 | 应用层必须校验产物（文件是否存在、引用能否核实） |
| 7 | Hook 类型选错做人工中断 | 恢复时逻辑重放异常 | 人工中断优先放 Node-style Hook，而非 Wrap-style |
| 8 | 简单任务过度规划 | 两步问答也生成 5 条 todo，token 浪费 | 按任务特征决定是否启用；简单场景保持关闭 |

---

## 四、学习心得

1. **"清单"是 Agent 的短期工作记忆外挂**：规划不是让模型变聪明，而是给它一个外部结构来对抗上下文漂移。这和第 3 章的上下文管理是协同关系——总结会压缩消息，但 `todos` 字段独立存活，成为长任务中的"锚"。
2. **v0.7 的"按需启用"哲学值得点赞**：框架不再默认塞满所有能力，把选择权交还给应用。代价是开发者必须理解每个 Middleware 的职责边界（本章的 PatchToolCalls vs Retry vs Checkpointer 职责对照表非常实用）。
3. **进度标记 ≠ 结果正确**：Agent 自检能力有限，生产环境中"清单全绿"只能作为流程参考，产物校验（测试、引用核实）必须留在应用侧。
4. **由表及里学习路径清晰**：先用 `create_deep_agent()` 一行开箱即用，再揭开引擎盖看 LangChain 中间件组装，最后能手动用 `create_agent()` 组合——这条"先 harness 后 framework"的路线对入门者友好。

---

## 五、对教程的意见及建议

1. ✅ **优点**：职责边界表格（Patch/Retry/Checkpointer/校验）非常清晰，是本章最有价值的部分；v0.7 变更提醒（TodoList 显式启用、trigger=None）能防止大量"旧教程跑不通"的挫败感。
2. 💡 **建议 1**："代码实战"部分可增加**对照实验**——同一任务分别启用/不启用 TodoListMiddleware，对比工具调用次数、遗漏率、轨迹长度，让"规划是否有用"从定性变定量。
3. 💡 **建议 2**：SummarizationMiddleware 两版对比一节信息量很大，建议配一个最小可运行的对照代码（设置相同的 trigger/keep，打印模型实际收到的 messages），降低动手门槛。
4. 💡 **建议 3**：PatchToolCalls 一节提到"服务端可能已处理请求"，建议补充实际案例（如写操作取消后如何做幂等补偿），与第 9 章 interrupt() 呼应。
5. 💡 **建议 4**：规划粒度的提示词工程可单独成节——任务拆多细、何时该"合并小步骤"，是实践中影响 token 成本的关键。

