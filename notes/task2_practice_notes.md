# Task2 实践笔记：运行第一个 Deep Agent

> - **Task**：Task2 — 运行第一个 deep agent（第 1、2 章）
> - **教程仓库**：[datawhalechina/deepagents-in-action](https://github.com/datawhalechina/deepagents-in-action)
> - **本人仓库**：[wyxang0133/deepagents-in-action](https://github.com/wyxang0133/deepagents-in-action)
> - **环境**：Windows 11 + WSL2 (Ubuntu) + Python 3.12/3.13 (uv 虚拟环境) + AgentSeek 0.1.2
> - **日期**：2026-09-17

---

## 一、学习内容归纳

### 第 1 章：Deep Agents 的设计定位

Deep Agents 是 LangChain 团队开源的 Agent Harness（脚手架），核心思想是：**把"规划、上下文管理、子 Agent 委派"等复杂能力打包成开箱即用的运行时，开发者只需要写业务工具**。

与"自己拼 LangGraph 状态机"相比，Deep Agents 默认提供：

- **虚拟文件系统**（`write_file` / `read_file` / `ls`）：管理超长上下文，避免溢出
- **任务规划**（`TodoListMiddleware` 提供 `write_todos`）：长任务拆解
- **子 Agent 委派**（`task` 工具）：复杂子任务交给专门 Agent

一句话：**写 Agent 的门槛从"设计图结构"降到"写 Python 函数"**。

### 第 2 章：5 分钟构建第一个 Deep Agent

#### 1. 安装与配置

```bash
pip install deepagents langchain-openai   # Python 要求 3.11+
export SILICONFLOW_API_KEY="your-key"
export MODEL_NAME="Qwen/Qwen2.5-7B-Instruct"  # 当前免费、支持 Tools
```

推荐模型提供商为**硅基流动（SiliconFlow）**：国内直连、兼容 OpenAI 接口、有免费模型。

#### 2. 核心 API

```python
agent = create_deep_agent(
    model=model,            # ChatOpenAI 实例，或 "provider:model" 字符串
    tools=[my_tool],        # 自定义工具函数列表
    system_prompt="...",    # 系统提示词（人设）
)
result = agent.invoke({"messages": [{"role": "user", "content": "..."}]})
print(result["messages"][-1].content)
```

注意 v0.7 起**不再默认启用任务规划**，需要 `TodoListMiddleware()` 显式传入 `middleware` 参数才有 `write_todos`。

#### 3. 自定义工具三要素

一个普通 Python 函数 = 一个工具，关键是：

| 要素 | 作用 |
| --- | --- |
| 参数类型标注 | 告诉 Agent 每个参数传什么类型 |
| Docstring | 告诉 Agent 工具的用途和调用时机 |
| 默认值 | 标记可选参数，减少必填项 |

#### 4. `invoke()` 背后发生了什么

只写 1 行调用，Agent 自动完成 10+ 次工具调用：规划任务 → 搜索信息 → 用 `write_file` 管理上下文 → （可选）委派子 Agent → `read_file` 汇总写报告。这正是 Harness 的价值。

#### 5. AgentSeek（pre01 准备篇）与第 2 章的关系

这一点我花了一点时间才理清：

- **AgentSeek** 是"模板 + 生命周期工具"：`agentseek create deepagents/research` 生成完整研究应用（LangGraph 后端 + React 前端），并负责装依赖、启动服务。
- **第 2 章** 教你写 Agent 的**核心代码**：`create_deep_agent(...)`。
- 二者关系：AgentSeek 生成的项目里 `src/research_deepagent/agent.py` **内部就是 `create_deep_agent` 写的**，本质相同。AgentSeek 管"项目怎么搭起来、怎么启动"，第 2 章管"Agent 代码怎么写"，互不冲突。

---

## 二、代码运行记录

### 运行环境

- WSL2 (Ubuntu)，项目目录：`~/research_deepagent`（由 `agentseek create deepagents/research --checkout main --no-input` 生成）
- 项目依赖已通过 `agentseek task sync` 装好（uv 管理的 `.venv`）
- 模型：SiliconFlow `Qwen/Qwen2.5-7B-Instruct`（免费、支持工具调用）

### Hello Agent 代码（`hello_agent.py`）

```python
import os
from langchain_openai import ChatOpenAI
from deepagents import create_deep_agent

model = ChatOpenAI(
    model=os.environ.get("MODEL_NAME", "Qwen/Qwen2.5-7B-Instruct"),
    api_key=os.environ["SILICONFLOW_API_KEY"],
    base_url="https://api.siliconflow.cn/v1",
)

def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

agent = create_deep_agent(
    model=model,
    tools=[get_weather],
    system_prompt="You are a helpful assistant.",
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "北京今天天气怎么样？"}]}
)
print(result["messages"][-1].content)
```

运行命令（在项目根目录）：

```bash
uv run python hello_agent.py
```

### 运行结果

Agent 正确识别出需要调用 `get_weather` 工具，传入参数 `city="北京"`，最终回复：

```
北京的天气总是阳光明媚！(It's always sunny in 北京!)
```

（Qwen2.5-7B 会先用工具拿到英文结果，再组织中文回答。）

---

## 三、踩坑填坑记录（本次重点）

### 坑 1：在系统 Python 里跑 → `ModuleNotFoundError: No module named 'langchain_openai'`

**现象**：在项目目录下直接 `python` 进入 REPL 粘贴代码，import 就报错。

**原因**：直接敲 `python` 用的是**系统 Python 3.14.4**，里面什么都没装；AgentSeek 用 `uv sync` 把依赖装在了项目的 `.venv` 虚拟环境里，两者互不相通。

**填坑**：不要用系统 Python，用下面任一方式让代码跑在**项目虚拟环境**里：

```bash
uv run python hello_agent.py            # 方式一：uv run 自动找 .venv
# 或
source .venv/bin/activate && python hello_agent.py   # 方式二：手动激活
```

**经验**：在 uv 管理的项目里，看到 `python` 前先想一下"这是哪个环境的 python"。

### 坑 2：`uv run` 不会自动加载 `.env` → `KeyError: 'OPENAI_API_KEY'`

**现象**：脚本里写 `os.environ["OPENAI_API_KEY"]`（因为 AgentSeek 模板的 `.env` 用的是这个名字），`uv run` 运行时直接 KeyError。

**原因**：`.env` 文件只是**文本**，需要被 source 或被 dotenv 库读取才会变成环境变量；`uv run` 并不会自动加载项目根的 `.env`。

**填坑**（二选一）：

```bash
# 方案 A：终端手动导出 .env 再运行
set -a && source .env && set +a
uv run python hello_agent.py

# 方案 B：脚本改用已 export 的 SILICONFLOW_API_KEY（更简单，推荐）
api_key = os.environ["SILICONFLOW_API_KEY"]
```

### 坑 3：环境变量名"双重标准"——`OPENAI_API_KEY` vs `SILICONFLOW_API_KEY`

**现象**：AgentSeek 模板 `.env` 用 `OPENAI_API_KEY`（因为它是 OpenAI 兼容接口的通用命名），而教程第 2 章用 `SILICONFLOW_API_KEY`，两边都是同一个硅基流动 Key，名字却不一样，很容易困惑到底该写哪个。

**本质**：变量名是**你自己定的约定**，Key 的值才是硅基流动平台上那个 `sk-` 开头的串。`base_url` 指向硅基流动，用哪个变量名传 Key 都一样。

**建议**：在脚本里统一用一个名字（比如教程的 `SILICONFLOW_API_KEY`），并在笔记里写明"模板的 `OPENAI_API_KEY` 与它等价"，避免后来的人（或未来的自己）再踩一遍。

### 坑 4（安全警示）：API Key 不要明文外传

我在排错时把带真实 Key 的终端输出整段发给了 AI 助手，存在泄露风险。

**正确姿势**：
- 发给他人/AI 前，把 `sk-...` 的内容打码（`sk-***`）；
- 如果已经发出，立刻去平台后台**作废并重新生成**新 Key；
- Key 只写在 `.env` 里（该文件已被 `.gitignore` 忽略），绝不提交到 Git。

---

## 四、对教程的意见与建议

1. **建议在第 2 章开头加一小节"如果你已经用 AgentSeek 完成了 pre01"**：明确说明 quickstart 的代码可以直接在 `research_deepagent` 项目里用 `uv run` 跑，并指出"系统 python vs uv .venv"的区别。AgentSeek 路线和裸装 pip 路线的读者会在第 2 章交汇，需要一个"汇合点"说明。
2. **建议统一环境变量命名**：要么教程注明"AgentSeek 模板中等价于 `OPENAI_API_KEY`"，要么模板生成时加一行 `SILICONFLOW_API_KEY=${OPENAI_API_KEY}`，减少认知负担。
3. **建议补充 uv 用户的最小命令**：安装一节目前只给了 pip/uv pip/poetry，如果读者已经在 uv 项目里，直接 `uv run python xxx.py` 是最短路径，值得点明。
4. **第 2 章"实战：构建研究助手"一节自己也提示了可跳过**，对新手来说 Tavily 注册是多一个外部依赖，建议把"纯本地计算器 Agent"一节标为"必做"，研究助手标为"选做"，学习路径会更清晰。

---

## 五、学习心得

1. **"函数即工具"是门槛最低的设计**：类型标注 + docstring + 默认值三要素，让 Agent 自己理解何时调用、传什么参数。写工具的核心竞争力变成了"把 docstring 写清楚"。
2. **Harness 的价值在于"你少写的代码"**：我只提供了 1 个 `get_weather` 工具，框架却自动附带了文件系统、任务规划、子 Agent 等能力——这正是"上下文工程"被框架化后的体验。
3. **环境问题是新手第一大杀手**：本次 3 个坑全部与环境有关（虚拟环境、.env 加载、变量命名），没有一个是代码本身的错误。以后排错先问"代码在哪个环境、用哪个 Key、哪个变量名"，再去看逻辑。
4. **小模型够用，但要有预期**：Qwen2.5-7B 能稳定跑通工具调用，但教程也提醒复杂任务（规划、多 Agent）建议换 GLM-5.2 等强模型——学习期先用免费模型跑通流程，再按需升级，是性价比最高的路径。

---

*下一步（Task3）：继续完成"研究助手"实战（接入 Tavily 搜索 + `TodoListMiddleware`），并开启 LangSmith 追踪观察 Agent 的完整决策链路。*
