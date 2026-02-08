<div align="center">

# AgentRecorder

**Agent 轨迹录制工具 - 将 Agent 框架日志转换为标准化轨迹格式**
**Convert agent framework logs into a standardized trajectory format**

[![PyPI](https://img.shields.io/pypi/v/knowlyr-recorder?color=blue)](https://pypi.org/project/knowlyr-recorder/)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![MCP](https://img.shields.io/badge/MCP-3_Tools-purple.svg)](#mcp-server)

[快速开始](#快速开始) · [支持的框架](#支持的框架) · [Schema 文档](#schema-文档) · [MCP Server](#mcp-server)

</div>

---

**GitHub Topics**: `agent`, `trajectory`, `recorder`, `openhands`, `swe-agent`, `mcp`, `benchmark`

将 OpenHands、SWE-agent 等 Agent 框架的执行日志转换为统一的标准化轨迹格式，便于分析、对比和复现。

## 核心能力 / Core Capabilities

```
Agent 日志 (OpenHands/SWE-agent/...) → 适配器解析 → 标准化 Trajectory → JSONL 输出
```

### 设计特点 / Design Highlights

| 特点 | 说明 |
|------|------|
| **适配器模式** | 每个 Agent 框架一个适配器，易于扩展 |
| **标准化 Schema** | 统一的 Pydantic 数据模型，类型安全 |
| **JSONL 输出** | 一行一条轨迹，便于流式处理 |
| **CLI + MCP** | 命令行和 MCP Server 双入口 |

### 解决的问题 / Problems Solved

| 痛点 | 现状 | AgentRecorder |
|------|------|---------------|
| **格式不统一** | 每个框架自定义日志格式 | 统一 Trajectory Schema |
| **难以对比** | 不同框架结果无法直接比较 | 标准化后可直接对比 |
| **复现困难** | 日志缺乏结构化 | 完整记录每步 thought/action/result |
| **分析耗时** | 手动解析各种日志 | 一键批量转换 |

## 安装 / Installation

```bash
pip install knowlyr-recorder
```

可选依赖：

```bash
pip install knowlyr-recorder[mcp]   # MCP 服务器
pip install knowlyr-recorder[dev]   # 开发依赖
pip install knowlyr-recorder[all]   # 全部功能
```

## 快速开始 / Quick Start

### CLI 使用 / CLI Usage

```bash
# 转换单个日志文件
knowlyr-recorder convert ./logs/output.jsonl -f openhands -o trajectory.jsonl

# 验证日志格式
knowlyr-recorder validate ./logs/output.jsonl

# 批量转换目录
knowlyr-recorder batch ./logs/ -f openhands -o trajectories.jsonl

# 查看 Schema
knowlyr-recorder schema
```

### Python API 使用 / Python API

```python
from agentrecorder import Recorder
from agentrecorder.adapters import OpenHandsAdapter

# 创建录制器
recorder = Recorder(OpenHandsAdapter())

# 转换单个文件
trajectory = recorder.convert("path/to/log.jsonl")

# 批量转换
trajectories = recorder.convert_batch("path/to/logs/")

# 保存为 JSONL
trajectory.to_jsonl("output/trajectories.jsonl")

# 从 JSONL 加载
from agentrecorder.schema import Trajectory
loaded = Trajectory.from_jsonl("output/trajectories.jsonl")
```

## 支持的框架 / Supported Frameworks

| 框架 | 状态 | 适配器 | 日志格式 |
|------|------|--------|----------|
| [OpenHands](https://github.com/All-Hands-AI/OpenHands) | Stub | `OpenHandsAdapter` | JSONL (action/observation) |
| [SWE-agent](https://github.com/princeton-nlp/SWE-agent) | Stub | `SWEAgentAdapter` | JSON (history/info) |
| Aider | 计划中 | - | - |
| Moatless | 计划中 | - | - |

### 添加新适配器 / Adding New Adapters

```python
from agentrecorder.adapters.base import BaseAdapter
from agentrecorder.schema import Trajectory

class MyAgentAdapter(BaseAdapter):
    def parse(self, log_path: str) -> Trajectory:
        # 实现日志解析逻辑
        ...

    def validate(self, log_path: str) -> bool:
        # 实现格式验证逻辑
        ...
```

## Schema 文档 / Schema Documentation

### Trajectory 数据模型

```
Trajectory
├── task: TaskInfo          # 任务信息
│   ├── task_id             # 任务 ID
│   ├── description         # 任务描述
│   ├── type                # 任务类型 (bug_fix, code_edit, ...)
│   ├── language            # 编程语言
│   ├── difficulty          # 难度等级
│   ├── repo                # 目标仓库
│   ├── base_commit         # 基础 commit
│   └── test_command        # 测试命令
├── agent: str              # Agent 框架名称
├── model: str              # LLM 模型名称
├── steps: list[Step]       # 执行步骤列表
│   └── Step
│       ├── step_id         # 步骤编号
│       ├── thought         # Agent 思考过程
│       ├── tool_call       # 工具调用
│       │   ├── name        # 工具名称
│       │   └── parameters  # 调用参数
│       ├── tool_result     # 工具结果
│       │   ├── output      # 输出内容
│       │   ├── exit_code   # 退出码
│       │   └── error       # 错误信息
│       ├── timestamp       # 时间戳
│       └── token_count     # Token 消耗
├── outcome: Outcome        # 执行结果
│   ├── success             # 是否成功
│   ├── tests_passed        # 通过测试数
│   ├── tests_failed        # 失败测试数
│   ├── total_steps         # 总步骤数
│   └── total_tokens        # 总 Token 数
└── metadata: dict          # 额外元数据
```

### JSONL 输出示例

```jsonl
{"task":{"task_id":"django__django-11099","description":"Fix URL resolver","type":"bug_fix","language":"python","difficulty":"medium","repo":"django/django","base_commit":"abc123","test_command":"python -m pytest tests/"},"agent":"openhands","model":"claude-sonnet-4-20250514","steps":[{"step_id":1,"thought":"Let me look at the failing test","tool_call":{"name":"bash","parameters":{"command":"cat tests/test_urls.py"}},"tool_result":{"output":"...","exit_code":0,"error":null},"timestamp":"2026-01-15T10:30:00Z","token_count":150}],"outcome":{"success":true,"tests_passed":42,"tests_failed":0,"total_steps":8,"total_tokens":12500},"metadata":{"run_id":"run-001"}}
```

---

## MCP Server / Claude Integration

在 Claude Desktop / Claude Code 中直接使用。

### 配置 / Config

添加到 `~/Library/Application Support/Claude/claude_desktop_config.json`：

```json
{
  "mcpServers": {
    "knowlyr-recorder": {
      "command": "uv",
      "args": ["--directory", "/path/to/agent-recorder", "run", "python", "-m", "agentrecorder.mcp_server"]
    }
  }
}
```

### 可用工具 / Tools

| 工具 | 功能 |
|------|------|
| `convert_logs` | 将 Agent 日志转换为标准化轨迹格式 |
| `validate_logs` | 验证日志文件格式 |
| `get_schema` | 返回轨迹的 JSON Schema 定义 |

---

## 命令参考 / Command Reference

| 命令 | 功能 |
|------|------|
| `knowlyr-recorder convert <log> -f <framework>` | 转换单个日志文件 |
| `knowlyr-recorder validate <log>` | 验证日志格式 |
| `knowlyr-recorder batch <dir> -f <framework> -o <out>` | 批量转换 |
| `knowlyr-recorder schema` | 输出 JSON Schema |

---

## 项目架构 / Project Structure

```
src/agentrecorder/
├── __init__.py          # 包入口
├── schema.py            # 标准化轨迹数据模型
├── recorder.py          # 核心录制器
├── cli.py               # CLI 命令行
├── mcp_server.py        # MCP Server (3 工具)
└── adapters/
    ├── __init__.py      # 适配器注册
    ├── base.py          # 适配器基类
    ├── openhands.py     # OpenHands 适配器
    └── sweagent.py      # SWE-agent 适配器
```

---

## AI Data Pipeline 生态

> 覆盖 AI 数据工程全流程，均支持 CLI + MCP，可独立使用也可组合成流水线。

| Tool | Description | Link |
|------|-------------|------|
| **AI Dataset Radar** | Competitive intelligence for AI training datasets | [GitHub](https://github.com/liuxiaotong/ai-dataset-radar) |
| **DataRecipe** | Reverse-engineer datasets into annotation specs & cost models | [GitHub](https://github.com/liuxiaotong/data-recipe) |
| **DataSynth** | Seed-to-scale synthetic data generation | [GitHub](https://github.com/liuxiaotong/data-synth) |
| **DataLabel** | Lightweight, serverless HTML labeling tool | [GitHub](https://github.com/liuxiaotong/data-label) |
| **DataCheck** | Automated quality checks & anomaly detection | [GitHub](https://github.com/liuxiaotong/data-check) |
| **AgentRecorder** | Convert agent logs to standardized trajectories | You are here |

---

## License

[MIT](LICENSE)

---

<div align="center">
<sub>将 Agent 执行日志转化为可分析、可复现的标准化轨迹</sub>
</div>
