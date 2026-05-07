# Trace 业务解读 — Retrieval-heavy 对照样本

> 与 `03_tree_and_summary.md` 配对：03 是"什么都没干（无 retrieval）"的极小样本，本文是"跑了完整工具链 + 检索三层"的真实样本，用来体现 §4.3.1 / §4.3.2 / §6.6 描述的能力。

- **conversation**: `01KQ7129HD1XGB1XWBT9QTF58W`
- **traceId**: `3eb5c7aa3b3671242a9346e18166c689`
- **agent**: `agent_core_v2_01KQ712953BF6K4GX3RCVPVA0M`
- **总 span 数**: **50**（用 `_search { term: traceId.keyword }` 拉的全量；`by-conversation` 只能拉到 14 条）
- **总时长**: **75740 ms**
- **跨服务**: agent-factory(Go) ↔ agent-executor(Python) ↔ ontology-query-svc / dolphin
- **采集时间**: 2026-04-27（截至 2026-05-07 仍在线可查）

---

## 一、用户问题（system + user message 摘录）

> System prompt 摘录（~80 行 tool 描述 + 业务规则）：
>
> ```
> ## Goals：
> - 你需要：先仔细思考和分析用户的问题，然后决定由自己回答问题还是使用工具来处理问题，务必在调用工具前仔细思考。
> ## Available Tools:
> - find_skills(instance_identities, object_type_id, ...): 技能召回
> - search_schema(query, ...): schema 检索
> - query_object_instance(ot_id, condition, ...): 行级实例查询
> - query_instance_subgraph(relation_type_paths, ...): 子图遍历
> - builtin_skill_load(skill_id): 加载技能包
> - _date: 当前日期
> ```

> User message：
>
> > **Material MAT-001 hit critical stock level. Use find_skills to identify applicable skills, query the BKN for evidence (supplier capability, etc.), pick…**

也就是一个"供应链处置助手"的真实业务场景，要求 agent 自主跑工具链查证据 + 给处置方案。

---

## 二、Span 拓扑树（按时间顺序）

```
/api/agent-factory/v1/app/:app_key/chat/completion        75740.6ms Uns [agent-factory ]
  └ invoke_agent                                             21.8ms Ok  [agent-factory ]
    └ agentrunsvc.(*agentSvc).GetHistoryAndMsgIndex            1.1ms Ok  [agent-factory ]
      └ conversationdbacc.(*ConversationRepo).Create             1.0ms Uns [agent-factory ]
    └ agentrunsvc.(*agentSvc).UpsertUserAndAssistantMsg        2.7ms Ok  [agent-factory ]
      └ conversationmsgdbacc.(*ConversationMsgRepo).Create       1.0ms Uns [agent-factory ]
      └ conversationdbacc.(*ConversationRepo).Update             0.7ms Uns [agent-factory ]
      └ conversationmsgdbacc.(*ConversationMsgRepo).Create       0.9ms Uns [agent-factory ]
    └ agentrunsvc.(*agentSvc).EnsureSandboxSession             8.3ms Ok  [agent-factory ]
    └ agentrunsvc.(*agentSvc).GenerateAgentCallReq             0.0ms Ok  [agent-factory ]
    └ agentrunsvc.(*agentSvc).Process                      75717.9ms Ok  [agent-factory ]
      └ agentrunsvc.(*agentSvc).AfterProcess                     0.4ms Ok  [agent-factory ]
      └ agentrunsvc.(*agentSvc).handleMessageAndTempArea         2.9ms Ok  [agent-factory ]
        └ conversationmsgdbacc.(*ConversationMsgRepo).Update       1.6ms Uns [agent-factory ]
        └ conversationdbacc.(*ConversationRepo).GetByID            0.6ms Uns [agent-factory ]
        └ conversationdbacc.(*ConversationRepo).Update             0.5ms Uns [agent-factory ]
  └ v2agentexecutoraccess.(*v2AgentExecutorHttpAcc).Call     0.1ms Ok  [agent-factory ]
    └ POST /api/agent-executor/v2/agent/run                   12.0ms Uns [agent-executor]
      └ process_options                                          0.3ms Ok  [agent-executor]
      └ check_agent_permission                                   3.7ms Ok  [agent-executor]
      └ result_output                                        75692.9ms Ok  [agent-executor]
        └ invoke_agent                                       75690.3ms Ok  [agent-executor]
          └ process_input                                        0.3ms Ok  [agent-executor]
          └ process_tool_input                                   0.2ms Ok  [agent-executor]
          └ run_dolphin                                      75686.7ms Ok  [agent-executor]
            └ build_llm_config                                     0.1ms Ok  [agent-executor]
            └ build (search_memory_prompt / get_doc_retrieval_prompt …)  1.0ms Ok  [agent-executor]
            └ build_skills                                         0.1ms Ok  [agent-executor]
            └ build_tools                                         23.3ms Ok  [agent-executor]
            └ start_memory_build_thread                            0.6ms Ok  [agent-executor]

execute_tool _date                                          1.3ms Ok  [agent-executor] tool=_date
execute_tool find_skills                                  104.2ms Ok  [agent-executor] tool=find_skills · args={"object_type_id":"material","instance_identities":[{"sku":"MAT-001"}],...}
execute_tool search_schema                                115.5ms Ok  [agent-executor] tool=search_schema · args={"query":"material supplier stock capability","response_format":"json"}
execute_tool query_object_instance                         40.7ms Ok  [agent-executor] tool=query_object_instance · args={"ot_id":"material","condition":{"field":"sku","operation":"==","value":"MAT-001"}}
execute_tool query_object_instance                         36.5ms Ok  [agent-executor] tool=query_object_instance · args={"ot_id":"supplier","condition":{"field":"supplier_id","operation":"==","value":"SUP-1"}}
execute_tool query_object_instance                         35.5ms Ok  [agent-executor] tool=query_object_instance · args={"ot_id":"skills","condition":{"field":"skill_id","operation":"==","value":"substitute_swap"}}
execute_tool builtin_skill_load                            13.8ms Ok  [agent-executor] tool=builtin_skill_load · args={"skill_id":"substitute_swap"}                ❌ skill 不存在
execute_tool query_object_instance                         38.8ms Ok  [agent-executor] tool=query_object_instance · args={"ot_id":"material","condition":{"field":"material_risk","operation":"==","value":"critical"}}
execute_tool builtin_skill_load                            23.6ms Ok  [agent-executor] tool=builtin_skill_load · args={"skill_id":"supplier_expedite"}              ❌ skill 不存在
execute_tool builtin_skill_load                            23.4ms Ok  [agent-executor] tool=builtin_skill_load · args={"skill_id":"standard_replenish"}            ❌ skill 不存在
execute_tool query_instance_subgraph                       32.1ms Ok  [agent-executor] tool=query_instance_subgraph · args={"relation_type_paths":[{"object_types":["material","supplier"],"relation_types":["supplied_by"]}]}  ❌ 400 BadRequest
execute_tool query_instance_subgraph                       21.4ms Ok  [agent-executor] tool=query_instance_subgraph · args={"relation_type_paths":[{"object_types":[{"object_type_id":"material"},...]}]}  ❌ 400 BadRequest
execute_tool query_object_instance                         55.1ms Ok  [agent-executor] tool=query_object_instance · args={"ot_id":"supplier","response_format":"json","limit":10}
chat deepseek-v3.2                                      20172.2ms Ok  [agent-executor] model=deepseek-v3.2 · in=9489 · out=612 · finish=stop
```

> 注：`execute_tool *` 与 `chat` 这 14 条 span 在 OpenSearch 里有 `gen_ai.conversation.id`，所以 `by-conversation` 能拉到 14/50；其余 36 条管道 span（HTTP 入口 / Go 内部 RPC / dolphin 装配等）需要走 traceId 兜底才能拉全。

---

## 三、数据证据链（13 步）

每行的"命中数据"都是 `gen_ai.tool.call.result` 解析出来的真实内容（字段名、值、`_score` 命中分都齐）。

| 步 | 工具 | 对象类 | 过滤 / 参数 | 耗时 | 命中数据（关键字段） |
|---|---|---|---|---:|---|
| 1 | `_date` | — | — | 1.3ms | `2026-04-27` |
| 2 | `find_skills` | material | identities=[{sku:MAT-001}] | 104ms | 1 个候选：`substitute_swap`（替代料切换 — Pick best substitute via Python scoring then call MES）|
| 3 | `search_schema` | — | query="material supplier stock capability" | 116ms | object_types: `material` / `supplier` / `skills`；relation_types: `supplied_by` / `applicable_skill` |
| 4 | `query_object_instance` | material | sku == MAT-001 | 41ms | name=Battery Cell, current_stock=40, safety_stock=100, material_risk=critical, supplier_id=SUP-1, bound_skill_id=substitute_swap, **`_score=1.386`** |
| 5 | `query_object_instance` | supplier | supplier_id == SUP-1 | 37ms | name=Acme Corp, capability=normal, **`_score=0.981`** |
| 6 | `query_object_instance` | skills | skill_id == substitute_swap | 36ms | name=替代料切换, description=Pick best substitute via Python scoring..., **`_score=0.981`** |
| 7 | `builtin_skill_load` | — | skill_id=substitute_swap | 14ms | ❌ `builtin_skill_load failed for skill 'substitute_swap': ...`（技能包不存在）|
| 8 | `query_object_instance` | material | material_risk == critical | 39ms | 3 条同级 critical 物料（含 MAT-001 自身 + 另外 2 条）|
| 9 | `builtin_skill_load` | — | skill_id=supplier_expedite | 24ms | ❌ 同上失败 |
| 10 | `builtin_skill_load` | — | skill_id=standard_replenish | 23ms | ❌ 同上失败 |
| 11 | `query_instance_subgraph` | — | paths=[material, supplier, supplied_by]（字符串数组形式）| 32ms | ❌ 400：`Binding Parameter Failed: cannot unmarshal string into Go struct field QueryRelationTypePath.relation_type_paths.object_types of type interfaces.ObjectTypeWithKeyField` |
| 12 | `query_instance_subgraph` | — | paths（修正成对象数组形式后再试）| 21ms | ❌ 同样 400 失败 |
| 13 | `query_object_instance` | supplier | top 10 | 55ms | 3 条 supplier 全表 |
| → | **chat** | — | model=deepseek-v3.2, T=0.7 | **20.2s** | input 9489 tok, output 612 tok, finish=stop |

---

## 四、LLM 输入输出（来自 chat span 的 inference event）

### 4.1 Input messages 结构

`gen_ai.input.messages` 是完整 28 条对话历史 + 当前问题，包括：
- 1 条 `system`（含 ~80 行工具描述）
- 1 条 `user`（业务剧本完整原文，未截断）
- 12 轮 `assistant ↔ tool`（每个工具调用 + 返回都完整入库）
- 上一轮 assistant 的中间回答（含"步骤 2：查询知识网络获取证据"等推理片段）

总 input = **9489 token**（每次 LLM 调用都把工具历史完整压回 prompt，所以这是常规量级）。

### 4.2 Assistant 最终输出（output_messages，612 tok）

```text
## 分析总结

基于查询到的知识网络证据，我可以提供以下处置方案：

### 证据收集结果：
1. **物料 MAT-001 信息**：
   - 名称：Battery Cell（电池单元）
   - 当前库存：40
   - 安全库存：100
   - 物料风险等级：critical（关键）
   - 供应商 ID：SUP-1
   - 绑定技能 ID：substitute_swap（替代料切换）

2. **供应商 SUP-1 信息**：
   - 名称：Acme Corp
   - 能力：normal（正常交付能力）

3. **其他关键物料**：
   - MAT-002（Power Modul...
   [输出后续被截断，本附录不展开]
```

→ **重要观察**：assistant 给出的"分析总结"里准确还原了 trace 第 4-5 步 `query_object_instance` 的命中行（current_stock=40 / safety_stock=100 / supplier=Acme Corp / capability=normal），同时也基于第 7-12 步的失败信息知道"技能包没找到"。**模型的每一句话都能在 trace span 上找到原始证据**，这就是"知识引用溯源"的实例。

---

## 五、性能拆解

| 类别 | 累计耗时 | 次数 | 说明 |
|---|---:|---:|---|
| LLM (chat) | **20172ms** | 1 | deepseek-v3.2，input 9489 tok，**这是单次延迟的最大块** |
| tool: query_object_instance | 207ms | 5 | 行级实例查询 |
| tool: search_schema | 116ms | 1 | schema 检索 |
| tool: find_skills | 104ms | 1 | 技能召回 |
| tool: builtin_skill_load | 61ms | 3 | 全部失败 |
| tool: query_instance_subgraph | 53ms | 2 | 全部失败 |
| db (ConversationRepo / Msg) | 6ms | 7 | agent-factory 持久化 |
| prompt-build (`*_prompt`) | 0ms | 6 | 模板装配（每个 0.1ms） |
| tool: _date | 1.3ms | 1 | — |
| **工具调用累计** | **~542ms** | 13 | |
| **可见工作累计** | **~20.7s** | — | |
| **不可见的"black hole"** | **~55s** | — | LLM 实际不止 20.2s？或 dolphin 内部还有未埋点的等待？需 vega/BKN OTel 埋点才能拆 |

总 75.7s ≈ 20.2s LLM + ~0.6s 工具 + ~55s 未拆解时间。**~55s 没有 span 覆盖**，最可能的归属是 LLM 多次 streaming 调用之间的等待 / dolphin 内部 sequencer 等待 —— 这条 trace 自带性能可观测能力**到 dolphin 边界为止**，再往深就要等 §五"vega/BKN OTel 埋点"补齐。

---

## 六、与样本 03（"你好"问候）的对照

| 维度 | 03（无 retrieval） | 04（retrieval-heavy，本样本） |
|---|---|---|
| 用户问题 | "你好，用一句话介绍你自己" | "MAT-001 critical stock level，跑工具链查证据出处置方案" |
| 总 span 数 | 37 | **50** |
| 总耗时 | 1228ms | **75740ms** |
| `execute_tool` 次数 | 0 | **13** |
| 涉及对象类 | — | material / supplier / skills（3 个）|
| 命中行级数据 | — | MAT-001 (Battery Cell) / SUP-1 (Acme Corp) / substitute_swap |
| 失败步骤 | 0 | 5（3 次 skill_load + 2 次 subgraph）|
| LLM token | in=1607 / out=25 | **in=9489 / out=612** |
| LLM 占总耗时 | 81.7% | 26.7% |
| `by-conversation` 拉到 / 应有 | 17/37（46%） | **14/50（28%）** |
| 演示的能力 | 跨语言串联 + 计费 + 错误根因 | + 检索证据链 + 失败可观测 + 复杂决策溯源 |

两条 trace 互补 —— **03 是"骨架完整但语义为空"的对照基线**，**04 是"业务真实跑起来"的产品级证据链**。

[kweaver-sdk v0.7.3 / PR#116](https://github.com/kweaver-ai/kweaver-sdk/pull/116) 已 ship 的 `kweaver agent trace --view evidence` 输出的就是本附录第三节那张 13 步表的格式（SDK 团队另外多做了第 4 个 `--view reasoning` 视图，把 LLM ReAct 推理过程展开成时序，详见现状文档 §6.6.4）。

---

## 七、复现命令

### 7.1 一键命令（推荐，v0.7.3+）

```bash
export NODE_TLS_REJECT_UNAUTHORIZED=0
export KWEAVER_BASE_URL=https://<TRACE_AI_HOST>
export KWEAVER_TOKEN=__NO_AUTH__

# 13 步证据链（含 _score 命中分）—— 本附录第三节就是这条命令的输出
kweaver agent trace 01KQ7129HD1XGB1XWBT9QTF58W --view evidence

# 50-span 拓扑树（字符画）
kweaver agent trace 01KQ7129HD1XGB1XWBT9QTF58W --view tree

# LLM ReAct 推理痕迹（spec 外的彩蛋）
kweaver agent trace 01KQ7129HD1XGB1XWBT9QTF58W --view reasoning

# 4 视图全出
kweaver agent trace 01KQ7129HD1XGB1XWBT9QTF58W --view all

# 给程序消费的 JSON
kweaver agent trace 01KQ7129HD1XGB1XWBT9QTF58W --json
```

### 7.2 通用调用（手写 DSL，看原始数据时用）

```bash
TID=3eb5c7aa3b3671242a9346e18166c689

# 拉全 50 条 span（按 traceId）
kweaver call /api/agent-observability/v1/traces/_search -X POST -d "{
  \"size\":1000,
  \"query\":{\"terms\":{\"traceId.keyword\":[\"$TID\"]}},
  \"sort\":[{\"startTime\":\"asc\"}]
}"

# 仅拉这次会话的 13 个 execute_tool span（看证据链原文）
kweaver call /api/agent-observability/v1/traces/_search -X POST -d '{
  "size": 50,
  "query": {"bool":{"must":[
    {"term": {"attributes.gen_ai.conversation.id.keyword": "01KQ7129HD1XGB1XWBT9QTF58W"}},
    {"term": {"attributes.gen_ai.operation.name.keyword": "execute_tool"}}
  ]}},
  "_source": ["name","attributes.gen_ai.tool.name","attributes.gen_ai.tool.call.arguments","attributes.gen_ai.tool.call.result","attributes.agent.tool.latency_ms"],
  "sort": [{"startTime": "asc"}]
}'
```
