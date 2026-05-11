---
title: trace-ai CLI 详细设计
status: draft
date: 2026-05-09
依赖: vision/trace-ai-continuous-learning-design.md（以下简称 vision）
覆盖: MVP-A 轨迹诊断、MVP-B eval-set 构建与测试、MVP-C 单路径迭代、post-MVP 多路径/飞轮能力的取舍
---

## §0 范围与背景

vision §6.4 / §7 把 trace-ai above-L0 的能力划成 M4 Curation / M5 Eval-Set Builder / M6 Experiment Engine / M7 Replay / M8 Publish Registry / M9 Post-deploy Verify / MX1 Schema。本文档按用户价值路径重切 MVP：MVP-A 先做轨迹诊断，帮助用户识别 Decision Agent 这个"概率性程序"哪里写得不合理；MVP-B 再把诊断沉淀为 eval-set 并跑测试；MVP-C 支持沿一个方向做单路径持续迭代。多路径探索、Trial Forest、自动飞轮、M7 replay、M5 relabel、MX1 audit、M8/M9 均放到 post-MVP。

- **代码归属**：哪个文件 / 哪个仓库
- **命令字面值**：用户键入什么
- **模块边界**：内部子模块怎么切、依赖方向
- **共享层契约**：跨模块复用的内核组件
- **触发机制**：周期性任务怎么不靠 K8s CronJob 也能跑
- **用户操作路径**：operator 从写任务配置、启动实验、查看进展到取走 `outputs/` 产物，具体敲哪些命令

本文档**不**覆盖：每个 HTTP 接口的字段细节（属下游各 M-spec）、远端智能体层（DA / kweaver-eval / Triage Agent）的内部实现（属各自子 spec）、L0 数据面（M1 otelcol / M2 OpenSearch / M3 agent-observability）的服务实现（属 trace-ai 仓库后端）。

### 0.1 用户故事与操作路径

本文档后面按模块拆解实现，但用户不会按模块名工作。CLI 面向三段 MVP user story：

1. **Story A：轨迹诊断**。我有线上失败 / 可疑 trace，希望 KWeaver 帮我识别 agent program 的不合理性，并给出带证据的修改建议。
2. **Story B：从 0 构建 eval-set 并测试**。我把诊断出的失败模式沉淀成可复现 eval cases，并用当前 agent 配置跑测试确认问题存在。
3. **Story C：单路径持续迭代**。我沿一个明确修改方向迭代 prompt / skill / BKN / tool contract / workflow，并持续跑同一套 eval-set 看是否改善。

这三段对应真实用户路径：先诊断，再固化测试，最后单线优化。多路径并行实验和线上反馈飞轮不是 MVP 的第一目标，避免一开始就引入 Trial Forest、registry、周期 verify、自动 feed 聚合等系统复杂度。

#### 0.1.1 Story A：轨迹诊断

Story A 的目标不是跑实验，而是把 trace 变成"compiler warning 风格"的诊断报告：症状是什么、证据在哪、可能原因是什么、建议改哪里、如何验证。前置条件是 M3 trace 查询、schema mirror 与本地诊断规则；不要求 eval-set、实验文件夹或远端 replay。

```bash
# 1. 对单条失败 trace 做诊断
kweaver trace diagnose <trace_id> --out=diagnosis/refund-001.yaml

# 2. 对一批线上 trace 扫描并输出诊断候选
kweaver trace diagnose scan --time-range=24h --tenant=acme --out=diagnosis/latest/

# 3. 查看诊断规则，或校验团队自定义规则
kweaver trace diagnose rules list
kweaver trace diagnose rules validate diagnosis-rules/tool-loop.yaml
```

诊断报告必须面向用户可执行，而不是只输出 trace 摘要：

```yaml
diagnosis:
  symptom: repeated_tool_call_without_state_change
  likely_cause: missing_termination_condition_in_agent_program
  evidence:
    trace_id: tr_123
    spans: [sp_7, sp_8, sp_9]
  suggested_fix:
    target: decision_agent.prompt
    change: add stop condition after two equivalent failed retrievals
  confidence: medium
  verify_with:
    suggested_eval_case:               # eval-case 草稿，eval-set build 直接吃
      query_id: refund_001
      query: <从 trace 抽取的原始请求>
      assertions:
        - tool_call_count(retrieval) <= 2
```

`verify_with.suggested_eval_case` 是 Story A → Story B 的唯一衔接点：`eval-set build --diagnosis=<dir>` 直接读取并填充成 eval shard，无需用户手工拼接。

Story A 主要落在 M4 Diagnosis：规则 / 状态机 / schema signal 负责识别症状，诊断模板负责给建议。MVP-A 不做 LLM 自动修复，不做 replay，不生成实验候选。

#### 0.1.2 Story B：从 0 构建 eval-set 并测试

Story B 的目标是把诊断出的线上问题变成可复现测试资产，并确认当前 agent 配置在这些 case 上确实失败或风险较高。

```bash
# 1. 从诊断报告或 trace 查询构建 eval-set
kweaver trace eval-set build --diagnosis=diagnosis/latest/ --out=eval-sets/customer-support-v1

# 2. 本地 schema 校验
kweaver trace schema validate eval-sets/customer-support-v1/index.yaml --kind=eval-set-index

# 3. 用当前 agent 配置跑一次测试，生成 baseline test report
kweaver trace eval-set test eval-sets/customer-support-v1 --candidate=agent-current.yaml --out=test-runs/baseline/
```

Story B 主要落在 M5 Eval-Set Builder：构造 eval-set、脱敏、schema 校验、调用远端 DA / evaluator 跑测试。它仍不需要 M6 Experiment Engine，也不做多候选优化。

#### 0.1.3 Story C：单路径持续迭代

Story C 的目标是沿一个明确方向连续迭代，而不是同时探索 K 条路径。用户先根据诊断建议手工或半自动改一个候选配置，再用同一套 eval-set 验证是否改善。

```bash
# 1. 准备单路径 mission：只声明一个当前候选和一个下一步修改方向
cd my-experiment
$EDITOR mission.md

# 2. 启动 / 续跑单路径迭代
kweaver trace exp run . --mode=single-path

# 3. 查看配置、进展、健康度
kweaver trace exp show .
kweaver trace exp status .
kweaver trace exp doctor .

# 4. 需要停止时优雅中止
kweaver trace exp abort .
```

MVP-C 的 M6 只维护一条 candidate lineage：baseline → candidate-v1 → candidate-v2。它输出 `outputs/{bundle,manifest,provenance}.yaml`，但不做 Trial Forest、多路径 slot 分配、自动探索 / 利用调度或线上 feed 回流。

**mission.md v1 (single-path) 必填字段**：`goal` / `eval_sets[]` / `current_candidate` / `next_change`。其他多路径相关字段（`variation_points` / `resources` / `fallback_queries` 等）保留给 post-MVP，MVP-C 解析器忽略它们。

```yaml
schema_version: trace-mission/v1
goal: "降低退款流 agent 重复调用 tool 的概率"
eval_sets:
  - path: eval-sets/customer-support-v1/
    role: seed
current_candidate:
  path: candidates/baseline.yaml
next_change:
  target: decision_agent.prompt
  hypothesis: "加 stop condition 后两次失败检索即终止"
```

#### 0.1.4 日常操作场景

| 场景 | 用户目标 | 首选命令 | 读写行为 |
|---|---|---|---|
| 诊断单条 trace | 识别 agent program 的不合理性 | `kweaver trace diagnose <trace_id>` | 读 M3，写 diagnosis yaml |
| 扫一批可疑 trace | 批量输出诊断候选 | `kweaver trace diagnose scan ...` | 读 M3，写 diagnosis 输出 |
| 构建 eval-set | 把诊断结果变成可复现 cases | `kweaver trace eval-set build ...` | 读 diagnosis / trace，写 eval-set |
| 跑 baseline 测试 | 确认当前 agent 在 eval-set 上的问题 | `kweaver trace eval-set test ...` | 调 DA/evaluator，写 test report |
| 新建单路径迭代任务 | 描述一个修改方向和验收 eval-set | 手写 `mission.md`，然后 `kweaver trace exp show .` | show 只读 |
| 启动 / 续跑单路径迭代 | 让 M6 Coordinator 接管单 candidate lineage | `kweaver trace exp run . --mode=single-path` 或 `resume` | 写 `.trace-state/`、round 快照、outputs |
| 看当前跑到哪 | 一次性看 FSM state、round、pending jobs、最近错误 | `kweaver trace exp status .` | 只读 |
| 长时间盯进展 | 实时看 K×M 执行矩阵和最近事件 | post-MVP：`kweaver trace exp watch .` | 只读 |
| 排查为什么跑不动 | 校验 mission/eval-set/journal/lock/jobs/outputs | `kweaver trace exp doctor .` | MVP 只读，不自动修复 |
| 停止当前 driver | 让 driver 在下一个 checkpoint 优雅退出 | `kweaver trace exp abort .` | 写 `abort.signal` |
| 扫多个实验 | 看一批实验目录的概要状态 | post-MVP：`kweaver trace exp list <path...>` | 只读 |
| 保存候选产物 | 取走实验输出供外部发布平台使用 | 直接读取 `outputs/` | CLI 不写外部系统 |
| replay 对比 | 用新 trial 重跑历史 trace 看 diff | post-MVP：`kweaver trace replay ...` | 触发 DA replay；新 trace 喂回 M3 |
| 上线后复盘 | 人工或外部平台根据产物地址拉 trace 复盘 | 外部流程；post-MVP 可接 verify/replay | MVP 不内置 post-deploy verify |

#### 0.1.5 用户心智边界

- **诊断先于实验**：MVP-A 的入口是 trace，不是 mission；先把"哪里写得不合理"讲清楚。
- **测试资产由诊断沉淀**：MVP-B 把 diagnosis 转成 eval-set，并跑 baseline test。
- **迭代保持单路径**：MVP-C 只沿一个候选链前进；多路径探索和自动飞轮推迟。
- **任务配置由人写**：Story C 的 `mission.md` 是入口；`eval-sets/`、diagnosis report 是输入。
- **运行态由 CLI 写**：`.trace-state/` 下的 journal、lock、job 流水、round 快照都不要求用户手写。
- **进展靠只读命令看**：MVP-C 提供 `show/status/doctor`；`watch/list` 是后续观测增强，不和 `run/resume` 抢 lock。
- **发布和验证不进 MVP 边界**：M6 只把 bundle / manifest / provenance 写到 `outputs/`；保存地址、上线动作、上线后对账由用户或外部发布平台负责。中央 publish-registry、全局 bundle 索引、周期 `verify scan` 都推迟到 post-MVP。
- **trace 查询统一走 M3**：用户命令不直接暴露 OpenSearch DSL；生产 trace 的全文输出默认脱敏。

### 0.2 MVP 必须性切分

按"诊断 → 测试资产 → 单路径迭代 → 多路径飞轮"推进。MVP 只覆盖前三段，post-MVP 才做多路径和飞轮。

| 层级 | 必须性 | 包含能力 | 不包含什么 |
|---|---|---|---|
| **MVP-A：轨迹诊断** | 必须 | `trace diagnose <trace_id>`、`trace diagnose scan`、静态信号规则 schema + 至少 5 条 baseline、diagnose-provider wrapper（B2 调远端 LLM provider）、诊断报告 schema、M3 trace 读取、`schema validate` | 不要求 eval-set、实验文件夹、自动修复、replay |
| **MVP-B：eval-set 构建与测试** | 第二阶段 | `trace eval-set build --diagnosis=...`、`trace eval-set test`、脱敏、baseline test report | 不做 hindsight relabel；不做多候选优化 |
| **MVP-C：单路径迭代** | 第三阶段 | `trace exp run/resume/show/status/abort/doctor --mode=single-path`、ExpStore、RemoteJob submit/poll、`outputs/{bundle,manifest,provenance}.yaml` | 不做 Trial Forest、多路径并行、watch/list、git push、registry |
| **Post-MVP：多路径 / 飞轮增强** | 延后 | Trial Forest、多 Trial 并行、M7 replay、`eval-set relabel`、`schema audit`、`exp watch/list`、Git checkpoint/push、M8/M9、自动 curation feed | 只有单路径手工流程成为瓶颈时再做 |

MVP-A 验收口径：给定一条或一批线上 trace，CLI 能输出带证据、原因假设、修改建议和验证建议的诊断报告。MVP-B 验收口径：诊断报告能转成可复现 eval-set，并跑出 baseline test report。MVP-C 验收口径：用户沿一个候选方向持续迭代，状态可恢复，产物可交接。

## §1 总体形态

### 1.1 物理位置

CLI 是 **kweaver-sdk monorepo** 的一部分——不是独立二进制，不是独立仓库。所有 trace-ai 的 above-L0 能力作为 `kweaver` 命令的子命名空间暴露给用户：

```
kweaver-sdk/
  └── packages/typescript/
        ├── bin/kweaver.js              # 唯一二进制入口（既有）
        ├── src/
        │   ├── cli.ts                  # 顶层 dispatch（既有，新增 trace 路由）
        │   ├── commands/
        │   │   ├── (既有命令 agent / bkn / dataflow / ...)
        │   │   └── trace/              # ← 本文档定义的 trace-ai 子命名空间
        │   │         ├── exp.ts
        │   │         ├── diagnose.ts
        │   │         ├── eval-set.ts
        │   │         ├── replay.ts
        │   │         └── schema.ts
        │   ├── api/
        │   │   ├── (既有 api clients)
        │   │   └── trace/              # ← trace-ai 专属 HTTP 客户端
        │   │         └── observability.ts
        │   └── trace-core/             # ← trace-ai 专属内核（FSM / async-poll / schema / exp-store / exp-engine；git checkpoint post-MVP）
        │         ├── exp-store/
        │         ├── exp-engine/
        │         ├── diagnose/
        │         ├── eval-set/
        │         ├── replay/
        │         ├── schema/
        │         ├── remote-job.ts
        │         └── git-checkpoint.ts # post-MVP
        └── schema-mirrors.lock         # ← schema 静态契约同步源 SHA 锁
```

**单向依赖**：`commands/trace/*` → `trace-core/*` → `api/trace/*` → `auth/` + `config/`（既有基础设施）

### 1.2 命令树总览

命令按 cli_conventions §2 "顶层 + 子资源 + 动作" 三段式落地，分层交付：

```
# MVP-A：轨迹诊断
kweaver trace diagnose  <trace_id> | scan | rules list | rules validate          # M4 diagnosis
kweaver trace schema    validate                                                 # MX1 local validate

# MVP-B：eval-set 构建与测试
kweaver trace eval-set  build | test                                             # M5

# MVP-C：单路径迭代
kweaver trace exp       run | resume | show | status | abort | doctor            # M6 single-path

# Post-MVP：多路径 / 飞轮 / 批量观测 / 自动化增强
kweaver trace exp       watch | list
kweaver trace eval-set  relabel
kweaver trace replay    <trace_id|--experiment-id+--query>                       # M7
kweaver trace schema    audit                                                    # MX1 audit
```

`diagnose` 是用户面对线上 trace 的第一入口；`eval-set build/test` 把诊断沉淀为测试资产；`exp` 只负责 MVP-C 的单路径迭代。`replay`、`watch/list` 和 `schema audit` 都是 post-MVP 增强，不阻塞前三段。

M8 `bundle` 与 M9 `verify` 不进 MVP 命令树：bundle / manifest / provenance 直接作为 M6 的 `outputs/` 产物交接。

### 1.3 与现有 kweaver 命令的关系

- **不影响 `kweaver agent trace <conv_id>`**——那是 `agent` 资源的 `trace` **动作**（v0.7.4 已上线的 4 视图查询）；本文档定义的 `kweaver trace ...` 是顶层 **资源** 命名空间，承载 trace-ai 持续学习子系统。help 文本里点一句区分。
- **复用 `kweaver call` / `kweaver auth` / `kweaver config` 全部既有能力** —— profile / no-auth 哨兵 / TLS env / `--verbose` 这些都不重新造。
- **顶层 flag**（`--base-url` / `--token` / `--user` / `-bd`）继承现有 `cli.ts` 的 strip 逻辑。
- **测试约定** 沿用 `docs/cli_conventions.md` §7：解析器单测 + API 客户端单测 + e2e smoke。

## §2 共享层（trace-core 内核 + api/trace 客户端）

trace 子命令家族复用 kweaver-sdk 现成基础设施（`auth/` / `config` / `utils` / `ui`）之上，新增共享组件。MVP-A 需要 B1/B2/B5/B6（B2 用于调 diagnose-provider）；MVP-B 复用 B2 跑 eval-set test；MVP-C 再引入 B3 ExpStore。B4 延后为可选 git checkpoint。

### 2.1 共享层组件清单（B1–B6）

| 组件 | 路径 | 职责 | 谁用 |
|---|---|---|---|
| **B1 ObservabilityClient** | `src/api/trace/observability.ts` | M3 HTTP 包装：`/traces/{id}` / `/traces?...` / 时间窗 + 分页 + 配额错误码处理 | MVP-A：diagnose 读 trace；MVP-B：eval-set build；post-MVP：replay / watch / audit |
| **B2 RemoteJobClient** | `src/trace-core/remote-job.ts` | async submit + poll 抽象（vision §6.4.4）：`submit(target, payload) → job_id` / `poll(job_id) → status\|result`；MVP 期内置 sync 降级开关 `--sync` | MVP-A：M4 diagnose-provider；MVP-B：eval-set test；MVP-C：single-path executor/scorer/generator；post-MVP：relabel / replay |
| **B3 ExpStore** | `src/trace-core/exp-store/` | 实验文件夹持久化抽象（vision §6.4.3）：`mission.md` / `.trace-state/{events.jsonl, candidate-lineage.yaml, jobs.jsonl, lock.json, abort.signal, rounds/}` / `outputs/` 读写 + lockfile 协议（hostname+pid+30s 心跳） | MVP-C：M6 单路径迭代 |
| **B4 GitCheckpointDriver** | `src/trace-core/git-checkpoint.ts` | post-MVP 可选：git CLI 包装（spawn `git` 子进程，不引 nodegit / isomorphic-git）：commit / push / pull / `git ls-tree` / `git show`；约定式 commit message | post-MVP：M6 checkpoint；MVP-C 默认只写本地文件，git 由用户或外部流程处理 |
| **B5 SchemaRegistry** | `src/trace-core/schema/` | schema 静态契约 mirror 副本（`schema/v1/*.yaml`）+ ajv 校验器 + 别名兼容表 + 版本 / 兼容窗口；提供 `validate(kind, doc) → result` | 全 M 模块；MX1 直接暴露 |
| **B6 OutputFormatter** | `src/ui/trace/` | 复用 `src/ui/` 的 ink 设施；`--json` / `--pretty` / `--compact` 三档；长任务 progress（reuse ink-spinner） | 全 M 模块 |

**关键边界约束**：

1. **`trace-core/` 不依赖 `commands/trace/*`**——单向。让 driver 主体（M6 Coordinator）可作为库被未来其他东西调用（比如本地 dashboard / 第三方 wrapper）。
2. **B1 ObservabilityClient 是 above-L0 唯一的 trace 读入口**——任何 M 模块要读 trace **不允许**绕过它直接打 `kweaver call` 或 OpenSearch DSL，避免 schema-bypass。
3. **B5 SchemaRegistry 自带 schema 静态文件副本**——CLI 自包含、无运行时网络依赖；schema 升级 = SDK 发版。

### 2.1.1 B1 / M3 API prerequisite matrix

B1 是 CLI 侧唯一 trace 读入口，但它背后的 M3 能力在落地期会分阶段补齐。为避免各命令在缺接口时私自绕回 OpenSearch DSL，统一按下表处理：

| B1 方法 | M3 目标接口 | 谁用 | 阶段要求 | 缺失时 CLI 行为 |
|---|---|---|---|---|
| `getTrace(traceId)` | `GET /traces/{trace_id}` | diagnose 单条输入 / post-MVP replay | MVP-A | 不绕过；返回 `TRACE_DETAIL_API_REQUIRED`，提示先升级 M3 |
| `searchTraces(query)` | `GET /traces?...` 或受控 POST 查询 | diagnose scan / eval-set build / MX1 | MVP-A | 不暴露 OpenSearch DSL；只允许 B1 内部使用 M3 受控查询契约 |
| `searchTracesStream(query, page)` | `GET /traces?...&page_token=&page_size=` 或受控 POST 查询 | diagnose 大集合扫描 / eval-set build | MVP-A | 不一次性拉全量；缺分页时返回 `TRACE_PAGINATION_API_REQUIRED` |
| `listExperimentTraces(experimentId, filters)` | `GET /traces?experiment_id=...` | post-MVP replay 间接定位 / M6 watch | post-MVP | 降级为 `searchTraces` 的结构化过滤，不允许命令层拼 DSL |
| `diffTraces(a, b)` | `GET /traces/diff?a=&b=` | M7 replay | post-MVP | 缺失时 B1 内部拉两条 trace 做 naive diff，并输出 `diff_engine=local-naive` |
| `sampleTraces(window, sample)` | `GET /traces?time_window=&sample=` | MX1 audit | post-MVP | 报 `TRACE_SAMPLE_API_REQUIRED`；audit 不做全量扫描 |

**M3 错误码要求**：B1 需要区分 `SCHEMA_VALIDATION_FAILED` / `QUERY_TOO_LARGE` / `RATE_LIMITED` / `STORAGE_UNAVAILABLE` / `AUTH_FORBIDDEN`。如果 M3 仍返回泛化 5xx，B1 保留原始 HTTP status + response body 摘要，CLI 输出里标 `unclassified_m3_error=true`，避免误判成远端 job 或 CLI bug。

**流式读取契约**：`searchTracesStream` 是 B1 暴露给 M4 的唯一大集合入口，返回 `AsyncIterable<TraceBatch>`，每个 batch 带 `{items, next_page_token?, response_meta}`。CLI 默认 `page_size=500`，可由命令内核按内存压力下调；命令层只消费 async iterator，不直接感知 M3 分页字段。M3 必须保证同一查询在短时间窗口内的分页稳定性；若无法保证，B1 输出 `pagination_consistency=best_effort`，并要求调用方在报告里标注抽样/扫描口径。

### 2.1.2 B2 RemoteJobClient 契约

B2 是所有远端智能体 / 评分 / replay job 的唯一出口。MVP 的 HTTP 字段细节由各远端服务 spec 钉，但 CLI 侧抽象必须稳定：

```ts
submit(target, payload, { idempotencyKey, timeoutSec? }) -> {
  job_id: string,
  status: "queued" | "running" | "succeeded" | "failed" | "expired" | "cancelled",
  submitted_at: string,
  result_ttl_until?: string
}

poll(job_id) -> {
  job_id: string,
  status: "queued" | "running" | "succeeded" | "failed" | "expired" | "cancelled",
  result?: unknown,
  error?: { code: string, message: string, retryable: boolean },
  progress?: { completed: number, total: number }
}
```

**恢复语义**：

1. `idempotencyKey` 必填，由 CLI 用 `experiment_id + round + trial_id + query_id + stage` 计算；driver 崩溃后重复 submit 不应制造第二个远端 job。
2. 远端 job 结果 TTL 建议 ≥ 72h；`expired` 表示远端结果已不可取，M6 将该 cell 标为 `job_expired` 并按策略重做或跳过。
3. `failed.retryable=true` 走指数退避；超过阈值后写入 events.jsonl，不在内存里静默丢弃。
4. `cancelled` 只表示远端已停止；`exp abort` 的 MVP 行为是本地优雅退出，不强依赖远端 cancel 成功。
5. `--sync` 只允许 dev/debug 与小集合 relabel；正式 `exp run` 路径必须落 `jobs.jsonl` 的 job transport 记录与 `events.jsonl` 的业务推进事件，不能把 sync 结果绕过 journal。

### 2.1.3 trace 输出脱敏策略

trace 子命名空间默认面向生产 trace，输出层必须先按安全默认值收口：

- `--pretty` / `--compact` 默认对 prompt、LLM message、tool arguments、tool result、retrieval document 正文做摘要化：保留类型、长度、hash、前后短片段；不打印全文。
- `--json` 默认同样脱敏，避免程序消费路径成为泄露旁路。
- 显式 `--full` 只解除长度截断；敏感字段仍按 redaction rules 脱敏。
- 只有 `--unsafe-full` 才打印原文字段；命令必须二次提示或在非交互环境要求 `KWEAVER_TRACE_UNSAFE_FULL=1`。
- redaction rules 查找顺序：命令行 `--redaction-rules` > repo `<repo>/redaction-rules/` > 内置低保真规则。规则缺失时不阻断命令，但输出标 `redaction_rules=default`。

### 2.2 Schema 静态契约 mirror 同步机制

vision §7.MX1 钉死 schema/v1/ 是 git-tracked 静态契约，物理 SSOT 在 trace-ai 仓库；CLI 在 kweaver-sdk monorepo 持有 mirror 副本。同步机制：

```
SSOT:   trace-ai/schema/v1/*.yaml                                         ← 维护者写
                  │
                  │ (人工纪律：schema PR 同步推送 mirror)
                  ↓
Mirror: kweaver-sdk/packages/typescript/src/trace-core/schema/v1/*.yaml   ← CLI 用
Lock:   kweaver-sdk/packages/typescript/schema-mirrors.lock               ← 记录同步源 SHA
```

**`schema-mirrors.lock` 格式**（多源可扩展，MVP 阶段只有一项）：

```yaml
sources:
  - name: trace-ai-schema
    source-repo: github.com/<org>/trace-ai
    source-path: schema/v1/
    target-path: packages/typescript/src/trace-core/schema/v1/
    synced-at-sha: <40-char SHA>
    synced-at: 2026-05-09T...
```

**CI lint 校验**：kweaver-sdk CI 跑一个 lint 步骤，扫 `sources` 数组每条：①SHA 在 trace-ai 仓库存在；②mirror target-path 下文件 hash 跟源端 SHA 对应文件 hash 一致。Drift 立刻 CI 红。

**为什么不发 npm 包 / 不用 git submodule**：MVP 阶段 schema 改动频率低（minor 级 / 周月节奏），人工纪律 + lock 文件 + CI lint 已经够；等真有 drift 咬到再升级到自动化（CI 周期同步开 PR）。

### 2.3 产物边界：outputs/ 即交接面

MVP 阶段不引入中央 publish-registry，也不提供 bundle artifact resolver。实验结束后，M6 在实验目录 `outputs/` 下产出三类文本产物：

```
outputs/
  ├── bundle.yaml
  ├── manifest.yaml
  └── provenance.yaml
```

这三个文件就是 trace-ai CLI 与外部发布平台 / 人审流程的交接面。CLI 保证它们 schema 合规、来源可追溯；至于如何保存、上传、上线、回滚、周期验证，全部由用户或外部发布平台负责。这样 MVP 少掉一套全局索引、artifact 拉取协议、registry 并发写入协议与定时扫描器。

### 2.4 用户配置心智

CLI 实现会读写多类文件，但用户只需要按三类配置理解：

| 用户心智 | 实际载体 | CLI 行为 |
|---|---|---|
| **环境配置：我连哪里** | `kweaver config` / env | `auth` / `config` 统一读取；trace 子命令不另建配置系统 |
| **诊断配置：我怎么识别 agent program 问题** | `diagnosis-rules/` / `diagnosis-policy.yaml` | M4 读取规则，输出 diagnosis report |
| **测试资产：我要复现哪些问题** | `eval-sets/` | M5 生成 / 校验 / 测试 |
| **任务配置：我要沿哪个方向优化** | `mission.md` + `eval-sets/` | M6 `exp run/resume --mode=single-path` 的主入口 |
| **产物交接：我要把什么交给发布平台** | `outputs/{bundle,manifest,provenance}.yaml` | M6 生成；CLI 不上传、不索引、不周期验证 |

`.trace-state/*`、watermark、`outputs/`、`verification.yaml` 都是运行态或生成物；CLI 可以展示和校验，但用户默认不手写。`eval-sets/` 例外：既可以由用户预置，也可以由 M5 生成。

## §3 模块详设

### 3.1 M4 Diagnosis — `kweaver trace diagnose`

**命令树**（vision §7.M4）：

```
kweaver trace diagnose <trace_id>       [--rules=<path>] [--out=<file>]                # case mode
kweaver trace diagnose --traces=<id-list> [--rules=<path>] [--out=<file>]              # batch mode：多 trace 比较诊断
kweaver trace diagnose scan             [--policy=<path>] [--time-range=] [--tenant=] [--rules=<path>] [--out=<dir>]
kweaver trace diagnose rules list       [--rules-dir=<path>]
kweaver trace diagnose rules validate   <rule.yaml>

# post-MVP alias / extension：生产 feed 自动聚合后可恢复 curate 字面
kweaver trace curate scan               [--feed=<path>] [--update-watermark] ...
```

**边界（vs M6 Triage）**：M4 三种 mode 的 subject 都是 **agent program**——回答"代码哪里写得不合理"。**实验调度**（剪枝 / 收敛判定 / 探索 vs 利用 / slot 跨树分配）由 M6 Triage Agent 承担，subject 是实验进程；M6 Triage 可消费 M4 报告作为证据，但不重做 trace 级诊断。

**代码模块布局**：

```
src/commands/trace/diagnose.ts           # 参数解析 + dispatch（含 rules 子资源）
src/trace-core/diagnose/
  ├── planner.ts             # policy + CLI flags → DiagnosisPlan
  ├── policy.ts              # policy yaml loader
  ├── signal-probe.ts        # 静态信号层：rule / FSM / schema signal 检测
  ├── trace-loader.ts        # 通过 B1 拉单 trace / 批量 trace
  ├── provider-wrapper.ts    # ★ Diagnose Provider Wrapper：拼 payload / B2 submit / poll / 解析
  ├── report.ts              # 合成 diagnosis report yaml（static + provider 输出）
  ├── rules.ts               # rules yaml loader + ajv 校验
  ├── invariant.ts           # post-MVP：声明式不变量评估（在 BKN 上 query）
  ├── latent-failure.ts      # post-MVP：guard-code-as-oracle 检测
  ├── watermark.ts           # post-MVP：周期 scan 的增量游标读写
  └── feed-pickup.ts         # post-MVP：从显式 --feed 路径拾取 curation-feed.yaml
```

**依赖**：

- **B1 ObservabilityClient** — 拉单条 trace 或时间窗 / 租户范围内的 trace
- **B2 RemoteJobClient** — 调远端 diagnose-provider 走 async submit + poll（rule-only 模式可跳过）
- **B5 SchemaRegistry** — 校验 rules yaml + diagnose payload + diagnosis report schema
- **kweaver-sdk 现有 BKN api client** — post-MVP optional 反查不变量

**关键设计点**：

1. `diagnose <trace_id>` 是 MVP-A 主路径：trace + 静态信号 → diagnose-provider → diagnosis report。
2. **双轨架构**：静态信号层（本地）抓 surface pattern + 给 LLM 提供证据 anchor；LLM 诊断层（远端 provider）做语义级判断。CLI 内部是 thin binding——和 §3.3.4.1 Agent Synthesizer / §3.3.4.2 Triage Wrapper 同一套架构。
3. `diagnose scan` 是批量诊断：通过 B1 `searchTracesStream` 分页 streaming 处理，不落整集到内存；signal-probe 先跑（廉价），命中后再交给 provider-wrapper（贵），并发度受 `--max-parallel` 控制。
4. `--no-llm` 是离线降级开关：只跑静态信号层，diagnosis report 的 `provider` 字段标 `rule-only`，`likely_cause` / `suggested_fix` 用 rule yaml 自带的静态模板填充（不靠 LLM 生成）。MVP-A 默认开启 LLM 层。
5. `rules` 子资源符合 cli_conventions §2 "顶层 + 子资源 + 动作"——规则文件本身 git-tracked（约定 `<repo>/diagnosis-rules/`），CLI 不管 register API。
6. **MVP-A 不自动拉中央 feed**：`scan` 只读取本地规则。上线后发现的新失败模式先由人工沉淀为本地 rule；中央 feed 聚合留到 post-MVP。

**Diagnose Provider Wrapper（payload + provider 抽象）**：

provider 注册机制和 Agent Synthesizer 一致（capability check 在 `diagnose` 启动前跑，失败 fail-fast）。MVP 期支持两个 provider：

| provider | 适用场景 | 输出要求 |
|---|---|---|
| `claude-code` | 本地 / 半自动诊断：在受控 workspace 跑 LLM | 输出 diagnosis report（schema 见下） |
| `decision-agent` | 平台内 dogfood：用 KWeaver DA 诊断 | 输出同一 schema |

**Diagnose request payload**（CLI 拼，B2 submit）：

```yaml
schema_version: trace-diagnose-request/v1
target_trace:
  trace_id: tr_123
  trace_doc_ref: <M3 fetch 引用，或内联 trace 摘要>
static_signals:
  - rule_id: tool_loop_no_state_change
    spans: [sp_7, sp_8, sp_9]
    severity: high
  - rule_id: retrieval_empty_no_fallback
    spans: [sp_4]
    severity: medium
context:
  agent_id: agent_123
  tenant: acme
  agent_program_ref: <可选：当前 agent 配置 git ref>
```

**Diagnose response → diagnosis report**：provider 必须返回符合 `trace-diagnose-report/v1` schema 的对象，包含 `symptom / likely_cause / suggested_fix / confidence / evidence / verify_with.suggested_eval_case`。CLI 在 Report Assembler 阶段把静态信号也并入同一份 report 的 `static_signals` 段，方便审计。

**Payload 压缩约束**（batch / scan mode 必须；case mode 仍按以下规则避免 context 溢出）：

provider-wrapper 在拼 payload 前必须执行四条硬约束，**不允许**把原始 trace 直接灌进去——一条复杂 trace 几十到几百 span，K 条同送必撑爆 LLM context：

1. **Span 选择**：仅 candidate spans（signal-probe 标的可疑 span）+ 每个 candidate 前后 ≤2 个上下文 span 进 payload；其余 span 退化为骨架（`span_id / name / status / duration_ms`），不带 payload 字段
2. **大字段摘要**：`prompt` / `tool_args` / `tool_result` / `retrieval_doc` 正文一律替换为 `{hash, length, head: <前 200 字>, tail: <后 200 字>}`；`--full-content` 显式开关绕过（默认 off）
3. **Batch dedup**：batch mode payload 结构改为 `{shared_context, per_trace_overlay[]}`——同源（同 query / 同 agent 配置 / 同环境）部分提到 `shared_context`，per-trace 只放差异
4. **Token budget**：CLI 端用 tokenizer 估算 payload token 数；单次 ≤ 32k tokens；超限 fail-fast 提示用户拆批（`--max-traces-per-batch`），不静默截断

**不钉具体序列化格式**——provider wrapper 各自决定 JSON / YAML / 紧凑格式（如 TOON）。trace-ai 只钉 payload **结构语义 + token budget**；格式选择留给 provider 接口契约自身演进。

**DiagnosisPolicy 最小契约**：

```yaml
policy_id: prod-agent-diagnosis
scope:
  tenant: acme
  agent_id: agent_123
  time_window: 24h
rulesets:
  - diagnosis-rules/
```

`DiagnosisPlan` 是运行时派生对象，不需要用户手写。输出侧按 `trace_id + rule_id` 去重。

**核心流程与逻辑（以 `diagnose scan` 为例）**：

1. **环境准备与规则加载**：
   - `planner.ts` 读取 `--policy` 与 CLI flags，生成本次 `DiagnosisPlan`。
   - 从默认路径 `<repo>/diagnosis-rules/`、policy 声明的 rulesets、用户指定的 `--rules` 路径加载 YAML 规则。
2. **数据流式获取**：
   - 调用 B1 (ObservabilityClient)，传入时间窗和租户参数，获取 Trace 数据的异步分页流（`AsyncIterable<TraceBatch>`）。
3. **管道式过滤（Core Pipeline）**：
   - 建立一个 Transform Stream 管道，每条 Trace 依次流经：
     - `signal-probe`：解析 Span 属性，匹配交互/执行层异常规则。
     - `invariant`：optional；若规则涉及 BKN 不变量，调用 BKN API 验证该 Trace 是否满足“应读未读”等约束。
     - `latent-failure`：通过确定性规则反查隐性失败。
   - 任何一个环节触发规则，即为该 Trace 打上 `rule_id`、symptom、evidence spans、likely cause 和 suggested fix。
   - **短路机制**：一旦某条 Trace 触发了任意一条规则，立即流向输出缓冲区，不再继续做耗时的后续反查。
4. **结果输出**：
   - 命中的 Trace 及其诊断报告被收集，通过 `report.ts` 序列化为 YAML 格式。
   - 写入到指定的输出目录（默认为 `diagnosis-output/`），文件名包含时间戳。

### 3.2 M5 Eval-Set Builder — `kweaver trace eval-set`

**命令树**（vision §7.M5）：

```
kweaver trace eval-set build   [--diagnosis=<path>] [--queries=<path>] --out=<dir> [--with-reference]  # MVP-B
kweaver trace eval-set test    <eval-set-dir> --candidate=<path> [--out=<dir>] [--sync]                # MVP-B
kweaver trace eval-set relabel <eval-set-dir> [--sync] [--force]                                      # post-MVP
```

**代码模块布局**：

```
src/commands/trace/eval-set.ts            # 参数解析 + dispatch
src/trace-core/eval-set/
  ├── loader.ts              # EvalSetRef[] → EvalCase[]，支持 index + shard
  ├── query-picker.ts        # 从 diagnosis report / queries yaml 圈选
  ├── redactor.ts            # 自动脱敏（PII / 业务密文 patterns；规则 yaml-driven）
  ├── test-runner.ts         # MVP-B：薄包装，复用 M6 single-path executor 跑 1 round + 写 test report
  ├── relabel.ts             # post-MVP：hindsight relabel via async LLM（用 B2 RemoteJobClient）
  └── output.ts              # eval-set yaml 多文件目录写入
```

**依赖**：

- **B1 ObservabilityClient** — 圈选时拉 trace 拼上下文
- **B2 RemoteJobClient** — `eval-set test` 调远端 DA / evaluator；post-MVP relabel 也复用
- **B5 SchemaRegistry** — eval-set yaml schema 校验
- **不依赖 B4 GitCheckpointDriver**——eval-set 写到 `<repo>/eval-sets/<name>/`，git 由用户自己 commit/push

**关键设计点**：

1. eval-set 是一等输入，不只是 M5 产物。用户已有“请求 + 标准答案”可直接放入 `<repo>/eval-sets/<name>/`，由 `mission.md` 引用；M5 生成的数据也写成同一格式。
2. `build` 是无状态的纯函数——输入 (diagnosis report, queries yaml) → 输出 yaml 集合。**`--diagnosis=` 读取 M4 报告的 `verify_with.suggested_eval_case` 段直接生成 shard**，无需用户在 diagnosis 与 eval-set 之间手工搬数据。两态（带 reference / 不带）由 `--with-reference` flag 切。
3. `test` 是 MVP-B 的闭环验证：只跑一个 candidate，不比较多个候选，不生成优化建议。**实现上是 M6 single-path executor 的薄包装**（跑 1 round + 立即 Termination + 写 test report），避免在 M5 重写一套调度逻辑；MVP-C 跑多 round 时直接复用同一份代码。
4. `relabel` 不进 MVP-B；后续实现时必须 async（vision §6.4.4 钉死方向）。输出**沿用同一个 eval-set 目录原地改写**——relabel 的 audit 痕迹靠 git history 本身承载，不复制到新目录。
4. **redaction rules 不属于 schema/v1/ 范畴**——是企业敏感信息 pattern 库，由组织自建于 `<repo>/redaction-rules/`；vision §9.3 加注此约定。
5. **原地改写必须有 git 安全前置**：默认要求 `<eval-set-dir>` 位于 git repo 内，且将被改写的 eval-set 文件没有未提交改动；否则拒绝并提示用户先 commit / stash。`--force` 可跳过 dirty 检查，但输出必须标 `audit_risk=dirty_worktree`。

**用户预置 eval-set 目录契约**：

```
eval-sets/
  customer-support-v1/
    index.yaml
    refund.yaml
    shipping.yaml
```

`mission.md` frontmatter 引用：

```yaml
eval_sets:
  - path: eval-sets/customer-support-v1/
    role: seed
  - path: eval-sets/regression-v2/
    role: regression
```

`index.yaml`：

```yaml
schema_version: trace-eval-set-index/v1
eval_set_id: customer-support-v1
shards:
  - path: refund.yaml
  - path: shipping.yaml
```

shard 文件：

```yaml
schema_version: trace-eval-set/v1
cases:
  - query_id: refund_001
    input:
      user_message: "如何申请退款？"
    reference:
      answer: "请在订单详情页点击申请退款。"
    assertions:
      - type: contains
        value: "订单详情页"
    tags: ["refund"]
```

约束：`query_id` 在整个 eval set 内唯一；shard path 必须是目录内相对路径，禁止 `../` 越界；`role` MVP 枚举为 `seed | regression | holdout`。目录必须有 `index.yaml`；不做隐式读取 `*.yaml`，避免误读临时文件。

**核心流程与逻辑**：

**`eval-set build` 流程**：
1. **源数据圈选**：
   - `query-picker.ts` 聚合输入的 `--diagnosis`（来自 M4 的诊断报告）和 `--queries`。
   - 如果指定了 `--with-reference`，会尝试从历史 Trace 中提取“成功的标准输出”作为 Ground Truth（Reference）。
2. **敏感信息脱敏**：
   - 遍历圈选出的 Query 和上下文，调用 `redactor.ts`。
   - 根据 `<repo>/redaction-rules/` 中的规则，对 PII（个人隐私）和业务密文进行脱敏（替换为 Hash 或脱敏占位符）。
3. **写入与校验**：
   - 调用 `output.ts` 将脱敏后的数据写入指定的输出目录（`<repo>/eval-sets/<name>/`）。
   - 调用 B5 (SchemaRegistry) 校验生成的 YAML 是否符合 Eval-Set 的标准 Schema。

**`eval-set test` 流程**：
1. **加载与 preflight**：
   - 读取 eval-set index / shard，校验 schema、query_id 唯一性、shard path 与 role。
   - 读取 `--candidate` 指向的 agent 配置快照，校验必填资源引用。
2. **执行与评分**：
   - 通过 B2 RemoteJobClient 调 DA 执行 eval cases；`--sync` 只用于小集合调试。
   - 调 evaluator 输出 pass/fail、severity、diagnosis_ref、trace_id。
3. **报告输出**：
   - 写 `test-runs/<name>/report.yaml`，作为 Story C 的 baseline。
   - 不更新 eval-set，不生成新 candidate，不进入多路径优化。

**`eval-set relabel` 流程（post-MVP）**：
1. **加载与提交**：
   - 读取指定目录下的 Eval-Set 文件。
   - 执行 git preflight：确认目录在 git repo 内、目标文件 clean；`--force` 时只警告不阻断。
   - 调用 B2 (RemoteJobClient) 将需要打标的失败轨迹（如 latent failure）打包，异步提交给远端 LLM（或通过 `--sync` 同步处理）。
2. **结果轮询与就地改写**：
   - 若为异步，CLI 周期性轮询（Poll）Job 状态。
   - 任务成功后，拉回“原行为 vs 应有行为”的偏好对（Preference Pairs）。
   - **核心逻辑**：不创建新目录，直接**原地改写（In-place Rewrite）**原 Eval-Set 文件。审计和版本追踪完全依赖 Git History。

### 3.3 M6 Experiment Engine — `kweaver trace exp`

M6 只进入 MVP-C，且 MVP-C 只做**单路径迭代**：一条 candidate lineage 从 baseline 逐步演进，不做多 Trial 并行、Trial Forest、自动探索 / 利用 slot 分配。M6 仍带 FSM、长生命周期、跨进程重启和 async 远端编排，但复杂度被限制在单候选链。

**实验目录用户视图**：

```
my-experiment/
  ├── mission.md              # 必填：任务配置，用户主要编辑
  ├── diagnosis/              # 可选：MVP-A 诊断报告；可作为迭代证据
  ├── eval-sets/              # 用户预置或 M5 生成；可 review
  ├── outputs/                # 生成物：bundle.yaml / manifest.yaml / provenance.yaml
  └── .trace-state/           # 运行态：events/jobs/lock/rounds，用户不手写
```

`exp run --mode=single-path` 的冷启输入规则：加载 `mission.md` 中引用的 eval set 与 baseline test report；每个 round 只产生一个下一版 candidate。若没有 eval set，命令 fail-fast 并提示先执行 Story B；MVP-C 不从 mission fallback queries 隐式生成 eval set。

**`exp run` preflight 卡点**：

`exp run` / `resume` 在抢到 lock 后、进入 active FSM state 前，必须执行 preflight；失败时不 submit 远端 job，不生成 Trial。

1. 解析 `mission.md` frontmatter 与 fallback queries。
2. 解析 `eval_sets[]` 引用，加载 `index.yaml` 与所有 shard。
3. 调用 B5 SchemaRegistry 校验 `eval-set-index` / `eval-set`。
4. 检查 `query_id` 在所有参与本次实验的 eval set 内全局唯一。
5. 检查 shard path 不越界、role 枚举合法、必填字段存在。
6. 检查 provider capability（如 patch/suggestion provider、DA executor、evaluator 依赖）。

典型错误码：

```text
EVAL_SET_SCHEMA_INVALID
EVAL_SET_DUPLICATE_QUERY_ID
EVAL_SET_SHARD_PATH_INVALID
EVAL_SET_ROLE_INVALID
MISSION_EVAL_SET_NOT_FOUND
```

用户也可以提前显式校验：

```bash
kweaver trace schema validate eval-sets/customer-support-v1/index.yaml --kind=eval-set-index
kweaver trace schema validate eval-sets/customer-support-v1/refund.yaml --kind=eval-set
```

#### 3.3.1 命令树

```
kweaver trace exp run    [folder] --mode=single-path       # 抢 lock + 读 mission.md + 启动/续跑单路径 FSM
kweaver trace exp resume [folder]                         # 语义等价于 run（强调"续跑"），同一代码路径
kweaver trace exp show   [folder]                         # 只读：展示任务配置、派生输入、输出物引用
kweaver trace exp status [folder]                         # 只读：一次性折叠 events/jobs/lock，输出当前进展与状态摘要
kweaver trace exp abort  [folder]                         # 写 abort 信号；driver 在下一个 checkpoint 优雅退出
kweaver trace exp doctor [folder]                         # 只读：校验 mission/eval-set/journal/lock/jobs/outputs 健康度

# post-MVP
kweaver trace exp watch  [folder]                         # 只读：tail events.jsonl + 拉 M3 拼当前进度视图
kweaver trace exp list   [path...]                        # 扫 path 下所有实验文件夹列状态
```

#### 3.3.2 代码模块布局（两层）

底层是 `exp-store/` 持久化抽象（**所有对实验文件夹的读写必须经过它**），上层是 `exp-engine/` 编排逻辑。依赖方向固定为 `exp-engine/* → exp-store/*`；`exp-store/` 不知道 FSM、Generator、Scorer、Triage 等运行时概念。

```
src/commands/trace/exp.ts                       # 参数解析 + dispatch（MVP-C 6 个子命令；watch/list post-MVP）

src/trace-core/exp-store/
  ├── paths.ts                # canonical 路径解析
  ├── mission-md.ts           # mission.md 解析（YAML frontmatter + body fallback queries 段）
  ├── preflight.ts            # mission / eval-set / provider capability 启动前校验
  ├── events-jsonl.ts         # append-only 写 + replay 读（FSM 真源）
  ├── candidate-lineage-yaml.ts # MVP-C：baseline → candidate-vN 单路径快照
  ├── trial-forest-yaml.ts    # post-MVP：多路径派生关系拓扑 yaml（快照式覆写）
  ├── jobs-jsonl.ts           # 远端 job_id 流水（append-only）
  ├── round-yaml.ts           # rounds/round-N.yaml 读写
  ├── lock.ts                 # 心跳锁协议（hostname+pid+last_heartbeat_ts，30s 超时）
  ├── abort-signal.ts         # .trace-state/abort.signal 文件读写
  └── read-model.ts           # 只读 fold：mission + events + jobs + lock + outputs → ExperimentSnapshot

src/trace-core/exp-engine/
  ├── coordinator.ts          # FSM driver 主循环（元控制层）
  ├── fsm.ts                  # 6+1 状态枚举 + 转移表 + checkpoint 钩子
  ├── candidate-lineage.ts    # MVP-C：单候选链推进
  ├── trial-forest-ops.ts     # post-MVP：Trial Forest 拓扑增删改逻辑
  ├── generator.ts            # thin wrapper：调 Agent Synthesizer provider
  ├── executor.ts             # K×M 调度（执行层；B2 async + jobs.jsonl）
  ├── scorer.ts               # thin wrapper：调 kweaver-eval + 三轴合成 + safety hard gate
  ├── triage.ts               # thin wrapper：调远端 Triage Agent + 跨轮记忆 binding
  ├── termination.ts          # 本地 Termination Decider
  ├── git-checkpoint.ts       # post-MVP：Round-内 local commit / Round-末 push 编排
  ├── status.ts               # `exp show/status/list` 的快照摘要与表格模型
  ├── doctor.ts               # `exp doctor` 健康检查
  └── watch.ts                # `exp watch` 实现
```

#### 3.3.3 FSM 与持久化

**6 + 1 状态**（vision §6.3）：

```
Initializing → Generating → Executing → Scoring → Triaging → Deciding
                  ↑__________________________________________|
                              ↓
                        Publishing → [end]
                  ↓
              Aborted（任意 active state 可转入；非终态，可 resume）
```

**真源分层**：`.trace-state/events.jsonl` 是 FSM 的 **append-only journal**——每次状态迁移、每个 checkpoint、每次 abort 检测都 append 一行。`.trace-state/jobs.jsonl` 是远端 job 的 **append-only job journal**——专门兜住 submit / poll 的 crash point。MVP-C 的 `candidate-lineage.yaml` / `rounds/round-N.yaml` 是从 journal 派生的快照，崩坏可重建；post-MVP 多路径再引入 `trial-forest.yaml`。`lock.json` / `abort.signal` 是运行期控制文件，不作为业务真源。

**events.jsonl v1 最小事件契约**：

每行都是独立 JSON object，公共字段固定：

```json
{
  "schema_version": "trace-exp-event/v1",
  "event_id": "<ulid>",
  "event_type": "state_transition",
  "experiment_id": "<id>",
  "round": 1,
  "ts": "2026-05-09T10:00:00Z",
  "actor": {"host": "macbook-xupeng", "pid": 23451},
  "payload": {}
}
```

| event_type | 何时写 | payload 最小字段 | replay 语义 |
|---|---|---|---|
| `experiment_initialized` | 首次 run 解析 mission.md 后 | `mission_hash` | 建立 experiment_id / mission 绑定 |
| `state_transition` | FSM 状态变化 | `from` / `to` / `reason` | fold 出当前 FSM state |
| `candidate_generated` | Generator 返回下一版 candidate 后 | `candidate_id` / `parent_candidate_id` / `variation` | 重建 candidate-lineage |
| `trial_execution_completed` | Executor 收到 trial × query 结果后 | `trial_id` / `query_id` / `trace_id?` / `status` / `result_ref?` / `error?` | 推进 execution cell 状态 |
| `score_recorded` | Scorer 得到分数后 | `trial_id` / `query_id` / `score_ref` / `hard_gate` | 重建 round 评分输入 |
| `round_completed` | Triaging→Deciding 完成后 | `round` / `verdict` / `round_ref` | 标记 round 快照可用 |
| `bundle_ready` | Publishing 输出 bundle 后 | `bundle_id` / `bundle_hash` / `manifest_hash` | 标记 `outputs/` 产物可交接给外部发布平台 |
| `abort_requested` | abort.signal 被检测到 | `requested_at` / `requested_by` | 触发后续 `state_transition(to=Aborted)` |
| `abort_cleared` | resume 删除 abort.signal 后 | `cleared_at` / `cleared_by` | 审计用，不单独改变 FSM state |
| `lock_stolen` | 过期 lock 被接管 | `previous_actor` / `heartbeat_age_sec` | 审计用，不改变业务状态 |
| `synthesizer_capability_missing` | Generating 前置检查失败 | `provider` / `missing_capability` / `required_version?` | fail-fast，等待 operator 安装 / 升级后 resume |

**写入规则**：

1. append 必须使用单行原子追加；写失败则当前 tick 失败并重试，不允许先改快照再补 journal。
2. `event_id` 必须全局唯一；replay 遇到重复 `event_id` 只取第一条，保证崩溃重试幂等。
3. replay 遇到坏行：默认 fail-fast，提示 `kweaver trace exp doctor` 定位问题或人工修复；`watch/list/status` 可跳过坏行并标红。
4. `candidate-lineage.yaml` / `round-N.yaml` 的 `source_event_id` 必须指向生成它们的最新事件，便于检测快照落后；post-MVP 的 `trial-forest.yaml` 也遵守同一规则。

**jobs.jsonl v1 最小事件契约**：

`jobs.jsonl` 只记录远端 job transport 生命周期，避免把 B2 submit 成功但本地未落盘的断点变成不可恢复状态。它不表达 FSM 状态，也不替代 `events.jsonl` 里的业务事件。每行同样是独立 JSON object，公共字段为 `{schema_version, job_event_id, experiment_id, round, ts, actor, payload}`。

| job_event_type | 何时写 | payload 最小字段 | resume 语义 |
|---|---|---|---|
| `job_submit_intent` | 调 B2 submit **之前** | `idempotency_key` / `target` / `stage` / `trial_id` / `query_id?` | 若无后续 `job_submitted`，resume 用同一 idempotency key 重新 submit |
| `job_submitted` | B2 submit 返回后 | `idempotency_key` / `job_id` / `target` / `stage` / `trial_id` / `query_id?` | 建立 pending job |
| `job_polled` | poll 到非终态进度时 | `job_id` / `status` / `progress?` | 供 watch/list 展示，不推进 FSM |
| `job_completed` | poll 到终态后 | `job_id` / `status` / `result_ref?` / `error?` | 从 pending job 移除；随后按 stage append 对应 events.jsonl 业务事件 |

写入顺序固定为：`jobs.jsonl job_submit_intent` → B2 `submit(idempotencyKey)` → `jobs.jsonl job_submitted`。如果任意一步崩溃，resume 先 fold `events.jsonl` 得到业务状态，再 fold `jobs.jsonl` 补齐 pending jobs；遇到 intent-only 记录时，用相同 idempotency key 重新 submit，远端必须返回原 job 或等价 job。`events.jsonl` 仍是 FSM 真源，`jobs.jsonl` 只补远端 job 可恢复性。

**Replay 协议**（resume 的实现）：

1. 抢 lock
2. 读 events.jsonl 到内存，按事件类型 fold 出当前状态（state / round_n / 已完成业务事件）
3. 读 jobs.jsonl 到内存，按 idempotency key fold 出 pending jobs；对 intent-only 记录按同一 idempotency key 重提 B2 submit
4. 对每个 `pending_jobs[i]`：用 B2 RemoteJobClient.poll(job_id) 看远端状态——已完成 → append `jobs.jsonl job_completed`，再按 stage append 对应 `events.jsonl` 业务事件（如 `trial_execution_completed` / `score_recorded` / `round_completed`）；in-flight → 加回 polling 队列继续等
5. 进 Coordinator 主循环

第 3 / 4 步是 vision §6.4.4 "driver 离线时已发出去的 trial 在远端继续跑"的具体落地。

**写时序**（Coordinator 主循环每个 tick）：

```
检查 abort.signal → 检查 pending_jobs poll → FSM transition → events/jobs journal append
                    ↓
              必要时同步快照（candidate-lineage.yaml / rounds/round-N.yaml）
                    ↓
              MVP-C 只写本地文件；post-MVP 可触发 git checkpoint
```

#### 3.3.4 M6 内部子件的本地 / 远端归属

| 子件 | 类型 | CLI 进程内做什么 | 远端做什么 |
|---|---|---|---|
| **Coordinator** | 元控制（本地） | FSM 驱动 / events.jsonl append / candidate-lineage 维护 / checkpoint 编排（MVP-C 写本地文件；post-MVP 叠加 git commit） | — |
| **Agent Patch Generator** | 研判（thin wrapper） | 拼 payload（mission / current_agent / diagnosis evidence / baseline test report） / 选择 provider / B2 submit / 收一个 candidate patch 落进 candidate-lineage.yaml | 智能：基于现有 Agent 配置与诊断证据生成下一版候选 |
| **Executor** | 执行（编排） | K×M 任务批量 B2 submit → jobs.jsonl 流水 → 周期 poll | DA 跑实际 trial × query |
| **Scorer** | 研判（thin wrapper） | 收 trial 完成事件 → B2 submit kweaver-eval → 收三轴分 → 按 trial 角色合成（vs-parent / 跨派生链 / cross-round）+ safety hard gate 横切淘汰 | 智能：deterministic + LLM-judge 双轨打分 |
| **Triage Agent (post-MVP)** | 元层反思（thin wrapper） | Round 末把 N 轮历史 (candidate / scores / 上轮 triage 结论 / 可附带 M4 报告) + 跨轮记忆引用 B2 submit → 收**调度决策**（剪枝 / 探索 vs 利用 / 收敛判定 / slot 跨树分配）→ 落 rounds/round-N.yaml | 智能：subject = 实验进程；不重做 trace 级诊断（trace 诊断走 M4） |
| **Termination Decider** | 元控制（本地） | 读最新 round-N.yaml + guardrail 历史 → 三选一判定（饱和 / 收敛 / 用户介入） | — |

**关键约束**：研判层 wrapper（Agent Synthesizer / Scorer / Triage）的"智能"100% 在 provider / 远端；CLI 内部是纯 binding——CLI 实现不依赖任何 LLM SDK、不用本地 GPU、可以 `npm install` 完了 offline 跑（除了 B2 远端调用本身）。M4 的 **Diagnose Provider Wrapper**（详见 §3.1）是同一套架构的另一个实例：四个 wrapper 共用 B2 RemoteJobClient + provider capability check + payload schema 校验流程，代码一份多用。

#### 3.3.4.1 Agent Synthesizer provider 抽象

Agent Synthesizer 是 M6 的生成 / 优化智能层，不是 CLI 内部算法。它有两类任务：

1. **cold-start synthesis**：从 `mission.md` 的 goal / guardrail / resources / reference docs / queries 生成初始 Agent 配置、Knowledge Network 草案、Skill 选择与资源绑定。
2. **iterative optimization**：基于 `current_agent`、上一轮 Trial、trace evidence、eval score、Triage hints，派生下一轮 Trial 的配置变更。

MVP 支持两个 provider：

| provider | 适用场景 | 调用方式 | 输出要求 |
|---|---|---|---|
| `claude-code` | 本地 / 半自动工程生成：生成 repo patch、agent config、BKN schema 草案、skill glue code | CLI 通过 B2 job 包装调用本机或远端 Claude Code runner；runner 必须在受控 workspace 内执行 | 输出 `change_set` + 文件 patch / config patch + evidence refs |
| `decision-agent` | 平台内 dogfood：用 KWeaver Decision Agent 生成 / 优化自身 Agent 配置 | CLI 通过 B2 submit 到远端 Decision Agent；DA 自己打 OTel，trace 回流 M1 | 输出结构化 Trial proposal，不直接改本地文件 |

**kweaver-core 前置依赖**：

- Agent Synthesizer 要生成 Agent、Knowledge Network、Skill 绑定时，必须能调用已安装的 `kweaver-core` skill/CLI 能力；它是写入 / 校验 KWeaver 平台资产的执行面，不由 trace CLI 重写。
- CLI 启动 `exp run` 时做 provider capability check：`kweaver-core` 不存在、版本不满足、或缺少所需 skill 时，`Generating` 阶段 fail-fast，并写 `synthesizer_capability_missing` 事件。
- `mission.md` 可声明 `synthesizer.provider: claude-code | decision-agent`；未声明时按环境探测选择，优先 `decision-agent`（平台内闭环），其次 `claude-code`（本地工程生成）。
- provider 只能产出 proposal / patch；真正进入 candidate-lineage 前，CLI 必须用 B5 SchemaRegistry 校验 candidate schema，并检查 resource binding 是否引用存在的 KWeaver 资源。post-MVP 多路径 Trial Forest 也复用这条校验。

**标准输出结构**：

```yaml
trials:
  - trial_id: trial_...
    parent_trial_id: trial_parent_or_null
    provider: claude-code
    change_set:
      agent_config_patch: ...
      prompt_patch: ...
      knowledge_network_patch: ...
      skill_binding_patch: ...
      resource_binding_patch: ...
    rationale: ...
    evidence_refs:
      - trace_id: ...
      - round_ref: rounds/round-1.yaml
    predicted_fixes:
      - query_id: ...
        reason: ...
    predicted_risks:
      - query_id: ...
        reason: ...
    constraints_checked:
      - guardrail_id: ...
        result: pass
```

`predicted_fixes` / `predicted_risks` 是 manifest 对账的前置字段；缺失时该 Trial 可用于探索，但不能进入最终 `outputs/bundle.yaml`。

#### 3.3.4.2 Triage Agent 交互协议

研判层 3 件中，Triage Agent 的“智能”虽在远端，但 CLI 需负责拼装请求 Payload 并解析返回结果，将其持久化在 `rounds/round-N.yaml` 中，作为下一轮 `Generating` 的输入（Triage hints）。

**标准请求 Payload 结构**：
CLI 在 `Scoring` 阶段结束后，收集本轮数据，通过 B2 异步提交给远端 Triage Agent：

```yaml
experiment_id: "exp-123"
current_round: 2
cross_round_memory_ref: "mem_token_xxxx" # 跨轮记忆引用（来自上一轮的 new_memory_token）
trials:
  - trial_id: "trial-v1"
    parent_trial_id: "trial-v0"
    variation: { prompt_template: "v1" }
    scores:
      outcome: 0.8
      trajectory: 0.9
      guardrail: 1.0
    evidence_trace_ids: ["trace-1", "trace-2"] # 支持打分的证据 trace
```

**标准返回与持久化结构（`rounds/round-N.yaml`）**：
CLI 轮询到任务成功后，解析结果并覆写/生成对应的轮次快照文件：

```yaml
round: 2
status: completed
triage_conclusion:
  diagnoses:
    - "发现 Tool X 的调用参数在 Trial B 中经常出错，导致 trajectory 分数下降"
  hints: # 核心：给下一轮的 Generator 提供方向
    - "尝试收紧 Tool X 的参数范围"
    - "可以尝试开新树探索分支 Y"
  verdict: continue # 策略：continue 还是 publish
  new_memory_token: "mem_token_yyyy" # 传递给下一轮的记忆 Token
```

#### 3.3.5 lock / abort / resume / watch 协议

**lock.json 协议**（vision §6.4.5(a) cooperative lock，非分布式锁）：

```json
{
  "hostname": "macbook-xupeng",
  "pid": 23451,
  "started_at": "2026-05-09T10:00:00Z",
  "last_heartbeat_ts": "2026-05-09T10:00:30Z"
}
```

- `exp run` 启动：检查 lock.json
  - 不存在 → 创建，进 Coordinator
  - 存在且 `now - last_heartbeat_ts < 30s` → fail：`another driver running on <hostname>:<pid>`
  - 存在但 `now - last_heartbeat_ts >= 30s` → 视为弃锁，覆写 + events.jsonl append `lock_stolen` + 警告
- Coordinator 主循环每 10s 更新 `last_heartbeat_ts`
- 干净退出 → 删 lock.json；崩溃退出 → lock.json 残留，30s 后被下一个 driver 强夺

**abort 协议**：

- `exp abort` 不抢 lock，**直接写文件** `.trace-state/abort.signal`（内容：`{requested_at, requested_by}`）
- Coordinator 在 FSM 每个 transition 之间 **poll abort.signal**：检测到 → events.jsonl append `abort_requested` → append `state_transition(to=Aborted, reason=abort_requested)` → 释放 lock + 退出
- vision §6.3 的 Aborted 不是终态——下次 `exp run` / `exp resume` 删除 abort.signal 后 append `abort_cleared`，再从最近稳定业务 checkpoint 继续

**resume 与 run 的关系**：

- **同一代码路径**——`resume` 只是给用户的语义糖。底层都是抢 lock + events.jsonl replay + 继续 FSM。
- 区别仅在用户提示语：run 在空 events.jsonl 上启动会输出"new experiment from mission.md"；resume 会输出"resuming from round-N, state=Triaging"。

**show / status / doctor 边界**：

这三个命令补齐实验文件夹的只读观测面，全部不抢 lock、不写文件，默认不拉生产 trace 全文。

| 命令 | 回答的问题 | 读取来源 | 输出重点 |
|---|---|---|---|
| `exp show [folder]` | 这个任务配置成了什么 | `mission.md` / `eval-sets/` / `diagnosis/` / `outputs/` | experiment_id、goal、guardrails、eval set 摘要、candidate lineage、bundle/manifest 引用 |
| `exp status [folder]` | 当前跑到哪一步 | `.trace-state/events.jsonl` / `jobs.jsonl` / `lock.json` / `abort.signal` / `rounds/` / 可选 B1 trace 聚合 | FSM state、round、trial 数、K×M cell 进度、pending jobs、最近错误、driver 心跳、abort 状态 |
| `exp doctor [folder]` | 这个任务是否可继续运行 | `show + status` 的全部来源 + B5 schema 校验 + journal replay 校验 | preflight 结果、坏 journal 行、重复 event_id、快照滞后、悬挂 job、过期 lock、缺失 outputs |

`show` 偏配置，`status` 偏进展，`doctor` 偏可恢复性与可操作诊断。三者共用 `exp-store/read-model.ts` fold 出 `ExperimentSnapshot`，避免每个命令各自解析 journal。`doctor` 只报告修复建议，MVP 不自动改文件；未来可加 `exp doctor --repair`，但必须逐项确认或要求显式 flag。

**watch 边界**：

`watch` 不进 MVP-C。MVP-C 用 `status` 的一次性快照满足“看进展”需求；只有当长任务盯盘成为高频操作时，再实现实时 UI。

- **不抢 lock、不写任何文件**——只读 events.jsonl + jobs.jsonl（fs.watch 或 100ms polling fallback for cross-platform）+ B1 拉对应 round 的 trace 拼当前进度
- 多人同时 watch 互不干扰
- watch 的 UI 用 ink，三栏：FSM 状态时间轴 / 当前 round 的 K×M 矩阵进度 / 最近 5 行 events；底层同样复用 `ExperimentSnapshot`，只是在 fs watch / polling 下持续刷新

**list 边界**：

`list` 不进 MVP-C。MVP-C 的对象是单个实验文件夹；跨目录扫描属于团队协作和 dashboard 的前置能力。

- 不抢 lock。扫 `path...` 下含 `.trace-state/` 的子目录 → 对每个读取轻量 `ExperimentSnapshot` → 输出表格（experiment / state / round / pending-jobs / driver-host / heartbeat-age / last-error / bundle-ready）

#### 3.3.6 commit / push 节奏

MVP-C 不自动 commit / push；`ExpStore` 只负责把 `.trace-state/`、`rounds/` 与 `outputs/` 写成可 review 的本地文件。用户或外部平台自行决定何时 `git add/commit/push`。

post-MVP 如需自动 checkpoint，再由 `git-checkpoint.ts` 编排，三档触发：

| 触发 | 操作 | 提交信息约定 |
|---|---|---|
| FSM 进入新稳定状态 | local commit（不 push） | `exp <name> round-<N>: <prev>→<state>` |
| Round 末（Triaging→Deciding 完成） | local commit；push 仍建议交给外部流程 | `exp <name> round-<N> complete; trial-count=<K>; verdict=<continue\|publish>` |
| Publishing 输出 bundle 后 | local commit | `exp <name> bundle-<id> ready in outputs/` |
| Aborted | local commit | `exp <name> aborted at round-<N>:<state>` |

自动 push 不是 MVP-C 前提。若 post-MVP 加回 push，push 失败（网络断 / 权限拒）也不应阻断 FSM；commit 已落地即可，下次 checkpoint 时累积处理。

#### 3.3.7 关键 trade-offs

1. **`run` / `resume` 同一代码路径**——避免双套 FSM 启动逻辑漂移；用户语义靠提示语区分。
2. **events.jsonl 是 FSM 真源，jobs.jsonl 是远端 job journal，candidate-lineage.yaml / round-N.yaml 是派生快照**——崩坏可从 journal 重建。事件文件持续追加，单实验持续若干月可能涨到几十 MB；可接受，未来涨到 GB 级再加 segment 切分。
3. **B2 RemoteJobClient 是研判层 3 件的唯一出口**——保证 driver 离线 / 笔记本关停可恢复。MVP 期 `--sync` 降级开关只允许 dev / debug 用。
4. **lock 是 cooperative，不是分布式锁**——vision §6.4.5(a) 已认代价；冲突发生时该轮重做，不在 trace-core 加 ZooKeeper / etcd。
5. **watch / list 不抢 lock**——post-MVP 实现时也只能走读路径无锁，多人监控不干扰；可装到 CI dashboard 持续显示。
6. **MVP-C 不自动 commit / push**——本地文件是唯一必须交接面；git checkpoint 只在 post-MVP 手工流程成为瓶颈时加入。

### 3.4 Post-MVP：M7 Replay — `kweaver trace replay`

M7 不进 MVP-A/B/C。MVP-A 的诊断先用既有 trace 做规则 / 状态机 / schema signal 分析；replay 需要 DA 支持历史上下文重跑、新 trace 标注和 diff 契约，工程成本更高。等单路径诊断与测试流程跑顺后，再把 replay 作为深度诊断增强加入。post-MVP 首版仅实现 `compare` 单档；strict / explore 继续延后。

**命令树**（vision §7.M7，无 sub-resource，单命令双定位）：

```
kweaver trace replay <trace_id>                     [--trial <spec>] [--mode strict|compare|explore]
kweaver trace replay --experiment-id <id> --query <q>   --trial <spec>  [--mode ...]
```

**代码模块布局**：

```
src/commands/trace/replay.ts                # 参数解析 + 两种定位的 dispatch
src/trace-core/replay/
  ├── locator.ts             # trace_id 直定位 / experiment+query 间接定位 二选一
  ├── request-builder.ts     # 构造 replay payload（必带 replay_of=<原 trace_id> attribute）
  ├── diff.ts                # 拉新旧两条 trace 走 B1 → 逐 span diff
  └── output.ts              # diff 写本地 yaml 或 stdout
```

**依赖**：

- **B1 ObservabilityClient** — 两次：①前置拉原 trace 拼 replay 请求上下文；②replay 完成后拉新 trace 做 diff
- **B2 RemoteJobClient** — async submit replay 任务给远端 DA
- **不依赖 B3/B4/B5**

**关键设计点**：

1. **`replay_of=<原 trace_id>` 是契约性 attribute**——CLI 端必须强制带；DA 收到后必须原样标进新 trace 的 root span。schema/v1/trace.yaml 的 attributes 段加一条可选项约束。
2. **DA 自己打 OTel 喂回 M1，CLI 不直接 push OTLP**（与 vision §7.M7 一致）。
3. **mode 三档先单档实现**——post-MVP 首版仅实现 `compare`（最常用），`strict` / `explore` 留 enum 但报"not yet implemented"；保留 flag 是为了未来扩展不破坏命令字面值。
4. **diff 走 B1 提供的 `/traces/diff?a=&b=`**（vision §7.M3 扩张接口已包含）——CLI 端不本地实现 span diff 算法。如果 M3 一期没 ship `/traces/diff`，CLI 暂时拉两条 trace 在本地做 naive diff，留 TODO。

**核心流程与逻辑（以 `kweaver trace replay` 为例）**：

1. **目标定位**：
   - 调用 `locator.ts`。如果用户提供的是 `trace_id`，则直接使用。
   - 如果提供的是 `--experiment-id` 和 `--query`，则调用 B1 (ObservabilityClient) 查询对应的 Trace ID。
2. **上下文拉取与 Payload 构造**：
   - 调用 B1 获取原始 Trace 的详细内容（特别是输入的 User Query 和初始上下文）。
   - 调用 `request-builder.ts` 构造 Replay 请求体，**强制注入** `replay_of=<原 trace_id>` 属性。
3. **提交重放任务**：
   - 调用 B2 (RemoteJobClient) 将 Replay 请求异步提交给远端 Decision Agent (DA)。
   - 远端 DA 使用新的 Trial 配置重新执行该 Query，并将新的 Trace 上报到 M1。
4. **结果获取与 Diff**：
   - CLI 轮询 B2 直到任务完成，获取新生成的 `new_trace_id`。
   - 调用 B1 的 `/traces/diff?a=<old>&b=<new>` 接口（或在本地进行基础 Diff）。
   - `output.ts` 将步骤数、工具选择、检索命中、最终输出等差异格式化输出为 YAML 或直接在终端打印。

### 3.5 Deferred：M8 / M9 不进 MVP

为降低实现和维护成本，MVP 不实现 `kweaver trace bundle ...` 与 `kweaver trace verify ...` 两个命令族。

**MVP 取舍**：

1. **不做 publish-registry**：没有中央 bundle 仓库、没有 registry URL 配置、没有 git 并发写入协议。
2. **不做 artifact resolver**：CLI 不负责从 URL / S3 / git path 拉 bundle；实验产物就是本地 `outputs/` 目录。
3. **不做 post-deploy verify scan**：没有 cadence、没有 `deployed_at.yaml`、没有 CI 定时扫描器。
4. **不做自动 curation-feed 聚合**：新失败模式回流先由人工或外部流程处理；M4 只支持本地规则和显式 `--feed` 文件。

**保留的契约**：

- M6 仍必须在 `outputs/` 生成 `bundle.yaml`、`manifest.yaml`、`provenance.yaml`。
- `manifest.yaml` 仍是可验证发布意图的静态凭据，必须通过 B5 SchemaRegistry 校验。
- `provenance.yaml` 仍从 `.trace-state/events.jsonl` 抽取 trace_id / round / triage 引用，保证产物来源可审计。

**post-MVP 触发条件**：

只有当团队确实需要跨实验发现、跨团队共享、自动 post-deploy 对账，且“手动保存 outputs 地址 + 外部发布平台记录”已经成为瓶颈时，再恢复 M8/M9。届时优先考虑一个极薄的 artifact index，而不是直接引入可写 publish-registry。

### 3.6 MX1 Schema — `kweaver trace schema`

**命令树**：

```
kweaver trace schema validate <file> [--kind=<kind>]                          # MVP-A：开发者本地单文件 ajv 校验
kweaver trace schema audit    [--time-window=1h] [--sample=1000] [--out=]     # post-MVP：跨 span 不变量 / 漂移率 / 准入率 抽样报告
```

**代码模块布局**：

```
src/commands/trace/schema.ts                        # dispatch
src/trace-core/schema/                              # B5 SchemaRegistry 主体
  ├── index.ts               # 公共 API：validate(kind, doc) / loadSchema(kind, version) / aliasResolve(field)
  ├── validator.ts           # ajv 实例 + 别名兼容表 + 兼容窗口判定
  ├── alias-table.ts         # 字段别名兼容（如 session_id ↔ agent.session.id）
  ├── audit/                 # post-MVP
  │   ├── invariant.ts       # 跨 span 不变量评估（要看完整 trace 上下文）
  │   ├── drift.ts           # 漂移率指标（兼容窗口监控）
  │   └── admit-rate.ts      # L1/L2 准入率指标
  ├── audit-orchestrator.ts  # audit 子命令主流程（抽样 → 三件并行 → 合 report）
  ├── report.ts              # audit 报告 yaml 序列化
  └── v1/                    # ← schema 静态文件 mirror（trace.yaml / experiment.yaml / bundle.yaml / manifest.yaml / eval-set.yaml / eval-set-index.yaml）
```

**依赖**：

- **schema/v1/*.yaml mirror**（自带，build 时打进 dist）
- **B1 ObservabilityClient** — post-MVP audit 时按 `time-window + sample` 抽样拉 trace
- 不依赖 B2/B4

**外部触发：CI workflow（post-MVP，可在 trace-ai 仓库自带）**：

trace-ai 仓库 `.github/workflows/schema-audit.yml`：

```yaml
on:
  schedule: { cron: '0 */1 * * *' }   # 每小时
jobs:
  audit:
    steps:
      - run: npm install -g @kweaver-ai/kweaver-sdk
      - run: kweaver trace schema audit --time-window=1h --sample=1000 --out=audit-report.yaml
        env: ...
      - run: |
          # 报告 commit 到 trace-ai/schema-audit-reports/<ts>.yaml
          # 偏差超阈值时同时开 issue
          ...
```

**关键设计点**：

1. **静态 schema 文件**住 `src/trace-core/schema/v1/`，build 时由 tsconfig `include` 拷进 dist；`schema-mirrors.lock` 记录同步源 SHA。
2. **validate 与 audit 共享 ajv 实例**——一次 schema load，两个命令复用；冷启动加载 5–10ms 量级，可忽略。
3. **validate 的 kind 判定**：优先使用 `--kind`；未传时按文件名约定推断（`index.yaml` 且父目录位于 `eval-sets/*/` → `eval-set-index`；`bundle.yaml` / `manifest.yaml` / `verification.yaml` / `eval-set*.yaml` / `trace*.json` / `experiment*.yaml`）；仍无法推断则报 `SCHEMA_KIND_REQUIRED`，不猜。
4. **inline 校验仍然在 M1 otelcol 端做**（vision §7.M1 的 schema-check hook 不动）；MVP-A CLI 只做单文件本地校验给开发者用；post-MVP audit 再拿 inline 干不了的三件（跨 span / 漂移 / 准入）。两条路径互不干涉。
5. **audit 抽样不全量**——CI runner 离 M2 远，全量校验数据量过大；vision §7.MX1 本来就钉死抽样。`--sample=1000` 默认值在初期可调。

## §4 测试与发布策略

### 4.1 测试约定（沿用 cli_conventions §7）

每个新命令必须包含：

1. **解析器单测**（`test/trace-<cmd>-cmd.test.ts`）：覆盖每个 flag 的 happy path 与至少一个错误路径
2. **API 客户端单测**（`test/api/trace/<resource>.test.ts`）：mock `fetch`，断言 URL、method、headers、body
3. **e2e smoke**（`test/e2e/trace-<resource>.test.ts`）：跑通完整链路，环境变量缺失时跳过

**新增**（trace 子命名空间专属）：

4. **trace-core 单元测**（`test/trace-core/<module>.test.ts`）：FSM transition / lock / events.jsonl replay / exp-store 等内核组件独立测试
5. **schema mirror lint**（CI step）：`schema-mirrors.lock` 中每条 source 的 SHA + target hash 一致性
6. **状态恢复 golden tests**：MVP-C 固定一组 `events.jsonl + jobs.jsonl` 样本，断言 replay 后的 state / pending_jobs / candidate-lineage 完全一致
7. **crash-point tests**：MVP-C 覆盖 `job_submit_intent 已写但 submit 未完成`、`submit 成功但 jobs.jsonl job_submitted 未写`、`jobs.jsonl job_completed 已写但业务事件未写` 三类断点；git checkpoint 断点随 post-MVP B4 再补
8. **git checkpoint tests**：post-MVP；用临时 git repo 覆盖 M6 local commit、push 失败不阻断、后续重试 push 等路径
9. **安全输出 tests**：默认输出不得包含 prompt / tool result 全文；`--json` 与 pretty 路径同样遵守 redaction；`--unsafe-full` 需要显式开关

### 4.2 发布策略

- **整体跟 kweaver-sdk 现有节奏**：不独立发版；trace 子命名空间作为 kweaver-sdk 新 minor 上线
- **MVP-A 顺序**（建议，非强约束）：B5 + `schema validate` → B1 ObservabilityClient（`getTrace` / `searchTracesStream` 最小集）→ B2 RemoteJobClient（`--sync` 降级先用上）→ M4 静态信号层 + Diagnose Provider Wrapper → M4 `diagnose <trace_id>` / `scan`
  - 理由：先把 trace 读出来 + B2 通道立起来，再实现双轨诊断；用户最早能感知的价值就是 diagnosis report。
- **MVP-B 顺序**：M5 `eval-set build --diagnosis=...` → M5 `eval-set test`（复用 B2 + M6 single-path executor 雏形）
  - 理由：把诊断结果固化为可复现测试，并形成 baseline test report。test runner 是 MVP-C M6 executor 的薄包装，提前把这条调用路径打通。
- **MVP-C 顺序**：B3 ExpStore → M6 single-path FSM 完整化 → `exp run/resume/show/status/abort/doctor` → outputs schema 校验
  - 理由：B2 在 MVP-A 已就绪；MVP-C 只需补 ExpStore + 跨 round 编排。
- **Post-MVP 顺序**：Trial Forest / multi-trial 并行 / M7 `replay` / M5 `relabel` / MX1 `audit` / `exp watch/list` / B4 GitCheckpoint / M8/M9
  - 理由：跨团队协作、批量盯盘、多路径探索和自动飞轮都不是前三段 MVP 的必要条件。
- **每个 minor 都跑 schema-mirrors.lock CI lint**——drift 即红
- **MVP 期 B2 RemoteJobClient 内置 `--sync` 降级开关**给 dev 用；正式实验路径强制 async（vision §6.4.4 钉死方向）

### 4.3 文档

- `docs/cli_conventions.md`：现有文档 §1–§7 不动；追加 §8 trace 子命名空间约定（指向本文档）
- 每个 trace 子命令的 `--help` 文本：包含命令字面值 + 一条 example + 链接到本文档对应章节
- README / `kweaver --help` 顶层命令树更新（加 `kweaver trace ...` 一行）

## 附录 A：trace-core/ 目录全景

为方便实现期参考，集中列出 trace-core/ 下的所有目录与文件归属：

```
src/trace-core/
  ├── remote-job.ts                       # B2 共享：async submit + poll
  ├── git-checkpoint.ts                   # B4 post-MVP：git CLI checkpoint 包装

  ├── exp-store/                          # B3 共享：实验文件夹持久化抽象（M6 独占）
  │   ├── paths.ts
  │   ├── mission-md.ts
  │   ├── events-jsonl.ts
  │   ├── candidate-lineage-yaml.ts
  │   ├── trial-forest-yaml.ts              # post-MVP
  │   ├── jobs-jsonl.ts
  │   ├── round-yaml.ts
  │   ├── lock.ts
  │   ├── abort-signal.ts
  │   └── read-model.ts

  ├── schema/                             # B5 共享：SchemaRegistry
  │   ├── index.ts
  │   ├── validator.ts
  │   ├── alias-table.ts
  │   ├── audit/
  │   │   ├── invariant.ts
  │   │   ├── drift.ts
  │   │   └── admit-rate.ts
  │   ├── audit-orchestrator.ts
  │   ├── report.ts
  │   └── v1/                             # schema 静态文件 mirror

  ├── exp-engine/                         # M6 专属：实验运行引擎
  │   ├── coordinator.ts
  │   ├── fsm.ts
  │   ├── candidate-lineage.ts
  │   ├── trial-forest-ops.ts               # post-MVP
  │   ├── generator.ts
  │   ├── executor.ts
  │   ├── scorer.ts
  │   ├── triage.ts
  │   ├── termination.ts
  │   ├── git-checkpoint.ts
  │   ├── status.ts
  │   ├── doctor.ts
  │   └── watch.ts

  ├── diagnose/                           # M4 专属
  │   ├── planner.ts
  │   ├── policy.ts
  │   ├── signal-probe.ts
  │   ├── invariant.ts
  │   ├── latent-failure.ts
  │   ├── rules.ts
  │   ├── suggestion.ts
  │   ├── report.ts
  │   ├── watermark.ts                     # post-MVP
  │   └── feed-pickup.ts                   # post-MVP

  ├── eval-set/                           # M5 专属
  │   ├── query-picker.ts
  │   ├── redactor.ts
  │   ├── test-runner.ts
  │   ├── relabel.ts
  │   └── output.ts

  ├── replay/                             # M7 专属
  │   ├── locator.ts
  │   ├── request-builder.ts
  │   ├── diff.ts
  │   └── output.ts

src/commands/trace/                       # 各 M 模块的命令入口（薄层；只做参数解析 + dispatch 到 trace-core）
  ├── exp.ts
  ├── diagnose.ts
  ├── eval-set.ts
  ├── replay.ts
  └── schema.ts

src/api/trace/                            # B1 共享：M3 HTTP 客户端
  └── observability.ts

src/ui/trace/                             # B6 共享：trace 子命名空间专属输出格式器
  └── ...
```

## 附录 B：依赖矩阵（CLI 视角）

各 M 模块对共享层的依赖（行：M 模块；列：B 共享组件）：

| | B1 Obs | B2 RemoteJob | B3 ExpStore | B4 GitCheckpoint | B5 SchemaReg | B6 Output | BKN api（既有） | kweaver-core skill/CLI |
|---|---|---|---|---|---|---|---|---|
| **M4 diagnose** | MVP-A：读 trace | MVP-A：diagnose-provider async | | | ✓ 校 rules / payload / report | ✓ | post-MVP/optional：反查不变量 | |
| **M5 eval-set** | MVP-B：圈 query | MVP-B：test；post-MVP：relabel | | | ✓ 校 eval-set / report | ✓ | | |
| **M6 exp** | post-MVP：watch 拉 trace | MVP-C：single-path executor / scorer / patch generator async | ✓ 独占写 | post-MVP：commit/push | ✓ 校 candidate / outputs | ✓ | | ✓ synthesis 写入 / 校验平台资产 |
| **M7 replay** | post-MVP：拉新旧 trace + diff | post-MVP：DA replay async | | | ✓ replay payload schema | ✓ | | |
| **MX1 schema** | post-MVP：audit 抽样 | | | | self | ✓ | | |
