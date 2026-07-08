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
│   └── 10-PromptCaching前缀缓存.md
├── papers/             # 论文收集
│   └── quantization/   # 大模型量化论文分类整理
└── README.md
```

## 核心主题

- ReAct Loop 对话循环
- ToolRegistry 工具系统与动态schema
- Skills 技能系统（Curator → Skills知识沉淀）
- Guardrails 输入输出安全门
- 三层记忆架构 + MemoryManager编排器 + Fire-and-Forget异步后台
- Turn回合边界与生命周期
- MCP协议支持（Stdio/Streamable HTTP/SSE三种传输）
- Prompt Caching前缀缓存（字节级冻结 + cache_control断点，降低75%成本）
- 大模型量化论文分类（10个方向）
