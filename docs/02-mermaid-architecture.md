# Mermaid：流程/控制流 + 组件/视图（GitHub 可直接渲染）

> 本文所有图均使用 GitHub 支持的 Mermaid fenced code block：```mermaid

---

## 1) 端到端执行流程（从 `main.py` 到最终答案）

```mermaid
flowchart TD
  U[用户输入 / benchmark item] -->|Hydra 配置| MAIN[apps/miroflow-agent/main.py]

  MAIN --> PIPE[core/pipeline.execute_task_pipeline]
  PIPE --> LOG[TaskLog: 创建 + 记录 env/config]
  PIPE --> LLM[ClientFactory: OpenAIClient/AnthropicClient]
  PIPE --> TM[ToolManager: 连接 MCP servers]
  PIPE --> ORCH[Orchestrator.run_main_agent]

  ORCH -->|生成 system_prompt(含 tools schema)| SP[utils/prompt_utils.generate_mcp_system_prompt]
  ORCH --> LOOP{{主循环\nLLM ↔ Tools}}

  LOOP -->|LLM 返回文本 + 可能的 tool call| PARSE[utils/parsing_utils.parse_llm_response_for_tool_calls]
  PARSE -->|无 tool call 且满足停止条件| FINAL[最终总结/boxed]
  PARSE -->|有 tool call| CALL[ToolManager.execute_tool_call]

  CALL --> MCP[(MCP Server)]
  MCP --> EXT[(外部服务/环境\nSerper / Jina / E2B / ...)]
  EXT --> RESULT[tool result]
  RESULT --> FMT[OutputFormatter.format_tool_result_for_user]
  FMT -->|作为 user message 回灌| LOOP

  FINAL -->|summary prompt\n强制 \\boxed{}| OUT[Final summary + boxed answer]
  OUT --> SAVE[保存日志到 logs/]
```

---

## 2) 单轮交互的时序图（“一次工具调用”）

```mermaid
sequenceDiagram
  autonumber
  participant User as User/Benchmark
  participant Main as main.py + pipeline
  participant Orch as Orchestrator
  participant LLM as LLM Client
  participant TM as ToolManager
  participant MCP as MCP Server

  User->>Main: task_description (+ optional file)
  Main->>Orch: run_main_agent()
  Orch->>LLM: create_message(system_prompt, history, tool_schemas)
  LLM-->>Orch: assistant_response_text (may include <use_mcp_tool/>)
  Orch->>Orch: parse tool calls (regex + safe_json_loads)
  alt 有 tool call
    Orch->>TM: execute_tool_call(server, tool, args)
    TM->>MCP: call_tool(tool, args)
    MCP-->>TM: result / error
    TM-->>Orch: tool_result
    Orch->>LLM: 将 tool_result 作为 user message 回灌
  else 无 tool call
    Orch-->>Main: 进入最终总结或提前结束
  end
```

---

## 3) Orchestrator 主循环的“控制流/防护栏”（回滚、去重、上下文上限）

```mermaid
stateDiagram-v2
  [*] --> Init: 初始化 system_prompt + tool_defs
  Init --> TurnLoop

  state TurnLoop {
    [*] --> LLMCall
    LLMCall --> Parse
    Parse --> NoToolCall: tool_calls == empty
    Parse --> HasToolCall: tool_calls != empty

    NoToolCall --> StopOK: 正常结束(无工具需求)
    NoToolCall --> Rollback1: 检测到格式错误(MCP tags)或拒答关键词
    Rollback1 --> LLMCall: 回滚本轮 assistant 并重试

    HasToolCall --> DedupCheck: 提取 query_str 并做重复检测
    DedupCheck --> Rollback2: 检测到重复查询且未超回滚上限
    Rollback2 --> LLMCall: 回滚本轮并重试
    DedupCheck --> ExecTools: 允许执行

    ExecTools --> UnknownToolRollback: 返回 Unknown tool 且未超回滚上限
    UnknownToolRollback --> LLMCall
    ExecTools --> UpdateHistory: tool_result 回灌进 message_history
    UpdateHistory --> ContextCheck: ensure_summary_context
    ContextCheck --> TriggerSummary: 超出上下文上限
    ContextCheck --> LLMCall: 未超限继续下一轮
  }

  TriggerSummary --> FinalSummary
  StopOK --> FinalSummary

  FinalSummary --> BoxedRetry: 最终答案要求 \\boxed{}，必要时重试
  BoxedRetry --> Done
  Done --> [*]
```

---

## 4) 仓库“组件/视图”图（从目录理解责任边界）

```mermaid
flowchart LR
  subgraph Apps[apps/]
    A1[miroflow-agent\n执行/评测/日志]
    A2[collect-trace\n采集训练轨迹]
    A3[visualize-trace\n可视化日志/轨迹]
    A4[gradio-demo\n交互/部署示例]
    A5[lobehub-compatibility\nchat_template.jinja]
  end

  subgraph Libs[libs/]
    L1[miroflow-tools\nToolManager + MCP servers]
  end

  subgraph Assets[assets/]
    S1[qwen3_nonthinking.jinja\nQwen3 模板]
    S2[LOCAL-TOOL-DEPLOYMENT.md]
  end

  A1 -->|import| L1
  A2 -->|复用 system prompts / converter| A1
  A3 -->|读取 logs/| A1
  A5 -->|提供 ChatML/MCP 兼容模板| A1
  S1 -->|服务端/推理端模板| A1
```

---

## 5)（可选）工具生态视角：最小闭环（三个 MCP server）

```mermaid
flowchart TB
  subgraph Minimal["最小可用工具闭环（v1.0/v1.5 推荐）"]
    S[search_and_scrape_webpage\nSerper 搜索] --> W[网页 URL/摘要线索]
    W --> J[jina_scrape_llm_summary\n抓取 + 结构化抽取]
    J --> K[关键信息/引用片段]
    K --> P[tool-python\nE2B 沙盒执行/验证/计算]
  end
  Minimal --> LLM[LLM 生成答案/boxed]
```

