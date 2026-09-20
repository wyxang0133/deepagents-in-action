# Task3 实践笔记：虚拟文件系统与存储后端（第 3 章）

> - **Task**：Task3 — 虚拟文件系统与存储后端（学习第 3 章）
> - **教程**：[Deep Agents 实战 · 第 3 章](https://datawhalechina.github.io/deepagents-in-action/chapters/ch03-virtual-filesystem/)
> - **视频参考**：B 站 DataWhale 官方账号
> - **环境**：Windows 11 + WSL2 (Ubuntu) + `research_deepagent`（AgentSeek 项目，uv 管理）
> - **日期**：2026-09-20

---

## 〇、今日状态说明（诚实记录）

今天先听了 B 站视频课 + 精读网页教程，**代码实操还没跑**。说实话，这一章信息密度比前两章大，第一次听完有几个概念是模糊的：

- 虚拟文件系统的文件到底**存在哪**？（后来明白：由 Backend 决定，默认在内存/State 里，根本不是磁盘上的真文件）
- "大结果自动卸载"和"对话历史总结"两道防线，**谁先看**、什么关系？
- Backend、工具、权限三者怎么串起来？

下面按"看懂 → 存疑 → 计划验证"来整理。打卡后我会把实操结果补进来。

---

## 一、内容归纳整理

### 1. 为什么需要虚拟文件系统？

传统 Agent 的致命问题：**所有信息都塞进 prompt**，对话历史不断膨胀，最终上下文溢出。

Deep Agents 的方案是**给 Agent 一个文件系统**，让 Agent 像人一样工作：

- 不会把所有资料同时摊在桌面 → 分门别类存放
- 需要时按需取出（`read_file`）
- 用搜索快速定位（`grep` / `glob`）
- 便签记录中间结果（`write_file`）

这就是 Context Engineering：上下文不是被动堆积，而是被**主动工程化管理**。

### 2. 七个内置文件工具

| 工具 | 用途 | 类比 |
| --- | --- | --- |
| `ls` | 列出目录与元信息（大小、修改时间） | 打开文件夹看看 |
| `read_file` | 读取内容，支持 offset/limit 分片；**原生支持多模态**（图片/视频/音频/PDF/PPT） | 翻资料阅读 |
| `write_file` | 创建或**完整覆盖**文件 | 写新备忘录 |
| `edit_file` | 精确字符串替换 | 红笔改文档 |
| `delete` | 删除文件或目录 | 清理资料 |
| `glob` | 按模式匹配找文件（`**/*.py`） | 按标签找 |
| `grep` | 内容搜索，三种输出模式 | 全文检索 |

两个容易踩的细节：
- **局部修改要用 `edit_file`**，因为 `write_file` 会整体覆盖同路径文件
- `grep` 三种模式：`files_with_matches`（快速定位）/ `content`（看内容）/ `count`（统计）

### 3. 两道自动防线（本章最核心）

**防线一：大结果自动卸载（eviction）**
- 触发：工具输出 > 20,000 tokens（可用 `tool_token_limit_before_evict` 配置）
- 动作：完整内容写入虚拟文件系统 → 对话里替换为"文件路径 + 前 10 行预览" → 需要时 `read_file`/`grep` 读回
- **完全自动**，Agent 无感

**防线二：对话历史自动总结（summarization）**
- 触发：上下文达到模型窗口的 85%（默认）
- 动作：旧消息写入 Backend 保存 → LLM 生成结构化摘要（意图、产出物、下一步）→ 模型输入变成"摘要 + 近期消息"

结果：**Agent 始终拥有"精炼的工作记忆 + 可回溯的完整记录"**——这是我今天理解的最重要的一个画面。

### 4. 可插拔存储后端

"虚拟文件系统"是抽象，文件实际存哪由 Backend 决定。五种内置 + 一种组合：

| 后端 | 存储位置 | 持久化 | 适用场景 |
| --- | --- | --- | --- |
| `StateBackend`（默认） | LangGraph State | 线程内 ✓ / 跨线程 ✗ | 学习实验、"草稿纸" |
| `FilesystemBackend` | 本地磁盘 | 永久 ✓ | 本地编程助手 |
| `LocalShellBackend` | 本地磁盘 + `execute` | 永久 ✓ | 完全信任的本地机，**生产禁用** |
| `StoreBackend` | LangGraph Store | 跨线程/跨会话 ✓ | 长期记忆、用户偏好 |
| `CompositeBackend` | 按路径路由 | 混合 | `/memories/` → Store，其他 → State |
| 沙箱后端 | Modal/Daytona 等 | 隔离环境 | 安全执行代码 |

记忆锚点（从临时到持久化）：**State（便签纸）→ Filesystem（硬盘）→ Store（档案库）→ Composite（分区管理）→ Sandbox（保险箱）**。

### 5. 权限与安全

- **声明式权限 `FilesystemPermission`**：`mode` 有 `allow` / `deny` / `interrupt`（暂停等人工审批），规则**顺序求值、first-match-wins**，具体规则放前面
- **继承式**：`GuardedBackend` 继承现有后端重写 write/edit/delete
- **包装式**：`PolicyWrapper` 包装任意后端，加 deny 前缀拦截
- 共同要求：包装器**必须转发或拒绝 delete**，不能只保护 write/edit

---

## 二、学习心得与疑点

### 心得

1. **"文件系统"是比喻，不是实现**。文件可以根本不存在磁盘上（StateBackend 存内存）。理解"抽象 + 可插拔后端"这个分层，整章就通了。
2. **Context Engineering 是 Agent 框架的竞争核心**。前两章的"10+ 次工具调用"之所以可行，靠的就是这两道防线撑住上下文。
3. **读代码比读概念快**：`read_file` 支持分片 + 多模态，说明这套设计从一开始就按"真实工作负载"而非"玩具 demo"来做的。
4. 安全不是附加题：默认 `virtual_mode=True` 沙箱、`deny` 规则、`interrupt` 人工审批，是一条完整的信任链。

### 疑点（待实操验证）

1. **两道防线的触发顺序**：是先卸载大工具结果，还是先总结对话？还是各自独立计数？打算写一个返回超大结果的假工具，用 LangSmith 看 Trace 验证。
2. **v0.7 的兼容提醒**（`backend=` 必须传实例、工厂函数已移除）——手上的 deepagents 版本是否是 v0.7？直接 `uv pip show deepagents` 确认。
3. `StoreBackend` 的 `namespace` 在本地 `invoke()` 时 `rt.server_info` 是 None，教程给了兜底写法，想亲手跑一次确认报错长什么样、兜底是否生效。

---

## 三、代码运行记录（计划 + 待回填）

### 验证脚本 1：默认 StateBackend 的"草稿纸"行为

在 `~/research_deepagent` 下新建 `vfs_test.py`（复用 Task2 的 model 配置）：

```python
import os
from langchain_openai import ChatOpenAI
from deepagents import create_deep_agent

model = ChatOpenAI(
    model=os.environ.get("MODEL_NAME", "Qwen/Qwen2.5-7B-Instruct"),
    api_key=os.environ["SILICONFLOW_API_KEY"],
    base_url="https://api.siliconflow.cn/v1",
)

agent = create_deep_agent(model=model)  # 默认 StateBackend

result = agent.invoke({"messages": [{
    "role": "user",
    "content": "请用 write_file 创建一个 /workspace/notes.md，"
               "写入三行关于你自己的说明；然后用 ls 确认它存在；"
               "最后告诉我这个文件保存在哪里？"
}]})
print(result["messages"][-1].content)
```

预期观察点：Agent 会调用 `write_file` 和 `ls`（这两个工具我并没有显式传入！）；运行结束后**当前目录下不会出现 notes.md**——证明 StateBackend 是"虚拟"的。

运行命令（Task2 踩坑后已确认的正确姿势）：

```bash
export SILICONFLOW_API_KEY="sk-***"   # 你的 Key（注意：不要明文提交到 git）
export MODEL_NAME="Qwen/Qwen2.5-7B-Instruct"
uv run python vfs_test.py
```

### 验证脚本 2：FilesystemBackend 落盘

```python
from deepagents.backends import FilesystemBackend

agent = create_deep_agent(
    model=model,
    backend=FilesystemBackend(root_dir="./agent_workspace", virtual_mode=True),
)
# 同一 prompt，预期：./agent_workspace/notes.md 真实出现在磁盘上
```

预期对比：脚本 1 无文件落盘，脚本 2 落盘 → 亲手感受"同一个抽象、不同后端"。

> 实操结果、LangSmith Trace 截图：**待回填**

---

## 四、踩坑与预判

### 已踩（预判类，实操时重点盯）

1. **Python 环境问题（Task2 已踩过）**：在 WSL2 里必须用 `uv run python`，不能直接 `python`（系统 Python 3.14 没有依赖）。
2. **环境变量**：本项目 `uv run` 不会自动加载 `.env`，Key 用 `export SILICONFLOW_API_KEY`（不是模板里的 `OPENAI_API_KEY`，二者是同一个 Key、不同变量名）。
3. **版本兼容**：教程基于 v0.7。`backend=lambda rt: ...` 工厂函数已移除、必须传实例；`virtual_mode` 0.6 起必填。动手前先 `uv pip show deepagents` 对版本。

### 新预判

4. Qwen2.5-7B 小模型面对"多步文件操作"指令可能不稳定（教程第 2 章也提醒过复杂任务建议 GLM-5.2），如果脚本 1 跑不通，先怀疑模型能力而非代码错误，换 `zai-org/GLM-5.2` 重试。
5. `FilesystemBackend` 的 `root_dir="./agent_workspace"` 目录如果不存在是否会自动创建？实操时留意。

---

## 五、对教程的意见与建议

1. **建议在章首加一张"分层架构图"**：工具层（7 个工具）→ 后端层（5+ 种 Backend）→ 存储层（State/磁盘/Store），现在的顺序是先工具后后端，初学者容易把"虚拟文件"误以为"磁盘文件"，一张图能省很多误解。
2. **两道防线的交互关系建议画成一张时序图**：触发条件各自独立吗？eviction 和 summarization 同时接近阈值时谁优先？目前只给了两张并列的流程图。
3. **视频课与网页的侧重点可以更明显地区分**：视频适合建立直觉（本章的"办公桌类比"很好），但 API 细节（v0.7 兼容提醒、namespace 兜底）目前主要在网页，建议视频结尾明确提示"这些细节以网页为准"。
4. **建议章末给一个 10 分钟的"最小验证练习"**：比如本文档里的"StateBackend vs FilesystemBackend 落盘对比"，比直接上 CompositeBackend 更平缓。

---

## 六、下一步

1. 跑通验证脚本 1、2，回填运行结果与 Trace 截图
2. 用 LangSmith 观察一次 eviction 或 summarization 的真实触发
3. Task4：任务规划（`TodoListMiddleware` / `write_todos`）
