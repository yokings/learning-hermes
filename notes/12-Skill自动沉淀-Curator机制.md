# 12 - Skill自动沉淀：Agent自己积累经验不重复踩坑
> 主agent和子agent都可以通过`skill_manage`工具即时把成功经验写成Skill，后台Curator（策展人）定期空闲时自动合并整理，不需要重复学习复杂流程
> 学习日期：2026-07-08

---

## 核心结论一句话

**Hermes的Skill不是只能人写！Agent在对话过程中解决了一个复杂问题后，可以主动调用`skill_manage`工具把成功流程写成SKILL.md沉淀下来，下次再遇到同类问题直接调用skill_view加载SOP，不用重复探索踩坑。后台还有专门的Curator（策展人）子agent，每周在你不用电脑的时候自动整理合并零散的小skill成通用的大类技能，归档长期不用的，保持技能库干净好用。**

---

## 一、即时沉淀：对话过程中主动创建Skill

### 1.1 工具：skill_manage（内置工具，默认启用）

**代码位置**：[tools/skill_manager_tool.py](file:///e:/github/hermes-agent/tools/skill_manager_tool.py)

这是内置工具，主agent和子agent都能用，支持6个操作：

| action | 作用 |
|--------|------|
| `create` | 创建新Skill，自动生成标准目录结构（SKILL.md + references/templates/scripts/assets） |
| `edit` | 全量重写已有SKILL.md内容 |
| `patch` | 定点补丁修改（find-and-replace），不用全量重写 |
| `delete` | 归档Skill（移到.archive目录，不真删除，可恢复） |
| `write_file` | 在Skill目录下添加支持文件（参考文档/模板/脚本/图片） |
| `remove_file` | 删除支持文件 |

### 1.2 典型沉淀场景

比如你让Hermes解决了一个复杂问题："帮我把项目部署到Vercel，配置自定义域名和HTTPS，解决了CORS问题"。

Agent做完后，如果判断这是个可复用的流程，会自己：
```
1. 调用 skill_manage(action="create",
      name="vercel-deploy-custom-domain",
      description="部署Vercel项目+配置自定义域名+HTTPS+CORS问题解决SOP",
      content="""# Vercel自定义域名部署完整SOP
## 前置准备
- 域名已经在阿里云/Cloudflare解析
- Vercel项目已经push到GitHub
## 步骤
1. 在Vercel项目Settings→Domains添加你的域名
2. 按提示去域名服务商添加CNAME记录
...
## CORS问题解决
如果遇到跨域错误，在vercel.json添加headers配置：
```json
...
```
""")
2. 返回给用户："我已经把这次部署完整流程沉淀成了`vercel-deploy-custom-domain` skill，下次部署直接调用这个skill就行，不用再重新查文档踩坑了。"
```

下次你再说"帮我部署另一个项目到Vercel，配置自定义域名"，模型直接：
```
调用 skill_view(name="vercel-deploy-custom-domain")
→ 直接拿到完整SOP
→ 按步骤执行，不用再重新摸索试错
```

### 1.3 Skill是agent的"程序性记忆"

文档原文：
> Skills are the agent's procedural memory: they capture *how to do a specific type of task* based on proven experience. General memory (MEMORY.md, USER.md) is broad and declarative. Skills are narrow and actionable.

| 记忆类型 | 内容 | 用途 |
|---------|------|------|
| MEMORY.md/USER.md | 陈述性记忆：事实、偏好、约定 | 记住"用户喜欢深色主题"、"项目用pnpm" |
| Skill | 程序性记忆：怎么做某类事的流程、SOP、经验 | 记住"如何部署Vercel自定义域名"、"如何排查React打包内存溢出" |

这和人类的记忆分类完全一致：事实记在长期记忆里，做事的步骤沉淀成"技能"，下次直接用。

---

## 二、后台自动整理：Curator（策展人）机制

如果Agent每次做个小任务都创建一个新Skill，时间长了技能库会变成几百个零散的小条目，找都找不到——这就是Curator要解决的问题。

**代码位置**：[agent/curator.py](file:///e:/github/hermes-agent/agent/curator.py)

### 2.1 Curator是什么？

Curator是一个**后台空闲触发的辅助子agent**——不是定时cron，而是当：
1. 主agent已经空闲了至少2小时（默认`min_idle_hours=2`）
2. 距离上次Curator运行已经超过7天（默认`interval_hours=168`）
3. Curator功能启用（默认启用）

满足这三个条件时，Hermes会fork一个独立的辅助AIAgent（用auxiliary模型配置，不占用主会话，不打断Prefix Cache），后台自动整理技能库。

### 2.2 Curator的核心职责

| 职责 | 说明 |
|------|------|
| **自动生命周期管理** | 不消耗LLM token，纯确定性规则：30天不用标记stale，90天不用归档到.archive（可恢复，永不删除） |
| **合并成伞形技能（Umbrella Skill）** | 消耗少量aux模型token：把零散的小skill合并成通用的大类技能，比如把`vercel-deploy`、`vercel-cors`、`vercel-env`合并成一个大的`vercel` umbrella skill，小的变成子章节/references |
| **补丁修正** | 发现skill内容有错误/过时了，自动patch修复 |
| **归档无用skill** | 确认长期不用、没有价值的skill归档（不删除） |
| **钉住保护** | 用户pin过的skill完全不动，永远不自动归档/合并 |

### 2.3 严格的安全铁律（代码注释原文）

```
Strict invariants:
  - Only touches agent-created skills (不碰官方/Hub安装的skill)
  - Never auto-deletes — only archives. Archive is recoverable.（永不删除，只归档，可恢复）
  - Pinned skills bypass all auto-transitions（用户pin的skill完全跳过）
  - Uses the auxiliary client; never touches the main session's prompt cache（用独立的辅助模型，不碰主会话Prefix Cache）
```

### 2.4 默认配置（开箱即用，不用改）

```yaml
# config.yaml 里可以调整，默认值已经很合理
curator:
  enabled: true               # 默认启用
  interval_hours: 168         # 7天跑一次
  min_idle_hours: 2           # 空闲2小时才跑，不影响你用
  stale_after_days: 30        # 30天没用标记stale
  archive_after_days: 90      # 90天没用归档
  consolidate: false          # 默认关闭LLM合并，只做确定性的生命周期管理；
                              # 想开LLM合并整理设为true，会消耗少量aux模型token
  prune_builtins: true        # 内置skill太久不用也归档
```

### 2.5 合并原则：追求"类级别伞形技能"，反对"一次任务一个小skill"

Curator的系统prompt原文（curator.py#L365-L506）明确要求：

```
The goal of the skill collection is a LIBRARY OF CLASS-LEVEL INSTRUCTIONS AND EXPERIENTIAL KNOWLEDGE.
A collection of hundreds of narrow skills where each one captures one session's specific bug is a FAILURE — not a feature.
One broad umbrella skill with labeled subsections beats five narrow siblings for discoverability.

正确的目标形态：
  - 类级别大伞skill，内容丰富
  - references/ 下放具体参考资料
  - templates/ 下放模板
  - scripts/ 下放脚本
而不是：几百个每个只解决一次特定问题的微型skill。
```

合并方式有三种：
1. **并入已有伞形skill**：比如已有`vercel`，把`vercel-cors`内容patch进去加一节，然后归档小的
2. **创建新伞形skill**：没有合适的大伞，创建新的大类skill，把小的吸收进去
3. **降级为支持文件**：小skill内容太窄但有价值，移到大伞的`references/`目录下当参考资料

### 2.6 完成后处理

- 归档的skill会写总结："xx → vercel"，告诉你去哪找
- 自动更新cron任务里引用的skill路径，不会因为合并导致cron失效
- 生成REPORT.md报告，存在`~/.hermes/logs/curator/YYYYMMDD-HHMMSS/`下
- 下次启动Hermes会显示上次整理结果
- 你可以用 `hermes curator status` 看状态，`hermes curator pin <name>` 钉住重要skill不被自动整理，`hermes curator run` 手动跑一次

---

## 三、完整的Skill生命周期图

```
Agent解决复杂问题成功
    ↓
判断：这个流程可复用吗？
    ├─ 否 → 什么都不做
    └─ 是 → 调用 skill_manage(create) 创建新Skill
            ↓
          Skill存到 ~/.hermes/skills/<name>/SKILL.md
            ↓
以后遇到同类问题：
  调用 skill_view(name) → 加载SOP → 直接按步骤执行，不用重新摸索
            ↓
（7天后，Hermes空闲2小时以上）
            ↓
        Curator后台启动
          ├─ 确定性处理：30天不用标stale，90天归档
          ├─ （如果开启consolidate）LLM合并零散小skill成伞形大技能
          ├─ 钉住的skill跳过
          └─ 归档的skill移到.archive，永不删除可恢复
            ↓
        技能库保持干净、通用、好查找
            ↓
        引用了被归档skill的cron任务自动更新路径
```

---

## 四、用户如何控制？

你完全可以掌控技能库，不需要担心自动整理乱动你的东西：

| 命令 | 作用 |
|------|------|
| `hermes skills list` | 看所有已安装的skill |
| `hermes curator status` | 看Curator状态，上次运行结果，整理了哪些 |
| `hermes curator pin <skill-name>` | 钉住某个skill，永远不自动归档/合并 |
| `hermes curator unpin <skill-name>` | 取消钉住 |
| `hermes curator run --dry-run` | 预览Curator会做什么，不实际修改 |
| `hermes curator run` | 手动立即跑一次Curator整理 |
| `hermes curator pause` | 暂停自动整理 |
| `hermes curator resume` | 恢复自动整理 |
| 你自己也可以用skill_manage | 手动创建/编辑/删除skill，和agent一样 |

归档的skill在`~/.hermes/skills/.archive/`下，随时可以手动移回来恢复。

---

## 五、主agent和子agent都能创建Skill吗？

看黑名单：[delegate_tool.py#L45-L54](file:///e:/github/hermes-agent/tools/delegate_tool.py#L45-L54)
```python
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task", "clarify", "memory", "send_message", "execute_code", "cronjob"
])
```
✅ `skill_manage` **不在黑名单里**！子agent默认也有完整的skill创建/编辑权限，可以在完成子任务后沉淀自己领域的skill。

- 子agent创建的skill和主agent创建的存在同一个磁盘目录
- Curator统一整理，不分是谁创建的
- 主agent和所有子agent都能用到这些沉淀的经验

这意味着：你派一个子agent专门研究"如何解决React打包内存溢出问题"，它研究完解决了问题，不仅返回结果给你，还可以把完整排障流程沉淀成skill，下次遇到同样问题直接用。

---

## 六、为什么这个设计很优雅？

1. **不用专门的"学习阶段"**：在正常解决问题的过程中顺便沉淀经验，不额外消耗时间token
2. **经验跨会话永久保留**：一次学会，以后所有会话、所有子agent都能用，不会因为重启丢失
3. **自动整理保持可用**：不会越积越多乱成一团，Curator后台自动合并归档，保持技能库结构清晰
4. **安全可控**：永不删除只归档，pin住的不动，用户完全可控，可以dry-run预览
5. **共享给子agent**：主agent沉淀的经验子agent直接用，子agent解决问题沉淀的经验主agent也能用，整体越用越聪明
6. **不打扰用户**：Curator只在空闲时跑，用独立的aux模型，不打断你聊天，不碰主会话Prefix Cache，你完全感知不到

---

## 关键源码索引

| 功能 | 文件位置 |
|------|---------|
| skill_manage工具实现 | [tools/skill_manager_tool.py](file:///e:/github/hermes-agent/tools/skill_manager_tool.py) |
| Curator核心编排 | [agent/curator.py](file:///e:/github/hermes-agent/agent/curator.py) |
| Skill使用追踪（记录最后使用时间） | [tools/skill_usage.py](file:///e:/github/hermes-agent/tools/skill_usage.py) |
| Curator CLI命令 | [hermes_cli/curator.py](file:///e:/github/hermes-agent/hermes_cli/curator.py) |
| 子agent工具黑名单 | [tools/delegate_tool.py#L45-L54](file:///e:/github/hermes-agent/tools/delegate_tool.py#L45-L54) |
| 安全扫描（agent创建的skill可选扫描） | [tools/skills_guard.py](file:///e:/github/hermes-agent/tools/skills_guard.py) |

---

## 一句话总结

**Agent通过`skill_manage`工具在解决问题后即时把成功流程沉淀成可复用Skill，后台Curator每周空闲时自动合并零散小技能为通用大类、归档长期不用的，一次学会永久复用，主agent子agent都能用，越用越熟练，不用每次重复走复杂流程踩同样的坑。**