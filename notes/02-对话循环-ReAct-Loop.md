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
