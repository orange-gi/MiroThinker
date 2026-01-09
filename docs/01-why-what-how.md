# Why / What / How：MiroThinker（含 MiroFlow Agent 与 MCP 工具生态）

## Why（为什么做）

**研究型 Agent 的核心瓶颈**不在“会不会回答”，而在“能否可靠地获取外部信息、在长链路交互中纠错、并把工具调用组织成可复现的工作流”。本项目围绕这一目标提供：

- **一个原生支持工具交互的研究型模型**：`MiroThinker`（强调 interactive scaling：更长上下文 + 更深的 agent-environment 交互）。
- **一套可复现的 agent 执行与评测框架**：`apps/miroflow-agent` 用统一的控制流驱动 LLM↔Tools 循环，并把日志、评测、trace 采集做成工程化管线。
- **一套可插拔的工具与服务器集合**：`libs/miroflow-tools` 用 MCP（Model Context Protocol）把“搜索/抓取/代码执行/阅读/视觉/音频/推理”等能力模块化，便于组合与替换。

简言之：**让模型能稳定地“查、读、算、写”，并把过程可视化、可评测、可训练。**

---

## What（是什么）

从仓库结构看，本项目主要由三层组成：

### 1) Agent 应用层：`apps/miroflow-agent`

- **入口**：`apps/miroflow-agent/main.py` 使用 Hydra 读取配置并运行单任务或基准评测。
- **管线**：`apps/miroflow-agent/src/core/pipeline.py`
  - `create_pipeline_components()`：按 `conf/agent/*.yaml` 组装 ToolManager（主 agent 与可选 sub-agent）与 OutputFormatter。
  - `execute_task_pipeline()`：创建 `TaskLog` → 初始化 LLM client → 初始化 `Orchestrator` → 执行 `run_main_agent()`。
- **核心控制流**：`apps/miroflow-agent/src/core/orchestrator.py`
  - 主循环：LLM 产出 tool call → 执行工具 → 把工具结果回灌给 LLM → 直到停止或达到上限。
  - 关键工程能力：**回滚重试、重复查询去重、上下文长度保护（keep_tool_result）、最终答案 boxed 重试、Demo 模式截断抓取结果**等。

### 2) 工具层：`libs/miroflow-tools`

- **ToolManager**：`libs/miroflow-tools/src/miroflow_tools/manager.py`
  - 连接多个 MCP server（stdio 或 SSE），拉取 tool schemas，并执行工具调用。
  - 包含一些安全/评测相关策略（例如：对 HuggingFace dataset/space 的抓取屏蔽，用于避免评测泄漏）。
- **MCP Servers**：`libs/miroflow-tools/src/miroflow_tools/mcp_servers/` 与 `dev_mcp_servers/`
  - 典型组合（v1.0/v1.5 最小配置）：`search_and_scrape_webpage`（Serper 搜索）、`jina_scrape_llm_summary`（Jina 抓取+小模型抽取）、`tool-python`（E2B 沙盒执行）。

### 3) 周边应用：trace / demo / 模板

- **trace 采集**：`apps/collect-trace/`
- **trace 可视化**：`apps/visualize-trace/`
- **部署/交互 Demo**：`apps/gradio-demo/`
- **Chat 模板**：`assets/qwen3_nonthinking.jinja`、`apps/lobehub-compatibility/chat_template.jinja`

---

## How（怎么用 / 怎么扩展）

### 1) 运行一个任务（最短路径）

1. 启动（自托管）模型服务（例如用 sglang/vLLM，见根目录 `README.md`）。
1. 安装依赖并配置密钥（Serper/Jina/E2B 等）：
   - 在 `apps/miroflow-agent/` 里 `uv sync`
   - `cp .env.example .env` 并填入环境变量
1. 运行：
   - `uv run python main.py llm=qwen-3 agent=mirothinker_v1.5_keep5_max200 llm.base_url=http://localhost:61002/v1`

### 2) 用配置“拼装一个 Agent”

配置入口在 `apps/miroflow-agent/conf/`：

- **LLM**：`conf/llm/*.yaml`（例如 `qwen-3.yaml`）
- **Agent**：`conf/agent/*.yaml`（例如 `mirothinker_v1.5_keep5_max200.yaml`）
  - `main_agent.tools`：声明要启用的 MCP servers（例如 `search_and_scrape_webpage`、`jina_scrape_llm_summary`、`tool-python`）
  - `tool_blacklist`：黑名单某些工具（例如禁用 `download_file_from_sandbox_to_local`）
  - `keep_tool_result`：只保留最近 K 次工具结果回灌给 LLM，以节省上下文预算（但完整过程仍写入日志）

### 3) 扩展工具（新增一个 MCP server）

在 `libs/miroflow-tools/src/miroflow_tools/mcp_servers/` 新增 server 文件，用 `FastMCP` 暴露工具；然后在 `apps/miroflow-agent/src/config/settings.py` 的 `create_mcp_server_parameters()` 增加该 server 的组装逻辑，并在 `conf/agent/*.yaml` 里启用它。

### 4) 复现与评测（benchmark）

`apps/miroflow-agent/conf/benchmark/*.yaml` 定义数据集与评测参数；运行时加 `benchmark=<name>` 触发，例如：

- `uv run python main.py llm=qwen-3 agent=mirothinker_v1.5_keep5_max200 benchmark=gaia-validation-text-103 ...`

---

## 你应当重点记住的 5 个实现特征（理解项目的“抓手”）

1. **控制流核心在 `Orchestrator`**：它决定“什么时候调用工具、什么时候回滚、什么时候总结”。
2. **工具通过 MCP 统一接入**：Tool schemas 会被拼进 system prompt，LLM 产出 `<use_mcp_tool>...</use_mcp_tool>` 触发调用。
3. **上下文管理靠 `keep_tool_result`**：并不删除推理轨迹，只减少回灌的工具输出量，从而支持更长 horizon。
4. **输出目标明确（boxed answer）**：最终 summary prompt 强制 `\\boxed{}`，并有重试与回退策略。
5. **全链路可观测**：`TaskLog` 记录每一步 LLM/工具调用、耗时、token 统计，支持后续 trace 可视化与训练数据采集。

