# Prompt 工程：本项目如何“教会模型用工具、控上下文、产出可评测答案”

本项目的 prompt 工程不是单一模板，而是一组**配合控制流**工作的约束集合：system prompt 负责声明工具与格式；Orchestrator 负责校验、回滚、去重、压缩；最终 summary prompt 负责把输出收敛到可评测的 `\\boxed{}`。

---

## 1) Prompt 的三层结构（从“全局规则”到“最终答案”）

### 1.1 工具系统提示（Tool-Use System Prompt）

入口：`apps/miroflow-agent/src/utils/prompt_utils.py` 的 `generate_mcp_system_prompt(date, mcp_servers)`

核心点：

- **把工具“以 schema 的形式”注入 system prompt**：ToolManager 会读取各 MCP server 的 tools 列表与 `inputSchema`，拼进 system prompt。
- **强制工具调用输出格式**：用 XML 标签包裹：
  - `<use_mcp_tool> ... </use_mcp_tool>`
  - `<server_name>...</server_name>`
  - `<tool_name>...</tool_name>`
  - `<arguments>{...json...}</arguments>`
- **硬性约束**：“一次只能用一个工具”“工具调用必须放在回复末尾”“arguments 必须是合法 JSON（字符串引号需转义）”

这层 prompt 的作用是：**把“可用动作空间（tools）”和“可解析协议（XML + JSON）”固定下来**，让后续解析与执行可确定。

### 1.2 Agent-specific Prompt（主 agent / 子 agent 角色目标）

入口：`generate_agent_specific_system_prompt(agent_type=...)`

- `main`：强调“逐步用工具获得完整、准确、推理充分的答案”
- `agent-browsing`：强调“只返回可验证事实，不要推断/大而化之的总结”

这层 prompt 的作用是：**把 agent 的“工作风格”与“允许的输出形态”区分开**（尤其是 browsing 子 agent 作为“事实检索器”）。

### 1.3 最终总结 Prompt（收敛到可评测输出）

入口：`generate_agent_summarize_prompt(task_description, agent_type=...)`

主 agent 的总结 prompt 有几个非常“硬”的设计：

- **明确禁止再用工具**：避免最后一步再开新搜索导致不可复现/不可控。
- **强调“如果前面已经有答案就直接抽取”**：降低重复计算与跑偏概率。
- **强制 `\\boxed{}`**：便于 benchmark 评测脚本稳定抽取答案。
- **强约束输出形式**：尽量短、不要句末标点、数字不带单位/逗号、列表用逗号分隔等。

配合 `OutputFormatter._extract_boxed_content()`，系统会从最终输出里提取最后一个 `\\boxed{...}` 作为最终答案，并且在缺失时触发重试/回退。

---

## 2) “Prompt 不是靠写得更长”，而是靠控制流兜底

关键实现集中在 `apps/miroflow-agent/src/core/orchestrator.py` 与 `src/utils/parsing_utils.py`：

### 2.1 工具调用解析策略（容错解析）

`parse_llm_response_for_tool_calls()` 支持三类输入：

- **XML MCP 格式**：用正则抽取 `<use_mcp_tool>...`；arguments 用 `safe_json_loads()` 解析（带 `json_repair` 回退）。
- **OpenAI tool_calls 列表**：从 `tool_call.function.name` 里按 `server-tool` 切分。
- **OpenAI response API 的 function_call 结构**：从 `output` 列表抽取。

这意味着：**同一套 Orchestrator，可以在不同 provider/协议下复用**（qwen/openai/anthropic），只要最终能落到统一的 `[{server_name, tool_name, arguments}]`。

### 2.2 “格式错了就回滚”的策略（让模型对协议更敏感）

Orchestrator 会对两类异常做回滚重试：

- **响应里出现 MCP tags 但没解析出 tool_calls**：视为“格式错误”，回滚本轮 assistant 并重试。
- **出现拒答关键词**：回滚并重试（避免某些模型在长任务中早退）。

这种设计的意图是：**用环境反馈训练时可理解为一种在线纠错信号**：输出协议不合格 → 无法行动 → 回滚 → 再给一次机会。

### 2.3 去重策略（避免死循环搜索）

Orchestrator 会把常见工具的 query 抽成 `query_str`（例如 `google_search_<q>`、`scrape_website_<url>` 等），若同一轮/同一会话中重复出现，触发回滚或警告后继续。

这本质上是 prompt 工程的“外置约束”：**不用把“不要重复”写很长，而是直接在执行器层面阻止重复带来的 token 浪费**。

### 2.4 上下文管理（keep_tool_result）

两处配合：

- LLM client 发请求前，会对发送给模型的 history 做“工具结果裁剪”（只保留最近 K 个工具结果），见 `OpenAIClient._remove_tool_result_from_messages(...)`（调用点在 `_create_message()`）。
- 每一轮执行完工具后，会用 `ensure_summary_context()` 估算 token，若可能超过上限则回滚最后一对 assistant/user，并提前进入 summary。

这使得项目能用更长的“行动步数”，而不被“工具输出占满上下文”卡死。

---

## 3) 模板（Jinja / Chat Template）在这里扮演什么角色

仓库里有两份重点模板：

### 3.1 `assets/qwen3_nonthinking.jinja`

定位：更偏向 **Qwen 系列 ChatML/工具调用格式** 的模板层，特点：

- 当存在 tools 时，会把 tool signatures 放进 `<tools> ... </tools>`。
- 工具调用输出用 `<tool_call> {"name": ..., "arguments": ...} </tool_call>`（不是 MCP 的 `<use_mcp_tool>`）。

这通常用于：**推理服务端（vLLM/sglang）侧的 chat template**，让模型按期望的 tool calling 协议输出。

### 3.2 `apps/lobehub-compatibility/chat_template.jinja`

定位：为 **前端/平台兼容**（例如 LobeHub）提供模板，把“工具调用”以 MCP 风格 `<use_mcp_tool>` 串起来，便于下游解析。

两者共同点：都是把“工具定义”前置到 system，并规范“工具调用”的序列化格式。差异在于：**具体标签协议不同**（`<tool_call>` vs `<use_mcp_tool>`）。

实践建议：

- 如果你的执行器按 MCP XML 解析（本项目默认路径），尽量选用/对齐 `<use_mcp_tool>`。
- 如果你走 OpenAI function calling 协议，优先让模型输出结构化 tool_calls；否则就必须在文本里解析（项目当前也支持）。

---

## 4) 本项目 prompt 工程的“可复用模式”（你改 prompt 时应遵守的几条规律）

- **模式 A：协议要可解析、且失败要可回滚**  
  写 prompt 时最重要的是让解析器“稳定抽取到 server/tool/args”。一旦你改了 tag/字段名，必须同步 `parse_llm_response_for_tool_calls()`。

- **模式 B：把“约束”尽量放到执行器层**  
  去重、超限、Unknown tool、HF 抓取屏蔽等都是执行器策略，优先级高于 prompt 文案。

- **模式 C：最终输出要“短、硬、可抽取”**  
  基准评测需要稳定抽取答案，`\\boxed{}` + 输出格式约束是为了减少评测噪声。

- **模式 D：不同子 agent 用不同目标函数**  
  browsing 子 agent 的 prompt 明确禁止推断，这是为了让它更像“检索/阅读器”，把推理留给主 agent 汇总。

---

## 5) 一个最小 mental model：prompt 与代码如何配合

你可以把整个系统理解为：

- **prompt**：定义“你可以做什么（tools）+ 你必须怎么说（格式）+ 你的目标是什么（角色）”
- **Orchestrator**：决定“你什么时候必须重说（回滚）+ 你什么时候不能再说（超限总结）+ 你说的是否重复（去重）”
- **OutputFormatter**：决定“最终答案如何被抽取（boxed）”

如果你要做 prompt 工程优化，优先顺序建议是：

1. 先保证解析稳定（格式、JSON、字段名一致）
2. 再优化工具选择质量（更好的任务分解/检索 query）
3. 最后再做文风/措辞优化（对最终准确性影响通常最小）

