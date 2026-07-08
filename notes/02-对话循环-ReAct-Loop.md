# 02 - 对话循环 ReAct Loop
> Agent 的心脏：一次用户回合的完整生命周期
> 学习日期：2026-07-02

---

## ReAct 循环是什么？

ReAct = **Reasoning + Acting**（思考 + 行动），是 Hermes Agent 处理每条用户消息的核心循环：

```
用户发消息
    ↓
[1] 构建 System Prompt + 历史消息
    ↓
[2] 调用 LLM → 获取思考/工具调用/最终回复
    ↓
[3] 如果是最终回复 → 返回给用户，结束
    ↓
[4] 如果是工具调用 → 执行工具（支持并发）
    ↓
[5] 把工具结果加入消息历史 → 回到 [2]
    ↑
    ↓
（最多迭代 max_iterations 次，防止无限循环）
```

每一轮就是一次 "思考→行动→观察→再思考" 的循环，直到模型给出最终答案。

---

## Q1: 字节级缓存（Byte-level Prompt Caching）

### 问题背景

每次对话发送给模型的内容结构：

```
[System Prompt 人设/工具/技能]  ← 几千~几万字，99%不变
[历史对话]
[本次新消息]                    ← 只有这一点是新的！
```

如果不缓存：每轮都重新发送、重新计算全部token → 浪费钱、增加延迟。

### 什么是 Prompt Caching？

Anthropic/OpenAI提供的功能：
> 如果请求开头内容和上一次**字节级完全相同**，直接复用KV缓存——不用重新计算，速度快5-10倍，缓存部分token价格打1-2.5折。

关键词：**字节级完全相同**——差一个空格、标点、时间戳，缓存就失效。

价格对比（Claude）：
| 类型 | 价格 |
|------|------|
| 普通输入 | $3 / 百万token |
| 缓存命中 | $0.3 / 百万token（1折！） |

首个token延迟（TTFT）：2-3秒 → 200-300毫秒。

### Hermes 的三层 System Prompt 架构

`agent/system_prompt.py` 刻意把System Prompt分成三层，**会变的内容放最后**：

| 层级 | 内容 | 变化频率 | 缓存 |
|------|------|---------|------|
| **stable（稳定层）** | SOUL.md人设、工具指南、技能索引、平台提示 | 整个会话不变 | ✅ 100%命中 |
| **context（上下文层）** | 工作目录AGENTS.md、用户system_message | 切项目才变 | ⚡ 大部分命中 |
| **volatile（易变层）** | 时间戳、模型名、记忆快照、用户画像 | 每回合变 | ❌ 放最后，不影响前缀 |

结构示意：
```
┌─────────────────────────────────┐
│ [stable] 完全不变前缀            │ ← 最长，每次字节相同 → 缓存命中！
│ 人设、工具指南、技能索引...      │
├─────────────────────────────────┤
│ [context] 偶尔变                │
├─────────────────────────────────┤
│ [volatile] 每次变               │ ← 放最后，不破坏前缀缓存
│ 时间戳、记忆、model/provider行  │
└─────────────────────────────────┘
```

**关键设计**：缓存是**前缀匹配**——开头一样，后面不同没关系，前面照样命中！

### 缓存标记策略

`agent/prompt_caching.py` 使用 `system_and_3` 策略：

```python
def apply_anthropic_cache_control(messages, cache_ttl="5m"):
    # 断点1：System Prompt末尾 → 缓存人设+工具
    if messages[0].role == "system":
        _apply_cache_marker(messages[0], marker)
    # 断点2-4：最后3条非系统消息 → 缓存最近对话
    for idx in non_system_messages[-3:]:
        _apply_cache_marker(messages[idx], marker)
```

在System Prompt和最近3条消息打 `cache_control: {type: "ephemeral", ttl: "5m"}` 标记。

### 效果

源码注释原文：
> Reduces input token costs by **~75%** on multi-turn conversations within a single session.

多轮对话输入token成本降到原来的 **25%**。

### 哪些"坑"会打碎缓存？

很多框架不注意这些细节，导致缓存失效：
- ❌ 动态时间戳放System Prompt开头
- ❌ 每次注入随机ID/会话ID到前面
- ❌ 记忆注入到system而不是user message
- ❌ 工具列表每次顺序随机打乱
- ❌ 每轮重建System Prompt

### Hermes 的正确做法

1. System Prompt一次构建，SQLite缓存，整个会话复用
2. 易变内容全部放最后
3. 记忆注入到user message（不是system），保护system前缀稳定
4. 只有上下文压缩才重建System Prompt（此时缓存本来也该失效了）

---

## Q2: 可中断设计具体是怎样实现的？

### 为什么需要可中断？

用户在Agent执行长时间任务时（比如跑一个很长的终端命令、等API重试、浏览网页），应该能随时打断它——按Ctrl+C、发新消息、点停止按钮，都能立即终止当前工作，响应新指令。

没有可中断的话：用户发"停"，Agent还得等当前命令跑完才能看到"停"这个消息，体验非常差。

### 核心机制：线程级中断信号

`tools/interrupt.py` 实现了**按线程隔离的中断状态**——Gateway里多个Agent并发跑时，中断一个不会影响其他Agent。

```python
# 核心数据结构：一个集合，记录哪些线程被中断了
_interrupted_threads: set[int] = set()
_lock = threading.Lock()

def set_interrupt(active: bool, thread_id: int | None = None) -> None:
    """标记某个线程为中断/清除中断"""
    tid = thread_id if thread_id is not None else threading.current_thread().ident
    with _lock:
        if active:
            _interrupted_threads.add(tid)
        else:
            _interrupted_threads.discard(tid)

def is_interrupted() -> bool:
    """当前线程是否被中断了？（工具调用这个）"""
    tid = threading.current_thread().ident
    with _lock:
        return tid in _interrupted_threads
```

**设计要点**：
- 不用全局 `threading.Event()`，因为Gateway里多Agent并发——全局事件会"一中断全中断"
- 用 `set[thread_id]` 精确记录哪个线程要中断
- 工具只需调用 `is_interrupted()`，不用传参——它自动检查自己所在的线程

### 中断检查点分布在整个系统的关键位置

Hermes不是只在一个地方检查中断，而是在**所有可能阻塞的地方都埋了检查点**：

#### 1. ReAct循环每轮开始检查

`agent/conversation_loop.py` 主循环里：
```python
while api_call_count < agent.max_iterations:
    # 检查：用户发了新消息/按了Ctrl+C？
    if agent._interrupt_requested:
        interrupted = True
        _turn_exit_reason = "interrupted_by_user"
        break
    # ... 继续LLM调用/工具执行
```

#### 2. API重试退避时小步sleep

退避等待不一次性sleep完，而是每200ms醒一次检查中断：
```python
# 退避等待时：不睡5秒，而是200ms醒一次检查
for _ in range(int(backoff / 0.2)):
    if agent._interrupt_requested:
        # 用户中断了，直接退出
        _interrupt_text = "Operation interrupted during retry..."
        close_interrupted_tool_sequence(messages, _interrupt_text)
        agent.clear_interrupt()
        return {"final_response": _interrupt_text, "interrupted": True}
    time.sleep(0.2)  # Check interrupt every 200ms
```
> 这里很关键：如果直接 `time.sleep(5)` 等退避，用户最多要等5秒才能中断。分成25次200ms的小sleep，响应延迟 ≤200ms。

#### 3. 流式API调用中途可中断

Hermes封装了 `_interruptible_streaming_api_call` 和 `_interruptible_api_call`：
- 流式接收每个token chunk时检查中断
- 网络中断重试时检查中断
- 收到中断信号立即关闭HTTP流，放弃剩余响应

#### 4. 长时间运行的工具内部轮询检查

耗时工具（web搜索、视觉处理、代码执行、浏览器操作、MCP调用）内部都会循环检查：

```python
# tools/web_tools.py
while fetching_next_page:
    if is_interrupted():
        return {"output": "[interrupted]", "returncode": 130}
    # ...抓取下一页

# tools/browser_tool.py
if is_interrupted():
    browser.close()
    return "浏览器操作已被用户中断"

# tools/approval.py 等待用户审批时
while waiting_for_user_approval:
    if is_interrupted():
        return "审批等待被中断"
```

#### 5. 子代理心跳检测

子代理（delegate）有心跳机制，父Agent可以通过 `interrupt_subagent(sid)` 给子代理线程发中断信号，子代理的 `is_interrupted()` 会返回True并退出。

### 中断后的清理：close_interrupted_tool_sequence

中断不是简单break，Hermes会做**消息序列修复**：
- 如果正在等工具返回时中断，需要补一条"cancelled"的tool消息
- 保证消息序列的角色交替（不连续两个assistant消息）
- 这样模型不会因为消息格式错乱而出错

核心逻辑在 `close_interrupted_tool_sequence()`：
```python
def close_interrupted_tool_sequence(messages, interrupt_text):
    """修复中断时的消息序列，保证格式合法"""
    for tool_call in pending_tool_calls:
        messages.append(make_tool_result_message(
            tool_call.id,
            _cancelled_tool_result(interrupt_text)
        ))
```

### 子代理（Delegate）的中断传播

Gateway/多代理场景下，中断可以级联传播：
1. 用户中断父Agent →
2. 父Agent给所有活跃子Agent线程 `set_interrupt(True, child_tid)` →
3. 子Agent工具内部 `is_interrupted()` 返回True →
4. 子Agent清理退出 →
5. 父Agent收到子Agent返回的"cancelled"，继续处理用户新消息

### 可中断设计的关键原则总结

| 原则 | 具体做法 |
|------|---------|
| **线程粒度** | 按thread_id记录中断状态，不影响其他并发Agent |
| **小步sleep** | 所有阻塞等待都拆成≤200ms的小片段，每片检查 |
| **埋点 everywhere** | 循环、重试、网络流、长工具、审批等待——每个阻塞点都检查 |
| **工具友好** | `is_interrupted()` 无参数，工具不用传agent引用，零成本调用 |
| **优雅清理** | 中断后修复消息序列、关闭资源、补cancelled结果，不让消息历史坏掉 |
| **级联传播** | 父中断能传到子代理线程，不留下孤儿任务 |

**响应延迟目标：用户按下停止 → Agent响应 ≤200ms**

---

## Q3: 迭代预算与宽限调用（Iteration Budget + Grace Call）

### 为什么需要迭代预算？

ReAct循环理论上可以无限运行——模型一直返回工具调用，一直执行，永远不结束。为了防止死循环和成本失控，Hermes引入了**IterationBudget**（迭代预算）。

### 核心实现

`agent/iteration_budget.py` + `agent/conversation_loop.py#L604-L628`：

```python
# 主循环条件
while (api_call_count < agent.max_iterations and agent.iteration_budget.remaining > 0) 
       or agent._budget_grace_call:
    # ...

    # 宽限调用逻辑
    if agent._budget_grace_call:
        agent._budget_grace_call = False  # 消费宽限标志，这轮后就退出
    elif not agent.iteration_budget.consume():
        _turn_exit_reason = "budget_exhausted"
        agent._safe_print(
            f"\n⚠️  Iteration budget exhausted "
            f"({agent.iteration_budget.used}/{agent.iteration_budget.max_total} iterations used)"
        )
        break
```

### 默认配置
- 默认 `max_iterations = 8`（8次LLM API调用/回合）
- 这个数字是经验值：大部分任务在5-6次迭代内完成，8次足够处理复杂任务，又防止无限循环
- 用户可以通过配置调整

### 宽限调用（Grace Call）是什么？

预算耗尽时不是立即退出，而是给模型**最后一次机会**生成最终回复：
- `agent._budget_grace_call = True` → 允许再多一次LLM调用
- 这一次调用专门给模型生成总结/部分结果，而不是继续工具调用
- 防止刚好在工具调用完时预算耗尽，模型没有机会回复用户
- 如果宽限调用里模型又返回工具调用，预算真的耗尽，退出

---

## Q4: 多层错误恢复与重试机制

Hermes的错误恢复不是简单的"重试3次"，而是针对不同错误类型有专门的恢复策略，共**7层恢复机制**：

| 错误类型 | 检测方式 | 恢复策略 | 重试次数 |
|---------|---------|---------|---------|
| 1. 限流/5xx错误 | HTTP 429/500/503 | 自适应退避重试 + 抖动(jitter) | 默认5次 |
| 2. 网络中断（流中途断了） | PARTIAL_STREAM_STUB_ID | 续传提示词，继续生成 | 最多3次 |
| 3. 输出截断（finish_reason=length） | finish_reason == "length" | 追加"continue"提示，继续生成 | 文本截断3次；工具截断3次 |
| 4. 工具调用JSON截断 | finish_reason=length 且有tool_calls | 增加max_tokens重跑同一请求 | 3次，每次翻倍max_tokens |
| 5. 空响应（模型什么都没返回） | content=None/""且无tool_calls | 分情况恢复：部分流恢复→上轮内容回退→nudge提示→thinking预填充→空响应重试 | 3次 |
| 6. 思考预算耗尽（全在想没输出） | 有<think>块但后面无可见内容 | 直接返回用户友好提示，降低思考努力/换模型 | 不重试，直接退出 |
| 7. 主provider失败 | API错误/限流/额度耗尽 | 自动激活fallback provider链，重试计数重置 | fallback链长度 |

### 详细恢复逻辑

#### 4.1 自适应限流退避

`agent/retry_utils.py`：
```python
while retry_count < max_retries:
    try:
        response = call_api()
        break
    except RateLimitError:
        backoff = adaptive_rate_limit_backoff(retry_count)
        # 不一次性sleep完，每200ms检查一次中断
        for _ in range(int(backoff / 0.2)):
            if agent._interrupt_requested:
                return interrupted_response
            time.sleep(0.2)
        retry_count += 1
```
退避时间随重试次数指数增加，加随机抖动防止惊群效应。

#### 4.2 工具调用截断恢复（非常巧妙）

`conversation_loop.py#L1754-L1791`：
- 检测到tool_calls被截断（JSON不完整）
- **不把坏消息加入历史**（会污染上下文）
- 每次重试**翻倍max_tokens**，给模型更多空间写完JSON
- 第一次4096 → 第二次8192 → 第三次16384（上限32768）
- 重试3次还不行就报错告诉用户

```python
if truncated_tool_call_retries < 3:
    truncated_tool_call_retries += 1
    # Boost max_tokens on each retry
    _tc_boost_base = agent.max_tokens if agent.max_tokens else 4096
    _tc_boost = _tc_boost_base * (truncated_tool_call_retries + 1)
    agent._ephemeral_max_output_tokens = min(_tc_boost, 32768)
    continue  # 重跑同一个API调用，不追加坏消息
```

#### 4.3 空响应恢复（4层递进）

模型返回空内容（content=None且无tool_calls）时，按优先级尝试4种恢复：

```
空响应
  ↓
1. 部分流恢复？→ 如果已经流式输出了部分内容给用户，直接用已输出的
  ↓ 否
2. 上一轮内容回退？→ 上一轮全是housekeeping工具（memory/todo）+ 有文本内容，用那个
  ↓ 否
3. 工具后nudge？→ 最近5条有tool消息，加提示"请处理工具结果继续任务"
  ↓ 否/重试过
4. Thinking预填充？→ 模型有思考块但没输出，把思考块加入历史继续（prefill）
  ↓ 否/重试2次
5. 空响应重试？→ 重新调用API
  ↓ 3次都空
6. 激活fallback provider
```

**nudge提示词**（conversation_loop.py#L4449-L4457）：
```python
messages.append({
    "role": "user",
    "content": "You just executed tool calls but returned an empty response. "
               "Please process the tool results above and continue with the task.",
    "_empty_recovery_synthetic": True,
})
```
注意：追加空assistant消息"(empty)"保证消息序列合法（tool→assistant→user，不能直接tool→user）。

#### 4.4 思考预算耗尽（Thinking Budget Exhausted）

`conversation_loop.py#L1632-L1691`：
- 检测：模型返回了<think>块，但<think>后面**完全没有可见内容**（finish_reason=length）
- 说明模型把所有输出token都花在思考上了，根本没开始回答
- 这种情况继续续传也没用，直接返回用户友好提示：
  ```
  ⚠️ Thinking Budget Exhausted
  
  The model used all its output tokens on reasoning and had none left for the actual response.
  
  To fix this:
  → Lower reasoning effort: `/thinkon low` or `/thinkon minimal`
  → Or switch to a larger/non-reasoning model with `/model`
  ```

---

## Q5: Fallback Provider链（自动故障转移）

### 为什么需要Fallback？

单个API provider可能出现：额度耗尽、限流、宕机、地区不可用、模型下线等问题。Fallback链让Hermes自动切换到备用provider，用户无感知。

### 核心实现

`agent/fallback_manager.py` + `conversation_loop.py#L1006-L1013`：
```python
if is_rate_limited_or_error:
    nous_remaining = nous_rate_limit_remaining()
    if nous_remaining <= 0:
        f"⏳ {nous_msg} Trying fallback..."
        if agent._try_activate_fallback():
            # 激活fallback成功：重置所有重试计数
            retry_count = 0
            compression_attempts = 0
            _retry.primary_recovery_attempted = False
            # rewrite模型身份行，恢复prefix cache字节同一性
            rewrite_prompt_model_identity(agent, rt["model"], rt["provider"])
```

### Fallback关键设计
1. **重试计数重置**：切换到新provider后从头开始重试，新provider有完整机会
2. **字节同一性恢复**：`rewrite_prompt_model_identity()` 把prompt中的`Model:/Provider:`行改回原始值，保证prefix cache继续命中
3. **凭证池轮换**：同provider多个API key自动轮换，耗尽了再切下一个provider
4. **可以配置多个fallback**：比如 主Anthropic → 备用OpenRouter → 再备用本地Ollama

---

## Q6: 消息卫生（Message Sanitization）

在发给API之前，Hermes会做**多层消息清洗和修复**，保证消息格式100%合法，防止API报错：

`conversation_loop.py#L722-L876`：

| 清理操作 | 目的 |
|---------|------|
| **工具参数修复** `_sanitize_tool_call_arguments()` | 修复模型生成的不合法JSON（缺逗号、引号不闭合等） |
| **孤儿工具结果清理** `_sanitize_api_messages()` | 删掉没有对应tool_call的tool结果，补全缺失的tool结果占位符 |
| **纯思考回合丢弃** `_drop_thinking_only_and_merge_users()` | 删掉只有thinking没有内容/tool_calls的assistant消息，合并相邻user消息 |
| **Surrogate字符清理** `_sanitize_messages_surrogates()` | 清理Unicode代理对，防止OpenAI SDK报错 |
| **非ASCII清理** `_sanitize_messages_non_ascii()` | 某些provider不支持的特殊字符处理 |
| **Strict API工具清理** `_sanitize_tool_calls_for_strict_api()` | 严格模式下工具参数额外清洗 |
| **内部标记剥离** `api_msg.pop("_thinking_prefill", None)` | 剥离内部标记，不发给API |

### 孤儿工具结果问题

什么是孤儿？
- **孤立tool结果**：有tool消息，但找不到对应的assistant tool_call → 删掉
- **缺失tool结果**：有assistant tool_call，但没有对应的tool结果 → 补一个占位符结果：
  ```
  [System: Tool result missing — the stream timed out before it could be delivered. 
  Do NOT retry the same tool call. Continue.]
  ```
这防止了：
- 会话从磁盘加载时消息不完整
- 流中断导致tool_call/result不配对
- 模型生成非法消息序列导致API 400错误

---

## Q7: 上下文压缩（Context Compression）

当对话历史太长接近上下文窗口限制时，Hermes自动压缩旧消息：

`conversation_loop.py#L4300-L4328`：

```python
_real_tokens = estimate_request_tokens_rough(messages, tools=agent.tools or None)
if agent.compression_enabled and _compressor.should_compress(_real_tokens):
    agent._safe_print("  ⟳ compacting context…")
    messages, active_system_prompt = agent._compress_context(
        messages, system_message,
        approx_tokens=agent.context_compressor.last_prompt_tokens,
        task_id=effective_task_id,
    )
    # 压缩后创建新session，旧的归档
    conversation_history = None
```

### 压缩时机
- 主压缩器：上下文窗口 **50%** 时触发（提前压缩，给后续消息留空间）
- Gateway卫生安全网：**85%** 时触发兜底压缩
- 压缩不是一次性压完，而是渐进式，保持最近N轮消息原文，旧消息摘要

### 压缩后处理
1. **invalidate_system_prompt()**：清空缓存的system prompt，重新构建（reload memory）
2. **创建新session**：压缩后的消息写入新session，旧session归档保留
3. **Prefix Cache恢复**：system prompt重建后stable部分仍然字节稳定，缓存1-2轮内恢复命中
4. **压缩消息保护**：前3条非系统消息、后6条消息不压缩，保护开场指令和最近上下文

---

## Q8: 预飞压缩（Pre-flight Compression）

回合开始前先估算token数，超阈值就先压缩再开始LLM调用——而不是等到API报错再处理。

回合初始化时（conversation_loop.py）：
1. 估算当前messages+tools的总token数（含工具schema，工具多的时候能加20-30k token）
2. 如果超过压缩阈值，先做preflight compression
3. 然后才开始构建api_messages和LLM调用

---

## Q9: 思考Spinner（等待动画）

`conversation_loop.py#L952-L964`：

模型"思考"时显示可爱的动画，让用户知道Agent在工作：

```python
if not agent._has_stream_consumers() and agent._should_start_quiet_spinner():
    face = random.choice(KawaiiSpinner.get_thinking_faces())
    verb = random.choice(KawaiiSpinner.get_thinking_verbs())
    thinking_spinner = KawaiiSpinner(f"{face} {verb}...", spinner_type=spinner_type, print_fn=agent._print_fn)
    thinking_spinner.start()
```

- 有流式输出时不显示spinner（用户直接看到token输出）
- 非流式/安静模式下显示spinner
- 随机选择表情（🤔💭⭐🌟等）和动词（thinking, pondering, processing等）
- 收到第一个token时spinner自动停止

---

## Q10: 增量会话持久化

`conversation_loop.py#L4324-L4325`：

```python
# Save session log incrementally (so progress is visible even if interrupted)
agent._session_messages = messages
```

不是回合结束才保存会话，而是每次工具执行完/LLM调用后就增量保存：
- 如果进程崩溃/被kill，之前的对话不会丢失
- 恢复时从最后保存点继续
- 后台review等fork也能看到最新消息

---

## 关键源码位置

| 概念 | 文件位置 |
|------|---------|
| ReAct对话循环主逻辑 | `agent/conversation_loop.py` |
| 三层System Prompt架构 | `agent/system_prompt.py` |
| Prompt缓存标记策略 | `agent/prompt_caching.py` |
| **线程级中断机制** | **`tools/interrupt.py`** |
| 工具并发执行与中断 | `agent/tool_executor.py` |
| 中断后消息序列修复 | `agent/tool_dispatch_helpers.py` (close_interrupted_tool_sequence) |
| 子代理中断传播 | `tools/delegate_tool.py` (interrupt_subagent) |
| **迭代预算** | `agent/iteration_budget.py` |
| **重试退避策略** | `agent/retry_utils.py` (adaptive_rate_limit_backoff, jittered_backoff) |
| **重试状态跟踪** | `agent/turn_retry_state.py` (TurnRetryState) |
| **Fallback管理器** | `agent/fallback_manager.py` (_try_activate_fallback, rewrite_prompt_model_identity) |
| **消息卫生/清理** | `agent/` (_sanitize_api_messages, _sanitize_tool_call_arguments, _drop_thinking_only_and_merge_users) |
| **上下文压缩器** | `agent/context_compressor.py` (_compress_context, should_compress) |
| **可爱思考Spinner** | `agent/kawaii_spinner.py` (KawaiiSpinner) |
| **增量会话持久化** | `run_agent.py` (_persist_session, _session_messages) |
