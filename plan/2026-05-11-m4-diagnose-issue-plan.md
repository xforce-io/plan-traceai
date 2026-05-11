# M4 Diagnose 执行 issue 计划

| 字段 | 值 |
|------|-----|
| 创建日期 | 2026-05-11 |
| 状态 | living — 随实现进度调整 |
| 依赖 vision | [`trace-cli-detailed-design.md` §3.1](../vision/trace-cli-detailed-design.md)、[`trace-ai-continuous-learning-design.md` §7.M4](../vision/trace-ai-continuous-learning-design.md) |
| 执行仓库 | `~/dev/github/kweaver-sdk/packages/typescript/`（**不是** `kweaver-core/trace-ai/`，那边是 Go observability service） |

## 0. 这个文档是什么

把 vision §3.1 的 M4 diagnose 设计落成 **4 个业务导向的 issue**。每个 issue ship 完都能给用户讲清楚"现在你能多做一件什么事"——而不是"我们交付了 B5 + B1 这两块基础设施"。

文档**会随实现进度演化**：某个 issue 拆开后发现切错了 / 多余 / 不够，就直接改本文件，并在 §3 变更日志记一行原因。

## 1. 执行约定

- **顺序**：按下表 #1 → #4 串行做。每个 issue 启动前在 kweaver-sdk 那边用 superpowers `brainstorming` → `writing-plans` 跑一遍，产出该 issue 自己的实现 spec + plan。
- **第一刀（issue #1）原则**：vertical slice — 哪怕只识别 1 条规则，也要 trace 进 → 报告出端到端跑通；不允许"基础设施先 ship、用户价值留给下一个 PR"。
- **PR 体积控制**：单 issue 太大时在 PR 内按 commit 切（B5 / B1 / engine / CLI 各一个 commit），issue 不再切碎。
- **完成动作**：勾掉本文件状态、填 PR 链接、在 §3 变更日志写 1-3 行复盘（学到什么、下一刀要不要改）。

## 2. Issue 列表

### #1 【traceai】用户能对一条线上 trace 跑出诊断报告（rule-only MVP）

| 字段 | 值 |
|------|-----|
| 状态 | ⬜ pending |
| PR | — |
| 估算 | 4-5d |

**用户故事**：给定一条线上失败 trace 的 `trace_id`，跑 `kweaver trace diagnose <id> --no-llm --out=...`，得到符合 `trace-diagnose-report/v1` schema 的 yaml 报告，能看到 symptom / evidence spans / suggested_fix。

**范围**：

- B5 SchemaRegistry 最小实现（ajv 加载、schema 校验入口）
- B1 ObservabilityClient 最小实现（`getTrace(traceId)` + 鉴权）
- `diagnosis-rule/v1` 与 `trace-diagnose-report/v1` 两份 schema
- `signal-probe` 引擎骨架 + **1 条 baseline 规则**（推荐 `tool_loop_no_state_change`）
- `kweaver trace diagnose <trace_id>` 命令入口 + report assembler
- e2e 测试：用 `status_quo/附录-完整trace样本/` 的真实 trace 跑通

**明确不做**（推到后续 issue）：

- LLM provider 层 → #3
- 多于 1 条规则 → #2
- `diagnose scan` → #4
- `rules list/validate` 子命令 → #2

**验收**：

- e2e 通过：真实 trace 进、yaml 报告出
- 报告通过 schema validate
- 报告 `evidence.spans[]` 能定位到真实 `span_id`

---

### #2 【traceai】诊断能识别更多症状，团队能管理自己的诊断规则

| 字段 | 值 |
|------|-----|
| 状态 | ⬜ pending |
| PR | — |
| 估算 | 2-3d |

**用户故事**：团队 fork 几条规则到 `<repo>/diagnosis-rules/`，用 `kweaver trace diagnose rules validate <path>` 校验后，diagnose 能识别更多症状。

**范围**：

- 补 4 条 baseline 规则，覆盖维度参考 vision §3.1（候选：`retrieval_empty_no_fallback` / schema 缺失 / state mismatch / tool 异常路径 等；最终 4 条选型在 issue #1 落地、对真实 trace 跑过后再敲定）
- `kweaver trace diagnose rules list [--rules-dir=]`
- `kweaver trace diagnose rules validate <rule.yaml>`
- 规则文档：每条写清触发条件、evidence 提示、修复方向模板

**验收**：

- 5 条规则（含 #1 的那条）全部通过 schema validate
- 至少 3 条能在历史 trace 样本上命中

---

### #3 【traceai】诊断能给出语义级原因和修复建议（LLM 双轨上线）

| 字段 | 值 |
|------|-----|
| 状态 | ⬜ pending |
| PR | — |
| 估算 | 4-5d |

**用户故事**：用户去掉 `--no-llm`，diagnose 默认走双轨——静态信号定位证据 + LLM 给 `likely_cause` / `suggested_fix` / `confidence`。

**范围**：

- B2 RemoteJobClient（async submit + poll；MVP 内置 `--sync` 降级开关）
- Diagnose Provider Wrapper：payload 拼装 + B2 submit + 响应解析
- Payload 四条硬约束（参 vision §3.1）：span 选择、大字段摘要、token budget；**batch dedup 暂不做**（推到 #4）
- `claude-code` provider 接入（MVP 只接 1 个）
- `--no-llm` 仍保留作为 offline 降级路径

**明确不做**：

- `decision-agent` provider（按需后补）
- batch payload dedup → #4

**验收**：

- 用 #1 的同一条 trace 跑双轨：产出非空的 `likely_cause` + `suggested_fix`，`confidence` 字段非 null
- `--no-llm` 仍能跑（不破坏 #1 的能力）

---

### #4 【traceai】用户能批量诊断一段时间窗口内的可疑 trace

| 字段 | 值 |
|------|-----|
| 状态 | ⬜ pending |
| PR | — |
| 估算 | 3-4d |

**用户故事**：跑 `kweaver trace diagnose scan --time-range=24h --tenant=acme --out=diagnosis/latest/`，得到一个目录的诊断报告，内存稳定、可中断恢复。

**范围**：

- B1 `searchTracesStream(query, page)` 流式分页拉取
- scan pipeline：`signal-probe` 先跑（廉价）→ 命中再喂 `provider-wrapper`（贵）
- `--max-parallel` 并发控制
- batch payload `{shared_context, per_trace_overlay[]}` dedup
- 输出去重（按 `trace_id + rule_id`）

**验收**：

- 对 100+ trace 跑 scan：内存稳定、报告无重复、并发控制生效
- token budget 超限时 fail-fast 提示拆批，不静默截断

## 3. 变更日志

| 日期 | 变更 | 原因 |
|------|------|------|
| 2026-05-11 | 初始 4 个 issue（业务导向切分） | brainstorming 收敛：第一版组件导向被否（第 1 个 issue 无法 demo），改为每 issue 一项可感知能力 |

## 4. 跨 issue 共用资产

- `diagnosis-rules/` 目录约定（git-tracked yaml；规则文件本身不走 register API）
- `outputs/` / `diagnosis/` 输出布局约定（参 vision §3.4）
- B5 / B1 / B2 的最小 API：每个 issue 只补它需要的那部分，不预先一次性铺开

## 5. 下一步

issue #1 启动时的动作：

1. `cd ~/dev/github/kweaver-sdk/packages/typescript`
2. 探现状：看 `commands/agent.ts` / `api/conversations.ts` 的鉴权和 API client 写法、测试框架（vitest？jest？）、tsconfig
3. 用 superpowers `brainstorming` skill 走 issue #1 的实现 spec
4. 通过后用 `writing-plans` skill 出 implementation plan
5. 在 kweaver-sdk 仓库开 worktree / 分支实现
6. 完成后回本文件勾掉 + 填 PR + 写复盘
