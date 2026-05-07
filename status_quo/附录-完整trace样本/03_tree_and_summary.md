# Trace 业务解读 - 完整数据视图

- traceId: `de39cbe95e46cb7f28d85db9cf3a4dc9`
- 总 span 数: **37**
- 总时长: **1228.2 ms**
- 起止时间: `2026-05-07T02:08:00.144063+00:00` → `2026-05-07T02:08:01.372221+00:00`
- 跨服务: agent-factory(Go) ↔ agent-executor(Python)

## 一、Span 拓扑树（按时间顺序）

```
[agent-factory  ] /api/agent-factory/v1/app/:app_key/chat/completion            op=-               1228.2ms  Unset  spanId=08216f0a
  [agent-factory  ] invoke_agent                                                  op=invoke_agent      14.2ms  Ok  spanId=1d944e1f
    [agent-factory  ] agentrunsvc.(*agentSvc).GetHistoryAndMsgIndex                 op=-                  1.2ms  Ok  spanId=29e5d286
      [agent-factory  ] conversationdbacc.(*ConversationRepo).Create                  op=-                  1.1ms  Unset  spanId=b69375a1
    [agent-factory  ] agentrunsvc.(*agentSvc).UpsertUserAndAssistantMsg             op=-                  4.4ms  Ok  spanId=390a1a25
      [agent-factory  ] conversationmsgdbacc.(*ConversationMsgRepo).Create            op=-                  1.0ms  Unset  spanId=b9a23fba
      [agent-factory  ] conversationdbacc.(*ConversationRepo).Update                  op=-                  1.8ms  Unset  spanId=76ded87f
      [agent-factory  ] conversationmsgdbacc.(*ConversationMsgRepo).Create            op=-                  1.4ms  Unset  spanId=7887c215
    [agent-factory  ] agentrunsvc.(*agentSvc).EnsureSandboxSession                  op=-                  5.8ms  Ok  spanId=4679c73e
    [agent-factory  ] agentrunsvc.(*agentSvc).GenerateAgentCallReq                  op=-                  0.0ms  Ok  spanId=a04bc193
    [agent-factory  ] agentrunsvc.(*agentSvc).Process                               op=-               1213.0ms  Ok  spanId=30dc6bc0
      [agent-factory  ] agentrunsvc.(*agentSvc).AfterProcess                          op=-                  0.3ms  Ok  spanId=60ee7554
      [agent-factory  ] agentrunsvc.(*agentSvc).handleMessageAndTempArea              op=-                  2.1ms  Ok  spanId=c6e719a1
        [agent-factory  ] conversationmsgdbacc.(*ConversationMsgRepo).Update            op=-                  0.9ms  Unset  spanId=21ce0692
        [agent-factory  ] conversationdbacc.(*ConversationRepo).GetByID                 op=-                  0.6ms  Unset  spanId=62223c30
        [agent-factory  ] conversationdbacc.(*ConversationRepo).Update                  op=-                  0.5ms  Unset  spanId=f8601355
  [agent-factory  ] v2agentexecutoraccess.(*v2AgentExecutorHttpAcc).Call          op=-                  0.1ms  Ok  spanId=9e12ffd6
    [agent-executor ] POST /api/agent-executor/v2/agent/run                         op=-                 44.0ms  Unset  spanId=722d6063
      [agent-executor ] process_options                                               op=-                  0.1ms  Ok  spanId=7de7592e
      [agent-executor ] check_agent_permission                                        op=-                  2.3ms  Ok  spanId=dadd0fe6
      [agent-executor ] result_output                                                 op=-               1203.2ms  Ok  spanId=8a2b2b3d
        [agent-executor ] invoke_agent                                                  op=invoke_agent    1202.0ms  Ok  spanId=71bfbbcd
          [agent-executor ] process_input                                                 op=-                  0.3ms  Ok  spanId=def6d2be
          [agent-executor ] process_tool_input                                            op=-                  0.2ms  Ok  spanId=70139c73
          [agent-executor ] run_dolphin                                                   op=-               1199.6ms  Ok  spanId=b7e29a0f
            [agent-executor ] build_llm_config                                              op=-                  0.2ms  Ok  spanId=ee3f56c7
            [agent-executor ] build                                                         op=-                  1.7ms  Ok  spanId=680e6e19
              [agent-executor ] search_memory_prompt                                          op=-                  0.1ms  Ok  spanId=c5e4986d
              [agent-executor ] temp_file_prompt                                              op=-                  0.1ms  Ok  spanId=2c3e1121
              [agent-executor ] get_doc_retrieval_prompt                                      op=-                  0.1ms  Ok  spanId=192ba722
              [agent-executor ] get_graph_retrieval_prompt                                    op=-                  0.1ms  Ok  spanId=f27f11e7
              [agent-executor ] get_context_prompt                                            op=-                  0.1ms  Ok  spanId=945dacfc
              [agent-executor ] get_related_questions_prompt                                  op=-                  0.1ms  Ok  spanId=42f4c7f1
            [agent-executor ] build_skills                                                  op=-                  0.2ms  Ok  spanId=34c64759
            [agent-executor ] build_tools                                                   op=-                  2.4ms  Ok  spanId=ac32a8af
            [agent-executor ] chat deepseek-chat                                            op=chat            1004.1ms  Ok  spanId=e03af4b4
            [agent-executor ] start_memory_build_thread                                     op=-                  0.2ms  Ok  spanId=614525f1
```

## 二、关键 span 业务字段

### `agent-factory` · `invoke_agent`
- duration: **14.2 ms**  status: `Ok`
- spanId: `1d944e1f03ce947d`  parentSpanId: `08216f0ad921ccc9`
- 业务属性:
  ```json
  {
    "gen_ai.agent.id": "01KR0327YK66SBYB3PNV6J4FC3",
    "gen_ai.agent.run_id": "01KR03284GQ7QH3539XGKRVTVF",
    "gen_ai.agent.version": "latest",
    "gen_ai.conversation.id": "01KR03284JNGR99KEN5RXV9TCW",
    "gen_ai.operation.name": "invoke_agent",
    "user_id": "mocked_user_id"
  }
  ```

### `agent-factory` · `/api/agent-factory/v1/app/:app_key/chat/completion`
- duration: **1228.2 ms**  status: `Unset`
- spanId: `08216f0ad921ccc9`  parentSpanId: `(root)`
- 业务属性:
  ```json
  {
    "gen_ai.conversation.id": "01KR03284JNGR99KEN5RXV9TCW",
    "http.client_ip": "<CLIENT_HOST>",
    "http.flavor": "1.1",
    "http.method": "POST",
    "http.route": "/api/agent-factory/v1/app/:app_key/chat/completion",
    "http.scheme": "http",
    "http.status_code": 200,
    "http.user_agent": "python-requests/2.32.3",
    "net.host.name": "agent-factory",
    "net.sock.peer.addr": "192.169.0.1",
    "net.sock.peer.port": 50080
  }
  ```

### `agent-factory` · `agentrunsvc.(*agentSvc).EnsureSandboxSession`
- duration: **5.8 ms**  status: `Ok`
- spanId: `4679c73eb8d9831a`  parentSpanId: `1d944e1f03ce947d`
- 业务属性:
  ```json
  {
    "code.filepath": "/build/src/domain/service/agentrunsvc/ensure_sandbox_session.go:19"
  }
  ```

### `agent-factory` · `agentrunsvc.(*agentSvc).GenerateAgentCallReq`
- duration: **0.0 ms**  status: `Ok`
- spanId: `a04bc193a9c6a02f`  parentSpanId: `1d944e1f03ce947d`
- 业务属性:
  ```json
  {
    "code.filepath": "/build/src/domain/service/agentrunsvc/chat_req.go:23",
    "gen_ai.agent.id": "01KR0327YK66SBYB3PNV6J4FC3",
    "gen_ai.agent.run_id": "01KR03284GQ7QH3539XGKRVTVF",
    "gen_ai.conversation.id": "01KR03284JNGR99KEN5RXV9TCW",
    "user_id": "mocked_user_id"
  }
  ```

### `agent-executor` · `POST /api/agent-executor/v2/agent/run`
- duration: **44.0 ms**  status: `Unset`
- spanId: `722d6063664a6490`  parentSpanId: `9e12ffd63c562dd8`
- 业务属性:
  ```json
  {
    "http.client_ip": "127.0.0.1",
    "http.method": "POST",
    "http.route": "/api/agent-executor/v2/agent/run",
    "http.status_code": 200,
    "http.url": "http://localhost:30778/api/agent-executor/v2/agent/run"
  }
  ```

### `agent-executor` · `process_input`
- duration: **0.3 ms**  status: `Ok`
- spanId: `def6d2be94bae117`  parentSpanId: `71bfbbcd93fdf486`
- 业务属性:
  ```json
  {
    "agent_id": "01KR0327YK66SBYB3PNV6J4FC3",
    "agent_run_id": "01KR03284GQ7QH3539XGKRVTVF",
    "gen_ai.agent.id": "01KR0327YK66SBYB3PNV6J4FC3",
    "user_id": "mocked_user_id"
  }
  ```

### `agent-executor` · `invoke_agent`
- duration: **1202.0 ms**  status: `Ok`
- spanId: `71bfbbcd93fdf486`  parentSpanId: `8a2b2b3d6bcebd05`
- 业务属性:
  ```json
  {
    "agent_id": "01KR0327YK66SBYB3PNV6J4FC3",
    "agent_run_id": "01KR03284GQ7QH3539XGKRVTVF",
    "gen_ai.agent.id": "01KR0327YK66SBYB3PNV6J4FC3",
    "gen_ai.conversation.id": "01KR03284JNGR99KEN5RXV9TCW",
    "gen_ai.operation.name": "invoke_agent",
    "user_id": "mocked_user_id"
  }
  ```

### `agent-executor` · `run_dolphin`
- duration: **1199.6 ms**  status: `Ok`
- spanId: `b7e29a0f1b09303e`  parentSpanId: `71bfbbcd93fdf486`
- 业务属性:
  ```json
  {
    "agent_id": "01KR0327YK66SBYB3PNV6J4FC3",
    "agent_run_id": "01KR03284GQ7QH3539XGKRVTVF",
    "gen_ai.agent.id": "01KR0327YK66SBYB3PNV6J4FC3",
    "gen_ai.conversation.id": "01KR03284JNGR99KEN5RXV9TCW",
    "user_id": "mocked_user_id"
  }
  ```

### `agent-executor` · `build_skills`
- duration: **0.2 ms**  status: `Ok`
- spanId: `34c6475965debde2`  parentSpanId: `b7e29a0f1b09303e`
- 业务属性:
  ```json
  {
    "agent_id": "01KR0327YK66SBYB3PNV6J4FC3",
    "gen_ai.agent.id": "01KR0327YK66SBYB3PNV6J4FC3",
    "user_id": "mocked_user_id"
  }
  ```

### `agent-executor` · `build_tools`
- duration: **2.4 ms**  status: `Ok`
- spanId: `ac32a8afacf782d4`  parentSpanId: `b7e29a0f1b09303e`
- 业务属性:
  ```json
  {}
  ```

### `agent-executor` · `chat deepseek-chat`
- duration: **1004.1 ms**  status: `Ok`
- spanId: `e03af4b4c9d1c930`  parentSpanId: `b7e29a0f1b09303e`
- 业务属性:
  ```json
  {
    "agent.block.type": "chat",
    "agent.llm.latency_ms": 1003,
    "agent.reasoning.step": 1,
    "gen_ai.agent.id": "agent_core_v2_01KR0327YK66SBYB3PNV6J4FC3",
    "gen_ai.conversation.id": "01KR03284GQ7QH3539XGKRVTVF",
    "gen_ai.operation.name": "chat",
    "gen_ai.output.type": "text",
    "gen_ai.request.model": "deepseek-chat",
    "gen_ai.request.temperature": 1,
    "gen_ai.response.finish_reasons": [
      "stop"
    ],
    "gen_ai.usage.input_tokens": 1607,
    "gen_ai.usage.output_tokens": 25
  }
  ```
- event `gen_ai.client.inference.operation.details` 字段: `['gen_ai.input.messages', 'gen_ai.output.messages']`

### `agent-executor` · `start_memory_build_thread`
- duration: **0.2 ms**  status: `Ok`
- spanId: `614525f1bfdfec28`  parentSpanId: `b7e29a0f1b09303e`
- 业务属性:
  ```json
  {
    "agent_id": "01KR0327YK66SBYB3PNV6J4FC3",
    "agent_run_id": "01KR03284GQ7QH3539XGKRVTVF",
    "gen_ai.agent.id": "01KR0327YK66SBYB3PNV6J4FC3",
    "user_id": "mocked_user_id"
  }
  ```

### `agent-factory` · `agentrunsvc.(*agentSvc).handleMessageAndTempArea`
- duration: **2.1 ms**  status: `Ok`
- spanId: `c6e719a1a4ce2ae6`  parentSpanId: `30dc6bc0abcc05a2`
- 业务属性:
  ```json
  {
    "code.filepath": "/build/src/domain/service/agentrunsvc/chat_post_process.go:374",
    "gen_ai.agent.id": "01KR0327YK66SBYB3PNV6J4FC3",
    "gen_ai.agent.run_id": "01KR03284GQ7QH3539XGKRVTVF",
    "gen_ai.conversation.id": "01KR03284JNGR99KEN5RXV9TCW",
    "user_id": "mocked_user_id"
  }
  ```


## 三、LLM 真实输入 / 输出（来自 chat span 的 inference event）

### LLM 输入 (`gen_ai.input.messages`)

#### role = `system`

```text

## Goals：
- 你需要：先仔细思考和分析用户的问题，然后决定由自己回答问题还是使用工具来处理问题，务必在调用工具前仔细思考。tools中的工具就是你可以使用的全部工具。

## Available Tools:
- builtin_skill_load(skill_id): Load the entry information for a skill (Phase 1 — must be called first). Returns the full SKILL.md content, a list of available script paths, and a list of available reference file paths. Whenever you have a skill_id, always call this function first before deciding any follow-up actions.
- builtin_skill_read_file(skill_id, file_path): Read the full content of a specific file inside a skill package (Phase 2 — optional). Only call this after builtin_skill_load has returned a path list or SKILL.md has referenced a file path. Reads one file at a time; batch reads are not supported. Supported text formats: .md .txt .json .yaml .yml .py .sh .js .ts
- builtin_skill_execute_script(skill_id, entry_shell): Execute a script inside the skill's runtime environment (Phase 3 — optional). Must first call builtin_skill_load and read SKILL.md before deciding to call this. The entry_shell command is specified in SKILL.md (e.g. 'python scripts/analyze.py'). Use the exact entry_shell value from SKILL.md — do not construct the command yourself. Not every skill requires script execution; some only need SKILL.md or reference files. In online mode the script runs in the execution factory sandbox; in local testing mode it runs directly in the current environment.
- _date: Get current date

## Built-in Skill Capabilities

You have access to three built-in tools for working with skills:

### 1. builtin_skill_load(skill_id)
- **Purpose**: Load a skill package and get its documentation
- **When to use**: Always call this first when you have a skill_id
- **Returns**: The full SKILL.md content plus lists of available scripts and reference files

### 2. builtin_skill_read_file(skill_id, file_path)
- **Purpose**: Read a specific file inside the skill package
- **When to use**: Optional. Only call after you have obtained a file path from builtin_skill_load or from SKILL.md
- **Note**: One file per call; cannot batch

### 3. builtin_skill_execute_script(skill_id, script_path)
- **Purpose**: Execute a script from the skill package
- **When to use**: Optional. Only call after reading SKILL.md and deciding that script execution is needed
- **Note**: Not all skills require script execution

### Usage Guidelines
1. If you have a skill_id, **always start with** `builtin_skill_load(skill_id)`
2. After reading SKILL.md, decide independently whether to call `read_file`, `execute_script`, both, or neither
3. Both `builtin_skill_read_file` and `builtin_skill_execute_script` are **optional steps**

---



### Tools Usage Guidelines：
- 仔细阅读每个工具的描述和参数要求
- 根据问题的具体需求选择最合适的工具
- 在调用工具前确保参数完整和正确
- 如果不确定工具用法，可以先尝试简单的调用来了解


你是一个测试助手

        
```

#### role = `user`

```text
如果有参考文档，结合参考文档回答用户的问题。如果没有参考文档，根据用户的问题回答。
用户的问题为：你好，用一句话介绍你自己
```

### LLM 输出 (`gen_ai.output.messages`)
```json
[
  {
    "role": "assistant",
    "content": "你好！我是你的测试助手，可以帮你加载和执行技能、读取文件、调用工具来解答问题或完成任务。"
  }
]
```


## 四、性能拆解

| 阶段 | 耗时 | 占比 |
|---|---|---|
| 入口 HTTP `chat/completion` | 1228.2 ms | 100% |
| └ agent-factory `invoke_agent` 编排 | 14.2 ms | 1.2% |
| └ Go→Python RPC `/v2/agent/run` | 44.0 ms | 3.6% |
|     └ `run_dolphin` 装配+推理 | 1199.6 ms | 97.7% |
|         └ ★ LLM `chat deepseek-chat` | 1004.1 ms | 81.8% |
