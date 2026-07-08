# 08 - MCP协议支持
> 单事件循环 + 三传输 + 深度集成：MCP 协议详解与 Hermes 实现
> 学习日期：2026-07-02

---

## 一、什么是 MCP？

### 1.1 MCP 的由来与作用

**MCP = Model Context Protocol（模型上下文协议）**，是由 Anthropic（Claude 开发商）于 2024 年底开源的一套开放协议。

#### MCP 解决了什么问题？

在 MCP 出现之前，AI Agent 接入外部工具/数据源的生态是**碎片化的孤岛**：

```
MCP 之前的混乱生态：
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   ChatGPT    │  │   Claude     │  │   Cursor    │
│  Plugins     │  │  Tools API   │  │  Extensions │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       │ 每个平台有     │  每个平台有     │  每个平台有
       │ 自己的工具API  │  自己的格式     │  自己的插件系统
       ▼                ▼                ▼
   ┌──────────────────────────────────────────┐
   │     工具开发者：我要写 N 套适配器！        │
   │     为 ChatGPT 写一套，为 Claude 写一套... │
   └──────────────────────────────────────────┘
```

**MCP 的目标**：做 AI Agent 世界的 **"USB-C 接口"** —— 一次开发，处处可用。

```
MCP 之后的统一生态：
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   ChatGPT    │  │   Claude     │  │   Cursor    │
│     IDE      │  │   Hermes     │  │   Cline     │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
              ┌─────────▼─────────┐
              │   MCP 标准协议    │  ← 统一的接口标准
              └─────────┬─────────┘
                        │
       ┌────────────────┼────────────────┐
       ▼                ▼                ▼
  ┌─────────┐     ┌─────────┐     ┌─────────┐
  │ 文件系统│     │  数据库 │     │  Git    │
  │  MCP    │     │  MCP    │     │  MCP    │
  └─────────┘     └─────────┘     └─────────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
         工具开发者：写一次 MCP Server，所有 Agent 都能用！
```

#### MCP 的核心价值

| 角色 | MCP 带来的好处 |
|------|---------------|
| **Agent 开发者** | 不需要为每个工具单独写适配层，直接接入标准 MCP Client |
| **工具开发者** | 写一次 MCP Server，所有支持 MCP 的 Agent 都能直接用，不需要适配 N 个平台 |
| **用户** | 想加什么工具，配置一下 MCP Server 地址就行，生态丰富，即插即用 |
| **生态** | 类似 LSP（Language Server Protocol）统一了编辑器和语言工具链，MCP 正在统一 AI Agent 和外部工具/数据的连接方式 |

---

### 1.2 MCP 的架构构成

MCP 采用经典的 **Client-Server（客户端-服务器）架构**：

```
┌─────────────────────────────────────────────────────────────────┐
│                        MCP Host (宿主)                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  AI Agent (如 Hermes/Claude/Cursor)      │   │
│  │  ┌─────────────────────────────────────────────────┐    │   │
│  │  │              MCP Client                         │    │   │
│  │  │  （协议处理、工具调用、传输管理）                  │    │   │
│  │  └───────┬──────────────┬──────────────┬───────────┘    │   │
│  └──────────┼──────────────┼──────────────┼────────────────┘   │
└─────────────┼──────────────┼──────────────┼─────────────────────┘
              │              │              │
        ┌─────▼─────┐  ┌────▼─────┐  ┌─────▼─────┐
        │  MCP      │  │  MCP     │  │  MCP      │
        │ Server 1  │  │ Server 2 │  │ Server 3  │
        │ (Files)   │  │ (Git)    │  │ (Search)  │
        └───────────┘  └──────────┘  └───────────┘
```

#### 三个核心角色

| 角色 | 说明 | 例子 |
|------|------|------|
| **MCP Host** | 承载 MCP Client 的应用 | Hermes、Claude Desktop、Cursor IDE、Cline、Windsurf |
| **MCP Client** | Host 内置的协议客户端，负责和 Server 通信 | Hermes 中的 `mcp_tool.py`，Claude Desktop 内置的 MCP 客户端 |
| **MCP Server** | 提供具体工具/资源/Prompts 的服务 | filesystem-mcp、git-mcp、brave-search-mcp、github-mcp |

---

### 1.3 MCP 提供的三大能力

MCP Server 可以暴露三类能力给 Client：

```
MCP Server 能力矩阵
┌─────────────────────────────────────────────────────────────┐
│  1. Tools（工具）                                          │
│     ─────────                                              │
│     模型可以调用的函数，执行操作并返回结果                     │
│     例：read_file、search_web、run_command、create_issue    │
│     → 这是最常用、最重要的能力！                            │
├─────────────────────────────────────────────────────────────┤
│  2. Resources（资源）                                      │
│     ──────────                                             │
│     可被模型读取的数据源，类似"文件"                          │
│     例：file:///path/to/file、postgres://table/schema       │
│     → 让 Agent 能读取结构化/非结构化数据                     │
├─────────────────────────────────────────────────────────────┤
│  3. Prompts（提示模板）                                    │
│     ────────────                                           │
│     可复用的提示词模板，用户可以直接调用                       │
│     例：/explain-code、/write-tests、/review-pr             │
│     → 标准化常用工作流                                     │
└─────────────────────────────────────────────────────────────┘
```

Hermes 目前主要使用 **Tools（工具）** 这一类能力。

---

### 1.4 常用 MCP Server 一览

#### 官方维护的参考实现

| MCP Server | 作用 | 启动方式 |
|------------|------|---------|
| **filesystem** | 安全访问本地文件系统（读写文件、列目录） | `npx @modelcontextprotocol/server-filesystem /allowed/path` |
| **github** | 操作 GitHub（Issues、PRs、代码搜索） | 需要 GitHub Token |
| **brave-search** | Brave 搜索引擎联网搜索 | 需要 Brave API Key |
| **google-maps** | Google Maps 服务（地点、路线、距离） | 需要 Google Maps Key |
| **postgres** | 连接 PostgreSQL 数据库，只读查询 | 数据库连接串 |
| **sqlite** | 操作 SQLite 数据库 | 数据库文件路径 |
| **slack** | 收发 Slack 消息 | Slack Bot Token |
| **memory** | 基于向量数据库的持久化记忆 | 本地运行 |
| **fetch** | HTTP 网页抓取（获取 URL 内容） | `npx @modelcontextprotocol/server-fetch` |
| **puppeteer** | 浏览器自动化（Puppeteer） | 本地运行 Chromium |

#### 社区热门 MCP Server

| MCP Server | 作用 |
|------------|------|
| **mcp-server-git** | Git 仓库操作（看 diff、提交历史、blame） |
| **mcp-server-shell** | 执行 shell 命令（需要谨慎配置白名单） |
| **docker-mcp** | 管理 Docker 容器/镜像 |
| **arxiv-mcp** | 检索 arXiv 学术论文 |
| **mcp-searxng** | 开源 SearXNG 元搜索引擎 |
| **figma-mcp** | 读取 Figma 设计稿数据 |
| **linear-mcp** | Linear 项目管理（Issue、Cycle） |
| **notion-mcp** | 操作 Notion 页面/数据库 |

---

### 1.5 MCP 通信协议基础

MCP 基于 **JSON-RPC 2.0** 格式进行通信。

#### 连接建立流程（Initialize Handshake）

```
MCP Client                                MCP Server
    │                                         │
    │  1. initialize 请求                      │
    │  {                                      │
    │    "jsonrpc": "2.0",                    │
    │    "id": 1,                             │
    │    "method": "initialize",              │
    │    "params": {                          │
    │      "protocolVersion": "2024-11-05",   │
    │      "capabilities": {                  │
    │        "tools": {}                      │
    │      },                                 │
    │      "clientInfo": {                    │
    │        "name": "hermes",                │
    │        "version": "1.0"                 │
    │      }                                  │
    │    }                                    │
    │  }                                      │
    │ ──────────────────────────────────────► │
    │                                         │
    │  2. initialize 响应                     │
    │  {                                      │
    │    "jsonrpc": "2.0",                    │
    │    "id": 1,                             │
    │    "result": {                          │
    │      "protocolVersion": "2024-11-05",   │
    │      "capabilities": {                  │
    │        "tools": { "listChanged": true } │
    │      },                                 │
    │      "serverInfo": {                    │
    │        "name": "filesystem",            │
    │        "version": "1.0"                 │
    │      }                                  │
    │    }                                    │
    │  }                                      │
    │ ◄────────────────────────────────────── │
    │                                         │
    │  3. initialized 通知（握手完成）         │
    │ ──────────────────────────────────────► │
    │                                         │
```

#### 发现工具（tools/list）

```
Client:
{ "method": "tools/list", "id": 2 }

Server 返回：
{
  "tools": [
    {
      "name": "read_file",
      "description": "Read a file from disk",
      "inputSchema": {
        "type": "object",
        "properties": {
          "path": { "type": "string", "description": "File path" }
        },
        "required": ["path"]
      }
    },
    // ... 更多工具
  ]
}
```

工具的 `inputSchema` 就是 **JSON Schema**，模型直接看这个就知道怎么调用！

#### 调用工具（tools/call）

```
Client:
{
  "method": "tools/call",
  "id": 3,
  "params": {
    "name": "read_file",
    "arguments": { "path": "/etc/hostname" }
  }
}

Server 返回：
{
  "content": [
    { "type": "text", "text": "my-computer\n" }
  ]
}
```

#### 服务器通知（Notifications）

Server 可以主动发通知给 Client：
- `tools/list_changed`：工具列表变了，Client 需要重新 `tools/list`
- `notifications/message`：日志消息

---

## 二、MCP 传输层详解

**传输层（Transport）** 负责在 Hermes（MCP Client）和 MCP Server 之间传递上述 JSON-RPC 消息。MCP 标准定义了三种传输方式，Hermes 全部支持。

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Hermes MCP Client                               │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │                    MCPServerTask                              │  │
│  │  (每个MCP Server一个，运行在专用 mcp-event-loop 后台线程)       │  │
│  └──────────────┬──────────────────┬──────────────────┬─────────┘  │
│                 │                  │                  │            │
└─────────────────┼──────────────────┼──────────────────┼────────────┘
                  │                  │                  │
          ┌───────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐
          │   Stdio       │  │ Streamable   │  │     SSE      │
          │  (子进程通信)  │  │ HTTP (HTTP)  │  │ (Server-Sent  │
          │               │  │              │  │  Events)      │
          └───────┬───────┘  └──────┬───────┘  └──────┬───────┘
                  │                  │                  │
          ┌───────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐
          │ 本地 MCP       │  │ 远程 MCP      │  │ 远程 MCP      │
          │ Server 进程    │  │ Server        │  │ Server        │
          │ (npx/uvx/...)  │  │ (HTTPS)       │  │ (HTTP长连接)  │
          └───────────────┘  └──────────────┘  └──────────────┘
```

**核心实现文件**：[mcp_tool.py](file:///e:/github/hermes-agent/tools/mcp_tool.py)（近3000行，包含三种传输、重连、保活、OAuth、熔断器等完整实现）

---

### 传输方式对比总览

| 特性 | Stdio | Streamable HTTP | SSE |
|------|-------|-----------------|-----|
| **通信方式** | 本地子进程 stdin/stdout | HTTP 请求/响应 | HTTP 长连接 + Server-Sent Events |
| **适用场景** | 本地工具（filesystem、git等） | 现代远程MCP服务（推荐） | 旧版远程MCP服务、需要服务器推送 |
| **部署位置** | 本机启动子进程 | 远程服务器（公网/内网） | 远程服务器 |
| **连接方向** | Hermes 启动 Server | Hermes 主动请求 Server | Hermes 建立长连接，Server 推送 |
| **消息方向** | 全双工（双向管道） | 请求-响应，支持会话复用 | 单向推送（SSE）+ HTTP POST（客户端发消息） |
| **网络要求** | 无需网络 | 需要HTTP/HTTPS | 需要HTTP，长连接不能被代理切断 |
| **延迟** | 最低（本地进程间通信） | 较低（HTTP/2复用） | 低（长连接已建立） |
| **安全性** | 环境变量白名单过滤 + OSV恶意软件检测 | OAuth 2.1 PKCE + mTLS + 跨域重定向保护 + Content-Type预检 | 同Streamable HTTP |
| **SDK支持版本** | 所有版本 | mcp >= 1.0，新API mcp >= 1.24.0 | mcp 较早版本支持 |
| **是否默认** | 本地MCP默认 | 配置url时默认（无transport:sse） | 需显式配置 `transport: sse` |
| **stderr处理** | 重定向到日志文件 | 由HTTP服务器处理 | 由HTTP服务器处理 |
| **空闲超时** | 进程存活即连接存活 | 300s读超时 + 可配置keepalive | **sse_read_timeout=300s**（关键！） |
| **保活机制** | 可选ping/list_tools | 可配置keepalive_interval（默认180s） | 同Streamable HTTP |
| **自动重连** | 不支持（进程退出即结束） | 支持（最多5次，指数退避） | 支持（同Streamable HTTP） |
| **进程组清理** | 支持killpg清理孙子进程 | N/A | N/A |

---

### 2.1 Stdio 传输：本地子进程通信

#### 是什么？
Stdio（Standard Input/Output）传输通过**启动本地子进程**，直接用子进程的 stdin（标准输入）和 stdout（标准输出）作为双向通信管道。这是 MCP 最原始、最简单、延迟最低的传输方式。

**核心代码**：[_run_stdio()](file:///e:/github/hermes-agent/tools/mcp_tool.py#L1762-L1897)

#### 工作原理

```
Hermes Agent
    │
    │ 1. 启动子进程
    │    command: npx -y @modelcontextprotocol/server-filesystem /path
    │    env: 过滤后的安全环境变量
    │    stderr → ~/.hermes/logs/mcp-stderr.log
    │
    ▼
┌─────────────────────────────────────────┐
│  MCP Server 子进程                      │
│                                         │
│  stdin (Hermes → Server)  ──────────►  JSON-RPC 请求
│                                         │
│  stdout (Server → Hermes) ◄──────────  JSON-RPC 响应/通知
│                                         │
│  stderr ──────────────────────────────►  日志文件
│                                         │
└─────────────────────────────────────────┘
    │
    │ 2. MCP 握手：initialize 请求/响应
    │ 3. tools/list 发现工具
    │ 4. 进入保活循环，等待工具调用
    │
```

#### 配置示例

```yaml
# ~/.hermes/config.yaml
mcp_servers:
  # 文件系统工具
  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "e:\\github"]
    env: {}  # 额外环境变量
    timeout: 120
    keepalive_interval: 10  # 短TTL服务器用

  # Git 工具
  git:
    command: "uvx"
    args: ["mcp-server-git", "--repository", "."]
```

#### 关键设计细节

##### 1. 环境变量安全过滤（白名单机制）
**代码位置**：[_build_safe_env()](file:///e:/github/hermes-agent/tools/mcp_tool.py#L366-L386)

Stdio MCP Server 是本地启动的子进程，默认会继承 Hermes 的所有环境变量（包括API Key、Token等敏感信息）。为防止泄漏：
- 使用白名单机制，只保留安全的环境变量（PATH、HOME、USER、系统路径等）
- Unix 白名单和 Windows 大小写不敏感白名单分别处理
- 用户在 config 中显式指定的 `env` 变量会额外加入

```python
_SAFE_ENV_KEYS = {
    "PATH", "HOME", "USER", "SHELL", "LANG", "LC_ALL", "TMPDIR",
    # ... 其他安全变量
}
```

##### 2. OSV 恶意软件预检测
**代码位置**：[mcp_tool.py:1790-1806](file:///e:/github/hermes-agent/tools/mcp_tool.py#L1790-L1806)

spawn 子进程前，会查询 OSV（Open Source Vulnerabilities）数据库检测包是否有已知恶意软件：
- 12秒超时，**fail-open**：超时或网络问题只记日志，继续启动（不阻塞用户）
- 发现恶意软件直接拒绝启动

##### 3. stderr 重定向到日志文件
**代码位置**：[_get_mcp_stderr_log()](file:///e:/github/hermes-agent/tools/mcp_tool.py#L116-L184)

**问题**：FastMCP等框架启动时会打印横幅、JSON日志到stderr，如果直接继承Hermes的stderr，会破坏TUI界面（prompt_toolkit/Rich渲染）。

**解决方案**：所有stdio MCP子进程的stderr统一重定向到 `~/.hermes/logs/mcp-stderr.log`，调试时可以查看，但不污染用户界面。

##### 4. 命令路径解析（Fallback搜索）
**代码位置**：[_resolve_stdio_command()](file:///e:/github/hermes-agent/tools/mcp_tool.py#L518-L562)

对于 `npx`、`npm`、`node` 等不写绝对路径的命令，会做多路径fallback搜索：
- `hermes_home/node/bin`
- `~/.local/bin`
- `/usr/local/bin`
- 系统PATH

##### 5. 进程组追踪与孤儿清理
**代码位置**：[mcp_tool.py:1820-1897](file:///e:/github/hermes-agent/tools/mcp_tool.py#L1820-L1897)

**问题**：MCP SDK的子进程清理可能失败（特别是Linux上setsid()的子进程会逃逸cgroup，或wrapper脚本启动的孙子进程），导致孤儿进程残留。

**解决方案**：
- spawn前后做PID快照，记录新PID和PGID（Process Group ID）
- Windows跳过`os.getpgid`（POSIX-only）
- 退出时检查：如果PID/PGID仍存活，标记为孤儿进程，下次清理sweep时kill
- 使用`os.killpg()`清理整个进程组（包括孙子进程）

#### 适用场景

✅ **最佳使用场景**：
- 本地工具（filesystem、git、docker、数据库等）
- 需要低延迟的工具调用
- 不需要远程访问的工具
- 用npx/uvx直接运行的MCP Server

❌ **不适用场景**：
- 远程部署的MCP服务
- 需要OAuth认证的服务
- 多客户端共享的MCP服务

---

### 2.2 Streamable HTTP 传输：现代远程HTTP通信

#### 是什么？
Streamable HTTP 是 MCP 官方推荐的现代远程传输方式，基于标准HTTP/HTTPS协议，支持会话复用、双向流式通信，是目前远程MCP服务的首选。

**核心代码**：[_run_http()](file:///e:/github/hermes-agent/tools/mcp_tool.py#L1979-L2168) 中的非SSE分支

#### 工作原理

```
Hermes Agent                                    MCP Server (远程)
    │                                                │
    │ 1. Content-Type 预检（HEAD/GET，5s超时）         │
    │ ──────────────────────────────────────────────► │
    │ ◄────────────────────────────────────────────── │
    │    验证 Content-Type 是 application/json 或     │
    │    text/event-stream，不是普通网页              │
    │                                                │
    │ 2. OAuth 2.1 PKCE 认证（如果需要）               │
    │ ──────────────────────────────────────────────► │
    │                                                │
    │ 3. 建立HTTP连接                                 │
    │    POST /mcp  (initialize)                     │
    │ ──────────────────────────────────────────────► │
    │ ◄────────────────────────────────────────────── │
    │    Session-ID 头部（会话复用）                   │
    │                                                │
    │ 4. JSON-RPC over HTTP（可流式）                 │
    │    POST /mcp  (tools/list, tools/call)         │
    │ ──────────────────────────────────────────────► │
    │ ◄────────────────────────────────────────────── │
    │    响应（支持分块流式传输）                      │
    │                                                │
    │ 5. 保活探测（默认180s）                         │
    │    POST /mcp  (ping/list_tools)                │
    │ ──────────────────────────────────────────────► │
    │                                                │
```

#### 配置示例

```yaml
mcp_servers:
  # 远程MCP服务（Streamable HTTP，默认）
  my-remote-api:
    url: "https://my-mcp-server.example.com/mcp"
    headers:
      Authorization: "Bearer sk-xxx"
    timeout: 180
    connect_timeout: 60
    keepalive_interval: 30  # 短TTL服务器调小
    ssl_verify: true  # 设为false跳过SSL验证（仅用于自签名证书测试）
    # client_cert: /path/to/cert.pem  # mTLS客户端证书
    # client_key: /path/to/key.pem
```

#### 关键设计细节

##### 1. Content-Type 预检（防挂死）
**代码位置**：[_preflight_content_type()](file:///e:/github/hermes-agent/tools/mcp_tool.py#L1904-L1977)

**问题**：如果用户配置错URL（指向普通网页、404页面、登录页），SDK会等待完整的`connect_timeout`（默认60s）才超时，体验很差。

**解决方案**：SDK正式连接前，先用5s超时做HEAD/GET探针：
- 必须返回2xx
- Content-Type必须是`application/json`或`text/event-stream`
- 否则立即抛出`NonMcpEndpointError`（非重试错误），快速失败

```python
_MCP_CONTENT_TYPES = ("application/json", "text/event-stream")
```

##### 2. 新旧SDK API兼容
**代码位置**：[mcp_tool.py:2099-2168](file:///e:/github/hermes-agent/tools/mcp_tool.py#L2099-L2168)

MCP Python SDK在1.24.0版本更新了HTTP API（下划线命名），Hermes同时支持两个版本：
- **新API**（mcp >= 1.24.0）：`streamable_http_client`（下划线），允许传入自定义`httpx.AsyncClient`，完全控制TLS/重定向/hook
- **旧API**（mcp < 1.24.0）：`streamablehttp_client`（驼峰），SDK内部管理httpx client

优雅导入检测：
```python
try:
    from mcp.client.streamable_http import streamable_http_client
    _MCP_NEW_HTTP = True
except ImportError:
    _MCP_NEW_HTTP = False
```

##### 3. 跨域重定向安全（Authorization头剥离）
**代码位置**：[_strip_auth_on_cross_origin_redirect()](file:///e:/github/hermes-agent/tools/mcp_tool.py#L2106-L2114)

**安全风险**：HTTP重定向到不同origin（不同scheme/host/port）时，如果还带着Authorization头，会把token泄露给第三方。

**解决方案**：httpx event hook自动检测：
```python
if response.is_redirect and response.next_request:
    target = response.next_request.url
    if (target.scheme, target.host, target.port) != (original.scheme, original.host, original.port):
        response.next_request.headers.pop("authorization", None)
        response.next_request.headers.pop("Authorization", None)
```

##### 4. OAuth 2.1 PKCE 支持
**代码位置**：通过 [mcp_oauth_manager.py](file:///e:/github/hermes-agent/tools/mcp_oauth_manager.py) 实现
- 标准OAuth 2.1 PKCE流程（授权码模式+code challenge）
- Token缓存到磁盘，多server复用
- 401自动刷新token
- 支持动态客户端注册（DCR）

##### 5. mTLS客户端证书支持
支持配置`client_cert`和`client_key`做双向TLS认证，用于企业内部MCP服务。

##### 6. 会话过期处理与自动重连
**代码位置**：[_is_session_expired_error()](file:///e:/github/hermes-agent/tools/mcp_tool.py#L2687-L2750) / [_handle_session_expired_and_retry()](file:///e:/github/hermes-agent/tools/mcp_tool.py#L2752-L2813)

Streamable HTTP服务器可能GC空闲会话（返回"Invalid or expired session"等错误）。Hermes通过子串匹配12种错误标记检测会话过期，然后触发传输层重连（`_reconnect_event`），而不是走OAuth刷新流程。

##### 7. 读超时配置
- connect_timeout：默认60s（建立连接超时）
- read timeout：300s（等待响应/事件超时），适配长连接场景

#### 适用场景

✅ **最佳使用场景**：
- 远程部署的MCP服务（公网/内网）
- 需要OAuth认证的服务
- 生产环境推荐使用
- 需要负载均衡/HTTP/2复用的场景

❌ **不适用场景**：
- 本地工具（stdio更简单更快）
- 只支持旧SSE协议的服务器（用SSE传输）

---

### 2.3 SSE 传输：Server-Sent Events 长连接

#### 是什么？
SSE（Server-Sent Events）是一种基于HTTP的单向服务器推送技术。在MCP中，SSE传输使用：
- **一个长连接GET请求**：服务器通过这个连接推送JSON-RPC消息/通知给客户端
- **普通HTTP POST请求**：客户端发送JSON-RPC消息给服务器

这是MCP较早的远程传输方式，现在逐渐被Streamable HTTP取代，但仍有很多旧服务器使用。

**核心代码**：[_run_http() SSE分支](file:///e:/github/hermes-agent/tools/mcp_tool.py#L2026-L2097)

#### 工作原理

```
Hermes Agent                                    MCP Server
    │                                                │
    │ 1. 建立SSE长连接（GET /sse）                     │
    │    Accept: text/event-stream                    │
    │ ──────────────────────────────────────────────► │
    │ ◄────────────────────────────────────────────── │
    │    连接保持打开，服务器随时可以推送                │
    │    endpoint 事件 → 告诉客户端POST用的URL         │
    │                                                │
    │ 2. 客户端发消息（POST到endpoint URL）            │
    │    POST /message?session_id=xxx                 │
    │ ──────────────────────────────────────────────► │
    │ ◄────────────────────────────────────────────── │
    │    202 Accepted                                 │
    │                                                │
    │ 3. 服务器响应通过SSE长连接推送回来                │
    │ ◄────────────────────────────────────────────── │
    │    event: message                               │
    │    data: {"jsonrpc":"2.0", "result": ...}       │
    │                                                │
    │ 4. 保活：SSE连接长时间空闲（300s超时）           │
    │                                                │
```

#### 配置示例

```yaml
mcp_servers:
  # SSE传输（需显式指定 transport: sse）
  searxng:
    url: "http://localhost:8000/sse"
    transport: sse  # ← 关键！必须显式指定
    timeout: 180
    headers:
      Authorization: "Bearer xxx"
    ssl_verify: true
```

#### 关键设计细节

##### 1. sse_read_timeout=300s（关键修复！）
**代码位置**：[mcp_tool.py:2045](file:///e:/github/hermes-agent/tools/mcp_tool.py#L2041-L2046)

这是一个重要的bug修复（来自PR #5981）：

**问题**：最初代码用`tool_timeout`（默认60s）作为SSE读超时，但SSE服务器经常在事件之间保持空闲流数分钟（特别是Cloudflare Workers等平台约60s空闲断连），导致60s后连接被切断。

**修复**：
```python
_sse_kwargs = {
    "url": url,
    "headers": headers or None,
    "timeout": float(connect_timeout),
    "sse_read_timeout": 300.0,  # ← 5分钟空闲超时，匹配Streamable HTTP
}
```

##### 2. OAuth透传修复
**代码位置**：[mcp_tool.py:2047-2051](file:///e:/github/hermes-agent/tools/mcp_tool.py#L2047-L2051)

早期代码构建了`_oauth_auth`但忘记传给SSE路径，导致OAuth保护的SSE MCP服务器静默401。现在通过`auth=`参数正确透传。

##### 3. TLS/mTLS 支持（httpx_client_factory）
**代码位置**：[mcp_tool.py:2052-2082](file:///e:/github/hermes-agent/tools/mcp_tool.py#L2052-L2082)

SSE transport的`sse_client`不直接暴露`verify`/`cert`参数，所以Hermes通过自定义`httpx_client_factory`工厂函数注入TLS配置：
- 包装SDK默认行为（`follow_redirects=True`）
- 叠加`verify`（SSL验证开关）和`cert`（mTLS客户端证书）

```python
def _mcp_http_client_factory(headers=None, timeout=None, auth=None):
    kwargs = {
        "follow_redirects": True,
        "verify": _verify_for_factory,
    }
    # ... 叠加timeout/headers/auth/cert
    return httpx.AsyncClient(**kwargs)
```

##### 4. 跳过Content-Type预检
SSE路径跳过`_preflight_content_type()`预检，因为SSE端点合法返回`text/event-stream`，由SSE客户端自身处理。

#### SSE vs Streamable HTTP 对比

| 特性 | SSE | Streamable HTTP |
|------|-----|-----------------|
| **连接模型** | 一个长连接GET + 普通POST | 可复用HTTP请求，可选流式 |
| **服务器推送** | 原生支持（SSE事件） | 通过HTTP响应分块支持 |
| **现代性** | 旧方案，逐渐被淘汰 | 新的MCP标准推荐 |
| **代理/防火墙** | 长连接可能被代理切断（60s空闲） | 标准HTTP，代理友好 |
| **实现复杂度** | 较高（需要维护长连接） | 较简单（标准HTTP） |
| **sdk支持** | 所有版本 | mcp >= 1.0 |

#### 适用场景

✅ **使用场景**：
- 只支持SSE协议的旧版MCP服务器
- 需要服务器主动推送的场景
- 无法升级到Streamable HTTP的遗留系统

❌ **不推荐用于新服务**：新开发的MCP服务应该用Streamable HTTP。

---

## 三、Hermes MCP 实现的公共机制

### 3.1 专用后台事件循环线程架构

所有MCP Server连接都运行在**专用的单后台线程**上：
- 线程名：`mcp-event-loop`（daemon线程）
- 每个MCP Server对应一个`asyncio.Task`（`MCPServerTask.run()`）
- 主线程的工具调用通过`asyncio.run_coroutine_threadsafe()`调度到该循环
- 单线程asyncio模型避免了复杂的线程同步问题

```
主线程（对话循环）                mcp-event-loop 后台线程
     │                                 │
     │  call_tool()                    │
     │  run_coroutine_threadsafe() ───►│  asyncio 事件循环
     │                                 │  ┌───────────────┐
     │                                 │  │ MCPServerTask  │
     │  ◄──────────────────────────────│  │ (stdio/http/sse)│
     │    返回结果                      │  └───────────────┘
     │                                 │
```

### 3.2 保活机制（Keepalive）
**代码位置**：[_wait_for_lifecycle_event()](file:///e:/github/hermes-agent/tools/mcp_tool.py#L1680-L1760)

远程传输（HTTP/SSE）需要保活防止NAT/LB/服务器TTL过期断开连接：

- **默认间隔**：180s（3分钟），适配长LB/NAT空闲窗口（通常300-600s）
- **最小间隔**：5s（防止用户误配置太小）
- **短TTL服务器**：如Unreal Engine编辑器MCP约15s TTL，需要用户配置`keepalive_interval: 15`

保活探测策略（优先顺序）：
1. **首选**：`session.send_ping()`（协议级ping，几字节，最轻量）
2. **降级**：如果服务器返回`-32601 Method not found`（ping是可选的），降级为`session.list_tools()`作为探测
3. 记录`_ping_unsupported`标志，避免每次保活都触发-32601错误
4. 保活失败触发重连

### 3.3 自动重连机制
**代码位置**：[MCPServerTask.run()](file:///e:/github/hermes-agent/tools/mcp_tool.py#L2202-L2395)

```
首次连接：
  最多 3 次重试（_MAX_INITIAL_CONNECT_RETRIES）
  指数退避：1s → 2s → 4s → ... → 上限60s
  处理启动时的瞬态DNS/网络问题

已连接后断线：
  最多 5 次重连（_MAX_RECONNECT_RETRIES）
  指数退避，上限60s

不重试的情况：
  - OAuth认证失败（_is_auth_error检测）
  - NonMcpEndpointError（URL不是MCP端点）
  - shutdown事件触发
```

### 3.4 熔断器模式（Circuit Breaker）
**代码位置**：[mcp_tool.py:2448-2492](file:///e:/github/hermes-agent/tools/mcp_tool.py#L2448-L2492)

连续3次工具调用失败后，熔断器打开：
- 60s冷却期内直接返回"server unreachable"
- 防止模型陷入无限重试循环（曾经观测到90次重试的情况）

### 3.5 RPC 序列化锁（_rpc_lock）
每个`MCPServerTask`持有一个`asyncio.Lock`，将所有RPC请求（`list_tools`、`call_tool`等）串行化。

**原因**：stdio是单JSON-RPC流，如果在工具调用进行中收到`tools/list_changed`通知并触发`list_tools`，流可能卡死。HTTP传输也保守地应用了此锁。

### 3.6 动态工具发现（通知处理）
通过`ClientSession`的`message_handler`接收服务器通知：
- `ToolListChangedNotification` → 调度后台任务刷新工具列表（`_schedule_tools_refresh`），不阻塞通知处理
- `PromptListChangedNotification`/`ResourceListChangedNotification` → 仅日志记录（预留扩展）

### 3.7 Sampling 与 Elicitation 支持

| 功能 | 说明 |
|------|------|
| **Sampling** | MCP服务器可以请求客户端（Hermes）调用LLM，用于服务器端需要推理的场景（如摘要、提取）。通过辅助LLM客户端调用，支持速率限制、工具循环治理、模型白名单 |
| **Elicitation** | （mcp SDK 1.11.0+）MCP服务器可以向用户请求结构化输入（如支付授权、确认），通过Hermes审批系统路由到CLI/TUI/Telegram/Slack等多界面 |

---

## 四、Hermes 作为 MCP Server（对外提供服务）

Hermes不仅是MCP Client，也可以作为MCP Server对外暴露工具，目前只用**stdio传输**：

### 4.1 消息桥接 MCP Server
**文件**：[mcp_serve.py](file:///e:/github/hermes-agent/mcp_serve.py)
- 使用FastMCP框架，`server.run_stdio_async()`
- 对外暴露10个消息平台工具：conversations_list、conversation_get、messages_read、attachments_fetch、events_poll、events_wait、messages_send、channels_list、permissions_list_open、permissions_respond
- `EventBridge`组件：200ms轮询SQLite SessionDB发现新消息，维护内存事件队列
- 入口：`hermes mcp serve`命令

### 4.2 Hermes工具 MCP Server（供Codex使用）
**文件**：[hermes_tools_mcp_server.py](file:///e:/github/hermes-agent/agent/transports/hermes_tools_mcp_server.py)
- 向Codex app-server暴露Hermes的高级工具集（web_search、browser_*、vision_analyze、image_generate、skills、kanban_*等共27个工具）
- 通过闭包调用`handle_function_call(tool_name, kwargs)`分发到Hermes内部工具系统
- 日志输出到stderr（MCP协议要求stdout为协议通道），设置`HERMES_QUIET=1`避免横幅污染

---

## 五、如何选择传输方式？

```
你要连接的MCP Server在哪里？
│
├─ 本地运行（npx/uvx/本地命令）
│  └─► 用 Stdio 传输（最简单、最快、无需网络）
│
└─ 远程服务器（HTTP/HTTPS）
   │
   ├─ 新开发的/现代的MCP服务
   │  └─► 用 Streamable HTTP（默认，推荐，标准HTTP，代理友好）
   │
   └─ 旧的/只支持SSE的服务
      └─► 用 SSE 传输（配置 transport: sse）
```

---

## 六、完整配置示例

```yaml
# ~/.hermes/config.yaml
mcp_servers:
  # ===== Stdio 传输（本地） =====
  filesystem:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-filesystem", "e:\\github"]
    timeout: 120

  git:
    command: "uvx"
    args: ["mcp-server-git", "--repository", "."]

  # ===== Streamable HTTP 传输（远程，默认） =====
  brave-search:
    url: "https://mcp.brave.com/mcp"
    headers:
      Authorization: "Bearer ${BRAVE_API_KEY}"
    timeout: 60
    keepalive_interval: 60

  internal-tools:
    url: "https://internal-mcp.corp.example.com/mcp"
    client_cert: "C:/certs/client.pem"
    client_key: "C:/certs/client-key.pem"
    ssl_verify: true

  # ===== SSE 传输（旧版远程） =====
  legacy-sse:
    url: "http://localhost:9000/sse"
    transport: sse  # 必须显式指定
    timeout: 180
```

---

## 关键文件索引

| 文件 | 职责 |
|------|------|
| [mcp_tool.py](file:///e:/github/hermes-agent/tools/mcp_tool.py) | **核心**：MCP客户端、三种传输实现、重连、保活、OAuth、熔断器、sampling、elicitation |
| [mcp_oauth.py](file:///e:/github/hermes-agent/tools/mcp_oauth.py) | MCP OAuth 2.1 PKCE客户端实现 |
| [mcp_oauth_manager.py](file:///e:/github/hermes-agent/tools/mcp_oauth_manager.py) | OAuth管理器（token缓存、多server复用、401恢复） |
| [mcp_serve.py](file:///e:/github/hermes-agent/mcp_serve.py) | Hermes消息桥接MCP Server（stdio对外服务） |
| [hermes_tools_mcp_server.py](file:///e:/github/hermes-agent/agent/transports/hermes_tools_mcp_server.py) | Hermes工具MCP Server（供Codex调用，stdio） |
| [test_mcp_sse_transport.py](file:///e:/github/hermes-agent/tests/tools/test_mcp_sse_transport.py) | SSE传输回归测试（sse_read_timeout=300s、OAuth透传） |
