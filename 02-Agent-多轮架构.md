# 02 · Agent 多轮架构

> **一句话：Agent = 模型 + 编排层。模型本身无状态，状态全在编排层。**

## 循环

```
用户输入
  ↓
组装上下文 = system prompt + 目标状态 + 用户画像 + 最近 N 轮 + 检索片段 + 工具结果
  ↓
LLM 决策：① 回答 ② 追问澄清 ③ 调工具 ④ 更新记忆
  ↓（②③ 则执行 → 观察 → 回到 LLM，循环）
回答 + 写回状态（记忆 / 目标 / 用户信息）
```

## 组件

| 组件 | 作用 | 一句话 |
| --- | --- | --- |
| System prompt | 角色、边界、输出格式 | 你已知的那个 |
| **上下文组装** | 决定这一轮往 prompt 里塞什么 | 真正的工程核心 |
| **短期记忆** | 最近 N 轮原文 | 塞多了超长、塞少了断片 |
| **长期记忆** | 老对话摘要 + 向量库 | 需要时"检索"回来 |
| **用户画像** | 结构化用户信息 | 名字、处境、偏好 |
| **目标状态** | 显式 goal/subgoal 对象 | "还记得目标"靠它 |
| 工具 | 搜索、计算、API | function calling |
| 护栏 | 安全、拒答 | 代码分支，不是提示词 |

## 一轮里发生什么（伪代码）

```python
def turn(user_input, state):
    state.history.append(user_input)
    ctx = build_context(
        system=SYSTEM,
        goal=state.goal,                 # 每轮都带
        profile=state.profile,           # 每轮都带
        recent=last_n(state.history, 10),# 短期
        summary=state.summary,           # 旧的压成摘要
        retrieved=retrieve(user_input),  # RAG（可能由模型决定要不要调）
        tools=state.tool_results,
    )
    action = llm(ctx)
    if action.type == "tool":
        state.tool_results.append(run(action))
        return turn(user_input, state)   # 循环
    if action.type == "answer":
        state.summary = maybe_summarize(state)  # 超长才压
        return action.text
```

## 状态存哪

| 方式 | 适合 | 注意 |
| --- | --- | --- |
| 进程内存 | 单机 demo | 重启就没了 |
| Redis | 多实例、低延迟 | 要 TTL、要序列化 |
| Postgres/SQLite | 要持久、要查询 | 多一次 IO |
| 向量库 | 长期记忆检索 | 要和正文映射 |

## 失败模式

- **goal 漂移**：聊着聊着跑题 —— 每轮把 goal 放 prompt 顶部。
- **记忆串台**：把别人的信息塞进来 —— 按 user_id 隔离。
- **上下文超长**：截断把关键信息截没了 —— 摘要 + 检索，别硬截。
- **工具死循环**：调同一个工具反复失败 —— 设最大步数。
