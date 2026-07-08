# 03 - 工具系统 ToolRegistry
> 自注册、动态发现、多层安全的工具总线
> 学习日期：2026-07-02

---

## ToolRegistry 是什么？

ToolRegistry 是 Hermes 的**工具中央总线**——所有工具（内置工具、插件工具、MCP工具、技能工具）都注册在这里，Agent 通过它发现、校验、执行工具。

它解决三个核心问题：
1. **发现**：工具不用手动在列表里加，`@tool()` 装饰器自动注册
2. **安全**：多层检查（环境变量、check_fn、审批）决定工具是否可见/可执行
3. **动态**：工具可以热插拔、schema可以运行时生成，不用重启

---

## Q1: 工具发现与自注册机制

### AST 静态扫描 + import 副作用

核心在 `tools/registry.py` 的 `discover_tools()`：

```python
def discover_tools():
    """扫描所有tools目录下的.py文件，用AST找被@tool装饰的函数，然后import触发注册"""
    # 1. 静态扫描：不执行代码，用AST找所有 @tool() 装饰的函数
    for py_file in tools_dir.glob("*.py"):
        tree = ast.parse(py_file.read_text())
        for node in ast.walk(tree):
            if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef)):
                for dec in node.decorator_list:
                    if _is_tool_decorator(dec):
                        tool_modules.append(py_file.stem)
    
    # 2. import：触发 @tool() 装饰器执行，调用 registry.register()
    for mod in tool_modules:
        importlib.import_module(f"tools.{mod}")
```

**为什么用AST先扫一遍再import？**
- 避免import所有文件（有些工具依赖可选包，没装就import会报错）
- AST扫描是纯静态分析，不执行任何代码，安全快速
- 找到候选文件后才import，此时`@tool()`装饰器执行副作用完成注册

### @tool() 装饰器做了什么？

```python
def tool(name=None, schema=None, toolset=None, check_fn=None, 
         requires_env=None, is_async=False, max_result_size_chars=None,
         emoji=None, dynamic_schema_overrides=None):
    """工具注册装饰器"""
    def decorator(fn):
        # 1. 如果没传schema，自动从函数签名和docstring生成JSON Schema
        generated_schema = schema or _infer_schema_from_function(fn)
        
        # 2. 注册到全局registry
        registry.register(
            name=name or fn.__name__,
            handler=fn,
            schema=generated_schema,
            toolset=toolset or "core",
            check_fn=check_fn,
            requires_env=requires_env or [],
            is_async=is_async or asyncio.iscoroutinefunction(fn),
            dynamic_schema_overrides=dynamic_schema_overrides,
        )
        return fn
    return decorator
```

最简单的工具只需要：
```python
@tool(emoji="🔍")
def web_search(query: str) -> dict:
    """Search the web for information.
    
    Args:
        query: Search query string
    """
    return _search(query)
```
不需要手动在任何列表里加——import后自动注册。

### 多层安全检查栈

工具不是注册了就能用，从展示到执行会过多层检查：

| 层级 | 阶段 | 检查什么 | 作用 |
|------|------|---------|------|
| 1. **requires_env** | 展示时（get_definitions） | 需要的环境变量是否存在 | 没配API Key的工具直接隐藏 |
| 2. **check_fn** | 展示时 | 自定义回调返回True/False | 更灵活的能力检查（如平台限制） |
| 3. **toolset 门控** | 展示时 | 当前模型/模式是否启用该工具集 | vision工具集只给支持视觉的模型 |
| 4. **approval 审批** | 执行前 | 危险工具需要用户确认 | execute_command、write_file默认需要审批 |
| 5. **Tool Guardrails（运行时护栏）** | 执行中（before_call/after_call） | 检测工具调用死循环 | 重复失败、无进展重复读→警告/阻断，防止LLM无限循环烧钱 |

例子：
```python
@tool(
    requires_env=["GOOGLE_API_KEY"],  # 没配Key不显示
    check_fn=lambda: config.search_enabled,  # 配置关了不显示
    toolset="web",  # 可以整体禁用web工具集
)
def web_search(query: str) -> dict: ...
```

---

### Tool Guardrails：运行时死循环防护（第5层）

核心在 `agent/tool_guardrails.py`，是一个**纯函数无副作用**的控制器，追踪**每一回合（per-turn）**的工具调用模式。

#### 解决什么问题？

LLM在工具调用时容易陷入死循环：
- **完全相同的参数重复失败**：比如调terminal跑一个错命令，参数完全不变重试5次
- **同一个工具连续失败**：比如一直调web_search都失败，不换工具不换思路
- **幂等工具无进展重复读**：比如read_file同一个文件、参数完全相同、返回结果也完全相同，重复3次——信息已经拿到了，模型还在读

这些死循环会：烧API token、浪费用户时间、让Agent看起来"卡死了"。

#### 工具分类：幂等 vs 变更

护栏先把工具分成两类：

```python
IDEMPOTENT_TOOL_NAMES = frozenset({
    "read_file", "search_files", "web_search", "web_extract",
    "session_search", "browser_snapshot", ...  # 只读工具
})

MUTATING_TOOL_NAMES = frozenset({
    "terminal", "execute_code", "write_file", "patch", "todo",
    "memory", "skill_manage", "browser_click", ...  # 会改东西的工具
})
```

- **幂等工具**：只读不改，相同参数调用N次结果一样 → 可以检测"无进展重复"
- **变更工具**：会改变状态，相同参数可能结果不同（比如terminal多跑一次状态变了）→ 只检测"连续失败"，不检测无进展

#### 检测的3种循环模式

| 模式 | 触发阈值（默认warn/block） | 检测什么 |
|------|---------------------------|---------|
| **完全相同参数重复失败** | warn=2次 / block=5次 | 同一个工具+完全相同参数（sha256哈希canonical JSON）连续失败 |
| **同一工具连续失败** | warn=3次 / halt=8次 | 同一个工具（不管参数）连续失败N次 |
| **幂等工具无进展重复** | warn=2次 / block=5次 | 幂等工具+相同参数+返回结果哈希也相同（没新信息） |

参数哈希方式：`canonical_json = json.dumps(args, sort_keys=True, separators=(",",":"))` → sha256。参数顺序不同但内容相同算同一个调用。

结果哈希同理：canonical JSON → sha256，结果内容完全一样才算"没新进展"。

#### 决策动作：allow / warn / block / halt

控制器返回4种决策：
- **allow**：正常执行
- **warn**：执行，但在工具结果后面追加警告提示，告诉模型"你在循环了，换个思路"——不阻断，只是提醒
- **block**：执行前阻断，直接返回合成的错误结果，告诉模型"别再试了，这个调用已经完全相同失败N次了"
- **halt**：回合强制终止，模型必须停下来换策略

**关键设计**：
- **warn默认开启，hard_stop默认关闭**——CLI/TUI会话默认是"温和提醒"，不强制阻断；用户可以在config里开 `tool_loop_guardrails.hard_stop_enabled: true` 启用保险丝
- **per-turn追踪**：每轮用户新消息开始时 `reset_for_turn()` 重置计数——不跨回合累积
- **纯函数**：控制器只做计数和返回决策，不修改消息历史、不直接发警告——运行时代码（run_agent.py）决定怎么呈现决策
- **合成错误结果**：block时 `toolguard_synthetic_result()` 返回一个带guardrail元数据的JSON错误，模型看到就知道为什么被拦了
- **恢复提示**：警告信息不是只说"你循环了"，还给具体建议："先跑pwd && ls -la诊断、换绝对路径、换简单命令、用read_file代替terminal"

#### 调用时机：before_call / after_call

```
模型生成tool_call
    ↓
before_call(tool_name, args)  ← 执行前检查
    ├─ hard_stop开了？检查是否达到block阈值
    ├─ 幂等工具？检查是否无进展重复达到block阈值
    └─ allow → 执行工具
                    ↓
            工具执行，拿到result
                    ↓
after_call(tool_name, args, result, failed)  ← 执行后更新计数
    ├─ 失败了？增加exact_failure_count和same_tool_failure_count
    │   ├─ 达到warn阈值 → 返回warn（追加提示）
    │   └─ hard_stop开了且达到halt阈值 → 返回halt（终止回合）
    └─ 成功了？
        ├─ 幂等工具？计算结果哈希 → 和之前比 → 相同则增加no_progress计数
        │   └─ 达到warn阈值 → 返回warn
        └─ 变更工具？清除no_progress记录
```

---

### SKILL.md里的Guardrails章节：技能级安全约定

除了代码层面的工具调用护栏，**每个SKILL.md可以有自己的Guardrails章节**——这是技能作者写的操作安全指引，告诉模型使用这个技能时必须遵守的规则。

例如 `optional-skills/security/1password/SKILL.md` 里的Guardrails：

```markdown
## Guardrails

- Never print raw secrets back to user unless they explicitly request the value.
- Prefer `op run` / `op inject` instead of writing secrets into files.
- If command fails with "account is not signed in", run `op signin` again in the same tmux session.
- If desktop app integration is unavailable (headless/CI), use service account token flow.
```

这不是代码强制执行的——它是写在SKILL.md里的指令，模型加载技能后会看到并遵守。这种约定式护栏比代码硬编码更灵活：
- 不同技能有不同的安全规则（1password不能泄露密码、数据库操作要先备份、发邮件要二次确认）
- 不需要改框架代码，技能作者在SKILL.md里写就行
- 模型看到自然语言规则比看到程序式检查更能理解上下文

所以Hermes有两类Guardrails：
1. **框架级Guardrails**（`agent/tool_guardrails.py`）：代码硬编码，检测工具调用死循环，保护用户不被烧钱
2. **技能级Guardrails**（SKILL.md里的## Guardrails章节）：技能作者写的自然语言规则，指导模型安全操作特定工具/领域

---

## Q2: 什么是"动态schema（运行时配置驱动）"？

### 先说结论

**动态 Schema** 是指：工具给模型看的"函数说明书"（OpenAI function schema）**不是写死在代码里的静态常量**，而是在**每次要发给模型之前**，根据当前运行时配置、用户设置、已安装能力**动态生成**的。

简单说：**模型看到的工具说明，永远反映当前真实状态，而不是代码里写的默认值。**

### 为什么需要动态 Schema？—— 静态 Schema 的问题

如果 Schema 是写死的常量，会出现"模型被误导"的问题：

#### 例子1：delegate_task 的并发限制
- 默认并发上限 **3个子代理**，用户可配置成8
- 静态写"最多3个"，模型就只会开3个，用户改配置没用

#### 例子2：video_generate 视频生成
- 不同后端（Runway/Pika/本地）支持的分辨率、时长、模态不同
- 静态写"支持1080p/10s/图生视频"，但用户后端只支持720p/5s/文本，调用就报错

#### 例子3：MCP 服务器工具
- MCP工具是**运行时连接后才发现**的
- 不可能写死——用户随意配置MCP服务器

### Hermes 的实现机制

核心代码在 `tools/registry.py` 的 `ToolEntry` 和 `get_definitions()`：

#### 1. ToolEntry 增加 dynamic_schema_overrides 字段

```python
class ToolEntry:
    __slots__ = (
        "name", "toolset", "schema", "handler", "check_fn",
        "requires_env", "is_async", "description", "emoji",
        "max_result_size_chars", "dynamic_schema_overrides",  # ← 就是这个
    )

    def __init__(self, ..., dynamic_schema_overrides=None):
        # 可选的零参数回调函数，每次get_definitions()时调用
        # 返回一个dict，用来覆盖/补充静态schema
        self.dynamic_schema_overrides = dynamic_schema_overrides
```

- `schema`：写死的基础 Schema（静态模板）
- `dynamic_schema_overrides`：一个**无参回调函数**，每次生成工具列表给模型时调用

#### 2. get_definitions() 每次都调用动态回调

```python
def get_definitions(self, tool_names, ...):
    result = []
    for name in sorted(tool_names):
        entry = entries_by_name.get(name)
        # 四层安全检查...

        # 先拿静态schema
        schema_with_name = {**entry.schema, "name": entry.name}

        # 关键：如果有动态回调，每次都调用
        if entry.dynamic_schema_overrides is not None:
            try:
                overrides = entry.dynamic_schema_overrides()  # ← 每次重新算
                if isinstance(overrides, dict):
                    schema_with_name.update(overrides)  # ← 浅合并覆盖
            except Exception as exc:
                logger.warning("dynamic_schema_overrides for %s failed; using static", name)

        result.append({"type": "function", "function": schema_with_name})
    return result
```

关键点：
- **每次调用都重新执行回调**——不是启动时算一次，是每次给模型发工具列表都算
- 回调异常不崩溃：回退到静态 schema
- **浅合并**：overrides 的顶层key直接覆盖静态 schema

#### 3. 缓存联动：config变了自动失效

调用方用 **config.yaml 的修改时间（mtime）+ 文件大小** 作为缓存key：
- 用户改了config → key变了 → 缓存失效 → 重新调用get_definitions() → 重新执行回调
- 不需要重启，下一轮对话模型就看到新值

### 实际例子1：delegate_task 动态描述

```python
def _build_dynamic_schema_overrides() -> dict:
    max_children = _get_max_concurrent_children()  # 读config
    max_depth = _get_max_spawn_depth()
    orchestrator_on = _get_orchestrator_enabled()

    return {
        "description": f"Spawn isolated subagents, max {max_children} concurrent. "
                       f"Nesting: {'enabled (depth='+str(max_depth)+')' if max_depth>=2 else 'off'}",
    }

registry.register(
    name="delegate_task",
    schema=DELEGATE_TASK_SCHEMA,
    handler=_delegate_handler,
    dynamic_schema_overrides=_build_dynamic_schema_overrides,  # ← 传回调
)
```

**效果**：用户配置改了并发/嵌套限制，模型下一轮就知道了，不会尝试越界。

### 实际例子2：video_generate 按后端能力生成

```python
def _build_dynamic_video_schema() -> dict:
    provider = _read_configured_video_provider()
    if not provider:
        return {"description": "No video backend configured; calls will error. Suggest user run `hermes tools`."}
    
    caps = provider.capabilities()
    return {
        "description": f"Generate video. Active: {provider.name}, model: {caps.model}. "
                       f"Resolutions: {', '.join(caps.resolutions)}. "
                       f"Max duration: {caps.max_duration}s. "
                       f"Modalities: {'+'.join(caps.modalities)}."
    }
```

**效果**：模型总是知道当前后端真实支持什么，不会要求1080p而后端只支持720p。

### 实际例子3：MCP 动态注册（更彻底的动态）

MCP工具不止描述是动态的——**连工具本身都是运行时才注册的**：
- 连接MCP服务器后 `list_tools()` 拿到工具列表
- 动态 `registry.register()` 注册每个MCP工具
- 断开连接时 `registry.deregister()` 移除
- Registry的 `_generation` 版本号单调递增，调用方自动刷新缓存

### 对比静态 vs 动态

| 方面 | 静态 Schema | 动态 Schema |
|------|------------|------------|
| 定义时机 | 代码写死，import时确定 | 每次get_definitions()时生成 |
| 用户改配置 | 不生效，需重启 | 下一轮对话自动生效 |
| 外部能力变化 | 不知道，会乱调用 | 实时反映 |
| 异常处理 | 出错就报错 | 回调异常自动回退到静态 |
| 性能 | 0开销 | 一次无参函数调用（可忽略） |
| 缓存 | 永远缓存 | config mtime变了才重算 |

### 设计哲学

1. **模型是被"说明书"驱动的**——给它看什么，它就按什么来，说明书必须准
2. **不要信任静态默认值**——用户会改配置、环境会变、外部服务会掉线
3. **惰性计算**——描述不是启动时生成一次，是每次用之前算，保证新鲜
4. **优雅降级**——动态回调出错不能让工具消失，回退到静态

**核心思想：Schema 不是代码的一部分，而是"系统当前状态"对外的视图。**

---

## _generation 版本号：热插拔失效缓存

Registry内部维护一个单调递增的 `_generation` 计数器：
- 每次 `register()` / `deregister()` 时 +1
- 调用方（如model_tools.py）缓存工具定义时记录generation
- 如果发现generation变了，说明工具列表变了（MCP重连、插件热加载），立即刷新缓存

```python
class ToolRegistry:
    def __init__(self):
        self._generation = 0  # 单调递增版本号
    
    def register(self, ...):
        # ... 注册逻辑
        self._generation += 1
    
    def deregister(self, name):
        # ... 注销逻辑
        self._generation += 1
```

这就是为什么MCP工具可以运行时插拔不用重启——版本号变了，缓存自动失效。

---

## 关键源码位置

| 概念 | 文件位置 |
|------|---------|
| ToolRegistry 核心实现 | `tools/registry.py` |
| @tool() 装饰器 | `tools/registry.py` |
| AST静态发现工具 | `tools/registry.py` (discover_tools) |
| 多层安全检查（展示时） | `tools/registry.py` (get_definitions) |
| **Tool Guardrails运行时死循环防护** | **`agent/tool_guardrails.py`** |
| Guardrails在对话循环中集成 | `run_agent.py` / `agent/conversation_loop.py` |
| dynamic_schema_overrides 机制 | `tools/registry.py` (ToolEntry类) |
| _generation版本号热插拔 | `tools/registry.py` |
| delegate_task动态schema | `tools/delegate_tool.py` |
| video_generate动态schema | `tools/video_generation_tool.py` |
| MCP动态注册/注销 | `tools/mcp_tool.py` |
| 工具并发执行 | `agent/tool_executor.py` |
