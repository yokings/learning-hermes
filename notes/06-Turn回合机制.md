# 06 - Turn 回合机制
> Agent 交互的基本时间单元：从用户消息到最终回复的完整生命周期
> 学习日期：2026-07-02

---

## 什么是 Turn（回合）？

**Turn（回合）** 是 Hermes 中一次完整的用户-助手交互周期：

```
┌─────────────────────────────────────────────────────────────┐
│                         一个 Turn（回合）                      │
├─────────────────────────────────────────────────────────────┤
│  用户发送消息 → Agent思考/调用工具/执行 → 返回最终回复给用户      │
└─────────────────────────────────────────────────────────────┘
```

简单来说：**用户每发一条消息，Agent 处理完并给出最终回复，这就是一个回合**。

一个 Turn 内部可能包含多次 LLM 调用和多次工具调用（ReAct 循环），但对外只呈现为一次"问-答"交互。

---

## Turn 的生命周期（三阶段模型）

```
┌──────────────────┐     ┌──────────────────────────┐     ┌──────────────────┐
│   Turn 开始      │────▶│   工具调用循环（ReAct）    │────▶│   Turn 结束       │
│  (Prologue)      │     │                          │     │  (Finalize)      │
└──────────────────┘     └──────────────────────────┘     └──────────────────┘
       │                            │                              │
       ▼                            ▼                              ▼
  • 生成 turn_id                • 调用 LLM                     • 保存会话轨迹
  • 重置重试计数器              • 检查是否有工具调用           • 清理VM/浏览器资源
  • 初始化迭代预算              • 执行工具（并发）             • 持久化会话到DB
  • MCP工具刷新                 • 把结果加入消息历史           • 触发记忆/技能回顾
  • 预飞上下文压缩              • 循环直到终止条件             • 触发插件钩子
  • 插件 pre_llm_call           •                              • 返回结果给调用方
  • 记忆预取
```

---

## 阶段一：Turn 开始（Prologue）

**入口点**：[build_turn_context()](file:///e:/github/hermes-agent/agent/turn_context.py#L118-L486)

### 关键初始化动作（按顺序）：

| 步骤 | 代码位置 | 做什么 |
|------|---------|--------|
| 1. 生成 turn_id | [turn_context.py:207](file:///e:/github/hermes-agent/agent/turn_context.py#L207-L207) | `turn_id = f"{session_id}:{task_id}:{uuid8}"` 唯一标识本次回合 |
| 2. 重置计数器 | [turn_context.py:212-225](file:///e:/github/hermes-agent/agent/turn_context.py#L212-L225) | 重置所有重试计数器、工具护栏状态 |
| 3. 初始化预算 | [turn_context.py:244](file:///e:/github/hermes-agent/agent/turn_context.py#L244-L244) | `IterationBudget(max_iterations)` 防止无限循环 |
| 4. MCP工具刷新 | [turn_context.py:185-191](file:///e:/github/hermes-agent/agent/turn_context.py#L185-L191) | 检查是否有新连接的MCP服务器，更新工具集 |
| 5. 用户消息净化 | [turn_context.py:194-197](file:///e:/github/hermes-agent/agent/turn_context.py#L194-L197) | 移除 surrogate 字符等无效Unicode |
| 6. 预飞压缩 | [turn_context.py:333-410](file:///e:/github/hermes-agent/agent/turn_context.py#L333-L410) | 提前估算token数，超阈值则压缩上下文 |
| 7. 插件钩子 | [turn_context.py:413-437](file:///e:/github/hermes-agent/agent/turn_context.py#L413-L437) | 触发 `pre_llm_call` 插件注入上下文 |
| 8. 记忆预取 | [turn_context.py:466-472](file:///e:/github/hermes-agent/agent/turn_context.py#L466-L472) | 提前从记忆系统检索相关内容 |
| 9. 持久化用户消息 | [turn_context.py:323-331](file:///e:/github/hermes-agent/agent/turn_context.py#L323-L331) | 崩溃恢复：先把用户消息存到DB |

### Turn ID 格式：
```python
# 示例：session:abc123:task:def456:7f9d3c2a
turn_id = f"{agent.session_id or 'session'}:{effective_task_id}:{uuid.uuid4().hex[:8]}"
```
- `session_id`: 会话ID，跨turn保持
- `task_id`: 任务ID，同一session内可变化
- 最后8位hex：本次turn唯一标识

---

## 阶段二：工具调用循环（核心 ReAct Loop）

**入口点**：[conversation_loop.py:604](file:///e:/github/hermes-agent/agent/conversation_loop.py#L604-L629) 主循环

### 循环条件：
```python
while (api_call_count < agent.max_iterations and agent.iteration_budget.remaining > 0) 
       or agent._budget_grace_call:
```

默认 `max_iterations = 8`，即一个turn最多调用8次LLM。

---

## 关键问题：如何界定 Turn 结束？

**这是理解turn机制的核心问题！** Turn 结束有 **5 种终止条件**：

### 终止条件 1：模型返回最终文本回复（正常结束）✅

**这是最常见、最理想的结束方式。**

**判断逻辑**（[conversation_loop.py:4330-4332](file:///e:/github/hermes-agent/agent/conversation_loop.py#L4330-L4332)）：
```python
else:
    # No tool calls - this is the final response
    final_response = assistant_message.content or ""
    # ... break loop
```

**判定标准**：LLM 返回的消息中 **没有 tool_calls**，且 `finish_reason == "stop"`。

```
模型返回:
{
  "content": "这是我的最终答案...",  ← 有文本内容
  "tool_calls": null,                ← 没有工具调用！
  "finish_reason": "stop"            ← 正常结束
}
→ Turn 结束，返回给用户
```

**退出原因标记**：`_turn_exit_reason = "text_response(...)"`

---

### 终止条件 2：用户中断（User Interrupt）⚡

**代码位置**：[conversation_loop.py:608-614](file:///e:/github/hermes-agent/agent/conversation_loop.py#L608-L614)

```python
# Check for interrupt request (e.g., user sent new message)
if agent._interrupt_requested:
    interrupted = True
    _turn_exit_reason = "interrupted_by_user"
    break
```

**触发场景**：
- 用户按了 Ctrl+C
- 用户在Web界面发送了新消息
- 用户发送了 `/stop` 命令

**后续处理**（[turn_finalizer.py:184-186](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L184-L186)）：
- 如果最后一条消息是tool结果，会补一个synthetic assistant消息关闭工具序列，避免下一轮格式错误

---

### 终止条件 3：迭代预算耗尽（Budget Exhausted）⚠️

**代码位置**：[conversation_loop.py:625-629](file:///e:/github/hermes-agent/agent/conversation_loop.py#L625-L629)

```python
elif not agent.iteration_budget.consume():
    _turn_exit_reason = "budget_exhausted"
    break
```

**预算控制类**：[IterationBudget](file:///e:/github/hermes-agent/agent/iteration_budget.py)
- 默认 `max_iterations = 8`（可配置）
- 每次循环前 `consume()` 消耗1个预算
- 预算用完强制终止

**Grace Call 机制**：
预算耗尽后还会给模型 **最后一次机会**（`_budget_grace_call = True`），这次不带工具调用，让模型总结并给出最终回复（[turn_finalizer.py:53-70](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L53-L70)）。

---

### 终止条件 4：空响应重试失败（Empty Response）❌

**场景**：模型返回空内容（既没有文本也没有工具调用），重试多次后仍然失败。

**重试逻辑**（[conversation_loop.py:4393-4457](file:///e:/github/hermes-agent/agent/conversation_loop.py#L4393-L4457)）：
1. 检测到空响应后，给模型追加一个"nudge"提示："Please continue with your response..."
2. 最多重试 `_empty_content_retries` 次（默认3次）
3. 重试仍然为空 → 终止turn，返回错误提示

**相关退出原因**：
- `empty_response_after_retries`
- `thinking_prefill_exhausted`（模型把token全花在思考上了）

---

### 终止条件 5：工具调用截断（Truncated Tool Calls）✂️

**场景**：模型输出的tool_calls被截断（max_tokens不够）。

**处理逻辑**（[conversation_loop.py:1756-1787](file:///e:/github/hermes-agent/agent/conversation_loop.py#L1756-L1787)）：
1. 检测到 `tool_calls` 存在但不完整
2. 自动增加 `max_tokens` 重试
3. 重试3次仍失败 → 终止

---

## 阶段三：Turn 结束（Finalize）

**入口点**：[finalize_turn()](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L30-L481)

### 结束后的清理动作（按顺序）：

| 步骤 | 代码位置 | 做什么 |
|------|---------|--------|
| 1. 预算耗尽总结 | [turn_finalizer.py:53-70](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L53-L70) | 如果是预算耗尽，调用无工具版本让模型总结 |
| 2. 保存轨迹 | [turn_finalizer.py:150-154](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L150-L154) | `_save_trajectory()` 保存完整对话轨迹 |
| 3. 清理资源 | [turn_finalizer.py:156-161](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L156-L161) | 关闭VM、浏览器等task资源 |
| 4. 修复中断序列 | [turn_finalizer.py:170-186](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L170-L186) | 如果是中断，补全工具调用序列 |
| 5. 持久化会话 | [turn_finalizer.py:188](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L188-L188) | `_persist_session()` 写入SQLite和JSON日志 |
| 6. 诊断日志 | [turn_finalizer.py:193-235](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L193-L235) | 详细记录turn结束原因、api调用数、工具数等 |
| 7. 文件变更提示 | [turn_finalizer.py:237-260](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L237-L260) | 如果有文件写入失败，追加提示到回复末尾 |
| 8. 异常解释 | [turn_finalizer.py:262-309](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L262-L309) | 如果异常结束，给用户友好解释 |
| 9. 插件钩子 | [turn_finalizer.py:313-354](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L313-L354) | 触发 `transform_llm_output`、`post_llm_call` |
| 10. 提取推理 | [turn_finalizer.py:356-371](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L356-L371) | 提取当前turn的reasoning，不跨turn |
| 11. 构建结果 | [turn_finalizer.py:374-421](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L374-L421) | 组装result字典返回 |
| 12. 记忆同步 | [turn_finalizer.py:436-442](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L436-L442) | 把本轮对话同步到外部记忆 |
| 13. 后台回顾 | [turn_finalizer.py:446-454](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L446-L454) | 后台触发记忆/技能回顾（不阻塞响应） |
| 14. on_session_end | [turn_finalizer.py:466-479](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L466-L479) | 触发插件 `on_session_end` 钩子 |

---

## Turn 退出原因大全（_turn_exit_reason）

这些原因会被日志记录，用于调试和用户提示：

| 退出原因 | 类型 | 说明 |
|---------|------|------|
| `text_response(N)` | ✅ 正常 | 模型给出N字符的最终文本回复 |
| `interrupted_by_user` | ⚡ 中断 | 用户主动中断（Ctrl+C / 新消息） |
| `budget_exhausted` | ⚠️ 预算 | 迭代预算用完（max_iterations次） |
| `max_iterations_reached(N/M)` | ⚠️ 预算 | 达到最大迭代次数，正在请求总结 |
| `partial_stream_recovery` | ↻ 恢复 | 流中断，使用已传输的部分内容 |
| `fallback_prior_turn_content` | ↻ 恢复 | 工具调用后空响应，使用上一轮有内容的回复 |
| `empty_response_after_retries` | ❌ 失败 | 空响应重试多次仍失败 |
| `thinking_prefill_exhausted` | ❌ 失败 | 模型把所有token花在thinking上，无输出 |
| `truncated_response_after_retries` | ❌ 失败 | 响应被截断，重试3次仍失败 |
| `truncated_tool_call_after_retries` | ❌ 失败 | 工具调用被截断，重试3次仍失败 |
| `unknown` | ❓ 未知 | 初始值，正常流程不会走到这里 |

---

## Turn 边界的可视化判断

### 消息历史中的 Turn 边界

```
messages[] 数组：
┌─────────────────────────────────────────────────────────┐
│  [system] 系统 prompt                                    │ ← 跨turn共享
├─────────────────────────────────────────────────────────┤
│  [user] 第一条消息                                       │ ┐
│  [assistant] ... (有tool_calls)                          │ │ Turn 1
│  [tool] ...结果                                          │ │
│  [assistant] 最终回复（无tool_calls）                    │ ┘
├─────────────────────────────────────────────────────────┤
│  [user] 第二条消息  ← current_turn_user_idx 指向这里      │ ┐
│  [assistant] ... (有tool_calls)                          │ │ Turn 2（当前）
│  [tool] ...结果                                          │ │
│  [assistant] ... (思考中)                                │ │
│  ...                                                     │ ┘
└─────────────────────────────────────────────────────────┘
```

**边界判定技巧**（[turn_finalizer.py:366-368](file:///e:/github/hermes-agent/agent/turn_finalizer.py#L366-L368)）：
```python
for msg in reversed(messages):
    if msg.get("role") == "user":
        break  # turn boundary — 遇到user消息就是上一个turn的边界！
    # ... 提取当前turn的reasoning
```

**简单记忆法**：**看到 `role == "user"` 就是新turn的开始！**

---

## 多 Turn 会话 vs 单 Turn

| 概念 | 说明 | 生命周期 |
|------|------|---------|
| **Session（会话）** | 一次完整的对话，包含多个turn | 从开始到 `/reset` 或超时 |
| **Turn（回合）** | 用户发一条消息，Agent回复一次 | 从一条user消息到assistant最终回复 |
| **Iteration（迭代）** | turn内部的一次LLM调用 | 一次API请求（可能带工具调用） |
| **Tool Call（工具调用）** | 一次具体的工具执行 | 模型发起→工具返回结果 |

### 层级关系：
```
Session
└── Turn 1（用户消息1）
    └── Iteration 1（LLM调用1）
        └── Tool Call A, B（并发）
    └── Iteration 2（LLM调用2）
        └── Tool Call C
    └── Iteration 3（LLM调用3）
        └── （无工具调用 → Turn 1 结束）
└── Turn 2（用户消息2）
    └── Iteration 1（LLM调用1）
        └── （无工具调用 → Turn 2 结束）
└── Turn 3（用户消息3）
    └── ...
```

---

## Turn 相关的状态重置

**每个新turn开始时，这些状态会被重置**（[turn_context.py:212-225](file:///e:/github/hermes-agent/agent/turn_context.py#L212-L225)）：

```python
agent._invalid_tool_retries = 0          # 无效工具重试次数
agent._invalid_json_retries = 0          # 无效JSON重试次数
agent._empty_content_retries = 0         # 空内容重试次数
agent._incomplete_scratchpad_retries = 0 # 不完整scratchpad重试
agent._codex_incomplete_retries = 0      # Codex不完整重试
agent._thinking_prefill_retries = 0      # Thinking预填充重试
agent._post_tool_empty_retried = False   # 工具后空回复已重试
agent._last_content_with_tools = None    # 上一次带工具的内容（用于fallback）
agent._last_content_tools_all_housekeeping = False  # 上一次工具是否都是内务工具
agent._mute_post_response = False        # 是否静音后续输出
agent._unicode_sanitization_passes = 0   # Unicode净化次数
agent._tool_guardrails.reset_for_turn()  # 工具护栏重置
agent._vision_supported = True           # 视觉支持重置
agent.iteration_budget = IterationBudget(agent.max_iterations)  # 预算重置
```

**注意**：`_turns_since_memory` 和 `_iters_since_skill` **不会重置**，它们跨turn累积，用于触发记忆/技能回顾。

---

## 代码最小示例：判断 Turn 是否结束

```python
def is_turn_complete(assistant_message: dict) -> bool:
    """判断assistant消息是否标志着turn结束。
    
    核心逻辑：没有tool_calls，且finish_reason不是length
    """
    has_tool_calls = bool(assistant_message.get("tool_calls"))
    finish_reason = assistant_message.get("finish_reason", "stop")
    
    # 有工具调用 → 继续循环，turn未结束
    if has_tool_calls:
        return False
    
    # 是因为长度截断 → 需要继续，turn未结束
    if finish_reason in {"length", "max_output_tokens"}:
        return False
    
    # 没有工具调用且正常结束 → turn完成！
    return True


# 使用示例
assistant_msg = {
    "content": "好的，我已经帮你查到了...",
    "tool_calls": None,
    "finish_reason": "stop"
}

if is_turn_complete(assistant_msg):
    print("✅ Turn结束，返回给用户")
else:
    print("⏳ Turn未结束，继续执行工具/再思考")
```

---

## 常见问题

### Q1: 为什么要有 max_iterations 限制？

防止模型陷入无限循环：
- 模型反复调用同一个工具
- 工具返回错误，模型不知道怎么处理
- 模型"思考"过度，停不下来

默认8次迭代是经验值，既保证能完成复杂任务（搜索→读文件→写文件→测试→提交），又防止死循环。

### Q2: Grace Call 是做什么的？

预算耗尽后，模型可能正中间在执行任务（比如刚调用完工具还没总结）。Grace Call 会**去掉所有工具**，再让模型做一次纯文本调用，说"请总结你目前的进展"，给用户一个交代，而不是粗暴中断。

### Q3: 用户中断后，正在执行的工具怎么办？

工具级中断：
- Hermes 有 thread-local 的中断信号
- 长时间运行的工具（如terminal命令）会检查中断信号并停止
- 已经在执行的系统命令可能需要等它完成，或由OS清理

### Q4: 为什么提取reasoning时要停在user消息处？

```python
# turn_finalizer.py:366-368
for msg in reversed(messages):
    if msg.get("role") == "user":
        break  # 不能跨turn！
```

防止把上一个turn的思考内容泄漏到当前turn。用户看到的"思考过程"应该只属于当前问题，否则会显示过时的、不相关的推理，造成混淆。

---

## 关键文件索引

| 文件 | 职责 |
|------|------|
| [turn_context.py](file:///e:/github/hermes-agent/agent/turn_context.py) | Turn 开始的prologue逻辑，构建TurnContext |
| [conversation_loop.py](file:///e:/github/hermes-agent/agent/conversation_loop.py) | 核心ReAct循环，终止条件判断 |
| [turn_finalizer.py](file:///e:/github/hermes-agent/agent/turn_finalizer.py) | Turn 结束的finalize逻辑 |
| [turn_retry_state.py](file:///e:/github/hermes-agent/agent/turn_retry_state.py) | Turn内的重试状态管理 |
| [iteration_budget.py](file:///e:/github/hermes-agent/agent/iteration_budget.py) | 迭代预算控制 |
