# 07 - 架构选型：Tool vs Skill vs Subagent（子代理）
> 什么样的功能该做成什么？核心决策指南
> 学习日期：2026-07-08

---

## 一句话先懂本质

| 概念 | 类比 | 是什么 | 是否调用LLM |
|------|------|--------|------------|
| **Tool（工具）** | 🖐️ 手 | 一个确定性Python函数，执行动作 | ❌ 不调用，纯代码执行 |
| **Skill（技能）** | 📖 手册/操作指南 | 一段Markdown文档+参考资料，按需注入上下文 | ❌ 不调用，只是让模型"读说明书" |
| **Subagent（子代理）** | 👨‍💼 雇了一个新助手 | 一个全新启动的独立Agent实例，有自己的对话历史 | ✅ 会，独立多轮ReAct循环，独立消耗token |

---

## 一、核心维度对比表

| 维度 | Tool（工具） | Skill（技能） | Subagent（子代理/Delegate） |
|------|-------------|--------------|---------------------------|
| **本质** | 确定性函数调用 | Markdown指令文档+辅助资料 | 独立AIAgent实例 |
| **是否调用LLM** | ❌ 否（纯Python代码） | ❌ 否（仅文件读取注入） | ✅ 是（独立多轮LLM对话） |
| **上下文** | 父agent同一上下文 | 仅注入文档到父上下文 | ✅ 全新独立上下文，与父隔离 |
| **Token消耗** | 几乎为0（仅返回结果字符串） | 0（仅读取文件，注入文档占用父上下文） | 高，独立完整ReAct循环 |
| **实现形式** | `tools/xxx.py`，`registry.register()`注册 | `skills/<name>/SKILL.md` + references/templates | 本身就是名为`delegate_task`的Tool，内部启动新Agent |
| **粒度** | 原子操作，一次调用完成 | 知识/流程封装 | 任务级，可多轮推理可调用多个工具 |
| **并行支持** | 支持（ToolExecutor批量执行） | 不涉及并行 | ✅ 原生支持，可同时派多个子代理并行 |
| **后台运行** | 工具本身不支持（可以用fire-and-forget但需自己实现） | 不涉及 | ✅ `background=true`原生支持 |
| **递归限制** | 无（工具不能调用其他工具，模型决定调用） | 无（skill不能调用其他skill，模型决定加载） | ✅ 有深度限制，默认最多1层（orchestrator角色可放宽） |
| **危险工具黑名单** | 受父agent审批门控制 | 无（只是文档） | 强制黑名单：禁止递归委托、禁止写memory、禁止cron、禁止需要用户交互的操作 |

---

## 二、详细说明

### 2.1 Tool（工具）：手，执行确定性动作

#### 是什么？
Tool是最基础的能力单元——就是一个Python函数，注册到ToolRegistry后，模型通过function calling调用，输入参数，执行代码，返回结果字符串。

**核心机制**：
- 模块导入时通过`registry.register()`自注册
- 每个工具有JSON Schema（OpenAI function calling格式）描述参数
- 模型生成tool_call → ToolRegistry.dispatch分发 → 调用handler → 捕获异常 → 返回结果字符串
- 结果直接加入父agent的消息历史，继续循环

#### ✅ 适合做成Tool的场景
| 场景特征 | 例子 |
|---------|------|
| **确定性动作** | 读写文件、执行shell命令、发HTTP请求、查数据库 |
| **需要立即副作用** | 修改文件、发送消息、创建issue、调用API |
| **原子操作** | 一次调用完成，不需要多步推理 |
| **输入输出明确** | 参数固定，返回结果结构清晰 |
| **低延迟、高频使用** | 读文件、列目录、搜索这类毫秒级操作 |

#### Hermes内置典型工具
- 文件操作：`read_file`、`write_file`、`search_files`
- 终端：`run_terminal_cmd`
- 网络：`web_search`、`web_fetch`、浏览器工具
- 系统：`memory_save`、`todo_write`、`cronjob`
- 元工具：`delegate_task`（子代理本身也是工具！）、`skill_view`（加载技能本身也是工具！）

---

### 2.2 Skill（技能）：手册，注入程序性知识

#### 是什么？
Skill **不是可执行代码**！它是"渐进式知识披露"系统——把专家知识、SOP流程、参考文档、模板等写在Markdown里，模型需要的时候再加载阅读。

**核心三层渐进披露**：
| 层级 | 内容 | 何时加载 | Token占用 |
|------|------|---------|----------|
| Tier 1 | 技能名 + 一句话描述 | 启动时注入system prompt | 极低，每个技能几十个token |
| Tier 2 | SKILL.md完整内容 | 模型主动调用`skill_view(name)`或用户输入`/skillname` | 几k token，按需加载 |
| Tier 3 | references/、templates/下的附件 | 模型进一步调用`skill_view(name, file_path="xxx")` | 按需，用多少加载多少 |

模型读完SKILL.md后，**仍然用普通的Tool来执行工作**——Skill只是给了它一份详细说明书，它本身不执行任何代码。

#### ✅ 适合做成Skill的场景
| 场景特征 | 例子 |
|---------|------|
| **领域知识/SOP流程** | "如何做符合团队规范的代码审查"、"如何写commit message"、"如何部署到Vercel" |
| **可复用的提示词工作流** | 把常用的复杂prompt模式固化，`/commit`直接生成规范提交信息 |
| **模板和参考文档** | PR模板、API文档摘要、格式规范 |
| **跨会话知识持久化**（比memory更结构化） | system prompt明确说："learned workflows belong in skills, not memory" |
| **不需要新代码，靠正确提示就能做好的事** | 比如"写夏朝历史网页"这种，模型本身有能力做，只是需要明确的风格/结构规范 |

#### ❌ 不适合做Skill的
- 需要实际执行代码/动作 → 这是Tool
- 需要大量独立推理/搜索/试错 → 这是Subagent
- 需要动态计算/实时数据 → 这是Tool

---

### 2.3 Subagent（子代理）：新助手，独立完成子任务

#### 是什么？
每次调用`delegate_task`，Hermes都会：
1. **构建一个全新的AIAgent实例**：全新对话历史、独立task_id（独立终端/文件缓存）
2. **裁剪工具集**：只给需要的toolsets，强制黑名单禁止危险操作
3. **独立运行完整ReAct循环**：自己思考→自己调用工具→自己迭代
4. **只返回最终摘要给父agent**：父看不到中间过程，只拿到结果总结

两种执行模式：
- **同步（默认）**：父阻塞等待，支持并行派多个子代理同时跑
- **后台（background=true）**：立即返回，子代理在daemon线程跑，完成后结果自动作为新消息注入，不阻塞用户继续聊天

#### ✅ 适合做成Subagent的场景
| 场景特征 | 为什么不能用Tool/Skill？ | 例子 |
|---------|------------------------|------|
| **子任务需要多轮推理、会产生大量中间结果** | 父上下文放不下，或者中间过程不需要父看到 | "深度审查这5个文件"、"调研最近3天arxiv的量化论文" |
| **任务可以拆成多个独立子问题并行** | Tool只能做原子操作，Skill不能并行 | "同时研究夏朝和商朝的历史，都写成网页" |
| **子任务适合用不同模型** | 可以用便宜快模型跑简单搜索子任务，父用强模型汇总 | 搜索用Haiku/GPT-4o-mini，汇总用Sonnet/GPT-4o |
| **长时间后台任务，用户不等结果** | Tool/Skill都是同步的，会阻塞对话 | "帮我监控这个issue，有更新告诉我"、"后台生成商朝历史网页" |
| **子任务需要独立试错/探索** | 父上下文不需要被试错过程污染 | "帮我调试这个bug，试各种方法，修好了告诉我" |

#### ❌ Hermes代码明确说"不要委托"的反模式
来自`_build_child_system_prompt()`对orchestrator角色的提示：
1. **单步机械工作** → 父直接用Tool做，不要雇新助手
2. **一两步工具调用能搞定的小事** → 直接做，启动子代理开销太大
3. **原封不动把整个任务甩给一个子代理（pass-through）** → 父没有附加价值，不如直接做
4. **递归委托子代理再委托子代理** → 默认禁止，深度最多1层，防止无限fork爆炸

---

## 三、决策树：来了一个需求，该选哪个？

按顺序问自己这几个问题：

```
用户需求来了
│
├─ ❶ 这是一个确定性的动作（读/写/执行/API请求/计算）？
│   └─ ✅ → 做成 Tool
│
├─ ❷ 这是"如何做X"的程序性知识/SOP/模板/规范，
│   │  模型本身有能力做，只是需要明确指引？
│   └─ ✅ → 做成 Skill
│
├─ ❸ 这个任务需要多轮推理/搜索/试错？
│   │  中间过程会产生大量上下文，父不需要看？
│   ├─ 是 → ✅ 用 Subagent（delegate_task）
│   └─ 否 → 父直接用Tool做就行
│
├─ ❹ 这个任务能拆成2+个独立子问题，可以并行跑？
│   └─ ✅ → 用多个 Subagent 并行
│
├─ ❺ 这是个耗时任务，用户不需要等结果，完成了再通知就行？
│   └─ ✅ → 用 Subagent background=true（异步后台）
│
└─ ❻ 想把某个常用工作流固化成一键命令？
    └─ ✅ → 写成 Skill（/skillname一键加载）
```

---

## 四、典型例子对照

| 需求 | 选什么 | 为什么 |
|------|--------|--------|
| 读一个文件的内容 | Tool | 确定性原子操作 |
| 执行npm install安装依赖 | Tool | 确定性命令执行 |
| "团队的commit message规范是什么" | Skill | 知识/SOP，模型读完按规范写 |
| "按照我们的代码审查规范，审查这个PR" | Skill + Tool | Skill给审查规范，模型读了后调用文件/diff工具执行 |
| "搜一下最近3天的AI论文，整理成摘要" | Subagent | 需要多轮搜索→过滤→整理，中间过程多，独立上下文跑最合适 |
| "同时生成夏朝和商朝的历史网页" | 多个Subagent并行 | 两个任务独立，并行跑更快，互不干扰 |
| "后台跑着生成商朝网页，我继续问你别的问题" | Subagent background=true | 不阻塞主对话，完成自动通知 |
| "帮我写一个计算斐波那契的Python函数" | 直接生成，什么都不用加 | 模型本身就能做，不需要额外工具/技能/子代理 |
| "怎么部署静态网页到Vercel" | Skill | 流程性知识，一步步指引，按指引执行终端/git命令 |
| "帮我调试这个编译错误，试各种方法直到修好" | Subagent | 需要多轮试错，每轮改代码→编译→看错误→再改，独立上下文跑更干净 |

---

## 五、三者可以组合使用！

不是互斥的，可以组合：

| 组合模式 | 说明 |
|---------|------|
| **Skill + Tool**（最常见） | Skill给SOP/规范/模板，模型读了后调用各种Tool完成工作。大部分技能都是这个模式 |
| **Subagent + 特定Tool集** | 子代理只给它需要的工具（比如搜索子代理只给web_search/web_fetch），不要给全套工具，省token也更安全 |
| **Subagent + Skill** | 子代理启动时可以指定加载特定Skill，给子代理专门的工作指引 |
| **父Orchestrator + 多个Subagent并行** | 父agent不做具体工作，只负责拆任务→派子代理→汇总结果。适合大型复杂任务 |
| **Subagent + background** | 后台子代理跑完→结果回流→父agent可以继续用Tool/Skill处理结果 |

---

## 六、常见误区与反模式

### ❌ 误区1：什么都做成Tool
- 症状：把"如何写代码审查"做成一个Tool，里面硬编码一堆prompt逻辑
- 问题：不需要新代码！写个Skill（SKILL.md）就搞定了，不用写Python，不用注册，不用重启，用户都能自己改
- 修正：程序性知识/流程规范 → Skill

### ❌ 误区2：什么都做成Subagent
- 症状：读个文件、执行个ls都delegate_task给子代理
- 问题：启动子代理开销大（新agent初始化+多轮LLM调用），响应慢，token成本高
- 修正：一两步能搞定的原子操作 → 直接用Tool

### ❌ 误区3：Skill里写可执行代码逻辑
- 症状：在SKILL.md里写复杂的条件逻辑、"如果X就调用Y，否则调用Z"
- 问题：Skill只是文档，模型可能不严格遵守，而且动态逻辑应该在Tool里
- 修正：确定性的逻辑/计算/动作 → Tool；知识/流程/模板 → Skill

### ❌ 误区4：Subagent递归无限嵌套
- 症状：子代理又派子代理又派子代理...
- 问题：token成本爆炸，上下文不透明，调试困难，容易失控
- 修正：Hermes默认深度限制1层（leaf角色不能再派子代理），orchestrator最多2-3层，不要改大

### ❌ 误区5：Skill太大，一次加载几百k参考文档
- 症状：把整个API文档都塞进SKILL.md
- 问题：浪费token，模型读不完
- 修正：用渐进披露，主SKILL.md只写核心流程，参考文档放references/下，让模型按需调用skill_view加载

---

## 七、设计哲学一句话总结

- **Tool解决"怎么做"（How）的问题**：提供动作能力
- **Skill解决"做什么、按什么标准做"（What/Standard）的问题**：提供知识和流程规范
- **Subagent解决"谁来做、并行做、隔离做"（Who/Parallel/Isolation）的问题**：提供任务级隔离和并行能力

**能用Tool直接做的不要开Subagent；靠提示就能做好的不要写Tool代码；知识流程放Skill，动作执行放Tool，独立重任务放Subagent。**

---

## 关键源码位置

| 概念 | 文件 |
|------|------|
| Tool注册与分发 | [tools/registry.py](file:///e:/github/hermes-agent/tools/registry.py) |
| Skill加载工具 | [tools/skills_tool.py](file:///e:/github/hermes-agent/tools/skills_tool.py) |
| Skill系统提示构建 | [agent/prompt_builder.py#L1354](file:///e:/github/hermes-agent/agent/prompt_builder.py#L1354) |
| Subagent主实现 | [tools/delegate_tool.py](file:///e:/github/hermes-agent/tools/delegate_tool.py) |
| 异步后台Subagent | [tools/async_delegation.py](file:///e:/github/hermes-agent/tools/async_delegation.py) |
| Subagent工具黑名单 | [delegate_tool.py#L45-L54](file:///e:/github/hermes-agent/tools/delegate_tool.py#L45-L54) |
| 内置Skills示例 | [skills/](file:///e:/github/hermes-agent/skills/) |
