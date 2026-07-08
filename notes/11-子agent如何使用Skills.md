# 11 - 子agent如何继承使用主agent的Skills
> Skill是磁盘上的Markdown文件，子agent通过继承skill_view工具，按需读取共享文件系统上的SKILL.md——和主agent完全一样，不需要复杂的复制注入
> 学习日期：2026-07-08

---

## 核心结论一句话

**子agent不需要主agent"注入"或"复制"skill，它直接继承了`skill_view`/`skills_list`/`skill_manage`三个工具，共享同一个`~/.hermes/skills/`磁盘目录，需要哪个SOP/知识时自己调用skill_view加载即可，和主agent的使用方式完全相同。**

---

## 一、工具继承机制

### 1.1 子agent工具集 = 父工具集 ∩ 非黑名单

子agent构建时（[delegate_tool.py#L1047-L1082](file:///e:/github/hermes-agent/tools/delegate_tool.py#L1047-L1082)）：

```python
# 第一步：继承父agent启用的toolsets
if toolsets:
    # 用户指定了toolsets → 和父工具集求交集（子agent不能拥有父没有的工具）
    expanded_parent = _expand_parent_toolsets(parent_toolsets)
    child_toolsets = [t for t in toolsets if t in expanded_parent]
else:
    # 用户没指定toolsets → 直接继承父的所有启用toolsets
    child_toolsets = _strip_blocked_tools(parent_enabled)

# 第二步：黑名单过滤掉危险工具
child_toolsets = _strip_blocked_tools(child_toolsets)
```

### 1.2 DELEGATE_BLOCKED_TOOLS 黑名单里没有skill相关工具！

**代码位置**：[delegate_tool.py#L45-L54](file:///e:/github/hermes-agent/tools/delegate_tool.py#L45-L54)

```python
# Tools that children must never have access to
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",  # no recursive delegation（leaf模式禁止递归委托）
    "clarify",        # no user interaction（不能直接问用户澄清）
    "memory",         # no writes to shared MEMORY.md（不能写共享记忆）
    "send_message",   # no cross-platform side effects（不能发消息）
    "execute_code",   # children should reason step-by-step, not write scripts
    "cronjob",        # no scheduling more work in the parent's name
])
```

✅ `skill_view`、`skills_list`、`skill_manage` **不在黑名单里**，完整继承给子agent！

---

## 二、skills toolset 包含哪些工具

**代码位置**：[toolsets.py#L164-L168](file:///e:/github/hermes-agent/toolsets.py#L164-L168)

```python
"skills": {
    "description": "Access, create, edit, and manage skill documents with specialized instructions and knowledge",
    "tools": ["skills_list", "skill_view", "skill_manage"],
    "includes": []
},
```

三个工具全部继承给子agent：

| 工具 | 作用 | 子agent能用吗？ |
|------|------|----------------|
| `skills_list` | 列出所有已安装的skill，支持搜索 | ✅ 能用 |
| `skill_view` | 读取指定SKILL.md完整内容，注入到当前上下文 | ✅ 能用 |
| `skill_manage` | 创建/编辑/安装/更新skill（管理类） | 🔶 默认能用（在黑名单外），但子agent一般不需要修改skill，主agent才会做 |

主agent默认启用的场景（coding/hermes-acp/hermes-cli）都包含`skills` toolset，所以子agent默认就能用：

| 主场景 | 包含skills toolset吗？ |
|--------|-----------------------|
| coding（代码工作区自动选择） | ✅ 包含（见toolsets.py#L351） |
| hermes-acp（编辑器集成） | ✅ 包含（见toolsets.py#L383） |
| hermes-cli（默认CLI） | ✅ 包含（见toolsets.py#L407） |
| hermes-api-server | ✅ 包含（见toolsets.py，L407附近） |

---

## 三、共享文件系统：不需要复制skill文件

Skills是**磁盘上的普通Markdown文件**，存放在：
- 内置skills：随Hermes发布的`skills/`目录
- 用户安装skills：`~/.hermes/skills/`目录
- 项目skills：`./.hermes/skills/`目录

子agent是同一个进程内的新AIAgent实例，**和主agent共享同一个文件系统**，直接读相同路径的文件，不需要复制、不需要网络传输：

```
主agent进程
├─ 主agent实例
│  └─ skill_view工具 → 读 ~/.hermes/skills/git-commit/SKILL.md
└─ 子agent实例（后台线程/独立运行）
   └─ skill_view工具 → 读 同一个 ~/.hermes/skills/git-commit/SKILL.md
                          ↑
                     磁盘文件是共享的，不需要复制！
```

---

## 四、子agent使用skill的完整流程

```
用户："帮我按照我们的commit规范提交这三个bugfix"
  ↓
主agent（模型）判断：这个任务需要按规范写commit message，比较重，可以派给子agent
  ↓
调用 delegate_task(goal="按照项目commit规范提交这三个bugfix")
  ↓
_build_child_agent()构建子agent：
  ├─ 继承toolsets（包含skills）
  ├─ 黑名单过滤后，skill_view/skills_list保留
  ├─ 子agent有独立的空会话历史
  └─ system prompt是精简版：告诉它任务目标是什么，工作区路径在哪
  ↓
子agent开始运行自己的ReAct循环：
  ├─ 第一步：思考 → "我需要先知道项目的commit规范是什么"
  ├─ 第二步：调用 skills_list(pattern="commit") → 找到git-commit skill
  ├─ 第三步：调用 skill_view(name="git-commit") → 读取SKILL.md内容
  │           ↓（和主agent完全一样的调用，读同一个文件）
  ├─ SKILL.md内容作为tool结果返回，子agent读到了完整的commit规范
  ├─ 第四步：按照规范，调用read_file看diff → 调用terminal执行git commit
  ├─ ... 多轮工具调用 ...
  └─ 完成，返回结果摘要给主agent
  ↓
主agent拿到子agent结果，汇总后回复用户："已经按照规范提交了三个bugfix..."
```

---

## 五、为什么不把skill内容直接预注入子agent的system prompt？

你可能会问：为什么不直接把子agent需要的skill提前读出来塞到子agent的system prompt里，要让子agent自己调用skill_view？

这是故意的设计，好处有三个：

### 1. 上下文效率：按需加载，不浪费token
- 如果预注入所有skill，子agentsystem prompt会带上几十上百个SKILL.md，几万token，子agent还没开始干活上下文就满了
- 按需调用skill_view：子agent需要哪个读哪个，不需要的不读，上下文最小化

### 2. 职责分离：主agent不需要判断子agent需要什么skill
- 主agent派任务时不需要知道"这个任务需要用到哪些skill"，这是子agent自己的事
- 子agent在执行过程中自己判断需要什么知识，自己加载，主agent只需要给goal就行
- 解耦，主agent不需要理解子任务的细节

### 3. 一致性：和主agent行为完全相同
- 主agent就是"需要时调用skill_view加载"的模式
- 子agent用完全相同的方式，不需要特殊处理，模型在主agent和子agent里的行为一致
- 代码不需要两套逻辑，维护简单

---

## 六、特殊设计：子agent的system prompt虽然精简，但不影响使用skill

子agent的system prompt（[delegate_tool.py#L683-L739](file:///e:/github/hermes-agent/tools/delegate_tool.py#L683-L739)）是精简的专用版本，和主agent的完整SOUL/system prompt不同：

```
You are a focused subagent working on a specific delegated task.

YOUR TASK:
{goal}

WORKSPACE PATH:
{path}

Complete this task using the tools available to you. When finished, 
provide a clear, concise summary...
```

它没有主agent那么多引导提示，但**这没关系**——因为：
- `skill_view`工具的schema描述里已经说明了"这个工具用来加载skill操作手册/SOP/领域知识"
- 模型经过function calling微调，看到工具名和描述就知道怎么用
- 子agent在执行过程中遇到需要规范/知识的地方，自然会想到调用skill_view，和主agent一样

---

## 七、Skill继承关系总结图

```
磁盘上共享的Skills文件系统
┌─────────────────────────────────────────────────┐
│ ~/.hermes/skills/                               │
│  ├─ git-commit/SKILL.md                         │
│  ├─ code-review/SKILL.md                        │
│  ├─ react-expert/SKILL.md                       │
│  └─ ... 所有已安装的skill...                     │
└───────────────────┬─────────────────────────────┘
                    │
                    │ 都能通过skill_view读取
                    │
        ┌───────────┴───────────┐
        │                       │
        ▼                       ▼
┌───────────────┐       ┌───────────────────────┐
│ 主agent       │       │ 子agent                │
│ ───────────   │       │ ─────────────────────  │
│ 有skill_view  │       │ 继承skill_view工具     │
│ 按需加载      │       │ 按需加载，和主agent一样 │
│ 可以编辑skill │       │ leaf模式下一般不需要编辑 │
└───────────────┘       └───────────────────────┘
        │                       │
        └───────────┬───────────┘
                    │
                    ▼
          模型自己判断什么时候需要哪个skill
          调用skill_view加载到上下文使用
```

---

## 八、关键代码索引

| 功能 | 文件位置 |
|------|---------|
| 子agent工具继承 | [delegate_tool.py#L1047-L1089](file:///e:/github/hermes-agent/tools/delegate_tool.py#L1047-L1089) |
| DELEGATE_BLOCKED_TOOLS黑名单 | [delegate_tool.py#L45-L54](file:///e:/github/hermes-agent/tools/delegate_tool.py#L45-L54) |
| skills toolset定义 | [toolsets.py#L164-L168](file:///e:/github/hermes-agent/toolsets.py#L164-L168) |
| skill_view工具实现 | [skills_tool.py](file:///e:/github/hermes-agent/tools/skills_tool.py) |
| 子agent构建 `_build_child_agent` | [delegate_tool.py#L991-L1325](file:///e:/github/hermes-agent/tools/delegate_tool.py#L991-L1325) |
| 子agent专用system prompt | [delegate_tool.py#L666-L739](file:///e:/github/hermes-agent/tools/delegate_tool.py#L666-L739) |

---

## 一句话总结

**子agent用skill的方式和主agent完全一样：继承skill_view工具，需要哪个SOP/知识时自己从共享磁盘读，不需要主agent提前注入或复制——上下文按需加载，效率最高，行为一致。**