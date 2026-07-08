# Hermes Agent 学习笔记与资源

深入分析 Hermes Agent 核心架构与设计，配合最小代码示例和完整学习路线。

## 内容结构

```
learning-hermes/
├── notes/              # 核心学习笔记
│   ├── 01-整体架构概览.md
│   ├── 02-对话循环-ReAct-Loop.md
│   ├── 03-工具系统-ToolRegistry.md
│   ├── 04-技能系统-Skills.md
│   ├── 05-记忆系统-三层架构+编排器.md
│   ├── 06-Turn回合机制.md
│   ├── 07-架构选型-Tool-vs-Skill-vs-Subagent.md
│   ├── 08-MCP协议支持.md
│   ├── 09-异步后台-FireAndForget.md
│   ├── 10-PromptCaching前缀缓存.md
│   ├── 11-子agent如何使用Skills.md
│   └── 12-Skill自动沉淀-Curator机制.md
├── papers/             # 论文收集
│   └── quantization/   # 大模型量化论文分类整理（近两年顶会10个方向）
└── README.md
```

## 学习顺序建议

按编号顺序读即可，从整体到局部：
1. **01 整体架构概览** - 建立全局认知，知道各个模块在哪
2. **02 对话循环 ReAct Loop** - Agent的心脏，主循环怎么跑
3. **03 工具系统 ToolRegistry** - 工具注册、JSON Schema、Function Calling、扩展生态
4. **04 技能系统 Skills** - Skill是什么、怎么用
5. **11 子agent如何使用Skills** - 继承机制、共享文件系统
6. **12 Skill自动沉淀 Curator** - 经验如何自动积累，后台整理
7. **05 记忆系统** - 三层记忆架构、MemoryManager编排
8. **06 Turn回合机制** - 回合边界、生命周期、终止条件
9. **07 架构选型决策** - Tool/Skill/Subagent什么时候用哪个
10. **08 MCP协议支持** - 三种传输、协议概念、生态
11. **09 异步后台 Fire-and-Forget** - 非阻塞设计、所有使用场景
12. **10 Prompt Caching前缀缓存** - 字节级冻结、降本75%的工程细节

## 适合人群

- 想从零理解工业级Agent系统如何设计的开发者
- 研究Agent/LLM系统方向的学生/研究者
- 想自己写Agent框架，想参考成熟设计的工程师
- 需要前置知识：Python基础，了解LLM和Function Calling基本概念即可

## 核心主题

- ReAct Loop 对话循环
- ToolRegistry 工具系统与动态schema
- Skills 技能系统（Curator → Skills知识沉淀）
- Guardrails 输入输出安全门
- 三层记忆架构 + MemoryManager编排器 + Fire-and-Forget异步后台
- Turn回合边界与生命周期
- **架构选型：Tool vs Skill vs Subagent 决策指南**
- MCP协议支持（Stdio/Streamable HTTP/SSE三种传输）
- Prompt Caching前缀缓存（字节级冻结 + cache_control断点，降低75%成本）
- 大模型量化论文分类（10个方向）
