---
title: trace-ai CLI 详细设计
status: draft
date: 2026-05-09
依赖: vision/trace-ai-continuous-learning-design.md（以下简称 vision）
覆盖: MVP CLI（M4 / M5 / M6 / M7 / MX1）+ post-MVP 资产/验证能力（M8 / M9）的取舍
---

## §0 范围与背景

vision §6.4 / §7 曾把 trace-ai above-L0 的 7 个模块（M4 Curation / M5 Eval-Set Builder / M6 Experiment Engine / M7 Replay / M8 Publish Registry / M9 Post-deploy Verify / MX1 Schema）钉成 CLI 形态。为降低 MVP 的实现和维护成本，本文档把 MVP 收敛为 5 个 CLI 模块：M4 / M5 / M6 / M7 / MX1。M8 / M9 不进 MVP，仅作为 post-MVP 方向保留。

- **代码归属**：哪个文件 / 哪个仓库
- **命令字面值**：用户键入什么
- **模块边界**：内部子模块怎么切、依赖方向
- **共享层契约**：跨模块复用的内核组件
- **触发机制**：周期性任务怎么不靠 K8s CronJob 也能跑
- **用户操作路径**：operator 从写任务配置、启动实验、查看进展到取走 `outputs/` 产物，具体敲哪些命令

本文档**不**覆盖：每个 HTTP 接口的字段细节（属下游各 M-spec）、远端智能体层（DA / kweaver-eval / Triage Agent）的内部实现（属各自子 spec）、L0 数据面（M1 otelcol / M2 OpenSearch / M3 agent-observability）的服务实现（属 trace-ai 仓库后端）。

### 0.1 用户故事与操作路径

本文档后面按 MVP 模块（M4 / M5 / M6 / M7 / MX1）拆解实现，但用户不会按模块名工作。CLI 面向两个连续的 user story：

1. **Story A：0→1 跑通单个实验**。我已经有一个 Agent 优化任务和一组 eval cases，要把实验跑起来，并能随时看清任务配置、进展、状态与恢复点。
2. **Story B：1→N 持续迭代飞轮**。在单个实验可运行之后，我要把生产 trace、失败样本筛选、eval set 扩充、本地实验产物和外部上线反馈接起来，让下一轮实验持续吸收线上反馈。

Story A 是 MVP 的可用性底座；Story B 建立在 Story A 之上。换句话说，M6 `exp` 是 0→1 的核心，M4/M5 把生产 trace 和 eval set 输入接进来；M8/M9 这类资产 registry / 上线后自动验证先不做，避免 MVP 过早承担跨团队索引、发布状态、周期调度和 git 并发写入。

#### 0.1.1 Story A：0→1 跑通单个实验

Story A 的目标不是完整飞轮，而是先让一个实验文件夹具备“可配置、可运行、可观察、可恢复”的闭环。前置条件是：M3 trace 查询、schema mirror、B2 async job、至少一个 Agent Synthesizer provider 已可用；eval set 可以由用户预置，不强依赖 M4/M5 先生成。

```bash
# 1. 准备实验目录：用户主要编辑 mission.md，并引用已有 eval-sets/
cd my-experiment
$EDITOR mission.md

# 2. 启动或续跑持续优化实验
kweaver trace exp run .

# 3. 一次性查看配置、进展和健康度
kweaver trace exp show .
kweaver trace exp status .
kweaver trace exp doctor .

# 4. 需要长时间盯进展时再进入实时 UI
kweaver trace exp watch .

# 5. driver 需要停下时写 abort.signal，让 Coordinator 在 checkpoint 优雅退出
kweaver trace exp abort .
```

Story A 只要求 M6 + B1/B2/B3/B5/B6 的最小组合。`curate scan`、`eval-set build/relabel` 属于 Story B 的输入增强；发布资产中心和 post-deploy verify 不进入 MVP。

#### 0.1.2 Story B：1→N 持续迭代飞轮

Story B 的目标是把 Story A 产出的实验能力接入生产反馈。一次完整飞轮按下列顺序发生：

```bash
# 1. 准备实验目录：用户主要编辑 mission.md；eval-sets 可预置，也可后续生成
cd my-experiment
$EDITOR mission.md

# 2. 可选：从生产 trace 筛出失败/疑似失败样本
kweaver trace curate scan --policy=curation-policy.yaml --out=curation-output/

# 3. 可选：把筛出的样本构造成 eval set，或对既有 eval set 做 hindsight relabel
kweaver trace eval-set build --curation=curation-output/latest.yaml --out=eval-sets/customer-support-v2
kweaver trace eval-set relabel eval-sets/customer-support-v2

# 4. 启动或续跑持续优化实验
kweaver trace exp run .

# 5. 观察任务配置、当前进展和健康度
kweaver trace exp show .
kweaver trace exp status .
kweaver trace exp watch .
kweaver trace exp doctor .

# 6. 实验结束后，M6 已在 outputs/ 下生成 bundle.yaml / manifest.yaml / provenance.yaml
#    用户或发布平台自行保存这些产物地址；MVP CLI 不维护 registry，也不做 post-deploy verify
```

用户不需要直接编辑 `.trace-state/`。`outputs/` 是实验终态产物目录，用户可以复制、上传或交给外部发布平台；MVP 不提供跨团队 bundle 索引、artifact 拉取命令、周期 verify scan 或 registry 写入。

#### 0.1.3 日常操作场景

| 场景 | 用户目标 | 首选命令 | 读写行为 |
|---|---|---|---|
| 新建任务 | 描述要优化什么、用哪些 eval set / guardrail / provider | 手写 `mission.md`，然后 `kweaver trace exp show .` | show 只读 |
| 启动 / 续跑任务 | 让 M6 Coordinator 接管实验 FSM | `kweaver trace exp run .` 或 `kweaver trace exp resume .` | 写 `.trace-state/`、round 快照、outputs |
| 看当前跑到哪 | 一次性看 FSM state、round、pending jobs、最近错误 | `kweaver trace exp status .` | 只读 |
| 长时间盯进展 | 实时看 K×M 执行矩阵和最近事件 | `kweaver trace exp watch .` | 只读 |
| 排查为什么跑不动 | 校验 mission/eval-set/journal/lock/jobs/outputs | `kweaver trace exp doctor .` | MVP 只读，不自动修复 |
| 停止当前 driver | 让 driver 在下一个 checkpoint 优雅退出 | `kweaver trace exp abort .` | 写 `abort.signal` |
| 扫多个实验 | 看一批实验目录的概要状态 | `kweaver trace exp list <path...>` | 只读 |
| 从生产失败中补样本 | 把新失败模式变成 curation/eval 输入 | `kweaver trace curate scan ...` | 读 M3 trace，写 curation 输出 |
| 保存候选产物 | 取走实验输出供外部发布平台使用 | 直接读取 `outputs/` | CLI 不写外部系统 |
| 上线后复盘 | 人工或外部平台根据产物地址拉 trace 复盘 | `curate scan` / `replay` / 外部流程 | MVP 不内置 post-deploy verify |

#### 0.1.4 用户心智边界

- **任务配置由人写**：`mission.md` 是入口；`eval-sets/`、`curation-policy.yaml`、`curation-rules/` 是可选输入。
- **运行态由 CLI 写**：`.trace-state/` 下的 journal、lock、job 流水、round 快照都不要求用户手写。
- **进展靠只读命令看**：`show/status/watch/list/doctor` 是 operator 的日常观测面，不和 `run/resume` 抢 lock。
- **发布和验证不进 MVP 边界**：M6 只把 bundle / manifest / provenance 写到 `outputs/`；保存地址、上线动作、上线后对账由用户或外部发布平台负责。中央 publish-registry、全局 bundle 索引、周期 `verify scan` 都推迟到 post-MVP。
- **trace 查询统一走 M3**：用户命令不直接暴露 OpenSearch DSL；生产 trace 的全文输出默认脱敏。

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
        │   │   └── trace/              # ← 本设计新增的 trace-ai 子命名空间
        │   │         ├── exp.ts
        │   │         ├── curate.ts
        │   │         ├── eval-set.ts
        │   │         ├── replay.ts
        │   │         └── schema.ts
        │   ├── api/
        │   │   ├── (既有 api clients)
        │   │   └── trace/              # ← trace-ai 专属 HTTP 客户端
        │   │         └── observability.ts
        │   └── trace-core/             # ← trace-ai 专属内核（FSM / git / async-poll / schema / exp-store / exp-engine）
        │         ├── exp-store/
        │         ├── exp-engine/
        │         ├── curate/
        │         ├── eval-set/
        │         ├── replay/
        │         ├── schema/
        │         ├── remote-job.ts
        │         └── git-state.ts
        └── schema-mirrors.lock         # ← schema 静态契约同步源 SHA 锁
```

**单向依赖**：`commands/trace/*` → `trace-core/*` → `api/trace/*` → `auth/` + `config/`（既有基础设施）

### 1.2 命令树总览

MVP 模块按 cli_conventions §2 "顶层 + 子资源 + 动作" 三段式落地：

```
kweaver trace exp       run | resume | show | status | watch | abort | list | doctor # M6
kweaver trace curate    scan | rules list | rules validate                       # M4
kweaver trace eval-set  build | relabel                                          # M5
kweaver trace replay    <trace_id|--experiment-id+--query>                       # M7（动作即资源）
kweaver trace schema    validate | audit                                         # MX1
```

`exp show/status/doctor` 是 M6 的只读观测面：补足“当前任务配置是什么、跑到哪一步、健康不健康”的一次性查询能力；`watch` 只负责实时 UI，`list` 只负责多实验概要扫描。

M8 `bundle` 与 M9 `verify` 不进 MVP 命令树。MVP 中 bundle / manifest / provenance 是 M6 的 `outputs/` 产物，不需要单独 CLI 模块承接。

### 1.3 与现有 kweaver 命令的关系

- **不影响 `kweaver agent trace <conv_id>`**——那是 `agent` 资源的 `trace` **动作**（v0.7.4 已上线的 4 视图查询）；本设计新增的 `kweaver trace ...` 是顶层 **资源** 命名空间，承载 trace-ai 持续学习子系统。help 文本里点一句区分。
- **复用 `kweaver call` / `kweaver auth` / `kweaver config` 全部既有能力** —— profile / no-auth 哨兵 / TLS env / `--verbose` 这些都不重新造。
- **顶层 flag**（`--base-url` / `--token` / `--user` / `-bd`）继承现有 `cli.ts` 的 strip 逻辑。
- **测试约定** 沿用 `docs/cli_conventions.md` §7：解析器单测 + API 客户端单测 + e2e smoke。

## §2 共享层（trace-core 内核 + api/trace 客户端）

trace 子命令家族复用 kweaver-sdk 现成基础设施（`auth/` / `config/` / `utils/` / `ui/`）之上，新增 6 个共享组件，被多个 M 模块共享。

### 2.1 共享层组件清单（B1–B6）

| 组件 | 路径 | 职责 | 谁用 |
|---|---|---|---|
| **B1 ObservabilityClient** | `src/api/trace/observability.ts` | M3 HTTP 包装：`/traces/{id}` / `/traces?experiment_id=` / `/traces/diff` / 时间窗 + 分页 + 配额错误码处理 | M4 / M5 / M6(watch) / M7 / MX1(audit) |
| **B2 RemoteJobClient** | `src/trace-core/remote-job.ts` | async submit + poll 抽象（vision §6.4.4）：`submit(target, payload) → job_id` / `poll(job_id) → status\|result`；MVP 期内置 sync 降级开关 `--sync` | M5(relabel) / M6(executor/scorer/triage/generator) / M7 |
| **B3 ExpStore** | `src/trace-core/exp-store/` | 实验文件夹持久化抽象（vision §6.4.3）：`mission.md` / `.trace-state/{events.jsonl, trial-forest.yaml, jobs.jsonl, lock.json, abort.signal, rounds/}` / `outputs/` 读写 + lockfile 协议（hostname+pid+30s 心跳） | M6 独占 |
| **B4 GitStateDriver** | `src/trace-core/git-state.ts` | git CLI 包装（spawn `git` 子进程，不引 nodegit / isomorphic-git）：commit / push / pull / `git ls-tree` / `git show`；约定式 commit message；MVP 只用于实验文件夹 checkpoint | M6 |
| **B5 SchemaRegistry** | `src/trace-core/schema/` | schema 静态契约 mirror 副本（`schema/v1/*.yaml`）+ ajv 校验器 + 别名兼容表 + 版本 / 兼容窗口；提供 `validate(kind, doc) → result` | 全 M 模块；MX1 直接暴露 |
| **B6 OutputFormatter** | `src/ui/trace/` | 复用 `src/ui/` 的 ink 设施；`--json` / `--pretty` / `--compact` 三档；长任务 progress（reuse ink-spinner） | 全 M 模块 |

**关键边界约束**：

1. **`trace-core/` 不依赖 `commands/trace/*`**——单向。让 driver 主体（M6 Coordinator）可作为库被未来其他东西调用（比如本地 dashboard / 第三方 wrapper）。
2. **B1 ObservabilityClient 是 above-L0 唯一的 trace 读入口**——任何 M 模块要读 trace **不允许**绕过它直接打 `kweaver call` 或 OpenSearch DSL，避免 schema-bypass。
3. **B5 SchemaRegistry 自带 schema 静态文件副本**——CLI 自包含、无运行时网络依赖；schema 升级 = SDK 发版。

### 2.1.1 B1 / M3 API prerequisite matrix

B1 是 CLI 侧唯一 trace 读入口，但它背后的 M3 能力在落地期会分阶段补齐。为避免各命令在缺接口时私自绕回 OpenSearch DSL，统一按下表处理：

| B1 方法 | M3 目标接口 | 谁用 | MVP 要求 | 缺失时 CLI 行为 |
|---|---|---|---|---|
| `getTrace(traceId)` | `GET /traces/{trace_id}` | M7 / 排障类输出 | **必须** | 不绕过；返回 `TRACE_DETAIL_API_REQUIRED`，提示先升级 M3 |
| `searchTraces(query)` | `GET /traces?...` 或受控 POST 查询 | M4 / M5 / MX1 | **必须** | 不暴露 OpenSearch DSL；只允许 B1 内部使用 M3 受控查询契约 |
| `searchTracesStream(query, page)` | `GET /traces?...&page_token=&page_size=` 或受控 POST 查询 | M4 大集合扫描 | **必须** | 不一次性拉全量；缺分页时返回 `TRACE_PAGINATION_API_REQUIRED` |
| `listExperimentTraces(experimentId, filters)` | `GET /traces?experiment_id=...` | M6 watch / M7 间接定位 | **必须** | 降级为 `searchTraces` 的结构化过滤，不允许命令层拼 DSL |
| `diffTraces(a, b)` | `GET /traces/diff?a=&b=` | M7 | post-MVP 可降级 | MVP 可在 B1 内部拉两条 trace 做 naive diff，并输出 `diff_engine=local-naive` |
| `sampleTraces(window, sample)` | `GET /traces?time_window=&sample=` | MX1 audit | **必须** | 报 `TRACE_SAMPLE_API_REQUIRED`；audit 不做全量扫描 |

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
| **任务配置：我要优化什么** | `mission.md` + 可选 `eval-sets/` | M6 `exp run/resume` 的主入口 |
| **规则配置：我怎么筛 trace / 怎么验收** | `curation-policy.yaml` / `curation-rules/` / `manifest.yaml` | M4 读取筛选策略；M6 输出 manifest 供外部发布/验证 |
| **产物交接：我要把什么交给发布平台** | `outputs/{bundle,manifest,provenance}.yaml` | M6 生成；CLI 不上传、不索引、不周期验证 |

`.trace-state/*`、watermark、`outputs/`、`verification.yaml` 都是运行态或生成物；CLI 可以展示和校验，但用户默认不手写。`eval-sets/` 例外：既可以由用户预置，也可以由 M5 生成。

## §3 模块详设

### 3.1 M4 Curation — `kweaver trace curate`

**命令树**（vision §7.M4）：

```
kweaver trace curate scan         [--policy=<path>] [--time-range=] [--tenant=] [--rules=<path>] [--feed=<path>] [--out=<dir>] [--update-watermark]
kweaver trace curate rules list   [--rules-dir=<path>]
kweaver trace curate rules validate <rule.yaml>
```

**代码模块布局**：

```
src/commands/trace/curate.ts             # 参数解析 + dispatch（含 rules 子资源）
src/trace-core/curate/
  ├── planner.ts             # policy + CLI flags → CurationPlan
  ├── policy.ts              # policy yaml loader + watermark key 计算
  ├── signal-probe.ts        # 三层信号探针（interaction / execution / environment）
  ├── invariant.ts           # 声明式不变量评估器（在 BKN 上 query "若 X 则必读 Y / 必写 Z"）
  ├── latent-failure.ts      # guard-code-as-oracle 检测（默认不调 LLM）
  ├── rules.ts               # rules yaml loader + ajv 校验
  ├── watermark.ts           # 周期 scan 的增量游标读写
  ├── feed-pickup.ts         # 从显式 --feed 路径拾取 curation-feed.yaml（post-MVP 可接 registry）
  └── output.ts              # yaml 序列化（curation-output/<ts>.yaml）
```

**依赖**：

- **B1 ObservabilityClient** — 拉时间窗 / 租户范围内的 trace
- **B5 SchemaRegistry** — 校验 rules yaml 自身合规
- **kweaver-sdk 现有 BKN api client**（`src/api/knowledge-networks.ts`）— 反查不变量直接复用，**不在 trace-core 里重写 BKN client**

**关键设计点**：

1. `scan` 是数据流命令——大 trace 集合通过 B1 `searchTracesStream` 分页 streaming 处理，不落整集到内存。`signal-probe` / `invariant` / `latent-failure` 三件以 pipeline 形式串成 transform stream，单条 trace 进、判定结果出。
2. M4 只有局部 **CurationPlanner**，不新增全局 TriageController。planner 把 `--policy`、CLI flags、watermark、本地 `--feed` 合成一次性的 `CurationPlan`，然后交给 pipeline 执行。
3. `rules` 子资源符合 cli_conventions §2 "顶层 + 子资源 + 动作"——规则文件本身 git-tracked（约定 `<repo>/curation-rules/`），CLI 不管 register API。
4. **MVP 不自动拉中央 feed**：`scan` 只读取本地规则与显式 `--feed=<path>`。上线后发现的新失败模式先由人工或外部流程写成本地 `curation-feed.yaml`，用户下一轮显式传给 `curate scan --feed=...`。中央 feed 聚合留到 post-MVP。
5. watermark 只用于周期 scan，不用于一次性排障。显式 `--time-range` 默认不读写 watermark；只有 `--policy` + `--update-watermark` 同时出现时，成功输出后才推进 watermark。

**CurationPolicy 最小契约**：

```yaml
policy_id: prod-agent-daily
scope:
  tenant: acme
  agent_id: agent_123
  time_window: 24h
rulesets:
  - curation-rules/
feed:
  path: curation-feed.yaml
watermark:
  enabled: true
  safety_lag: 10m
```

`CurationPlan` 是运行时派生对象，不需要用户手写。watermark key = `policy_id + scope_hash`，value = `{trace_end_time, trace_id}`；只在 output 写入成功且 schema 校验通过后推进到 `upper_bound - safety_lag`。失败不推进，下一次允许重复扫描；输出侧按 `trace_id + rule_id` 去重。

**核心流程与逻辑（以 `curate scan` 为例）**：

1. **环境准备与规则加载**：
   - `planner.ts` 读取 `--policy` 与 CLI flags，生成本次 `CurationPlan`。
   - 从默认路径 `<repo>/curation-rules/`、policy 声明的 rulesets、用户指定的 `--rules` 路径加载 YAML 规则。
   - 若指定 `--feed` 或 policy 声明 feed path，则读取本地 `curation-feed.yaml`，提取失败模式并转换为动态规则，合并到规则集中。
   - 若启用 `--update-watermark`，根据 policy watermark 与 safety lag 计算本次扫描的上下界。
2. **数据流式获取**：
   - 调用 B1 (ObservabilityClient)，传入时间窗和租户参数，获取 Trace 数据的异步分页流（`AsyncIterable<TraceBatch>`）。
3. **管道式过滤（Core Pipeline）**：
   - 建立一个 Transform Stream 管道，每条 Trace 依次流经：
     - `signal-probe`：解析 Span 属性，匹配交互/执行层异常规则。
     - `invariant`：若规则涉及 BKN 不变量，调用 BKN API 验证该 Trace 是否满足“应读未读”等约束。
     - `latent-failure`：通过确定性规则反查隐性失败。
   - 任何一个环节触发规则，即为该 Trace 打上 `rule_id` 和原因标签。
   - **短路机制**：一旦某条 Trace 触发了任意一条规则，立即流向输出缓冲区，不再继续做耗时的后续反查。
4. **结果输出**：
   - 命中的 Trace 及其元数据被收集，通过 `output.ts` 序列化为 YAML 格式。
   - 写入到指定的输出目录（默认为 `curation-output/`），文件名包含时间戳。

### 3.2 M5 Eval-Set Builder — `kweaver trace eval-set`

**命令树**（vision §7.M5）：

```
kweaver trace eval-set build   [--queries=<path>] [--curation=<path>] --out=<dir> [--with-reference]
kweaver trace eval-set relabel <eval-set-dir> [--sync] [--force]
```

**代码模块布局**：

```
src/commands/trace/eval-set.ts            # 参数解析 + dispatch
src/trace-core/eval-set/
  ├── loader.ts              # EvalSetRef[] → EvalCase[]，支持 index + shard
  ├── query-picker.ts        # 从 mission.md fallback queries + M4 curation yaml 圈选
  ├── redactor.ts            # 自动脱敏（PII / 业务密文 patterns；规则 yaml-driven）
  ├── relabel.ts             # hindsight relabel via async LLM（用 B2 RemoteJobClient）
  └── output.ts              # eval-set yaml 多文件目录写入
```

**依赖**：

- **B1 ObservabilityClient** — 圈选时拉 trace 拼上下文
- **B2 RemoteJobClient** — relabel 走 async submit + poll；`--sync` 给开发者本地小集合用
- **B5 SchemaRegistry** — eval-set yaml schema 校验
- **不依赖 B4 GitStateDriver**——eval-set 写到 `<repo>/eval-sets/<name>/`，git 由用户自己 commit/push

**关键设计点**：

1. eval-set 是一等输入，不只是 M5 产物。用户已有“请求 + 标准答案”可直接放入 `<repo>/eval-sets/<name>/`，由 `mission.md` 引用；M5 生成的数据也写成同一格式。
2. `build` 是无状态的纯函数——输入 (fallback queries, curation 子集) → 输出 yaml 集合。两态（带 reference / 不带）由 `--with-reference` flag 切。
3. `relabel` 必须 async（vision §6.4.4 钉死方向）；输出**沿用同一个 eval-set 目录原地改写**——relabel 的 audit 痕迹靠 git history 本身承载，不复制到新目录。
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

**核心流程与逻辑（以 `eval-set build` 和 `relabel` 为例）**：

**`eval-set build` 流程**：
1. **源数据圈选**：
   - `query-picker.ts` 聚合输入的 `--queries`（mission fallback queries）和 `--curation`（来自 M4 的输出）。
   - 如果指定了 `--with-reference`，会尝试从历史 Trace 中提取“成功的标准输出”作为 Ground Truth（Reference）。
2. **敏感信息脱敏**：
   - 遍历圈选出的 Query 和上下文，调用 `redactor.ts`。
   - 根据 `<repo>/redaction-rules/` 中的规则，对 PII（个人隐私）和业务密文进行脱敏（替换为 Hash 或脱敏占位符）。
3. **写入与校验**：
   - 调用 `output.ts` 将脱敏后的数据写入指定的输出目录（`<repo>/eval-sets/<name>/`）。
   - 调用 B5 (SchemaRegistry) 校验生成的 YAML 是否符合 Eval-Set 的标准 Schema。

**`eval-set relabel` 流程**：
1. **加载与提交**：
   - 读取指定目录下的 Eval-Set 文件。
   - 执行 git preflight：确认目录在 git repo 内、目标文件 clean；`--force` 时只警告不阻断。
   - 调用 B2 (RemoteJobClient) 将需要打标的失败轨迹（如 latent failure）打包，异步提交给远端 LLM（或通过 `--sync` 同步处理）。
2. **结果轮询与就地改写**：
   - 若为异步，CLI 周期性轮询（Poll）Job 状态。
   - 任务成功后，拉回“原行为 vs 应有行为”的偏好对（Preference Pairs）。
   - **核心逻辑**：不创建新目录，直接**原地改写（In-place Rewrite）**原 Eval-Set 文件。审计和版本追踪完全依赖 Git History。

### 3.3 M6 Experiment Engine — `kweaver trace exp`

M6 是 MVP 模块里**唯一真正复杂**的——带 FSM、长生命周期、跨进程重启、跨机器接力、async 远端编排。

**实验目录用户视图**：

```
my-experiment/
  ├── mission.md              # 必填：任务配置，用户主要编辑
  ├── curation-policy.yaml    # 可选：规则配置，生产 trace 周期筛选时使用
  ├── curation-rules/         # 可选：本实验规则；可叠加显式 --feed
  ├── eval-sets/              # 用户预置或 M5 生成；可 review
  ├── outputs/                # 生成物：bundle.yaml / manifest.yaml
  └── .trace-state/           # 运行态：events/jobs/lock/rounds，用户不手写
```

`exp run` 的冷启输入规则：优先加载 `mission.md` 中 `eval_sets` 引用的 `seed` eval set；如果没有任何 eval set，则使用 mission fallback `queries` 生成最小 eval set。Round 1+ 可以同时使用 `seed`、`regression` 与 M4/M5 追加的新 cases；`holdout` 只用于最终验证，不参与生成方向。

**`exp run` preflight 卡点**：

`exp run` / `resume` 在抢到 lock 后、进入 active FSM state 前，必须执行 preflight；失败时不 submit 远端 job，不生成 Trial。

1. 解析 `mission.md` frontmatter 与 fallback queries。
2. 解析 `eval_sets[]` 引用，加载 `index.yaml` 与所有 shard。
3. 调用 B5 SchemaRegistry 校验 `eval-set-index` / `eval-set`。
4. 检查 `query_id` 在所有参与本次实验的 eval set 内全局唯一。
5. 检查 shard path 不越界、role 枚举合法、必填字段存在。
6. 检查 provider capability（如 synthesizer provider / kweaver-core 依赖）。

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
kweaver trace exp run    [folder]                         # 抢 lock + 读 mission.md + 启动/续跑 FSM
kweaver trace exp resume [folder]                         # 语义等价于 run（强调"续跑"），同一代码路径
kweaver trace exp show   [folder]                         # 只读：展示任务配置、派生输入、输出物引用
kweaver trace exp status [folder]                         # 只读：一次性折叠 events/jobs/lock，输出当前进展与状态摘要
kweaver trace exp watch  [folder]                         # 只读：tail events.jsonl + 拉 M3 拼当前进度视图
kweaver trace exp abort  [folder]                         # 写 abort 信号；driver 在下一个 checkpoint 优雅退出
kweaver trace exp list   [path...]                        # 扫 path 下所有实验文件夹列状态
kweaver trace exp doctor [folder]                         # 只读：校验 mission/eval-set/journal/lock/jobs/outputs 健康度
```

#### 3.3.2 代码模块布局（两层）

底层是 `exp-store/` 持久化抽象（**所有对实验文件夹的读写必须经过它**），上层是 `exp-engine/` 编排逻辑。依赖方向固定为 `exp-engine/* → exp-store/*`；`exp-store/` 不知道 FSM、Generator、Scorer、Triage 等运行时概念。

```
src/commands/trace/exp.ts                       # 参数解析 + dispatch（8 个子命令）

src/trace-core/exp-store/
  ├── paths.ts                # canonical 路径解析
  ├── mission-md.ts           # mission.md 解析（YAML frontmatter + body fallback queries 段）
  ├── preflight.ts            # mission / eval-set / provider capability 启动前校验
  ├── events-jsonl.ts         # append-only 写 + replay 读（FSM 真源）
  ├── trial-forest-yaml.ts    # 派生关系拓扑 yaml（快照式覆写）
  ├── jobs-jsonl.ts           # 远端 job_id 流水（append-only）
  ├── round-yaml.ts           # rounds/round-N.yaml 读写
  ├── lock.ts                 # 心跳锁协议（hostname+pid+last_heartbeat_ts，30s 超时）
  ├── abort-signal.ts         # .trace-state/abort.signal 文件读写
  └── read-model.ts           # 只读 fold：mission + events + jobs + lock + outputs → ExperimentSnapshot

src/trace-core/exp-engine/
  ├── coordinator.ts          # FSM driver 主循环（元控制层）
  ├── fsm.ts                  # 6+1 状态枚举 + 转移表 + checkpoint 钩子
  ├── trial-forest-ops.ts     # Trial Forest 拓扑增删改逻辑
  ├── generator.ts            # thin wrapper：调 Agent Synthesizer provider
  ├── executor.ts             # K×M 调度（执行层；B2 async + jobs.jsonl）
  ├── scorer.ts               # thin wrapper：调 kweaver-eval + 三轴合成 + safety hard gate
  ├── triage.ts               # thin wrapper：调远端 Triage Agent + 跨轮记忆 binding
  ├── termination.ts          # 本地 Termination Decider
  ├── git-checkpoint.ts       # Round-内 local commit / Round-末 push 编排
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

**真源分层**：`.trace-state/events.jsonl` 是 FSM 的 **append-only journal**——每次状态迁移、每个 checkpoint、每次 abort 检测都 append 一行。`.trace-state/jobs.jsonl` 是远端 job 的 **append-only job journal**——专门兜住 submit / poll 的 crash point。`trial-forest.yaml` / `rounds/round-N.yaml` 是从 journal 派生的快照，崩坏可重建；`lock.json` / `abort.signal` 是运行期控制文件，不作为业务真源。

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
| `trial_generated` | Generator 返回 Trial 后 | `trial_id` / `parent_trial_id` / `variation` | 重建 trial-forest |
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
4. `trial-forest.yaml` / `round-N.yaml` 的 `source_event_id` 必须指向生成它们的最新事件，便于检测快照落后。

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
              必要时同步快照（trial-forest.yaml / rounds/round-N.yaml）
                    ↓
              checkpoint 触发 git commit（local，无 push）
```

#### 3.3.4 6 个内部子件的本地 / 远端归属

| 子件 | 类型 | CLI 进程内做什么 | 远端做什么 |
|---|---|---|---|
| **Coordinator** | 元控制（本地） | FSM 驱动 / events.jsonl append / Trial Forest 内存维护 / git commit 编排 | — |
| **Agent Synthesizer / Generator** | 研判（thin wrapper） | 拼 payload（mission / current_agent / trace evidence / Triage hints） / 选择 provider / B2 submit / 收 K Trial 落进 trial-forest.yaml | 智能：冷启生成 Agent / Knowledge Network / Skill 绑定；或基于现有 Agent 配置与 trace 派生优化 Trial |
| **Executor** | 执行（编排） | K×M 任务批量 B2 submit → jobs.jsonl 流水 → 周期 poll | DA 跑实际 trial × query |
| **Scorer** | 研判（thin wrapper） | 收 trial 完成事件 → B2 submit kweaver-eval → 收三轴分 → 按 trial 角色合成（vs-parent / 跨派生链 / cross-round）+ safety hard gate 横切淘汰 | 智能：deterministic + LLM-judge 双轨打分 |
| **Triage Agent** | 研判（thin wrapper） | Round 末把当 round 数据 + 跨轮记忆引用 B2 submit → 收诊断 / 改进方向 / 趋势 → 落 rounds/round-N.yaml | 智能：跨轮记忆维护 / 全局视野 / 砍枯枝 / slot 跨树分配 |
| **Termination Decider** | 元控制（本地） | 读最新 round-N.yaml + guardrail 历史 → 三选一判定（饱和 / 收敛 / 用户介入） | — |

**关键约束**：研判层 3 件（Agent Synthesizer / Scorer / Triage）的"智能"100% 在 provider / 远端；CLI 内部是纯 binding——CLI 实现不依赖任何 LLM SDK、不用本地 GPU、可以 `npm install` 完了 offline 跑（除了 B2 远端调用本身）。

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
- provider 只能产出 proposal / patch；真正进入 Trial Forest 前，CLI 必须用 B5 SchemaRegistry 校验 Trial schema，并检查 resource binding 是否引用存在的 KWeaver 资源。

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
| `exp show [folder]` | 这个任务配置成了什么 | `mission.md` / `eval-sets/` / `curation-policy.yaml` / `outputs/` | experiment_id、goal、guardrails、eval set 摘要、synthesizer provider、bundle/manifest 引用 |
| `exp status [folder]` | 当前跑到哪一步 | `.trace-state/events.jsonl` / `jobs.jsonl` / `lock.json` / `abort.signal` / `rounds/` / 可选 B1 trace 聚合 | FSM state、round、trial 数、K×M cell 进度、pending jobs、最近错误、driver 心跳、abort 状态 |
| `exp doctor [folder]` | 这个任务是否可继续运行 | `show + status` 的全部来源 + B5 schema 校验 + journal replay 校验 | preflight 结果、坏 journal 行、重复 event_id、快照滞后、悬挂 job、过期 lock、缺失 outputs |

`show` 偏配置，`status` 偏进展，`doctor` 偏可恢复性与可操作诊断。三者共用 `exp-store/read-model.ts` fold 出 `ExperimentSnapshot`，避免每个命令各自解析 journal。`doctor` 只报告修复建议，MVP 不自动改文件；未来可加 `exp doctor --repair`，但必须逐项确认或要求显式 flag。

**watch 边界**：

- **不抢 lock、不写任何文件**——只读 events.jsonl + jobs.jsonl（fs.watch 或 100ms polling fallback for cross-platform）+ B1 拉对应 round 的 trace 拼当前进度
- 多人同时 watch 互不干扰
- watch 的 UI 用 ink，三栏：FSM 状态时间轴 / 当前 round 的 K×M 矩阵进度 / 最近 5 行 events；底层同样复用 `ExperimentSnapshot`，只是在 fs watch / polling 下持续刷新

**list 边界**：

- 不抢 lock。扫 `path...` 下含 `.trace-state/` 的子目录 → 对每个读取轻量 `ExperimentSnapshot` → 输出表格（experiment / state / round / pending-jobs / driver-host / heartbeat-age / last-error / bundle-ready）

#### 3.3.6 commit / push 节奏

由 `git-checkpoint.ts` 编排，三档触发：

| 触发 | 操作 | 提交信息约定 |
|---|---|---|
| FSM 进入新稳定状态 | local commit（不 push） | `exp <name> round-<N>: <prev>→<state>` |
| Round 末（Triaging→Deciding 完成） | local commit + push | `exp <name> round-<N> complete; trial-count=<K>; verdict=<continue\|publish>` |
| Publishing 输出 bundle 后 | local commit | `exp <name> bundle-<id> ready in outputs/` |
| Aborted | local commit | `exp <name> aborted at round-<N>:<state>` |

push 失败（网络断 / 权限拒）不阻断 FSM——commit 已落地，下次 commit 时累积一起 push。**git 的 eventual consistency 是 vision §6.4 心智的一部分**。

#### 3.3.7 关键 trade-offs

1. **`run` / `resume` 同一代码路径**——避免双套 FSM 启动逻辑漂移；用户语义靠提示语区分。
2. **events.jsonl 是 FSM 真源，jobs.jsonl 是远端 job journal，trial-forest.yaml / round-N.yaml 是派生快照**——崩坏可从 journal 重建。事件文件持续追加，单实验持续若干月可能涨到几十 MB；可接受，未来涨到 GB 级再加 segment 切分。
3. **B2 RemoteJobClient 是研判层 3 件的唯一出口**——保证 driver 离线 / 笔记本关停可恢复。MVP 期 `--sync` 降级开关只允许 dev / debug 用。
4. **lock 是 cooperative，不是分布式锁**——vision §6.4.5(a) 已认代价；冲突发生时该轮重做，不在 trace-core 加 ZooKeeper / etcd。
5. **watch / list 不抢 lock**——读路径无锁，多人监控不干扰；可装到 CI dashboard 持续显示。
6. **commit 不 push 的 Round 内策略**——避免每个 FSM 状态转移都污染远端 git history；Round 末才 push 一次。

### 3.4 M7 Replay — `kweaver trace replay`

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
2. **DA 自己打 OTel 喂回 M1，CLI 不直接 push OTLP**——vision §7.M7 此处措辞已修订（见 §5 patch P9）。
3. **mode 三档 MVP 期单档实现**——MVP 仅实现 `compare`（最常用），`strict` / `explore` 留 enum 但报"not yet implemented"；保留 flag 是为了未来扩展不破坏命令字面值。
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

**命令树**（砍 CronJob 后）：

```
kweaver trace schema validate <file> [--kind=<kind>]                          # 开发者本地：单文件 ajv 校验（含 eval-set / eval-set-index）
kweaver trace schema audit    [--time-window=1h] [--sample=1000] [--out=]     # CI 周期调用：跨 span 不变量 / 漂移率 / 准入率 报告
```

**代码模块布局**：

```
src/commands/trace/schema.ts                        # dispatch
src/trace-core/schema/                              # B5 SchemaRegistry 主体
  ├── index.ts               # 公共 API：validate(kind, doc) / loadSchema(kind, version) / aliasResolve(field)
  ├── validator.ts           # ajv 实例 + 别名兼容表 + 兼容窗口判定
  ├── alias-table.ts         # 字段别名兼容（如 session_id ↔ agent.session.id）
  ├── audit/
  │   ├── invariant.ts       # 跨 span 不变量评估（要看完整 trace 上下文）
  │   ├── drift.ts           # 漂移率指标（兼容窗口监控）
  │   └── admit-rate.ts      # L1/L2 准入率指标
  ├── audit-orchestrator.ts  # audit 子命令主流程（抽样 → 三件并行 → 合 report）
  ├── report.ts              # audit 报告 yaml 序列化
  └── v1/                    # ← schema 静态文件 mirror（trace.yaml / experiment.yaml / bundle.yaml / manifest.yaml / eval-set.yaml / eval-set-index.yaml）
```

**依赖**：

- **schema/v1/*.yaml mirror**（自带，build 时打进 dist）
- **B1 ObservabilityClient** — audit 时按 `time-window + sample` 抽样拉 trace
- 不依赖 B2/B4

**外部触发：CI workflow（trace-ai 仓库自带）**：

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
4. **inline 校验仍然在 M1 otelcol 端做**（vision §7.M1 的 schema-check hook 不动）；CLI 这边只做：①单文件本地校验给开发者用；②周期 audit 拿 inline 干不了的三件（跨 span / 漂移 / 准入）。两条路径互不干涉。
5. **audit 抽样不全量**——CI runner 离 M2 远，全量校验数据量过大；vision §7.MX1 本来就钉死抽样。`--sample=1000` 默认值在初期可调。

## §4 测试与发布策略

### 4.1 测试约定（沿用 cli_conventions §7）

每个新命令必须包含：

1. **解析器单测**（`test/trace-<cmd>-cmd.test.ts`）：覆盖每个 flag 的 happy path 与至少一个错误路径
2. **API 客户端单测**（`test/api/trace/<resource>.test.ts`）：mock `fetch`，断言 URL、method、headers、body
3. **e2e smoke**（`test/e2e/trace-<resource>.test.ts`）：跑通完整链路，环境变量缺失时跳过

**新增**（trace 子命名空间专属）：

4. **trace-core 单元测**（`test/trace-core/<module>.test.ts`）：FSM transition / lock / events.jsonl replay / git-state 等内核组件独立测试
5. **schema mirror lint**（CI step）：`schema-mirrors.lock` 中每条 source 的 SHA + target hash 一致性
6. **状态恢复 golden tests**：固定一组 `events.jsonl + jobs.jsonl` 样本，断言 replay 后的 state / pending_jobs / trial-forest 完全一致
7. **crash-point tests**：覆盖 `job_submit_intent 已写但 submit 未完成`、`submit 成功但 jobs.jsonl job_submitted 未写`、`jobs.jsonl job_completed 已写但业务事件未写`、`round_completed 已写但 git commit 失败` 四类断点
8. **git checkpoint tests**：用临时 git repo 覆盖 M6 local commit、push 失败不阻断、后续重试 push 等路径
9. **安全输出 tests**：默认输出不得包含 prompt / tool result 全文；`--json` 与 pretty 路径同样遵守 redaction；`--unsafe-full` 需要显式开关

### 4.2 发布策略

- **整体跟 kweaver-sdk 现有节奏**：不独立发版；trace 子命名空间作为 kweaver-sdk 新 minor 上线
- **MVP 顺序**（建议，非强约束）：MX1 + B1/B5 → M4 → M5 → M7 → M6
  - 理由：先把 schema 静态契约 + M3 客户端 + 单文件校验立起来；再做无状态的 curate / eval-set；M7 replay 可直接吃现有真实 trace，较早验证 B1 + B2 + DA replay 契约，也能暴露 trace 上下文是否足够重放；最后进入最大头 M6。M8/M9 不进 MVP。
  - 这条是实现落地顺序，不是用户操作顺序。§0.1.1 Story A 要等 M6 及其依赖就绪后才成立；在此之前，M4/M5/M7 可以作为独立命令先行交付和验证。
- **每个 minor 都跑 schema-mirrors.lock CI lint**——drift 即红
- **MVP 期 B2 RemoteJobClient 内置 `--sync` 降级开关**给 dev 用；正式实验路径强制 async（vision §6.4.4 钉死方向）

### 4.3 文档

- `docs/cli_conventions.md`：现有文档 §1–§7 不动；本设计完成后追加 §8 trace 子命名空间约定（指向本文档）
- 每个 trace 子命令的 `--help` 文本：包含命令字面值 + 一条 example + 链接到本文档对应章节
- README / `kweaver --help` 顶层命令树更新（加 `kweaver trace ...` 一行）

## §5 对 vision 的 patch 清单

> **状态**：2026-05-11 重新做 MVP 架构减法。此前 P1–P15 的历史 patch 记录不再逐条保留，避免与当前 MVP 形态冲突。以本节 P16 为准。

| # | vision 段落 | 改动 |
|---|---|---|
| **P16** | §6.4 / §7.M8 / §7.M9 / §7.4 / §8 | MVP 收敛为 M4/M5/M6/M7/MX1；删除 MVP publish-registry URL 配置、`bundle submit/list/show` registry 形态、`verify scan/check` registry 形态与 publish-registry CI；M6 `outputs/` 成为与外部发布平台的唯一交接面；M8/M9 改为 post-MVP deferred |

---

## 附录 A：trace-core/ 目录全景

为方便实现期参考，集中列出 trace-core/ 下的所有目录与文件归属：

```
src/trace-core/
  ├── remote-job.ts                       # B2 共享：async submit + poll
  ├── git-state.ts                        # B4 共享：git CLI 包装

  ├── exp-store/                          # B3 共享：实验文件夹持久化抽象（M6 独占）
  │   ├── paths.ts
  │   ├── mission-md.ts
  │   ├── events-jsonl.ts
  │   ├── trial-forest-yaml.ts
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
  │   ├── trial-forest-ops.ts
  │   ├── generator.ts
  │   ├── executor.ts
  │   ├── scorer.ts
  │   ├── triage.ts
  │   ├── termination.ts
  │   ├── git-checkpoint.ts
  │   ├── status.ts
  │   ├── doctor.ts
  │   └── watch.ts

  ├── curate/                             # M4 专属
  │   ├── planner.ts
  │   ├── policy.ts
  │   ├── signal-probe.ts
  │   ├── invariant.ts
  │   ├── latent-failure.ts
  │   ├── rules.ts
  │   ├── watermark.ts
  │   ├── feed-pickup.ts
  │   └── output.ts

  ├── eval-set/                           # M5 专属
  │   ├── query-picker.ts
  │   ├── redactor.ts
  │   ├── relabel.ts
  │   └── output.ts

  ├── replay/                             # M7 专属
  │   ├── locator.ts
  │   ├── request-builder.ts
  │   ├── diff.ts
  │   └── output.ts

src/commands/trace/                       # 各 M 模块的命令入口（薄层；只做参数解析 + dispatch 到 trace-core）
  ├── exp.ts
  ├── curate.ts
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

| | B1 Obs | B2 RemoteJob | B3 ExpStore | B4 GitState | B5 SchemaReg | B6 Output | BKN api（既有） | kweaver-core skill/CLI |
|---|---|---|---|---|---|---|---|---|
| **M4 curate** | ✓ 读 trace | | | | ✓ 校 rules | ✓ | ✓ 反查不变量 | |
| **M5 eval-set** | ✓ 圈 query | ✓ relabel | | | ✓ 校 eval-set | ✓ | | |
| **M6 exp** | ✓ watch 拉 trace | ✓ Agent Synthesizer / Executor / Scorer / Triage async | ✓ 独占写 | ✓ commit/push | ✓ 校 Trial proposal | ✓ | | ✓ synthesis 写入 / 校验平台资产 |
| **M7 replay** | ✓ 拉新旧 trace | ✓ DA replay | | | | ✓ | | |
| **MX1 schema** | ✓ audit 抽样 | | | | self | ✓ | | |
