# 09 - 异步后台（Fire-and-Forget）模式
> 发射即忘：非阻塞写入 + 有界排空 + 永不丢写的后台执行机制
> 学习日期：2026-07-03

---

## 一、什么是 Fire-and-Forget？

### 1.1 字面含义
**Fire-and-Forget**（发射即忘，也叫"发后即忘"）是一种异步执行模式：
- **Fire（发射）**：把任务提交给后台执行
- **Forget（忘记）**：提交完就不管了，**不等待结果、不阻塞当前流程、不关心什么时候完成**

就像寄一封平信：你把信投进邮筒（fire），然后就去做别的事了（forget），不需要站在邮局等信送到，也不需要邮递员给你打电话确认。

---

### 1.2 Hermes 中为什么需要 Fire-and-Forget？

**血的教训（来自代码注释）**：
> 一个配置错误的 Hindsight 记忆守护进程阻塞了 **298秒** 才失败！

如果把记忆同步（`sync_turn`）放在turn结束的**同步路径**上，会导致：

```
同步执行的糟糕体验（没有fire-and-forget）：
用户发消息
  ↓
LLM思考 → 工具调用 → 生成最终回复
  ↓
[用户已经看到回复了！]
  ↓
❌ 开始同步记忆到外部Provider（网络调用）
  ↓
外部Provider网络慢/配置错误 → 阻塞298秒...
  ↓
Agent一直显示"running"状态
  ↓
用户等不及了，发下一条消息
  ↓
触发激进中断（aggressive interrupt）→ 体验极差！
```

**Fire-and-Forget 解决这个问题**：

```
Fire-and-Forget 的正确体验：
用户发消息
  ↓
LLM思考 → 工具调用 → 生成最终回复
  ↓
✅ 用户立即看到回复！TTFT（首token延迟）= 真正的响应时间
  ↓
[后台默默开始同步记忆]（用户完全感知不到）
  ↓
用户可以立刻发下一条消息，不需要等待
  ↓
[后台任务] 同步完成/失败 → 只打日志，不影响用户
```

---

### 1.3 核心原则：用户响应优先

| 操作 | 应该同步还是异步？ | 原因 |
|------|------------------|------|
| **LLM调用** | 同步（必须等结果） | 用户需要看到回复才能继续 |
| **内置memory写文件** | 同步（很快） | 本地文件写入毫秒级，不影响体验 |
| **外部Provider sync_turn** | 🔥 Fire-and-Forget 异步 | 网络可能慢/挂掉，绝不能阻塞用户 |
| **外部Provider prefetch预热** | 🔥 Fire-and-Forget 异步 | 为下一回合做缓存，现在不需要结果 |
| **工具调用结果返回** | 同步（必须等结果） | 模型需要工具结果才能继续推理 |
| **日志/遥测上报** | Fire-and-Forget 异步 | 后台默默上报，用户不需要等 |

**一句话原则**：**用户在等结果的操作必须同步，用户不需要立刻知道结果的操作全部后台做！**

---

## 二、Fire-and-Forget 的核心实现

Hermes记忆系统的fire-and-forget实现在 [memory_manager.py](file:///e:/github/hermes-agent/agent/memory_manager.py#L618-L646) 的 `_submit_background()` 方法，基于 `ThreadPoolExecutor` 单工作线程池。

### 2.1 核心数据结构

```python
class MemoryManager:
    def __init__(self):
        # 惰性创建的单工作线程线程池
        self._sync_executor: Optional[ThreadPoolExecutor] = None
        # 双检锁保护executor的首次创建
        self._sync_executor_lock = threading.Lock()
```

| 字段 | 作用 |
|------|------|
| `_sync_executor` | **惰性创建**的线程池；`max_workers=1`、线程名前缀 `mem-sync` |
| `_sync_executor_lock` | 双检锁（double-checked locking），线程安全地惰性初始化 |

#### 为什么 max_workers=1（单线程）？
不是多线程更快吗？**不，这里单线程才是正确的！**

**关键原因：保证写入顺序**
```
必须保证的顺序：
Turn 1 的 sync_turn → Turn 2 的 sync_turn → Turn 3 的 sync_turn

如果用多线程（max_workers>1）：
Turn 1 提交 → 网络慢...
Turn 2 提交 → 网络快，先完成！
Turn 3 提交
→ 顺序错乱！记忆里Turn 2的内容比Turn 1先落库 → 回忆时顺序乱了

单线程（max_workers=1）天然串行化：
任务队列：[Turn1, Turn2, Turn3] → FIFO顺序执行 → 永远不会乱序
```

单线程的额外好处：
- Provider不需要自己保证线程安全
- 避免竞态条件
- 语义简单可预测

#### 为什么惰性创建？
- 只使用内置记忆（builtin）的常见路径，**根本不创建额外线程**
- 零额外开销，只有需要时才创建
- 资源友好

---

### 2.2 `_submit_background` — Fire-and-Forget 核心派发

**代码位置**：[memory_manager.py:618-646](file:///e:/github/hermes-agent/agent/memory_manager.py#L618-L646)

```python
def _submit_background(self, fn) -> None:
    """Run ``fn`` on the manager's background worker.

    The executor is created lazily and shared across calls. If the
    executor can't be created or has already been shut down, ``fn``
    runs inline as a last-resort fallback — losing the async benefit
    but never losing the write itself. ``fn`` must do its own
    per-provider error handling; this wrapper only guards executor
    plumbing.
    """
    executor = self._get_sync_executor()
    if executor is None:
        # Executor不可用（关闭/创建失败）→ 降级为内联同步执行
        # 慢，但正确：绝不丢写！
        try:
            fn()
        except Exception as e:
            logger.debug("Inline memory background task failed: %s", e)
        return
    try:
        # 🔥 真正的fire-and-forget：submit()返回的Future直接丢弃！
        # 不保留Future、不await、不回调、不等待结果
        executor.submit(fn)
    except RuntimeError:
        # get和submit之间executor被关闭了（关闭竞态）→ 降级内联
        try:
            fn()
        except Exception as e:
            logger.debug("Inline memory background task failed: %s", e)
```

---

### 2.3 四个关键设计决策

#### 决策1：Future直接丢弃（真正的fire-and-forget）

```python
executor.submit(fn)  # ← 返回的 concurrent.futures.Future 被直接扔掉！
```

对比其他做法：
| 做法 | 问题 |
|------|------|
| `future.result()` | ❌ 阻塞等待，失去异步意义 |
| `future.add_done_callback()` | 可以，但增加复杂度；Hermes不需要回调，失败只打日志 |
| 保存future后面检查 | 需要管理future列表，增加复杂度；这里不需要 |
| **直接丢弃future** | ✅ 最简单！fn内部自己处理异常，失败只打日志 |

#### 决策2：永不丢写（best-effort delivery）

两层降级兜底：

```
正常路径：executor.submit(fn) → 后台执行
    ↓ 如果executor为None（创建失败/已shutdown）
降级路径1：try: fn() → 同步内联执行（慢，但不丢）
    ↓ 如果同步执行也抛异常
降级路径2：logger.debug() → 只打debug日志，不崩溃
```

**设计理念**：
- 后台异步是**性能优化**，不是**正确性前提**
- 异步失败了，大不了慢一点同步做，但**绝对不能丢失这次记忆写入**
- 就像邮筒坏了，你不会把信扔了，而是自己跑一趟邮局送过去

#### 决策3：业务异常在fn内部处理

```python
# 注意：_submit_background本身不catch业务异常！
# 传入的_run闭包必须自己对每个provider做try/except
```

为什么？因为`_submit_background`只负责"管道层"的异常保护（executor创建失败、shutdown竞态），业务层的provider异常必须在fn内部处理：

```python
# sync_all中构造的_run闭包（示例）
def _run():
    for provider in self._providers:
        try:
            provider.sync_turn(user, asst, ...)  # 每个provider独立try/except
        except Exception as e:
            logger.warning("Memory provider '%s' sync_turn failed: %s", 
                         provider.name, e)
            # 一个provider失败，继续下一个，不影响其他provider
```

#### 决策4：双检锁惰性创建executor

**代码位置**：[memory_manager.py:647-661](file:///e:/github/hermes-agent/agent/memory_manager.py#L647-L661)

```python
def _get_sync_executor(self) -> Optional[ThreadPoolExecutor]:
    """Lazily create the single-worker background executor."""
    if self._sync_executor is not None:
        return self._sync_executor  # 快速路径：无锁
    with self._sync_executor_lock:
        if self._sync_executor is None:  # 双检：拿到锁后再检查一次
            try:
                self._sync_executor = ThreadPoolExecutor(
                    max_workers=1,
                    thread_name_prefix="mem-sync",  # 线程命名方便调试
                )
            except Exception as e:
                logger.warning("Failed to create memory sync executor: %s", e)
                return None
        return self._sync_executor
```

双检锁（Double-Checked Locking）的好处：
- 第一次之后的调用走无锁快速路径，性能好
- 多线程同时第一次调用时，只有一个线程真正创建
- 线程安全

---

## 三、两个 Fire-and-Forget 入口点

### 3.1 `sync_all`：回合结束后写记忆

**代码位置**：[memory_manager.py:558-614](file:///e:/github/hermes-agent/agent/memory_manager.py#L558-L614)

Turn完成、回复已经交付给用户之后调用，把本轮对话写入所有记忆Provider。

```python
def sync_all(self, user_content, assistant_content, *, session_id="", messages=None):
    # 第一步：剥离技能脚手架（/skill调用时的大段SKILL.md正文）
    clean_query = self._strip_skill_scaffolding(user_content or "")
    clean_response = self._strip_skill_scaffolding(assistant_content or "")
    
    if not clean_query.strip():
        return  # 没有用户真实内容，跳过（比如纯/skill调用）
    
    # 第二步：构造_run闭包（在后台线程执行）
    def _run():
        for provider in self._providers:
            try:
                # 反射检查provider的sync_turn接受哪些参数（向后兼容老provider）
                if self._provider_sync_accepts_messages(provider):
                    provider.sync_turn(clean_query, clean_response, 
                                     session_id=session_id, messages=messages)
                else:
                    provider.sync_turn(clean_query, clean_response, 
                                     session_id=session_id)
            except Exception as e:
                logger.warning(
                    "Memory provider '%s' sync_turn failed: %s",
                    provider.name, e,
                )
    
    # 第三步：🔥 fire-and-forget提交！
    self._submit_background(_run)
```

**前置处理**：剥离技能脚手架
- 用户调用 `/skill xxxx` 时，系统会把整个SKILL.md正文注入到消息里
- 这部分不是用户的真实指令，不应该存进记忆污染embedding
- `_strip_skill_scaffolding()` 把技能模板文本剥离，只留用户真实输入

**调用位置**：[run_agent.py 约L3145-3149]，在turn结束、回复已交付给用户之后的收尾路径上。

---

### 3.2 `queue_prefetch_all`：预热下一回合的召回

**代码位置**：[memory_manager.py:517-542](file:///e:/github/hermes-agent/agent/memory_manager.py#L517-L542)

紧跟在`sync_all`之后调用，让Provider为**下一回合**提前做召回/缓存（比如embedding检索预计算、向量库预热）。

```python
def queue_prefetch_all(self, user_content: str, *, session_id: str = "") -> None:
    clean_query = self._strip_skill_scaffolding(user_content or "")
    
    def _run():
        for provider in self._providers:
            try:
                provider.queue_prefetch(clean_query, session_id=session_id)
            except Exception as e:
                logger.debug(
                    "Memory provider '%s' queue_prefetch failed: %s",
                    provider.name, e,
                )
    
    self._submit_background(_run)
```

为什么要做prefetch预热？
- 下一回合用户发消息时，`prefetch()`需要同步快速返回缓存结果
- `queue_prefetch`在后台提前把下一回合可能需要的内容算好、缓存好
- 下一回合`prefetch()`直接拿缓存，不阻塞TTFT

**调用位置**：[run_agent.py 约L3150-3153]，紧跟`sync_all`之后，两者连续fire到后台。

---

### 3.3 调用链全景（Turn结束路径）

```
run_conversation 结束
  ↓
finalize_turn() → _persist_memory_and_prefetch()  [run_agent.py 约L3105-L3153]
  │
  ├─ 守卫检查：interrupted? 空内容? 多模态扁平化?
  │
  ├─→ memory_manager.sync_all(user_text, response_text, ...)
  │       ↓
  │       _strip_skill_scaffolding()
  │       构造 _run 闭包（对每个provider try/except）
  │       ↓
  │       🔥 _submit_background(_run)
  │           ↓
  │           ThreadPoolExecutor(max_workers=1, "mem-sync")
  │               ↓ （队列FIFO顺序执行）
  │               for p in providers:
  │                   p.sync_turn(...)  ← 网络/磁盘写入
  │
  └─→ memory_manager.queue_prefetch_all(user_text, ...)  （sync之后）
          ↓
          构造 _run 闭包
          ↓
          🔥 _submit_background(_run)
              ↓
              （排在sync任务之后，同一后台线程）
              for p in providers:
                  p.queue_prefetch(...)  ← 为下一回合预热缓存
```

**整个过程调用方立即返回，用户侧零等待！** 异常被消化进日志，用户完全感知不到后台在做什么。

---

## 四、有界排空：Fire-and-Forget 不是"永远不等"

Fire-and-Forget不是说"永远不等待结果"——在**会话边界、进程退出、测试断言**时，我们需要确定性地等待后台任务完成（排空队列）。

### 4.1 `flush_pending`：测试用的屏障

**代码位置**：[memory_manager.py:663-684](file:///e:/github/hermes-agent/agent/memory_manager.py#L663-L684)

```python
def flush_pending(self, timeout: Optional[float] = None) -> bool:
    """Block until queued sync/prefetch work has drained.

    Single-worker executor means submitting a sentinel and waiting on
    it guarantees every previously-submitted task has run. Returns
    True if the barrier completed within ``timeout``, False on timeout.
    """
    executor = self._sync_executor
    if executor is None:
        return True
    try:
        # 提交一个哨兵空任务
        fut = executor.submit(lambda: None)
    except RuntimeError:
        return True  # executor已关闭，没有待处理任务
    try:
        fut.result(timeout=timeout)  # 等待哨兵完成
        return True
    except Exception:
        return False
```

**为什么这个技巧有效？**
因为`max_workers=1`单线程FIFO队列：

```
提交顺序：
  [Task1(Turn1 sync), Task2(Turn1 prefetch), Task3(哨兵lambda: None)]

执行顺序（FIFO）：
  Task1 → Task2 → Task3 → 哨兵完成！

当哨兵执行完返回时，说明它前面所有任务都已经完成了！
```

这是一个利用单线程串行性质的聪明屏障（barrier）技巧。

**使用场景**：测试代码中大量使用，获得确定性的provider状态：
```python
# 测试代码示例
memory_manager.sync_all("user", "assistant", ...)
memory_manager.flush_pending(timeout=5)  # 等5秒让后台任务跑完
# 现在可以断言provider状态了，确定性！
assert "预期内容" in memory_provider.last_synced_content
```

---

### 4.2 `shutdown_all` / `_drain_sync_executor`：进程退出时的有界等待

**代码位置**：[memory_manager.py:999-1055](file:///e:/github/hermes-agent/agent/memory_manager.py#L999-L1055)

进程/会话结束时，不能直接退出让后台任务半路中断——需要给一个有界的排空时间，但也不能无限等一个卡死的provider。

```python
_SYNC_DRAIN_TIMEOUT_S = 5.0  # 最多等5秒

def shutdown_all(self) -> None:
    self._drain_sync_executor()  # 先排空后台executor
    for provider in reversed(self._providers):
        try:
            provider.shutdown()  # 再逆序关闭provider
        except Exception as e:
            logger.warning(...)

def _drain_sync_executor(self) -> None:
    with self._sync_executor_lock:
        executor = self._sync_executor
        self._sync_executor = None  # 置空，阻止新任务提交
    if executor is None:
        return
    
    # 第一步：停止接收新任务，取消队列中尚未开始的任务
    # wait=False：不阻塞调用线程！
    # cancel_futures=True：取消排队中还没开始的任务
    # 注意：已经在执行的那个任务不会被取消，继续跑
    executor.shutdown(wait=False, cancel_futures=True)
    
    # 第二步：在daemon watcher线程上有界等待
    drainer = threading.Thread(
        target=lambda: self._bounded_executor_wait(executor),
        daemon=True,  # daemon线程：主线程退出时它自动终止
        name="mem-sync-drain",
    )
    drainer.start()
    drainer.join(timeout=_SYNC_DRAIN_TIMEOUT_S)  # 最多等5秒

@staticmethod
def _bounded_executor_wait(executor):
    executor.shutdown(wait=True)  # 这个线程上真正等待
```

---

#### Shutdown流程的关键保证

| 保证 | 实现方式 |
|------|---------|
| **不阻塞调用线程** | `executor.shutdown(wait=False)` + watcher线程join最多5秒 |
| **有界等待** | 固定5秒超时，卡死的provider绝不会无限期阻塞退出 |
| **最后一次sync有机会落地** | shutdown前先drain，给当前在跑的sync最多5秒完成 |
| **排队任务不丢？不，取消排队任务** | `cancel_futures=True`取消队列里还没开始的，这些不重要（下一回合的prefetch预热） |
| **daemon线程兜底** | watcher和worker都是daemon线程，超过5秒还没跑完就随解释器退出终止 |
| **provider逆序关闭** | 先drain完写入，再逆序shutdown provider（类似栈的后进先出，保证依赖顺序） |

#### 为什么可以接受5秒后还没跑完就杀掉？
- 已经在跑的那个任务是**最后一回合的sync**——能跑完最好，跑不完也没关系
- 下次会话启动时记忆系统会恢复到一致状态
- 比起让用户等5秒以上才能退出进程，丢最后一次记忆写入是可接受的权衡
- daemon线程保证进程不会变成僵尸进程

---

## 五、对比：同步 vs Fire-and-Forget vs 异步await

| 模式 | 阻塞当前线程？ | 等待结果？ | 适用场景 | Hermes中的例子 |
|------|--------------|-----------|---------|---------------|
| **同步执行** | ✅ 阻塞 | ✅ 等结果 | 用户必须等结果才能继续 | LLM调用、工具调用、内置memory写文件 |
| **🔥 Fire-and-Forget** | ❌ 不阻塞 | ❌ 不等（提交就忘） | 用户不需要立刻知道结果，失败可接受 | 外部记忆sync、prefetch预热、遥测上报 |
| **异步await（asyncio）** | ❌ 不阻塞（协程挂起） | ✅ 等结果（但挂起不阻塞线程） | 需要结果但不想阻塞线程 | MCP传输、HTTP请求 |
| **后台+回调/Future** | ❌ 不阻塞 | 🔶 可选等待/回调 | 需要后续处理结果 | 异步任务需要后续聚合时 |

### 为什么 Hermes 记忆系统用 Fire-and-Forget 而不是 asyncio？

1. **线程模型简单**：`run_conversation` 主循环是同步风格代码（或在专用asyncio循环上），用 `ThreadPoolExecutor` 不需要把整个调用链改成async
2. **失败语义简单**：失败只打日志，不需要重试、不需要传播错误、不需要用户知道
3. **单线程顺序保证**：`max_workers=1`天然FIFO，比asyncio队列更直观
4. **惰性创建**：不需要就不创建线程，builtin路径零开销
5. **排空简单**：`flush_pending`哨兵屏障比asyncio的gather简单

---

## 六、不变量与设计保证总结

| 不变量 | 实现手段 |
|--------|---------|
| **用户响应不被阻塞** | sync/prefetch全部走 `_submit_background` fire-and-forget |
| **写入按Turn顺序落地** | `max_workers=1` 单线程FIFO串行执行 |
| **惰性创建无竞态** | `_sync_executor_lock` 双检锁 |
| **永不丢写（best-effort）** | executor不可用时降级为同步内联执行 |
| **Shutdown不崩** | submit捕获 `RuntimeError` 并内联兜底 |
| **一个Provider故障不影响其他** | `_run`闭包内按provider独立try/except |
| **进程退出不被卡死** | 5秒有界drain + daemon线程兜底 |
| **测试可确定性断言** | `flush_pending` 哨兵屏障 |
| **技能脚手架不污染记忆** | `_strip_skill_scaffolding` 统一剥离 |
| **新老Provider兼容** | 反射检查方法签名 `_provider_sync_accepts_messages` |

---

## 七、代码最小示例：自己实现Fire-and-Forget

### 最简单的Fire-and-Forget（Python）

```python
from concurrent.futures import ThreadPoolExecutor
import threading
import time

class SimpleFireAndForget:
    def __init__(self):
        self._executor = None
        self._lock = threading.Lock()
    
    def _get_executor(self):
        if self._executor is not None:
            return self._executor
        with self._lock:
            if self._executor is None:
                self._executor = ThreadPoolExecutor(max_workers=1, 
                                                    thread_name_prefix="bg-worker")
            return self._executor
    
    def submit(self, fn):
        """Fire-and-forget: 提交就忘，不阻塞、不等待、不保留future"""
        executor = self._get_executor()
        try:
            executor.submit(fn)  # future直接丢弃！
        except RuntimeError:
            # executor关闭了，降级同步执行
            try:
                fn()
            except Exception:
                pass  # 业务异常fn自己处理
    
    def flush(self, timeout=5):
        """测试/关闭时用：等待所有排队任务完成"""
        if self._executor is None:
            return True
        try:
            sentinel = self._executor.submit(lambda: None)
            sentinel.result(timeout=timeout)
            return True
        except Exception:
            return False

# ========== 使用示例 ==========
bg = SimpleFireAndForget()

def slow_task(name):
    print(f"开始执行 {name}")
    time.sleep(2)  # 模拟慢网络/IO
    print(f"完成 {name}")

print("提交任务1")
bg.submit(lambda: slow_task("Task1"))
print("提交任务2")
bg.submit(lambda: slow_task("Task2"))
print("✅ 提交完了，主线程继续做别的事，不需要等！")

# ... 用户可以立刻继续，后台默默跑
time.sleep(1)
print("主线程还在跑，后台任务在默默执行...")

# 退出前可以flush等一下
bg.flush(timeout=5)
print("所有后台任务完成（或超时）")
```

---

### 反例：这些不是Fire-and-Forget

❌ **错误1：等待future.result()**
```python
# 这不是fire-and-forget，这是同步阻塞！
future = executor.submit(fn)
result = future.result()  # ❌ 阻塞等待，失去异步意义
```

❌ **错误2：多线程不保证顺序**
```python
# max_workers>1时顺序无法保证，可能导致记忆写入错乱
ThreadPoolExecutor(max_workers=8)  # ❌ 对于需要顺序的场景
```

❌ **错误3：异常不处理导致线程死亡**
```python
def bad_fn():
    raise Exception("oops")  # ❌ 如果_run闭包不catch，异常会打到线程上
                            # 虽然ThreadPoolExecutor不会让进程崩溃，但任务丢了
executor.submit(bad_fn)
```

✅ **正确：fn自己做异常隔离**
```python
def safe_run():
    for task in tasks:
        try:
            task()
        except Exception as e:
            logger.warning(f"Task failed: {e}")  # 一个失败，继续其他
executor.submit(safe_run)
```

---

## 八、适用场景与不适用场景

### ✅ 适用Fire-and-Forget的场景

| 场景 | 原因 |
|------|------|
| 遥测/日志上报 | 失败了用户也不关心，不影响功能 |
| 缓存预热/预计算 | 为未来做准备，现在不需要结果 |
| 外部记忆同步 | 网络可能慢，用户已经看到回复了 |
| 非关键通知 | 比如更新badge计数、推送通知 |
| 索引更新/搜索优化 | 下次搜索时新内容能搜到就行，现在不用等 |
| 审计日志落盘 | 后台慢慢写，不阻塞响应 |

### ❌ 不适用Fire-and-Forget的场景

| 场景 | 应该用什么 | 原因 |
|------|----------|------|
| 用户需要结果才能继续 | 同步/await | 比如工具调用，模型需要结果才能推理 |
| 事务性写入，必须成功 | 同步+确认 | 比如支付、下单，必须确认成功 |
| 需要返回错误给用户 | 同步/await | 失败要给用户看错误提示 |
| 资源清理/释放 | 同步（有界等待） | 比如关闭文件、释放锁，必须做 |
| 需要聚合多个任务结果 | Future/gather/回调 | 需要等所有结果回来再处理 |

---

## 设计哲学一句话总结

**Fire-and-Forget 的精髓不是"异步"，而是"用户优先"：把所有不影响用户当下体验的操作全部扔到后台默默做，用户拿到响应就可以继续——后台任务能跑完最好，跑不完也没关系，大不了降级同步做，但绝对不能让用户等。**

---

## 关键文件索引

| 文件 | 职责 |
|------|------|
| [memory_manager.py](file:///e:/github/hermes-agent/agent/memory_manager.py#L618-L684) | **核心实现**：`_submit_background`、`_get_sync_executor`、`flush_pending`、`_drain_sync_executor`、`shutdown_all` |
| [run_agent.py](file:///e:/github/hermes-agent/run_agent.py) | 调用点：turn结束时调用`sync_all`和`queue_prefetch_all` |
