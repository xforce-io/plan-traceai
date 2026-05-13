# M4 Diagnose 执行 issue 计划

| 字段 | 值 |
|------|-----|
| 创建日期 | 2026-05-11 |
| 状态 | living — 随实现进度调整 |
| 依赖 vision | [`trace-cli-detailed-design.md` §3.1 / §3.3.4](../vision/trace-cli-detailed-design.md)、[`trace-ai-continuous-learning-design.md` §7.M4](../vision/trace-ai-continuous-learning-design.md) |
| 执行仓库 | `~/dev/github/kweaver-sdk/packages/typescript/`（**不是** `kweaver-core/trace-ai/`，那边是 Go observability service） |

## 0. 这个文档是什么

把 vision §3.1 的 M4 diagnose 设计落成 **2 个业务导向的 issue**。每个 issue ship 完都能给用户讲清楚"现在你能多做一件什么事"——而不是"我们交付了 B5 + B1 这两块基础设施"。

文档**会随实现进度演化**：某个 issue 拆开后发现切错了 / 多余 / 不够，就直接改本文件，并在 §3 变更日志记一行原因。

## 1. 执行约定

- **顺序**：按下表 #1 → #2 串行做。每个 issue 启动前在 kweaver-sdk 那边用 superpowers `brainstorming` → `writing-plans` 跑一遍，产出该 issue 自己的实现 spec + plan。
- **第一刀（issue #1）原则**：vertical slice + "做就一次做对" — 5 条 symbolic baseline + 1 条 rubric 演示规则 + 跨 trace-ai 公共 agent 抽象 + claude-code subprocess provider + 引擎 + CLI + validate 一次性 ship。后续 issue 不需要回头改规则引擎或 agent 抽象结构。
- **PR 体积控制**：issue #1 体量大（12-14d），落地分 **2 个 PR**：PR-A 全部 symbolic 路径（含 5 条规则 + 引擎 + B1 + B5 + CLI + validate）；PR-B 在 PR-A 之上加 agent 抽象层 + claude-code provider + 1 条 rubric 规则。两个 PR 都关同一个 issue。
- **完成动作**：勾掉本文件状态、填两个 PR 链接、在 §3 变更日志写 1-3 行复盘（学到什么、下一刀要不要改）。

## 2. Issue 列表

### #1 【traceai】用户能对一条线上 trace 跑出诊断报告（symbolic + rubric 双 pillar 完整版）

| 字段 | 值 |
|------|-----|
| 状态 | ✅ done（2026-05-12，user-story 验收口径） |
| 验收口径澄清 | "MVP-A 已完成"指 **用户故事 A 验收**：给一条 trace，CLI 输出带证据 / 原因 / 修改建议 / 验证建议的诊断报告。**spec 字面承诺中两件件刻意推到后续阶段**：B2 RemoteJobClient（推到 MVP-B，M5 test runner 真消费者）+ `kweaver trace schema validate` 子命令（推到 MVP-B，依赖 MX1 SSOT YAML 先建——M5 eval-set 是第一个真消费者）。两份 vision/detail spec 已于 2026-05-12 同步修订，状态记录见本文件 §3 change log 2026-05-12 行。 |
| Issue | [kweaver-ai/kweaver-sdk#120](https://github.com/kweaver-ai/kweaver-sdk/issues/120)（CLOSED） |
| PR | [#121](https://github.com/kweaver-ai/kweaver-sdk/pull/121) merged（PR-A symbolic）/ [#122](https://github.com/kweaver-ai/kweaver-sdk/pull/122) merged（PR-B agent + rubric + within-trace synthesizer） |
| 估算 | 14-16d |

**用户故事**：给定一条线上 trace 的 `trace_id`，跑：

```bash
kweaver trace diagnose <trace_id> --out=diagnosis/refund-001.yaml
```

得到符合 `trace-diagnose-report/v1` schema 的 yaml 报告。报告里既有 **symbolic 规则**抓的机械模式（5 条 baseline 覆盖循环 / 错误 / 降级 / 截断 / 成本），也有 **rubric 规则**经由本地 claude-code agent 给出的语义级判定（demo 1 条）。`--no-llm` 仍可用于 offline / CI 路径，rubric 规则会被跳过 + 警告。

**关键设计决定**（brainstorming 收敛）：

| 决定 | 选择 | 理由 |
|---|---|---|
| M3 读 trace 路径 | 复用现有 `POST /api/trace-ai/_search`，按 `traceId` term 查 | 不被后端 PR 卡 |
| Schema 校验库 | **zod**（非 vision 字面 ajv） | TS-first、类型自动从 schema 推、比 ajv 短一倍 |
| 规则表达形式 | **混合 symbolic + rubric**：yaml 元信息 + 二选一的 `predicate`（builtin TS 谓词）/ `rubric`（结构化判定契约） | 1 条规则做 DSL 是过度设计；symbolic 抓机械模式，rubric 抓需要"读懂 trace"的语义判定 |
| Agent 抽象位置 | **`agent-providers/` 公共层**（不绑 M4） | M4 / M6 Synthesizer / 未来 Triage / 其他 trace-ai 子模块都将复用同一个 `AgentProvider`；diagnose 只是第一个 binding |
| Agent transport | **claude-code subprocess** （issue #1 唯一 provider） | dogfood claude-code、零远端依赖、可独立 ship；远端 `decision-agent` provider 留 stub + TODO |
| 规则分层 | builtin ship with CLI；团队 yaml 当前只能引用 `predicate: builtin:<name>` 或 `rubric: <inline>` | 团队自创 TS 谓词需要安全沙箱，留到后续 issue |

**5 条 builtin symbolic 规则**：

| # | rule_id | 抓的反模式 | 触发判定 |
|---|---|---|---|
| 1 | `tool_loop_no_state_change` | 死循环 | 同 tool_name 连续调用 ≥ 3 次，args 等价，两次之间 conversation/state 字段无变化 |
| 2 | `tool_error_swallowed` | 错误吞没 | tool span `status=error` 后，下一个 LLM span 的 prompt 中无错误信息 |
| 3 | `retrieval_empty_no_fallback` | 缺降级 | retrieval span `result_count=0` 后，下一步直接 LLM 生成（无重试 / 改 query / 换 source） |
| 4 | `llm_response_truncated_no_continue` | 截断未续 | LLM span `finish_reason=length` 后，无续传 span |
| 5 | `excessive_tool_calls_per_turn` | 失控成本 | 单个 user turn 内 tool 调用总数 > 阈值（默认 10） |

**1 条 builtin rubric 规则（demo 双 pillar 价值）**：

| rule_id | 配套 symbolic | rubric judge_question |
|---|---|---|
| `tool_retry_intent_mismatch` | `tool_loop_no_state_change` | "Given the user's intent and the tool retry pattern, was this a legitimate retry strategy, stale-results handling failure, or prompt confusion?" |

—— 同一段循环模式：symbolic 报"机械上有循环"；rubric 报"经判定属于哪类失败"。展示双 pillar 互补。

**范围**：

PR-A（symbolic 路径，6-8d）：
- B5 SchemaRegistry 最小实现（zod-based；rule schema + report schema）
- B1 ObservabilityClient 最小实现：`getTrace(traceId)` 走 `_search` + traceId term 查
- `diagnosis-rule/v1` 与 `trace-diagnose-report/v1` zod schema
- signal-probe 引擎 + `predicate-registry`（builtin 谓词命名引用解析）
- 5 条 builtin symbolic 规则（每条：yaml + TS 谓词 + 合成 fixture）
- `kweaver trace diagnose <trace_id>` 命令 + report assembler（含模板渲染）
- `kweaver trace diagnose rules validate <rule.yaml>` 子命令
- e2e：5 条合成 fixture 各命中预期；status_quo 真 fixture 全不命中

PR-B（agent + rubric + 内部综合，6-8d，依赖 PR-A）：
- `agent-providers/` 公共抽象层：`AgentProvider` / `JudgmentRequest` / `JudgmentResponse` / `AgentRegistry` / `PromptTemplateRegistry`
- `providers/claude-code-subprocess.ts`：spawn `claude` CLI，prompt 注入 + 结构化输出 schema 提示 + JSON 解析重试 + 超时 + 不在 PATH 时 fail-fast
- `providers/stub.ts`：测试用，`KWEAVER_DIAGNOSE_AGENT_PROVIDER=stub` 切到 fixture 回放
- `providers/decision-agent-remote.ts`：仅 stub + TODO 注释（post-MVP 实现）
- `trace-ai/diagnose/agent-binding.ts`：Stage-2 — Rubric → AgentProvider.invoke → RubricJudgment
- `trace-ai/diagnose/synthesizer.ts`：Stage-3 within-trace — (meta, findings) → Summary；agent 模式走 `builtin:within-trace-synthesizer-v1` prompt template，`--no-llm` / agent 失败时走 deterministic template fallback；`run.synthesizer_mode` 记录走了哪条
- 1 条 builtin rubric 规则（`tool_retry_intent_mismatch`）+ rubric prompt template + synthesizer prompt template + 合成 fixture
- `--no-llm` 反转：默认走双 pillar + agent synthesizer；`--no-llm` 时 rubric 跳过 + warn，synthesizer 退到 template 模式仍输出 summary
- e2e（PR-B 增量）：rubric fixture 走 stub provider；synthesizer 在 stub provider / template 两种模式都断言 summary 字段非空、cross_finding_links 在 symbolic+rubric 同 span 命中时被填

**明确不做**（推到后续 issue 或 post-MVP）：

- `decision-agent` 远端 provider 真实现（仅 stub）
- 团队自定义 TS 谓词（需要安全沙箱）
- `diagnose rules list` 子命令（团队也能 `ls diagnosis-rules/`，不阻塞）
- `diagnose scan` 批量模式（时间窗 / 租户过滤）→ issue #2
- `diagnose --traces=<id-list>` 显式 ID 列表 batch 形态（vision §3.1 L383 写过）→ issue #2 作为 scan 的并列 flag 一并实现
- payload 四条硬约束的"batch dedup"（scan / batch 才需要）

**验收**：

- PR-A：5 条合成 fixture e2e 命中预期；真 fixture 0 命中；validate 命令两条路径都通；输出报告通过 `trace-diagnose-report/v1` schema validate
- PR-B：rubric 规则在 stub provider 上跑通（CI 用）、在本地装好 claude CLI 时跑通（手测）；`--no-llm` 路径不破坏 PR-A 能力；agent 抽象的接口稳定（写在 spec 里，不可改）
- 整体：AGENTS.md 同步：src/cli.ts top-level help / src/commands/trace.ts / skills/kweaver-core/references/ / README

---

### #2 【traceai】用户能批量诊断一段时间窗口内的可疑 trace（scan 模式 + 跨 trace 综合）

| 字段 | 值 |
|------|-----|
| 状态 | 🟡 待 PR merge（scope 已收敛：仅 batch 形态，scan 时间窗形态明确不做） |
| Issue | [kweaver-ai/kweaver-sdk#123](https://github.com/kweaver-ai/kweaver-sdk/issues/123)（OPEN，待 PR merge 后 close） |
| PR | [#124](https://github.com/kweaver-ai/kweaver-sdk/pull/124) OPEN — batch `--traces=<id-list>` + Stage-4 cross-trace synthesizer + Stage-1 gate + 单 agent 校验 + artifacts |
| 估算 | 5-7d |

**用户故事**：

- 时间窗形态：`kweaver trace diagnose scan --time-range=24h --tenant=acme --out=diagnosis/latest/`
- 显式 ID 列表形态：`kweaver trace diagnose --traces=conv1,conv2,conv3 --out=diagnosis/ticket-42/`（手头已经攥着一组 conv_id 时的交互场景，例如从 ticket 或日志摘出来一批；不走时间窗 / 租户过滤）

两种形态共用同一条 pipeline，差别仅在 trace 来源（streaming 拉取 vs 显式列表枚举）。两者都输出一个目录的诊断报告 + 一份 `scan-summary.yaml` 跨 trace 综合报告，内存稳定、可中断恢复。综合报告告诉用户"这段时间内最严重的失败模式是 X，受影响最重的 agent 是 Y，建议优先修 Z"——不是一坨 trace 报告罗列。

**范围**（✅ = PR #124 已 ship）：

- ✅ `--traces=<id-list>` 解析 + 逐个 `getTrace` 拉取（batch 模式用，复用 PR-A 的 B1.getTrace）；支持 `--traces=@/path/to/file`
- ✅ batch pipeline：Stage-1 symbolic 规则先跑（廉价，作为 triage gate，rubric `gates_on` 字段）→ 命中再喂 Stage-2 rubric 规则（贵）→ Stage-3 within-trace 综合（每条 trace 一份）
- ✅ **Stage-4 cross-trace 综合**：`cross-trace-synthesizer.ts` 聚合 N 份 trace 报告 → `scan-summary.yaml`（rule 频次、agent 排名、改进建议）；LLM 失败时回退到 deterministic 聚合摘要（`fallbackSummary`）
- ✅ `--max-parallel` 并发控制（[1, 64] 校验，避免 0 / 负数 死循环）
- ✅ 单 agent 校验：所有 traces 必须属于同一 agent，否则 fail-fast
- ✅ Artifacts 持久化（默认开，`--no-artifacts` 可关）；per-trace yaml/md 写入 `<out>/traces/` 子目录

**明确不做**（2026-05-12 收敛，scan 能力本期到此为止）：

- B1 `searchTracesStream(query, page)` 流式分页
- `kweaver trace diagnose scan --time-range=24h --tenant=<...>` 时间窗形态
- batch payload `{shared_context, per_trace_overlay[]}` dedup
- 输出去重（按 `trace_id + rule_id`）

—— 理由：batch (`--traces=`) 形态已能覆盖手头攥着 conv_id 列表的实战场景；时间窗扫描需要先确认后端 `_search` 分页 + 租户过滤契约，且当前没有强需求驱动；payload dedup / 输出去重在量级上来之前是过度优化。如未来出现真实场景再开新 issue。

**验收**：

- 对 100+ trace 跑 scan：内存稳定、报告无重复、并发控制生效
- token / context 超限时 fail-fast 提示拆批，不静默截断
- `scan-summary.yaml` 在 `--no-llm` 模式下走 deterministic 聚合模板（rule 频次、severity 排序、agent 出现次数）；agent 模式额外有 LLM 生成的 narrative summary
- cross-trace 综合的 schema (`scan-summary/v1`) 与 within-trace summary 共享部分字段结构（headline / fix_priority / cross_links），让用户跨级别看报告时心智一致

## 3. 变更日志

| 日期 | 变更 | 原因 |
|------|------|------|
| 2026-05-11 | 初始 4 个 issue（业务导向切分） | brainstorming 收敛：第一版组件导向被否（第 1 个 issue 无法 demo），改为每 issue 一项可感知能力 |
| 2026-05-11 | issue #1 已开 — 链接 kweaver-sdk#120 | — |
| 2026-05-11 | 重切 4→3 issue | "做就一次做对" — 单 issue 1 条规则太薄；5 条规则 + validate 一次 ship 完整可用的 rule-only diagnose |
| 2026-05-11 | 锁定决定：M3 走 _search term 查 / schema 校验用 zod / 规则用 yaml+TS 谓词混合 / report 走 meta+findings[] / 默认 builtin+cwd 混合装载 | brainstorming 澄清问题逐一收敛 |
| 2026-05-11 | **重大重切：3→2 issue。原 #1 (rule-only) 与 #2 (LLM 双轨) 合并为新 #1。补 rubric 规则类型 + 跨 trace-ai 公共 agent 抽象 + claude-code subprocess provider；scan 上提为 #2。估算 12-14d 分 2 PR 落地** | 用户指出之前设计盲点：只有 symbolic 规则是不够的，需要 rubric + agent 判定；agent 抽象应跨 trace-ai 模块复用而不是闷在 diagnose 里。承认是真盲点 |
| 2026-05-11 | spec §"Industry Alignment" 加固：明确"Stage-1 triage（symbolic）+ Stage-2 verdict（rubric）"分层叙事；规则 yaml 加 `taxonomy` 块（Signals 3 轴 + MS 6 类）；rubric `output_schema` 强制 `first_violating_step_id` 字段 | 用户 challenge "行业是不是这么做"，调研发现 LangSmith / Phoenix / Braintrust / Langfuse / OpenAI Evals / Anthropic / MS taxonomy / arXiv 2604.00356 (Signals) 全部走两段式 deterministic + judge，且 vision §3.1 引用过的 Signals 论文几乎是这个设计的"已发表对照组"。趁早把两段式框架写进 spec，避免实现期再返工 |
| 2026-05-11 | 加 Stage-3 within-trace synthesizer 到 #1（spec + plan + issue），#2 scope 增加 Stage-4 cross-trace synthesizer。#1 估算 12-14d → 14-16d；#2 估算 3-4d → 5-7d | 用户 challenge "只见树木不见森林"——单 trace 的 N findings 是局部，需要 forest view。within-trace synthesizer 处理"一份报告里多条 finding 的去重串联"；cross-trace synthesizer（scan）处理"100+ trace 的模式聚合 + 排名 + 优先级"。架构上是 agent-providers/ 公共层的第二个 binding，和 agent-binding.ts 平级 |
| 2026-05-12 | 把 vision §3.1 L383 的 `diagnose --traces=<id-list>` 显式 ID 列表形态对齐回 issue #2 范围 | 之前 3→2 重切时把 "batch (`--traces=`)" 隐式吞进了 scan 描述，留下三个文档口径不一致；用户 challenge 后追加：scan（时间窗）和 batch（显式列表）作为同一 pipeline 的两个入口，一并在 #2 落地 |
| 2026-05-12 | 把 `trace-core/` 容器拆成两个对等顶层目录：`agent-providers/`（peer of `api/`）+ `trace-ai/`（peer of `bkn / dataflow / vega`，目前内含 `diagnose/`） | 用户 challenge: trace-core 这个"-core"命名让 trace-ai 看起来跟其他业务模块不对等，且把 `agent/`（跨模块基础设施，未来 M6 / Triage 也要用）埋在 trace-ai 内部一层。趁 PR-B 还没 merge、只有一个 consumer，做最小成本的目录重整。机械替换 + 全 suite 1103/1103 验证 |
| 2026-05-12 | **Issue #1 完成**：PR-A [#121](https://github.com/kweaver-ai/kweaver-sdk/pull/121) + PR-B [#122](https://github.com/kweaver-ai/kweaver-sdk/pull/122) 双双 merged，issue [#120](https://github.com/kweaver-ai/kweaver-sdk/issues/120) closed | M4 双 pillar 完整版（symbolic + rubric + within-trace synthesizer）落地 |
| 2026-05-12 | **Issue #2 切分实施**：先发 PR [#124](https://github.com/kweaver-ai/kweaver-sdk/pull/124) 仅覆盖 batch (`--traces=<id-list>`) 形态 + Stage-4 cross-trace synthesizer + Stage-1 gate + 单 agent 校验 + artifacts；scan 时间窗形态（`scan --time-range=24h --tenant=`）+ B1 `searchTracesStream` + payload dedup + 输出去重 推后到 follow-up issue/PR | batch 形态优先级更高（手头攥着一组 conv_id 的实战场景），且不依赖后端 streaming 查询接口；scan 时间窗需要先确认 `_search` 分页 + 租户过滤的后端契约。CI 全绿（Python pass / TypeScript pass） |
| 2026-05-12 | **Scope 终态收敛**：scan 能力本期止于 PR #124 的 batch 形态。时间窗 / streaming / payload dedup / 输出去重 从"follow-up TODO"改判为"明确不做"，列入 §2 #2 的"明确不做"块。如未来出现真实场景再开新 issue | 用户决定：当前没有时间窗扫描的强需求驱动；避免在 vision 落地阶段过度铺基础设施。M4 整体收尾在 PR #124 merge |
| 2026-05-12 | **待验证假设：MX1 audit 子能力可下沉到 M4**。精确措辞：MX1 audit 三件（跨 span 不变量 / 漂移率 / L1/L2 准入率）= M4 schema-rules pack（新规则类型 `schema_invariant`）+ 新 cross-trace synthesizer template + B1 时间窗输入；MX1 模块保留 SSOT YAML (`schema/v1/`) + `kweaver trace schema validate` 单文件校验。**vision spec 暂不动**，待"现状 schema 缺口量化"任务（写 3-5 条 schema invariant 规则跑 status_quo 样本）跑完验证后，一次性改 `trace-cli-detailed-design.md` §3.6 / §4.2 / 附录 A / 附录 B + `trace-ai-continuous-learning-design.md` §7.MX1 + 散落引用 + 本工作区 README 活清单 | 用户 challenge：MX1 audit 跟 M4 + cross-trace synthesizer 在机制上几乎完全重叠（规则跨 span 求值 + 跨 trace 聚合）；3 个我之前 hand-wave 的差异（时序漂移 / 时间窗抽样 / 输出形态）重审后都不构成独立模块的必要性——漂移靠 git-track scan-summary 历史 + diff 脚本；时间窗抽样是 B1 缺口（M4 scan 和 audit 共享），不是 audit 独占；输出形态只需另写一个 synthesizer prompt template |
| 2026-05-12 | **顺带发现：MVP-A spec 偏差**——vision §3.6 / §4.2 / line 776 承诺 MVP-A 提供 `kweaver trace schema validate`，M4 plan 静默 de-scope 了（CLI 当前只有 `trace diagnose rules-validate`，校验 rule yaml 而非 trace/eval-set/bundle yaml）。规模小（B5 SchemaRegistry 已就绪），与上一行的"现状 schema 缺口量化"验证任务一并补回 | 重审 MX1 章节时核对 CLI 实际命令清单发现 |
| 2026-05-12 | **MVP-B / M5 范围澄清 + 格式约束细化**（detail §3.2 + vision §7.M5 同步） | 用户提问"MVP-B 到底是只从轨迹中总结 eval-set 还是接受客户上传的内容，格式如何约束和统一"。spec 原立场是"两种来源都接受，统一格式"，但留了 4 个洞：① `--queries=<path>` 输入格式没给；② assertions 类型只举了 `contains`；③ `--with-reference` flag 含义模糊；④ query_id 冲突语义没定。本日修订一次性补齐：① 引入 `trace-eval-set-input/v1` 简化 schema（input-only + 可选 query_id/tags），CLI lift 到 `trace-eval-set/v1`；② assertions 类型枚举 6 种（contains / not_contains / regex / tool_call_count / tool_call_order / semantic_match / latency_ms）；③ 砍 `--with-reference`（reference 提取属于 L2 relabel post-MVP）；④ `--on-conflict=fail\|skip\|overwrite` flag，default=fail；⑤ query_id 全 eval-set 唯一，未填用 hash 自动生成；⑥ reference 与 assertions 不可同时空（zod refinement）。**三种来源（用户手写完整 shard / build --diagnosis / build --queries）汇一个 schema 一个目录**，下游 test / exp 完全无感 |
| 2026-05-12 | **vision / detail spec 文档债务一次性修订**（A 类机械 + B 类降级，共 7 项；C 类 MX1 audit / SSOT 待 post-MVP 再决） | 修订内容：① detail spec 删 §0 banner、§1.1 物理位置树重写为 `trace-ai/` + `agent-providers/` 双根、附录 A 整体重写并标 ✓ MVP-A 已落地件；② detail §3.1 M4 章节同步双 pillar 实际架构、依赖删 B2、加 `agent-providers/`；③ detail §3.6 MX1 把 `schema validate` 从 MVP-A 降到 MVP-B（依赖 SSOT YAML 先建），audit 标"架构假设待重审"指向本日志；④ detail §4.2 MVP-A 发布顺序改为已落地实录，B2 / `schema validate` 推到 MVP-B 顺序；⑤ detail 附录 B 依赖矩阵加 `agent-providers` 列，M4 行 B2 单元格改"— MVP-A 不需要"；⑥ vision spec §1.1.1 / §6.4.1 / §7.M4 / §7.MX1 / §3.2 / §6.4.1 净效果 / 附录 A 等 8 处对齐降级（B2 → MVP-B、`schema validate` → MVP-B、`scan --time-range=` → post-MVP）；⑦ Issue #1 加"验收口径澄清"行，明确"MVP-A 已完成"=用户故事 A 验收，spec 字面承诺中 B2 / `schema validate` 推到 MVP-B。**改完两份 vision 文档与代码 100% 对齐**，C 类（MX1 audit 下沉假设 + SSOT YAML 何时建）仍按 2026-05-12 待验证假设行的路径处理 | 用户 challenge："vision 和 detail 都没包括，为什么要现在做" —— 暴露了 spec 偏离实际、"MVP-A 已完成"心智偏差。修订前先做结构总览，识别出 7 项文档债务（A 类 4 项机械 / B 类 2 项降级 / C 类 2 项决策待定，推后），按"A+B 一次性 ship，C 类暂缓"推进 |
| 2026-05-13 | **进 Story B (M5) 前 spec 聚焦式 review**：vision + detail M5 触面 + plan §172 C 类未决。**核心发现**：B2 RemoteJobClient + 远端 evaluator 是凭空抽象——kweaver 平台后端（DA agent-executor / agent-factory + trace-ai agent-observability）**不存在 async job 系统**（DA 只 sync streaming `chat/completion`）、**不存在独立 evaluator 服务**（grep 全两个仓库 0 命中）。M5 `eval-set test` 改走 sync sequential 调既有 `POST /api/agent-factory/v1/app/{agent_id}/chat/completion` + `GET /api/agent-observability/v1/traces/by-conversation?conversation_id=...` + 本地 6 种 assertion 评估 + `--max-parallel` 并发。**修订点**：① B2 RemoteJobClient 作为共享层组件**整体从 spec 移除**（§2.1 表格 / §2.1.2 contract / 附录 A `remote-job.ts` / 附录 B 矩阵 B2 列 / §1.1 物理位置树等），保留 §6.4.4 "async submit + poll" 作为**设计模式语言**给 M6/M7 后续 brainstorm 时绑具体组件；② MX1 `schema validate` 子命令**MVP-B 期 ship**（B5 zod 注册表薄包装，不依赖 SSOT YAML 先建）；③ MX1 SSOT YAML mirror **推到 "首个 polyglot 消费者出现时"**（M5 zod 内联自洽，当前无 polyglot 触发场景）；④ MX1 audit 下沉 M4 假设仍待 "现状 schema 缺口量化" 验证（未变）；⑤ `--candidate=<...>` 字面值**占位化**，MVP-B brainstorm 时决定 `agent_id[@version]` 裸标识 vs yaml 文件形态。**plan §172 三项 C 类**：B2 → 直接删（用户决策）；SSOT YAML → 推后（M5 不再是消费者）；MX1 audit → 不变（待量化任务） | 用户在 M5 brainstorm 第 2 题（candidate 寻址）时 challenge："如果这样的话，是不是先把设计文档再 review 修订一遍"——指出 B2 这种 size 的 spec offset 不应在 brainstorm 时一点点暴露，应先一次性把 M5 触面对齐。聚焦式范围避免 bikeshed |
| 2026-05-13 | **Story B (M5) brainstorm 收尾 + spec doc 落地**：spec doc 写在 `kweaver-sdk/docs/superpowers/specs/2026-05-13-m5-eval-set-builder-design.md`（706 行）。brainstorm 收敛 7 项决议（D0-D6）：D0 删 B2 + sync sequential test；D1 `trace-eval-set-input/v1` 加可选 reference/assertions（修 spec 隐 bug——原 build --queries= 自动留占位违反 refinement）；D2 redaction 默认内置低保真 PII + 覆盖链；D3 `--candidate=<agent_id>[@<version>]` 裸标识（去占位化）；D4 PR-A (build + schema validate) / PR-B (test + 6 assertions) 双 PR 切分；D5 PR-B 必 ship builtin rubric `answer-match-reference`（golden truth 核心）；D6 MX1 SSOT YAML 推到 polyglot 消费者出现时。spec doc + vision/detail D1-D5 同步修订一并 commit。M5 估算 10-14d（PR-A 5-7d + PR-B 5-7d，与 M4 14-16d 同口径减重） | 沿 brainstorming skill 流程：context exploration → clarifying questions (Q1-Q3) → design sections (§1-§7) → 过度工程审计两轮（砍 query-id-gen.ts / 退出码细分 / partial skip / retry 3x / severity / answer-match-reference 升 B5 注册表）→ spec doc 写盘 → 自检通过 |

## 4. 跨 issue 共用资产

- `diagnosis-rules/` 目录约定（git-tracked yaml；规则文件本身不走 register API；预留 `predicate: builtin:<name>` / `rubric: <inline>` 两种引用语法）
- `outputs/` / `diagnosis/` 输出布局约定（参 vision §3.4）
- B1 / B5：#1 内最小实现，#2 直接复用
- **`agent-providers/` 公共抽象**（issue #1 落地）：M4 / 未来 M6 Synthesizer / Triage / 其他需要"结构化判定"的 trace-ai 子模块都将复用 `AgentProvider`

## 5. 下一步

### Issue #1 — ✅ 完成（保留作历史参考）

1. ✅ 探现状
2. ✅ brainstorming 收敛所有关键设计决定
3. ✅ spec
4. ✅ implementation plan
5. ✅ PR-A [#121](https://github.com/kweaver-ai/kweaver-sdk/pull/121) merged
6. ✅ PR-B [#122](https://github.com/kweaver-ai/kweaver-sdk/pull/122) merged
7. ⬜ 复盘（待补，建议在 issue #2 收尾时一起回顾）

### Issue #2 — 🟡 待 PR merge 收尾

1. 🟡 PR [#124](https://github.com/kweaver-ai/kweaver-sdk/pull/124) review + merge — batch (`--traces=`) 形态 + Stage-4 cross-trace synthesizer。CI 已绿。
2. ⬜ PR #124 merge 后 close issue [#123](https://github.com/kweaver-ai/kweaver-sdk/issues/123)（连同"scan 时间窗形态本期不做"的备注）
3. ⬜ 在本文件 §3 加 1 行 M4 整体收尾复盘

—— scan 时间窗 / streaming / payload dedup / 输出去重 已在 §2 #2 的"明确不做"块中记录，本期不再追加 issue。

### M4 之后

M4 diagnose 收尾后，按 vision spec / `~/lab/plan-traceai/README.md` §"活清单"挑下一个 milestone（候选：M6 Synthesizer / Schema 治理 / L1 Triage MVP 选型 / Falsifiable Manifest）。
