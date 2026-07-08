# 04 - 技能系统 Skills
> 渐进式披露 + 自学习闭环
> 学习日期：2026-07-02

---

## 什么是"渐进式披露"（Progressive Disclosure）？

### 问题：如果把所有技能内容都放进System Prompt

假设你有50个技能，每个技能SKILL.md平均500字：
- System Prompt 直接塞25,000字 → **每轮对话都要发送这25,000字**
- 大多数技能当前任务完全用不上（比如你在写代码，不需要"做PPT"、"写邮件"的技能）
- 成本爆炸、延迟高、模型注意力被稀释，真正有用的技能反而被忽略

### 渐进式披露的核心思想

**不要一次性把所有内容都给模型看，而是分层：**

```
第1层（System Prompt里，每次都发）：
  → 只放【技能索引】—— 技能名字 + 一句话简介（约50字/技能）
  → 50个技能总共约2500字，非常轻量

第2层（按需加载，用户用skill_view加载时才发）：
  → 完整的SKILL.md内容（步骤、命令、注意事项、模板...）
  → 通常500-2000字，只在需要时才占token
  → 加载后以user消息注入（不破坏system prompt缓存）
```

就像看书：目录在书的开头（每翻一页都能看到），但具体章节内容你翻到那一页才读。

---

## Hermes 的三层渐进式披露实现

核心在 `agent/prompt_builder.py` 的 `build_skills_system_prompt()`。

### 第1层：System Prompt 中的技能索引（轻量目录）

每次对话都发的，只有**名字+一句话描述**，按分类组织：

```markdown
## Skills (mandatory)
Before replying, scan the skills below. If a skill matches or is even partially relevant 
to your task, you MUST load it with skill_view(name) and follow its instructions.

<available_skills>
  coding:
    - code-review: 代码审查规范与检查清单
    - debug: 系统化调试工作流
    - test: 编写和运行测试
  productivity:
    - ppt-maker: 生成PPT演示文稿
    - email-writer: 专业邮件写作
  hermes:
    - hermes-agent: Hermes自身配置与故障排查
  [names only]: social-media, entertainment...  （不在当前上下文的分类只列名字）
</available_skills>

Only proceed without loading a skill if genuinely none are relevant to the task.
```

**特点**：
- 只有名字和一句话描述，50个技能≈2500token
- 按分类分组（coding/productivity/hermes/...）
- **分类描述**：每个分类下的DESCRIPTION.md提供分类简介
- 无关分类降级为"[names only]"——只列名字，不写描述，进一步压缩
- 明确指示模型："只要相关就必须用skill_view加载"

### 第2层：skill_view 按需加载完整内容

模型看到索引后，如果判断某个技能和当前任务相关，调用 `skill_view(name="code-review")` 工具：

```python
@tool(emoji="📖")
def skill_view(name: str) -> dict:
    """Load a skill's full SKILL.md content into context.
    Use this when a skill in the index appears relevant to the current task.
    """
    skill_dir = find_skill_dir(name)
    content = (skill_dir / "SKILL.md").read_text()
    content = preprocess_skill_content(content, skill_dir)
    
    # 以user消息形式注入对话历史——不修改System Prompt！
    return {
        "output": f"[Loaded skill: {name}]\n\n{content}",
        # 标记这是技能内容，让框架注入到正确位置
        "_skill_injection": True,
    }
```

**关键点**：技能内容以**user消息**注入，不是system消息——这样**不会破坏System Prompt前缀缓存**（第02篇讲的字节级缓存依然生效！）

### 第3层：references/ 支持目录（二次渐进）

技能目录结构：
```
skills/code-review/
├── SKILL.md              ← 第2层：主内容（500字，步骤+核心要点）
├── references/           ← 第3层：参考资料（不自动加载）
│   ├── security-checklist.md
│   ├── performance-patterns.md
│   └── language-specific/
│       ├── python.md
│       └── typescript.md
├── templates/            ← 模板文件（不自动加载）
├── scripts/              ← 辅助脚本（不自动加载）
└── assets/               ← 图片等资源（不自动加载）
```

SKILL.md里会告诉模型："关于安全检查的详细清单见references/security-checklist.md，需要时用read_file读取"。

这样连SKILL.md本身也保持精简——最核心的步骤放在主文件，更深入的参考资料按需二次读取。

---

## 过滤与裁剪：索引不只是列名字

技能索引不是简单罗列所有文件，而是经过**多层智能过滤**，只给模型看当前真正能用/有用的技能：

### 1. 平台过滤（platforms）

技能在YAML frontmatter声明支持的平台：
```yaml
---
name: ios-deploy
description: 部署到iOS App Store
platforms: [macos]  # 只在macOS上显示，Windows/Linux用户看不到
---
```
`skill_matches_platform()` 检查当前OS，不兼容的直接跳过。

### 2. 环境过滤（environments）

```yaml
---
name: s6-service-management
description: s6容器服务管理
environments: [s6, docker]  # 只在Docker容器/s6环境里显示
---
```
`skill_matches_environment()` 检测当前运行环境（kanban/docker/s6等），不相关的隐藏。

> 注意：这只是**展示时过滤**——用户显式调用 `skill_view(name)` 或 `--skills` 预加载时，就算不匹配也照样加载。显式请求=明确同意。

### 3. 工具依赖过滤（requires/fallback_for）

技能可以声明"需要什么工具才有用"或"作为什么工具的后备"：
```yaml
---
name: web-search-fallback
description: 无浏览器时的网页搜索后备方案
conditions:
  fallback_for_tools: [browser_navigate]  # 有浏览器时隐藏这个
  requires_tools: [web_search]            # 没web_search工具时也隐藏
---
```
`_skill_should_show()` 根据当前可用工具集判断：
- `fallback_for`：如果主工具可用，后备技能隐藏（避免冗余）
- `requires`：如果依赖的工具不可用，技能也隐藏（加载了也用不了）

### 4. 姿态驱动分类降级（compact_categories）

当前上下文专注写代码时（coding posture），非编程类技能（社交、娱乐、生活类）整个分类降级为"[names only]"——只保留名字，去掉描述，减少噪音：

```python
if category in demoted:
    # 只列名字，不写描述
    index_lines.append(f"  {category} [names only]: {', '.join(names)}")
    continue
# 正常分类：名字+描述
for name, desc in skills:
    index_lines.append(f"    - {name}: {desc}")
```

注释特别强调：**NEVER remove entries entirely**——永远不彻底隐藏。模型可能凭记忆想起"有个做PPT的技能"，名字还在就能加载；如果名字都删了，模型就彻底不知道有这个技能了。

### 5. 禁用列表（disabled）

用户可以在config里禁用某些技能，禁用的不出现在索引里（但显式加载仍可用）。

---

## 两级缓存：索引构建极快

`build_skills_system_prompt()` 有两层缓存，避免每次都扫磁盘：

| 缓存层 | 位置 | key | 失效条件 |
|--------|------|-----|---------|
| L1 | 进程内LRU字典 | (skills_dir, tools, toolsets, platform, disabled, compact_cats) | 进程重启/参数变了 |
| L2 | 磁盘快照 `.skills_prompt_snapshot.json` | 所有SKILL.md的mtime+size manifest | 技能文件增删改 |

冷启动第一次才扫整个skills目录解析frontmatter，之后直接用快照，毫秒级返回。

外部技能目录（skills.external_dirs）不做磁盘快照（只读且通常小），但也走L1缓存。

---

## 渐进式披露的好处总结

| 方面 | 全量塞入System Prompt | 渐进式披露 |
|------|---------------------|-----------|
| **System Prompt大小** | 25,000+ token | ~2,500 token（10倍小） |
| **每轮成本** | 全部按输入计费 | 只有索引发送（省90%） |
| **前缀缓存命中** | 加一个技能/改一个技能全失效 | System Prompt里只有索引，稳定不变 |
| **模型注意力** | 被无关技能稀释，找不到真正需要的 | 分类清晰，当前相关才加载 |
| **响应速度（TTFT）** | 长prompt处理慢 | 短prompt，首个token快 |
| **技能可扩展性** | 加技能成本线性增长，不敢多加 | 加技能几乎零成本（索引多一行） |
| **内容深度** | 塞不下太多细节，只能写精简版 | SKILL.md可以写几千字详细步骤，按需加载 |
| **上下文切换** | 不相关技能总是在，干扰当前任务 | 只有相关技能被加载进上下文 |

### 设计哲学

1. **Offer vs Load 分离**：索引是"橱窗"（Offer），告诉模型有什么；`skill_view`是"取货"（Load），真正拿进上下文
2. **永远不彻底隐藏**：即便是降级分类也保留名字，模型的记忆锚点不丢失
3. **展示时严格过滤，加载时宽松放行**：显式请求永远优先于自动过滤
4. **User消息注入**：加载的技能内容走user消息，不动System Prompt，保护前缀缓存
5. **两级缓存**：磁盘快照+进程LRU，构建索引几乎零开销
6. **二次渐进**：SKILL.md本身又分层，主内容精简，参考资料放references/按需读

**一句话总结：给模型看书的方式不是把整本书复印贴在墙上，而是给它一个目录，它需要哪章自己翻。**

---

## Curator：后台技能维护编排器（自学习闭环的核心）

Curator 是技能系统"自学习闭环"的关键组件——它让Hermes在**空闲时自动整理技能库**，不需要用户手动维护。

### Curator 是什么？

Curator是一个**空闲触发的后台任务**（不是cron守护进程）：
- 当Agent空闲超过一定时间（默认2小时）
- 且距离上次Curator运行超过间隔（默认7天）
- 它会fork一个独立的AIAgent（用辅助模型/auxiliary client，不影响主会话）
- 在后台审查Agent自己创建的技能，做整理、归档、合并
- 主会话完全无感知，不打断用户

### 核心职责

1. **自动生命周期状态转换**（确定性逻辑，不需要LLM）：基于技能最后活动时间戳，自动在active/stale/archived之间转换
2. **LLM驱动的合并整理**（伞状构建/umbrella-building）：识别冗余的窄技能，合并成更通用的"类级别伞技能"
3. **状态持久化**：运行状态、最后运行时间、报告都存在 `.curator_state` 和报告文件里

### 严格不变量（铁则）

Curator遵守4条不可违反的规则：
1. **只碰Agent创建的技能**：内置技能（bundled）、Hub安装的技能、外部目录技能一律不碰（除非显式开了prune_builtins）
2. **永不删除，只归档**：最大破坏性操作是移动到 `.archive/` 目录，归档可恢复
3. **Pinned技能跳过所有自动转换**：用户钉住的技能永远不动
4. **使用辅助客户端**：用单独的auxiliary模型/凭证，不触碰主会话的prompt缓存

---

### Curator 的完整调用流程

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. 触发时机：maybe_run_curator()                                │
│    入口：会话结束/turn结束时被调用                                │
│    检查3个门控：                                                │
│    ├─ curator.enabled == true （默认开启）                      │
│    ├─ not paused                                               │
│    ├─ last_run_at + interval_hours < now（默认7天）             │
│    └─ Agent已经空闲了 min_idle_hours（默认2小时）                │
│    不满足则直接返回，不运行                                      │
│    ※ 全新安装不会立即运行——会先seed last_run_at=now，等一个完整间隔│
└───────────────────────────┬─────────────────────────────────────┘
                            │ 门控通过
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 前置：自动备份（pre-run snapshot）                            │
│    curator_backup.snapshot_skills() 把当前skills目录打快照       │
│    备份失败不阻断运行（debug日志记录，继续）                      │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 阶段一：确定性状态转换（apply_automatic_transitions）         │
│    纯Python逻辑，不需要LLM，零成本                               │
│                                                                 │
│    遍历所有Agent创建的技能（skill_usage.agent_created_report()）│
│    跳过pinned的技能                                              │
│                                                                 │
│    根据last_activity_at时间戳判断：                              │
│    ├─ > stale_cutoff（默认30天未活动）→ STATE_STALE（标记过期）  │
│    ├─ > archive_cutoff（默认90天未活动）→ STATE_ARCHIVED（归档） │
│    └─ stale技能重新被使用 → STATE_ACTIVE（重新激活）             │
│                                                                 │
│    新发现的符合条件技能：seed_record_if_missing，锚定时钟从现在开始│
│    不会第一次运行就归档——避免刚升级就把老技能全清了                │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 更新.curator_state                                          │
│    记录last_run_at、run_count+1、自动转换摘要                    │
│    （dry-run模式不更新last_run_at/run_count，预览不影响调度）     │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 阶段二：LLM审查合并（_llm_pass，consolidate开启时才运行）     │
│    默认consolidate=false！默认只做确定性归档，不花LLM钱           │
│    需要在config里设 curator.consolidate: true 才启用             │
│    或者手动 hermes curator run --consolidate                    │
│                                                                 │
│    执行方式：                                                    │
│    ├─ 默认异步：spawn一个daemon thread，主调用立即返回           │
│    └─ synchronous=true时在当前线程运行（CLI命令用这个）          │
│                                                                 │
│    Fork一个独立的AIAgent实例（auxiliary client）：               │
│    ├─ 用独立模型/API Key（auxiliary provider配置）              │
│    ├─ 自己独立的消息历史，和主会话完全隔离                        │
│    ├─ 有skills_list/skill_view/skill_manage/terminal工具        │
│    └─ 不会污染主会话的任何状态                                    │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. LLM审查Prompt（CURATOR_REVIEW_PROMPT）                       │
│    给fork出的审查Agent的系统指令核心思想：                        │
│                                                                 │
│    目标：构建"类级别伞技能库"，而不是"每次会话一个窄技能"         │
│    （几百个一个bug一个技能的库是失败的库）                        │
│                                                                 │
│    工作方法：                                                    │
│    1. 扫描候选列表，找前缀聚类（hermes-config-*, gateway-*,      │
│       mcp-*, python-*, pr-* 等，预计10-25个cluster）            │
│    2. 对每个≥2个技能的cluster，问：                               │
│       "人类维护者会把这写成N个独立技能，还是1个伞技能+N小节？"    │
│    3. 三种合并方式选一种：                                        │
│       a. 合并到已有伞技能：patch现有umbrella加新小节，归档窄技能  │
│       b. 创建新伞技能：skill_manage create新class-level技能，归档│
│       c. 降级到references/templates/scripts：把窄技能内容移到    │
│          伞技能的支持目录，再归档窄技能                            │
│                                                                 │
│    硬规则（不能违反）：                                          │
│    - 不碰内置/Hub/外部技能                                       │
│    - 不删除，只归档                                              │
│    - 不碰pinned技能                                              │
│    - 不碰plan等受保护的内置技能（UX入口点）                       │
│    - 不能只看use_count=0就跳过——按内容判断重叠，不是按使用次数    │
│    - 包完整性：移动技能时要把references/templates/scripts/assets │
│      一起搬，不能留下断链                                         │
│                                                                 │
│    输出要求：                                                    │
│    - 人类可读摘要                                                │
│    - 结构化YAML块（consolidations列表 + prunings列表）            │
│    - 包含原因（不只是"similar"，要说明为什么合并）                │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. Dry Run模式（预览）                                           │
│    hermes curator run --dry-run                                 │
│                                                                 │
│    - 跳过自动状态转换（不标记stale/不归档）                       │
│    - LLM审查收到DRY-RUN BANNER指令                              │
│    - LLM只读，不能调用skill_manage patch/create/delete/terminal  │
│    - 仍然写REPORT.md，告诉你它会做什么                            │
│    - 不更新last_run_at/run_count——预览不会推迟下一次正式运行      │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 8. 产出：运行报告 + 状态更新                                     │
│                                                                 │
│    运行结束后：                                                  │
│    ├─ 写REPORT.md到curator报告目录                              │
│    │ （包含自动变更统计、LLM合并/归档列表、结构化YAML）           │
│    ├─ 更新state.last_report_path 指向报告路径                    │
│    ├─ 记录last_run_duration_seconds                             │
│    ├─ 更新last_run_summary                                      │
│    └─ 下次用户打开新会话时，如果有新报告且没看过，显示提示         │
│       （"curator: 3 archived, 5 consolidated — see report"）    │
└─────────────────────────────────────────────────────────────────┘
```

---

### 输入与产出

#### 输入（Curator读什么）

| 输入 | 来源 | 说明 |
|------|------|------|
| Agent创建的技能列表 | `skill_usage.agent_created_report()` | 所有由skill_manage/create产生的本地技能，含状态、活动时间、使用次数 |
| 技能元数据 | 每个SKILL.md的YAML frontmatter | name/description/state/pinned/created_at等 |
| 配置参数 | `config.yaml` 的 `curator.*` 段 | interval/min_idle/stale_after/archive_after/consolidate等 |
| 持久化状态 | `~/.hermes/skills/.curator_state` | last_run_at/paused/run_count/last_report_path |
| 使用统计 | skill_usage追踪 | view_count/use_count/patch_count/last_activity_at |
| 备份快照 | curator_backup | 运行前自动备份，可回滚 |

#### 产出（Curator写什么）

| 产出 | 位置 | 说明 |
|------|------|------|
| 状态变更（自动） | 技能元数据 | active→stale→archived 状态转换，reactivated |
| 归档技能 | `~/.hermes/skills/.archive/<name>/` | 移动整个技能目录（含references等所有文件），可恢复 |
| 合并后的伞技能 | 现有技能SKILL.md | patch新增小节，或create新的class-level技能 |
| 降级内容 | `umbrella技能目录/references/` `templates/` `scripts/` | 窄技能的具体内容移到支持目录 |
| 运行报告 | curator报告目录 | REPORT.md（人类可读）+ run.json（机器可读），含diff和YAML结构化摘要 |
| 运行状态 | `.curator_state` | last_run_at/run_count/last_summary/last_report_path |
| 备份快照 | curator备份目录 | 运行前的完整快照，出问题可回滚 |
| 用户提示 | 新会话开头 | 上次curator运行后首次开新会话，显示简短摘要告知变更 |

---

### 默认配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `curator.enabled` | `true` | 默认开启 |
| `curator.interval_hours` | `168`（7天） | 两次运行最小间隔 |
| `curator.min_idle_hours` | `2` | Agent至少空闲多久才触发 |
| `curator.stale_after_days` | `30` | 30天未活动标记为stale |
| `curator.archive_after_days` | `90` | 90天未活动自动归档 |
| `curator.prune_builtins` | `true` | 是否也归档长期不用的内置技能 |
| `curator.consolidate` | `false` | 是否启用LLM伞状合并（默认关，省LLM钱） |

---

### 为什么Curator设计成"空闲触发"而不是cron？

1. **不抢资源**：用户正在用Agent的时候绝不跑后台整理，避免和主会话抢模型API配额、抢CPU、抢网络
2. **不需要额外守护进程**：没有常驻cron daemon，Gateway多实例下也不会重复跑（状态文件记录了last_run_at）
3. **自然节奏**：用户用了一周攒了一堆技能，Agent空闲时整理一次，节奏刚好
4. **可手动触发**：用户想立即整理可以 `hermes curator run`，想预览用 `--dry-run`

---

### CLI命令

```bash
hermes curator status          # 查看当前状态、上次运行、配置参数
hermes curator run             # 立即运行一次（含自动归档，consolidate按config）
hermes curator run --dry-run   # 预览模式，只读不修改
hermes curator run --consolidate  # 强制启用LLM合并，忽略config
hermes curator pause           # 暂停自动运行
hermes curator resume          # 恢复自动运行
```

---

## 知识如何沉淀为技能：自学习闭环

知识沉淀不是一个独立的工具，而是**嵌入在System Prompt指令 + skill_manage工具 + 使用追踪 + Curator后台整理**这四个环节组成的完整闭环。

### 沉淀触发：System Prompt里的明确指令

在技能索引的提示词里，Hermes被明确指示：

```
After difficult/iterative tasks, offer to save as a skill.
If a skill you loaded was missing steps, had wrong commands, or needed
pitfalls you discovered, update it before finishing.
```

翻译过来就是两条规则：
1. **完成复杂/迭代任务后**，主动提议把工作流保存为技能
2. **加载技能使用时发现问题**（缺步骤、命令错了、踩了坑），**结束前必须更新**它

这不是强制自动保存——Agent会先询问用户"要不要把这次的方法存成技能下次用？"用户同意后才写。

### 沉淀工具：skill_manage 的6个动作

Agent通过调用 `skill_manage` 工具来操作技能文件，支持6种action：

| action | 作用 | 用途 |
|--------|------|------|
| `create` | 创建新技能目录 + SKILL.md（含YAML frontmatter） | 沉淀全新的工作流 |
| `edit` | 完全重写用户技能的SKILL.md | 大改版 |
| `patch` | 在SKILL.md或支持文件里做定向find-replace | 补步骤、修命令、加踩坑记录 |
| `delete` | 删除/归档技能（移动到.archive/，不物理删除） | 废弃技能（Curator用） |
| `write_file` | 在技能目录下添加/覆盖支持文件（references/templates/scripts/assets） | 沉淀具体模板、参考资料、脚本 |
| `remove_file` | 删除支持文件 | 清理无用附件 |

创建新技能时会生成标准目录结构：
```
~/.hermes/skills/<category>/<skill-name>/
├── SKILL.md              # 主文件：YAML frontmatter + 步骤说明
├── references/           # 参考资料（具体细节、API文档、踩坑记录）
├── templates/            # 可复制的模板文件
├── scripts/              # 可执行脚本
└── assets/               # 图片等资源
```

### 写入来源追踪（skill_provenance）

关键细节：**不是所有Agent创建的技能都会被Curator整理**。

Hermes用 `ContextVar` 区分技能的写入来源：

```python
# tools/skill_provenance.py
_write_origin: contextvars.ContextVar[str] = ContextVar(
    "skill_write_origin",
    default="foreground",  # 默认：前台会话（用户直接要求创建的）
)
BACKGROUND_REVIEW = "background_review"  # Curator后台review fork
```

- **前台会话（foreground）**：用户和Agent聊天时，Agent提议保存技能，用户同意后创建的——这些**属于用户**，Curator**不会**自动归档/合并它们
- **后台审查（background_review）**：Curator fork的AIAgent创建的伞技能、合并补丁——这些**标记为agent_created**，是Curator未来整理的对象

这是非常重要的设计：Curator只整理它自己后台创建的技能，不会乱动用户明确要求创建的东西。

### 使用追踪：skill_usage 记录所有活动

每次技能被操作，`tools/skill_usage.py` 都会更新 `.usage.json` sidecar文件：

| 计数器 | 什么时候+1 | 作用 |
|--------|-----------|------|
| `view_count` | skill_view被调用（加载技能内容） | 衡量技能被发现的次数 |
| `use_count` | 技能被slash命令/--skills预加载使用 | 衡量技能实际被执行的次数 |
| `patch_count` | skill_manage patch/edit被调用 | 衡量技能被更新的次数 |
| 时间戳 | `last_viewed_at` `last_used_at` `last_patched_at` | Curator判断"多久没用了"的依据 |
| `created_at` | 技能创建时间 | 新技能不会刚创建就被归档 |
| `created_by` | "agent" 或 "user" | provenance标记来源 |
| `state` | active/stale/archived | 生命周期状态 |
| `pinned` | true/false | 用户钉住的技能Curator不动 |

这些数据就是Curator判断"哪些技能长期没用"、"哪些是窄技能应该合并"的信号源。

---

### 完整自学习闭环

沉淀和Curator的关系是一个**持续迭代的闭环**：

```
┌───────────────────────────────────────────────────────────────────┐
│  ① 使用中（前台会话）                                              │
│     ├─ Agent遇到复杂任务 → 探索 → 试错 → 成功解决                 │
│     ├─ System Prompt指示："完成后提议保存为技能"                   │
│     ├─ 询问用户 → 用户同意 → skill_manage create/patch            │
│     └─ 使用已加载技能时发现缺步骤/错命令 → 结束前必须patch更新     │
└──────────────────────────────┬────────────────────────────────────┘
                               │ 新技能/补丁写入磁盘
                               ▼
┌───────────────────────────────────────────────────────────────────┐
│  ② 索引更新                                                       │
│     ├─ 新技能自动出现在System Prompt的<available_skills>索引里    │
│     ├─ 下次会话模型就能看到这个技能（渐进式披露第一层：名字+描述）  │
│     └─ 需要时skill_view加载完整内容（渐进式披露第二层）            │
└──────────────────────────────┬────────────────────────────────────┘
                               │ 技能被使用
                               ▼
┌───────────────────────────────────────────────────────────────────┐
│  ③ 使用追踪                                                       │
│     ├─ skill_view加载 → view_count+1, last_viewed_at更新          │
│     ├─ slash命令/--skills使用 → use_count+1, last_used_at更新     │
│     ├─ 发现问题打补丁 → patch_count+1, last_patched_at更新        │
│     └─ 最新的活动时间戳作为Curator判断的依据                      │
└──────────────────────────────┬────────────────────────────────────┘
                               │ Agent空闲 + 距上次运行≥7天
                               ▼
┌───────────────────────────────────────────────────────────────────┐
│  ④ Curator后台整理（空闲时自动运行）                               │
│     ├─ 阶段一（确定性）：                                         │
│     │   ├─ last_activity > 30天 → 标记为stale                    │
│     │   ├─ last_activity > 90天 → 移动到.archive/归档             │
│     │   └─ stale技能重新被使用 → 恢复为active                     │
│     │                                                             │
│     └─ 阶段二（LLM，需consolidate=true）：                        │
│         ├─ 扫描所有agent_created技能，找前缀聚类（hermes-*, mcp-*）│
│         ├─ 判断：这几个窄技能应该合并成一个伞技能吗？              │
│         ├─ 方式a：patch已有伞技能，加新小节，归档窄技能            │
│         ├─ 方式b：create新伞技能，归档窄技能                      │
│         ├─ 方式c：窄技能内容write_file到伞技能的references/       │
│         └─ 所有写入origin标记为background_review                  │
└──────────────────────────────┬────────────────────────────────────┘
                               │ 伞技能更通用、更好用
                               ▼
┌───────────────────────────────────────────────────────────────────┐
│  ⑤ 回到①——下次使用                                                │
│     ├─ 索引里出现合并后的通用伞技能（名字更易匹配）                │
│     ├─ 加载伞技能发现缺少具体步骤 → 使用中patch补充                │
│     └─ 又有新经验 → 继续沉淀 → 循环往复                           │
└───────────────────────────────────────────────────────────────────┘
```

### 为什么这个闭环设计很聪明？

| 设计决策 | 原因 |
|---------|------|
| **主动提议但不自动保存** | 让用户把关，避免存一堆垃圾技能 |
| **前台创建的技能Curator不碰** | 用户明确要的东西永远不动，避免Curator"好心办坏事" |
| **只整理Curator自己创建的合并产物** | 自生成自整理，不越界 |
| **归档而非删除** | 误归档了可以从.archive/恢复，可回滚 |
| **使用中发现问题必须立即patch** | 不要等下次——踩坑的记忆还新鲜时就补上 |
| **Curator默认只做确定性归档，LLM合并默认关闭** | 省LLM钱，确定性的事情不用LLM，用户想伞状合并再开 |
| **30天stale/90天archive是默认值，可配置** | 不同用户节奏不同，可调整 |
| **Pinned技能跳过所有自动操作** | 用户可以钉住重要技能不让动 |

这就是Hermes的"自学习"本质：**不是模型自己在权重里学习，而是通过文件系统上的SKILL.md文档，形成一个可检查、可编辑、可版本控制、可回滚的外部程序化记忆库**——每一次解决问题的经验都沉淀成可复用的文档，Curator在后台像图书管理员一样整理书架，让这个知识库越用越好用。

---

## 关键源码位置

| 概念 | 文件位置 |
|------|---------|
| **Curator核心实现** | **`agent/curator.py`** |
| Curator备份机制 | `agent/curator_backup.py` |
| 技能使用追踪/状态管理 | `tools/skill_usage.py` |
| skill_manage 工具（patch/create/delete/archive） | `tools/skill_manager_tool.py` |
| **写入来源追踪（ContextVar区分foreground/background）** | **`tools/skill_provenance.py`** |
| Curator CLI命令 | `hermes_cli/curator.py` |
| 触发入口（会话结束时调用maybe_run_curator） | `run_agent.py` |
| Cron集成（可选定时触发） | `cron/jobs.py` |
| Gateway后台触发 | `gateway/run.py` |
| 技能索引构建（渐进式披露核心） | `agent/prompt_builder.py` (build_skills_system_prompt, ~L1354) |
| 平台/环境过滤 | `agent/skill_utils.py` (skill_matches_platform, skill_matches_environment) |
| 工具依赖条件判断 | `agent/prompt_builder.py` (_skill_should_show) |
| 分类降级（compact_categories） | `agent/coding_context.py` (coding_compact_skill_categories) |
| skill_view 工具实现 | `tools/skills_tool.py` |
| SKILL.md预处理（模板变量/内联shell） | `agent/skill_preprocessing.py` |
| 技能排除（依赖/虚拟env目录） | `agent/skill_utils.py` (is_excluded_skill_path, is_skill_support_path) |
| 技能扫描快照读写 | `agent/prompt_builder.py` (_load_skills_snapshot, _write_skills_snapshot) |
| reload-skills 热加载 | `agent/skill_commands.py` (reload_skills) |
