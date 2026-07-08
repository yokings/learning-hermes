# 10 - Prompt Caching 前缀缓存
> 字节级缓存 + cache_control断点策略，多轮对话输入token成本降低约75%
> 学习日期：2026-07-08

---

## 一、为什么需要 Prompt Caching？

多轮对话中，每次请求都要把整个system prompt + 完整历史消息发给LLM：
```
Turn 1: [system_prompt] + [user1] + [assistant1]
Turn 2: [system_prompt] + [user1] + [assistant1] + [user2] + [assistant2]
Turn 3: [system_prompt] + [user1] + [assistant1] + [user2] + [assistant2] + [user3] + ...
        └──────── 前缀完全一样 ────────┘
```
前缀部分（system prompt + 早期历史）**每次都重复发送、重复计费**，占比随着对话轮次增加越来越大。Anthropic/Claude、Qwen等模型提供了Prefix Cache功能：前缀相同的部分命中缓存后按**0.1x价格计费**，可以节省75%+成本。

---

## 二、Hermes 两层缓存架构

Hermes采用**客户端+服务端**两层协同：

```
┌─────────────────────────────────────────────────────────────────┐
│ Layer 1: 客户端字节级缓存 (_cached_system_prompt)                 │
│ ─────────────────────────────────────────────────────────────  │
│  System Prompt 三层结构构建后冻结，session内跨所有turn复用        │
│  只有context compression后才失效重建 → 保证字节完全一致           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  stable    (永久不变)：SOUL/工具引导/技能/平台提示         │    │
│  │  context   (session恒定)：用户system_message/项目文件     │    │
│  │  volatile  (构建时冻结)：MEMORY快照/时间戳精确到**天**     │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ Layer 2: 服务端Prefix Cache (cache_control断点)                  │
│ ─────────────────────────────────────────────────────────────  │
│  注入4个 ephemeral 断点告诉服务端缓存哪些部分：                  │
│  断点1：System Prompt              (所有turn稳定)                │
│  断点2/3/4：倒数第3/2/1条消息      (滚动窗口)                    │
│  命中后：写入1.25-2x价格，读取0.1x价格（Anthropic定价）          │
└─────────────────────────────────────────────────────────────────┘
```

核心约束：**上游prefix cache以请求字节精确匹配作为key**，任何一个字节不同都会导致cache miss。整个设计围绕这个约束展开。

---

## 三、核心实现代码

### 3.1 断点注入：[prompt_caching.py](file:///e:/github/hermes-agent/agent/prompt_caching.py)

纯函数实现，不依赖AIAgent状态：

```python
import copy
from typing import Any, Dict, List

def _apply_cache_marker(msg: dict, cache_marker: dict, native_anthropic: bool = False) -> None:
    """给单条消息加cache_control，处理所有格式变体"""
    role = msg.get("role", "")
    content = msg.get("content")

    # tool消息：native布局直接挂顶层
    if role == "tool" and native_anthropic:
        msg["cache_control"] = cache_marker
        return

    # 空内容：直接挂顶层
    if content is None or content == "":
        msg["cache_control"] = cache_marker
        return

    # 字符串：包装成text block，在block上加marker
    if isinstance(content, str):
        msg["content"] = [
            {"type": "text", "text": content, "cache_control": cache_marker}
        ]
        return

    # list/multimodal：最后一个block上加marker
    if isinstance(content, list) and content:
        last = content[-1]
        if isinstance(last, dict):
            last["cache_control"] = cache_marker

def _build_marker(ttl: str) -> Dict[str, str]:
    """构建cache_control marker，支持5m/1h TTL"""
    marker: Dict[str, str] = {"type": "ephemeral"}  # 默认5分钟TTL
    if ttl == "1h":
        marker["ttl"] = "1h"  # 长会话用1小时TTL
    return marker

def apply_anthropic_cache_control(
    api_messages: List[Dict[str, Any]],
    cache_ttl: str = "5m",
    native_anthropic: bool = False,
) -> List[Dict[str, Any]]:
    """
    system_and_3策略：4个cache_control断点
    - 断点1：system prompt
    - 断点2-4：最后3条非system消息（滚动窗口）
    降低多轮对话输入成本约75%
    """
    messages = copy.deepcopy(api_messages)
    if not messages:
        return messages

    marker = _build_marker(cache_ttl)
    breakpoints_used = 0

    # 断点1：System prompt（永远是第一个）
    if messages[0].get("role") == "system":
        _apply_cache_marker(messages[0], marker, native_anthropic=native_anthropic)
        breakpoints_used += 1

    # 断点2-4：最后3条非system消息（滚动窗口）
    remaining = 4 - breakpoints_used
    non_sys = [i for i in range(len(messages)) if messages[i].get("role") != "system"]
    for idx in non_sys[-remaining:]:
        _apply_cache_marker(messages[idx], marker, native_anthropic=native_anthropic)

    return messages
```

**为什么是"最后3条消息"？** 形成滚动窗口——每新增一轮，窗口向前滑动一条，保证最近3条对话在相邻turn间稳定命中缓存。Anthropic API每次请求最多4个`cache_control`断点，system占一个，剩下3个刚好给最近3条。

---

### 3.2 缓存策略决策：[agent_runtime_helpers.py#L1251-L1353](file:///e:/github/hermes-agent/agent/agent_runtime_helpers.py#L1251-L1353)

根据provider/model/base_url自动决定是否启用缓存，以及用哪种marker布局：

```python
def anthropic_prompt_cache_policy(agent, *, provider=None, base_url=None,
                                  api_mode=None, model=None) -> tuple[bool, bool]:
    """
    返回 (should_cache, use_native_layout):
    - should_cache: 是否注入cache_control断点
    - use_native_layout: True=native Anthropic布局(marker在content block内)
                        False=envelope布局(marker在消息外壳，OpenRouter/Qwen用)
    """
    eff_provider = (provider or agent.provider) or ""
    eff_base_url = base_url or (agent.base_url or "")
    eff_api_mode = api_mode or (agent.api_mode or "")
    eff_model = (model or agent.model) or ""

    is_claude = "claude" in eff_model.lower()
    is_openrouter = base_url_host_matches(eff_base_url, "openrouter.ai")
    is_nous_portal = "nousresearch" in eff_base_url.lower()
    is_anthropic_wire = eff_api_mode == "anthropic_messages"
    is_native_anthropic = is_anthropic_wire and (
        eff_provider == "anthropic" or
        base_url_hostname(eff_base_url) == "api.anthropic.com"
    )

    # 1. 原生Anthropic → 启用，native布局
    if is_native_anthropic:
        return True, True

    # 2. OpenRouter/Nous Portal + Claude → 启用，envelope布局
    if (is_openrouter or is_nous_portal) and is_claude:
        return True, False

    # 3. Nous Portal + Qwen → 启用，envelope布局
    if is_nous_portal and "qwen" in eff_model.lower():
        return True, False

    # 4. 第三方Anthropic兼容网关（智谱GLM/LiteLLM）→ 启用，native布局
    if is_anthropic_wire and is_claude:
        return True, True

    # 5. MiniMax自家模型（M2.7/M2.5/M2.1）→ 启用，native布局
    if is_anthropic_wire:
        if provider_lower in {"minimax", "minimax-cn"} or \
           base_url_host_matches(eff_base_url, "api.minimax.io"):
            return True, True

    # 6. 阿里Qwen（OpenCode/DashScope OpenAI-wire）→ 启用，envelope布局
    if provider_lower in {"opencode", "opencode-go", "alibaba"} and "qwen" in model_lower:
        return True, False

    # 7. 其他OpenAI-wire端点（如Fireworks）→ 严格关闭，防止未知key报错
    return False, False
```

**策略矩阵**：

| 场景 | should_cache | 布局类型 |
|------|-------------|---------|
| 原生Anthropic Claude | ✅ | native |
| OpenRouter Claude | ✅ | envelope |
| Nous Portal Claude/Qwen | ✅ | envelope |
| 第三方Anthropic兼容网关（智谱GLM/LiteLLM） | ✅ | native |
| MiniMax M2.x | ✅ | native |
| 阿里Qwen（OpenCode/DashScope） | ✅ | envelope |
| 其他OpenAI-wire端点 | ❌ | - |

---

### 3.3 System Prompt三层结构+冻结快照：[system_prompt.py#L121-L504](file:///e:/github/hermes-agent/agent/system_prompt.py#L121-L504)

```python
# 三层结构说明：
#   stable   - 进程生命周期内永久不变：SOUL身份、工具引导、技能索引、模型执行纪律、平台提示
#   context  - 同一session内cwd固定，恒定：调用方传入的system_message + 项目上下文文件
#   volatile - 构建时快照，不在turn间变化：MEMORY.md快照、USER.md用户画像、时间戳
# 拼接后缓存到 agent._cached_system_prompt，整个session复用

# 🔑 关键细节：时间戳精确到"天"，不是分钟！
from hermes_time import now as _hermes_now
now = _hermes_now()
# Date-only (not minute-precision) so the system prompt is byte-stable
# for the full day. Minute-precision changes invalidate prefix-cache KV
# on every rebuild path. Credit: @iamfoz (PR #20451).
timestamp_line = f"Conversation started: {now.strftime('%A, %B %d, %Y')}"
# ↑ 输出示例："Wednesday, July 08, 2026" → 一天内字节完全不变！
# 模型需要知道精确时间可以通过工具查询，不需要写在system prompt里

def build_system_prompt(agent, system_message=None) -> str:
    """
    Called once per session (cached on agent._cached_system_prompt)
    and only rebuilt after context compression. Ensures system prompt
    is stable across all turns, maximizing prefix cache hits.

    Layers ordered cache-friendly: stable → context → volatile.
    The whole string is one cached block — never rebuilds parts mid-session.
    """
    parts = build_system_prompt_parts(agent, system_message=system_message)
    joined = "\n\n".join(p for p in (parts["stable"], parts["context"], parts["volatile"]) if p)
    return joined

def invalidate_system_prompt(agent) -> None:
    """压缩后失效缓存，强制下一turn重建，同时reload memory"""
    agent._cached_system_prompt = None
    if agent._memory_store:
        agent._memory_store.load_from_disk()
```

**为什么时间戳精确到天？**
如果精确到分钟，每轮system prompt字节都不一样，prefix cache永远miss！模型需要知道精确时间可以通过工具查询，不需要在system prompt里写死。这是一个非常经典的"字节级缓存"优化。

---

### 3.4 对话循环中每次调用前注入：[conversation_loop.py#L851-L860](file:///e:/github/hermes-agent/agent/conversation_loop.py#L851-L860)

```python
# 每次发给LLM前，如果启用了prompt caching，注入cache_control断点
if agent._use_prompt_caching:
    api_messages = apply_anthropic_cache_control(
        api_messages,
        cache_ttl=agent._cache_ttl,
        native_anthropic=agent._use_native_cache_layout,
    )

# 然后做sanitize/清理orphaned tool结果等安全检查
api_messages = agent._sanitize_api_messages(api_messages)
```

---

## 四、字节级缓存的8条铁律

整个体系围绕一个核心约束运作：**上游prefix cache以请求字节精确匹配作为key**，任何字节差异都会导致cache miss。

| 约束 | 实现方式 |
|------|---------|
| **1. 时间戳只精确到天** | 避免每分钟时钟滴答导致前缀失效 |
| **2. System prompt构建后冻结** | `_cached_system_prompt` 整个session不做turn间部分更新 |
| **3. 记忆快照在构建时固化** | mid-session写入不回写已缓存prompt（直到下次重建）→ 同时也是记忆系统冻结快照设计 |
| **4. Ephemeral内容仅在API调用时注入** | prefill/ephemeral_system_prompt不进入 `_cached_system_prompt` |
| **5. Background review fork逐字节继承** | 父agent的cached prompt、session_id、session_start、toolset必须完全一致 |
| **6. Fallback恢复后重写模型身份** | 让prompt字节回到原始状态恢复cache匹配 |
| **7. 内部fork可跳过MCP工具刷新** | `_skip_mcp_refresh` 标志防止tool schema变化导致字节差异 |
| **8. Platform hint解析在session开始/压缩时做** | 产生字节稳定的文本，不是每turn动态渲染 |

---

## 五、TTL配置与成本

配置在 `config.yaml` 中：
```yaml
prompt_caching:
  cache_ttl: "5m"   # 默认5分钟，连续对话用
  # cache_ttl: "1h" # 1小时，长会话/有停顿的对话
```

| TTL | 写入价格 | 读取价格 | 适用场景 |
|-----|---------|---------|---------|
| 5m | 1.25x | 0.1x | 连续对话，推荐默认 |
| 1h | 2.0x | 0.1x | 长会话、边想边聊、隔夜恢复 |

初始化时打印状态：
```
💾 Prompt caching: ENABLED (native Anthropic, 5m TTL)
💾 Prompt caching: ENABLED (Claude via OpenRouter, 5m TTL)
```

---

## 六、与上下文压缩的交互

上下文压缩发生在消息中段时：
1. System prompt字节保持不变 → 系统提示缓存**继续命中**
2. 调用 `invalidate_system_prompt()` 重建volatile部分（memory更新）
3. 滚动3条消息窗口在**1-2轮后自动重建**缓存命中

文档原话：
> After compression, the cache is invalidated for the compressed region but the **system prompt cache survives**. The rolling 3-message window re-establishes caching within 1–2 turns.

压缩触发点：
- 主压缩器：上下文窗口50%时触发
- Gateway卫生安全网：85%时触发兜底压缩

---

## 七、跨Fork缓存一致性（Cache Parity）

Background review后台子线程必须**逐字节继承**父agent的缓存字段，否则每条review消息会额外消耗完整system prompt成本（在Sonnet 4.5上测量约26%端到端开销）：

```python
# 必须逐字节继承：
agent._cached_system_prompt = "PARENT-SYSTEM-PROMPT-BYTES"  # verbatim
agent.session_start = parent.session_start
agent.session_id = parent.session_id
agent.enabled_toolsets = parent.enabled_toolsets
agent.disabled_toolsets = parent.disabled_toolsets
agent._skip_mcp_refresh = True  # 跳过MCP刷新防止schema变化
```

任何字节差异（新的时间戳、新session_id、变了的skills_prompt）都会导致上游prefix cache miss。

---

## 八、调用链路全景

```
Session start
    │
    ▼
agent_init.py
  ├─ anthropic_prompt_cache_policy() → (should_cache, native_layout)
  ├─ 读取 config.yaml prompt_caching.cache_ttl
  └─ build_system_prompt() → 写入 _cached_system_prompt（三层拼接）
    │
    │  ... 多轮对话 ...
    │
    ▼
conversation_loop.py（每turn发送前）
  ├─ 组装 api_messages（含 system 消息，使用 _cached_system_prompt）
  ├─ if _use_prompt_caching:
  │    apply_anthropic_cache_control(api_messages, ttl, native_layout)
  │    → deep copy，注入 4 个 cache_control 断点
  ├─ _sanitize_api_messages() 清理孤立工具对
  └─ 发送到 Provider
    │
    ├─ 原生Anthropic：cache_control在content blocks内
    ├─ OpenRouter/envelope：cache_control在消息/内容外层
    └─ 服务端KV cache字节前缀匹配，命中则0.1x价格
    │
    │  ... 若触发 context compression ...
    │
    ▼
invalidate_system_prompt()
  ├─ _cached_system_prompt = None
  ├─ reload memory from disk
  └─ 下一turn重新build，缓存1-2轮内恢复
```

---

## 设计哲学一句话总结

**Prompt Caching的精髓不是"加几个cache_control标记"，而是系统性地保证整个system prompt的字节同一性——从时间戳精确到天、三层结构冻结、跨fork逐字节继承，到Fallback后恢复、压缩后快速重建，所有设计都围绕"任何不必要的字节变化都是缓存miss和金钱浪费"这个核心。**

---

## 关键文件索引

| 文件 | 职责 |
|------|------|
| [prompt_caching.py](file:///e:/github/hermes-agent/agent/prompt_caching.py) | `apply_anthropic_cache_control()` 断点注入纯函数，`system_and_3`策略4断点 |
| [agent_runtime_helpers.py#L1251-L1353](file:///e:/github/hermes-agent/agent/agent_runtime_helpers.py#L1251-L1353) | `anthropic_prompt_cache_policy()` 策略决策，provider/模型/布局判定 |
| [system_prompt.py#L121-L504](file:///e:/github/hermes-agent/agent/system_prompt.py#L121-L504) | 三层结构（stable/context/volatile）、冻结快照、日期精确到天、失效逻辑 |
| [conversation_loop.py#L851-L860](file:///e:/github/hermes-agent/agent/conversation_loop.py#L851-L860) | 每轮API调用前注入cache_control |
| [agent_init.py#L505-L528](file:///e:/github/hermes-agent/agent/agent_init.py#L505-L528) | 初始化时探测缓存能力、读取TTL配置、启动打印状态 |
| [run_agent.py#L4814-L4873](file:///e:/github/hermes-agent/run_agent.py#L4814-L4873) | Qwen专用OpenAI-wire缓存路径处理 |
| [tests/run_agent/test_anthropic_prompt_cache_policy.py](file:///e:/github/hermes-agent/tests/run_agent/test_anthropic_prompt_cache_policy.py) | 5类端点缓存策略回归测试矩阵 |
| [tests/run_agent/test_background_review_cache_parity.py](file:///e:/github/hermes-agent/tests/run_agent/test_background_review_cache_parity.py) | Background fork字节级一致性测试（#25322） |
