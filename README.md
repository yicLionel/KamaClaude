# KamaClaude：本地 AI Agent 学习与实践

> 本仓库是我基于开源项目 [KamaClaude](https://github.com/youngyangyang04/KamaClaude) 整理的个人学习版本。
>
> 项目的主要代码和设计来自原作者。我将其同步到个人仓库，用于阅读源码、运行实验和学习 AI Agent 的工程实现，不代表这是我的原创项目。

## 项目简介

KamaClaude 是一个使用 Python 实现的本地 AI Agent 运行时。

它参考 Claude Code 一类编程 Agent 的工作方式，将模型调用、工具执行、权限审批、事件追踪、会话管理和上下文压缩放进同一套运行流程中。

与只调用一次大模型 API 的聊天程序不同，KamaClaude 可以让模型根据目标连续执行多个步骤：

```text
用户输入
  → CLI / TUI
  → kama-core daemon
  → Agent Loop
  → LLM 推理
  → 工具调用
  → 权限检查
  → 执行结果回填
  → 继续推理或返回结果
```

项目采用 daemon 与客户端分离的结构：

```text
kama-core
  └── 监听 127.0.0.1:7437
          ↑
   JSON-RPC 2.0 + NDJSON
          ↑
     kama CLI / kama-tui
```

![KamaClaude TUI](docs/images/2026-06-10_14-30-58.jpg)

## 主要功能

- ReAct 风格的 Agent Loop
- 流式 LLM 输出与多步工具调用
- CLI 和 Textual TUI 两种交互方式
- 基于 JSON-RPC 2.0 与 NDJSON 的进程通信
- 工具参数校验、权限审批和失败重试
- Session、Thread 和 Notes 会话记录
- 上下文水位检测与 Compact 压缩
- Events 与 Trace 运行过程记录
- Skills、Subagents 和 MCP 扩展
- Pytest、Mypy strict 与 Ruff 工程检查

## 项目结构

```text
KamaClaude/
├── src/kama_claude/
│   ├── cli/                 # 命令行客户端
│   ├── tui/                 # Textual 终端界面
│   └── core/
│       ├── agents/          # Agent 配置与加载
│       ├── bus/             # 命令、事件及协议模型
│       ├── compact/         # 上下文压缩
│       ├── events/          # EventBus
│       ├── llm/             # Anthropic 模型接入
│       ├── mcp/             # MCP 客户端与服务管理
│       ├── memory/          # 记忆加载
│       ├── permissions/     # 工具权限控制
│       ├── session/         # 会话管理与持久化
│       ├── skills/          # Skills 系统
│       ├── subagent/        # 子 Agent
│       ├── tools/           # 内置工具与注册中心
│       ├── trace/           # 调用链追踪
│       └── transport/       # TCP IPC 通信
├── tests/                   # 单元测试与集成测试
├── WIRE_PROTOCOL.md         # IPC 协议说明
├── RUNBOOK.md               # 运行与排错手册
├── pyproject.toml
└── Makefile
```

## 环境要求

- Python 3.12
- [uv](https://docs.astral.sh/uv/)
- Anthropic API Key

## 快速开始

### 1. 克隆项目

```bash
git clone https://github.com/yicLionel/KamaClaude.git
cd KamaClaude
```

### 2. 安装依赖

```bash
uv sync
```

### 3. 配置环境变量

```bash
cp .env.example .env
```

在 `.env` 中填写自己的 API Key：

```env
ANTHROPIC_API_KEY=your_api_key_here
```

不要把包含真实密钥的 `.env` 提交到 Git。

### 4. 启动 Core Daemon

前台启动：

```bash
uv run kama-core
```

也可以在后台启动：

```bash
uv run kama core start
uv run kama core status
```

### 5. 检查连接

在另一个终端中执行：

```bash
uv run kama ping
```

如果 daemon 正常运行，会返回 `pong` 和运行时间等信息。

## 使用方式

### 执行一次 Agent 任务

```bash
uv run kama run --goal "读取当前目录并总结项目结构"
```

### 进入多轮对话

```bash
uv run kama chat
```

### 启动 TUI

```bash
uv run kama-tui
```

### 查看运行 Trace

```bash
uv run kama trace
```

### 停止后台 Daemon

```bash
uv run kama core stop
```

## 开发与测试

运行单元测试：

```bash
uv run pytest tests/unit -v
```

运行集成测试：

```bash
uv run pytest tests/integration -v
```

部分真实模型集成测试需要配置 `ANTHROPIC_API_KEY`，未配置时会自动跳过。

运行代码检查：

```bash
uv run ruff check src tests scripts
uv run mypy src
```

执行项目提供的完整验证流程：

```bash
make verify-s0
```

修改 IPC 数据模型后，需要重新生成协议文档：

```bash
uv run python scripts/gen_protocol_doc.py
```

## 我的学习重点

我把这个仓库作为 AI Agent 工程的源码学习项目，主要关注以下问题：

1. Agent 如何把一个目标拆成多次模型调用和工具执行。
2. CLI、TUI 与常驻 daemon 如何通过 IPC 协作。
3. 工具执行前如何完成参数校验和权限审批。
4. EventBus、events 和 trace 如何记录完整执行过程。
5. 长会话如何管理历史消息、记忆和上下文窗口。
6. Skills、Subagents 与 MCP 如何接入现有 Agent Loop。

建议阅读时从以下调用链开始：

```text
CLI / TUI
  → SocketClient
  → SocketServer
  → CoreApp
  → SessionManager
  → AgentRunner
  → AgentLoop
  → AnthropicProvider / ToolRegistry
```

先跟踪一次完整的 `kama run`，再分别阅读权限、会话、压缩和扩展模块，会比直接逐文件阅读更容易建立整体认识。

## 使用提醒

- 本项目主要用于学习和实验，不建议直接作为生产系统使用。
- 调用真实模型可能产生 API 费用。
- Agent 内置文件读写和命令执行能力，运行前应检查工作目录及权限配置。
- 请勿提交 `.env`、API Key、Token 或其他敏感信息。
- 如果你基于本仓库继续开发，建议在 README 中记录自己的修改内容，区分原项目功能与个人实现。

## 项目来源

- 原项目：[youngyangyang04/KamaClaude](https://github.com/youngyangyang04/KamaClaude)
- 个人学习仓库：[yicLionel/KamaClaude](https://github.com/yicLionel/KamaClaude)

感谢原作者提供项目代码和学习资料。本仓库保留原项目的版权信息，不将原作者的工作声明为个人原创。

## License

本项目使用 [MIT License](LICENSE)。

原项目版权归原作者所有，具体信息以仓库中的 `LICENSE` 文件为准。
```

没有修改本地文件，也没有运行代码测试。
