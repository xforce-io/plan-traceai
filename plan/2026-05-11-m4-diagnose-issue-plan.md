# M4 Diagnose 执行 issue 计划

| 字段 | 值 |
|------|-----|
| 创建日期 | 2026-05-11 |
| 状态 | living — 随实现进度调整 |
| 依赖 vision | [`trace-cli-detailed-design.md` §3.1](../vision/trace-cli-detailed-design.md)、[`trace-ai-continuous-learning-design.md` §7.M4](../vision/trace-ai-continuous-learning-design.md) |
| 执行仓库 | `~/dev/github/kweaver-sdk/packages/typescript/`（**不是** `kweaver-core/trace-ai/`，那边是 Go observability service） |

## 0. 这个文档是什么

把 vision §3.1 的 M4 diagnose 设计落成 **3 个业务导向的 issue**。每个 issue ship 完都能给用户讲清楚"现在你能多做一件什么事"——而不是"我们交付了 B5 + B1 这两块基础设施"。

文档**会随实现进度演化**：某个 issue 拆开后发现切错了 / 多余 / 不够，就直接改本文件，并在 §3 变更日志记一行原因。

## 1. 执行约定

- **顺序**：按下表 #1 → #3 串行做。每个 issue 启动前在 kweaver-sdk 那边用 superpowers `brainstorming` → `writing-plans` 跑一遍，产出该 issue 自己的实现 spec + plan。
- **第一刀（issue #1）原则**：vertical slice + "做就一次做对" — 5 条 baseline 规则 + 引擎 + CLI 一次性 ship 完整可用的 rule-only diagnose；后续 issue 不需要回头改规则引擎结构。
- **PR 体积控制**：单 issue 太大时在 PR 内按 commit 切（B5 / B1 / engine / 5 条规则各一 / CLI / e2e），issue 不再切碎。
- **完成动作**：勾掉本文件状态、填 PR 链接、在 §3 变更日志写 1-3 行复盘（学到什么、下一刀要不要改）。

## 2. Issue 列表

### #1 【traceai】用户能对一条线上 trace 跑出诊断报告（rule-only 完整版）

| 字段 | 值 |
|------|-----|
| 状态 | 🟡 in progress |
| Issue | [kweaver-ai/kweaver-sdk#120](https://github.com/kweaver-ai/kweaver-sdk/issues/120) |
| PR | — |
| 估算 | 6-8d |

**用户故事**：给定一条线上 trace 的 `trace_id`，跑：

```bash
kweaver trace diagnose <trace_id> --no-llm --out=diagnosis/refund-001.yaml
```

得到符合 `trace-diagnose-report/v1` schema 的 yaml 报告，能识别 5 类常见 agent program 反模式（循环 / 错误处理 / 降级 / 截断 / 成本失控），并对每条命中给出 evidence spans + 修复方向 + 验证建议。团队拿到报告能立即用于 code review 与 eval-set 构建。

**关键设计决定**（brainstorming 收敛）：

| 决定 | 选择 | 理由 |
|---|---|---|
| M3 读 trace 路径 | 复用现有 `POST /api/trace-ai/_search`，按 `traceId` term 查 | 不被后端 PR 卡；issue #1 可独立 ship |
| Schema 校验库 | **zod**（不是 vision 字面写的 ajv） | TS-first、类型自动从 schema 推、写起来短一倍；以后 python 复用走 `zod.toJsonSchema()` |
| 规则表达形式 | **混合**：yaml 元信息 + 命名 TS 谓词（`predicate: builtin:<name>`） | 1 条规则做 DSL 是过度设计；纯 TS 又把契约漏到代码层；混合让 yaml 承担对外契约、TS 承担匹配逻辑，未来真正 DSL 上线可平滑替换 `predicate` 字段 |
| 规则分层 | builtin 规则 ship with CLI；团队 yaml 当前只能引用 builtin 谓词（先不开放自定义谓词） | 自创规则需要完整 DSL，留到后续 issue |

**5 条 builtin baseline 规则**：

| # | rule_id | 抓的反模式 | 触发判定 |
|---|---|---|---|
| 1 | `tool_loop_no_state_change` | 死循环 | 同 tool_name 连续调用 ≥ 3 次，args 等价，两次之间 conversation/state 字段无变化 |
| 2 | `tool_error_swallowed` | 错误吞没 | tool span `status=error` 后，下一个 LLM span 的 prompt 中无错误信息 |
| 3 | `retrieval_empty_no_fallback` | 缺降级 | retrieval span `result_count=0` 后，下一步直接 LLM 生成（无重试 / 改 query / 换 source） |
| 4 | `llm_response_truncated_no_continue` | 截断未续 | LLM span `finish_reason=length` 后，无续传 span |
| 5 | `excessive_tool_calls_per_turn` | 失控成本 | 单个 user turn 内 tool 调用总数 > 阈值（默认 10） |

**范围**：

- B5 SchemaRegistry 最小实现（zod-based；rule schema + report schema 各一份）
- B1 ObservabilityClient 最小实现：`getTrace(traceId)` 内部走 `_search` + `traceId` term 查，本地拼 trace tree
- `diagnosis-rule/v1` 与 `trace-diagnose-report/v1` 两份 zod schema
- signal-probe 引擎 + builtin predicate registry
- 5 条 builtin baseline 规则（每条：yaml 元信息 + TS 谓词 + 合成 fixture）
- `kweaver trace diagnose <trace_id>` 命令入口 + report assembler
- `kweaver trace diagnose rules validate <rule.yaml>` 子命令（团队 fork baseline 后必备）
- e2e 测试：5 条合成 fixture 各自命中预期规则；status_quo 那条真 fixture 跑 5 条规则全部不命中（反面验证）

**明确不做**（推到后续 issue）：

- LLM provider 层 → #2
- `diagnose rules list` 子命令（团队也能 `ls diagnosis-rules/`，不阻塞）
- 团队自定义 TS 谓词（需要完整 DSL，暂不开放）
- `diagnose scan` → #3

**验收**：

- 5 条合成 fixture 端到端跑通：`kweaver trace diagnose <id> --no-llm` → yaml 报告，每份报告命中预期规则
- 真 fixture（status_quo 那条）跑同 5 条规则：报告体里 0 命中（无误报）
- 所有输出报告通过 `trace-diagnose-report/v1` schema validate
- 报告 `evidence.spans[]` 能定位到真实 `span_id`
- `--no-llm` 路径无任何 LLM SDK 依赖，可 offline 跑
- AGENTS.md 同步：src/cli.ts top-level help、src/commands/trace.ts、skills/kweaver-core/references/、README

---

### #2 【traceai】诊断能给出语义级原因和修复建议（LLM 双轨上线）

| 字段 | 值 |
|------|-----|
| 状态 | ⬜ pending |
| PR | — |
| 估算 | 4-5d |

**用户故事**：用户去掉 `--no-llm`，diagnose 默认走双轨——静态信号定位证据 + LLM 给 `likely_cause` / `suggested_fix` / `confidence`。

**范围**：

- B2 RemoteJobClient（async submit + poll；MVP 内置 `--sync` 降级开关）
- Diagnose Provider Wrapper：payload 拼装 + B2 submit + 响应解析
- Payload 四条硬约束（参 vision §3.1）：span 选择、大字段摘要、token budget；**batch dedup 暂不做**（推到 #3）
- `claude-code` provider 接入（MVP 只接 1 个）
- `--no-llm` 仍保留作为 offline 降级路径

**明确不做**：

- `decision-agent` provider（按需后补）
- batch payload dedup → #3

**验收**：

- 用 #1 的合成 fixture 跑双轨：产出非空的 `likely_cause` + `suggested_fix`，`confidence` 字段非 null
- `--no-llm` 仍能跑（不破坏 #1 的能力）

---

### #3 【traceai】用户能批量诊断一段时间窗口内的可疑 trace

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
| 2026-05-11 | issue #1 已开 — 链接 kweaver-sdk#120 | — |
| 2026-05-11 | **重切：4 issue → 3 issue。#1 scope 扩为完整 rule-only（5 条 baseline + validate）；杀掉原 #2；原 #3/#4 上提为 #2/#3** | "做就一次做对，后续不用翻掉" — 单 issue 1 条规则太薄；5 条规则 + validate 一次 ship 完整可用的 rule-only diagnose，团队能立即用 |
| 2026-05-11 | 锁定关键决定：M3 走 `_search` term 查 / schema 校验用 zod / 规则用 yaml+TS 谓词混合方案 | brainstorming 澄清问题逐一收敛 |

## 4. 跨 issue 共用资产

- `diagnosis-rules/` 目录约定（git-tracked yaml；规则文件本身不走 register API；预留 `predicate: builtin:<name>` 引用语法）
- `outputs/` / `diagnosis/` 输出布局约定（参 vision §3.4）
- B1 / B2 的最小 API：每个 issue 只补它需要的那部分，不预先一次性铺开
- B5（zod-based schema 注册）：#1 内最小实现，#2/#3 直接复用

## 5. 下一步

issue #1 启动时的动作：

1. `cd ~/dev/github/kweaver-sdk/packages/typescript`
2. 探现状：已完成（见 brainstorming 记录） — 命令注册手写分发、API client 每资源一文件、Node 原生 test 框架、AGENTS.md 强约束（英文注释 / CLI 改动同步 4 处文档 / Makefile target / limit 默认值）
3. 用 superpowers `brainstorming` skill 走完 issue #1 实现 spec 的剩余澄清（报告 schema 字段、CLI flag 默认值等）
4. 通过后用 `writing-plans` skill 出 implementation plan，落到 `kweaver-sdk/docs/superpowers/specs/` + `docs/superpowers/plans/`
5. 在 kweaver-sdk 仓库开 worktree / 分支实现
6. 完成后回本文件勾掉 + 填 PR + 写复盘
