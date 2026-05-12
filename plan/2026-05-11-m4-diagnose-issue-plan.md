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
| 状态 | 🟡 in progress |
| Issue | [kweaver-ai/kweaver-sdk#120](https://github.com/kweaver-ai/kweaver-sdk/issues/120) |
| PR | — / — （PR-A symbolic / PR-B agent + rubric + within-trace synthesizer） |
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
| 状态 | ⬜ pending |
| PR | — |
| 估算 | 5-7d |

**用户故事**：

- 时间窗形态：`kweaver trace diagnose scan --time-range=24h --tenant=acme --out=diagnosis/latest/`
- 显式 ID 列表形态：`kweaver trace diagnose --traces=conv1,conv2,conv3 --out=diagnosis/ticket-42/`（手头已经攥着一组 conv_id 时的交互场景，例如从 ticket 或日志摘出来一批；不走时间窗 / 租户过滤）

两种形态共用同一条 pipeline，差别仅在 trace 来源（streaming 拉取 vs 显式列表枚举）。两者都输出一个目录的诊断报告 + 一份 `scan-summary.yaml` 跨 trace 综合报告，内存稳定、可中断恢复。综合报告告诉用户"这段时间内最严重的失败模式是 X，受影响最重的 agent 是 Y，建议优先修 Z"——不是一坨 trace 报告罗列。

**范围**：

- B1 `searchTracesStream(query, page)` 流式分页拉取（scan 模式用）
- `--traces=<id-list>` 解析 + 逐个 `getTrace` 拉取（batch 模式用，复用 PR-A 的 B1.getTrace）
- scan / batch pipeline：Stage-1 symbolic 规则先跑（廉价，作为 triage gate）→ 命中再喂 Stage-2 rubric 规则（贵）→ Stage-3 within-trace 综合（每条 trace 一份）
- **Stage-4 cross-trace 综合**（issue #2 新增）：`scan-synthesizer` 聚合 N 份 trace 报告 → `scan-summary.yaml`（rule 频次、agent 排名、模式聚类、top-K 改进建议）；agent 模式 + template fallback 同 within-trace synthesizer 一致
- `--max-parallel` 并发控制
- batch payload `{shared_context, per_trace_overlay[]}` dedup（多 trace 同送 agent 时去重共享上下文）
- 输出去重（按 `trace_id + rule_id`）

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

## 4. 跨 issue 共用资产

- `diagnosis-rules/` 目录约定（git-tracked yaml；规则文件本身不走 register API；预留 `predicate: builtin:<name>` / `rubric: <inline>` 两种引用语法）
- `outputs/` / `diagnosis/` 输出布局约定（参 vision §3.4）
- B1 / B5：#1 内最小实现，#2 直接复用
- **`agent-providers/` 公共抽象**（issue #1 落地）：M4 / 未来 M6 Synthesizer / Triage / 其他需要"结构化判定"的 trace-ai 子模块都将复用 `AgentProvider`

## 5. 下一步

issue #1 启动时的动作：

1. ✅ 探现状（已完成；见 brainstorming 记录）
2. ✅ brainstorming 收敛所有关键设计决定
3. 🟡 写正式 spec：`kweaver-sdk/docs/superpowers/specs/2026-05-11-m4-diagnose-issue1.md`
4. ⬜ spec self-review + 用户 review
5. ⬜ `writing-plans` 出 implementation plan，落到 `kweaver-sdk/docs/superpowers/plans/`
6. ⬜ 在 kweaver-sdk 仓库开 worktree / 分支实现（PR-A → PR-B）
7. ⬜ 完成后回本文件勾掉 + 填 PR + 写复盘
