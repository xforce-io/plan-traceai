---
title: trace-ai CLI 详细设计
status: draft
date: 2026-05-09
依赖: vision/trace-ai-continuous-learning-design.md（以下简称 vision）
覆盖: M4 / M5 / M6 / M7 / M8 / M9 / MX1 七个模块的 CLI 落地形态
---

## §0 范围与背景

vision §6.4 / §7 把 trace-ai above-L0 的 7 个模块（M4 Curation / M5 Eval-Set Builder / M6 Experiment Engine / M7 Replay / M8 Publish Registry / M9 Post-deploy Verify / MX1 Schema）钉成 CLI 形态。本文档承接 vision，给这 7 个模块的 CLI 落到具体形态：

- **代码归属**：哪个文件 / 哪个仓库
- **命令字面值**：用户键入什么
- **模块边界**：内部子模块怎么切、依赖方向
- **共享层契约**：跨模块复用的内核组件
- **触发机制**：周期性任务怎么不靠 K8s CronJob 也能跑

本文档**不**覆盖：每个 HTTP 接口的字段细节（属下游各 M-spec）、远端智能体层（DA / kweaver-eval / Triage Agent）的内部实现（属各自子 spec）、L0 数据面（M1 otelcol / M2 OpenSearch / M3 agent-observability）的服务实现（属 trace-ai 仓库后端）。

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
        │   │         ├── bundle.ts
        │   │         ├── verify.ts
        │   │         └── schema.ts
        │   ├── api/
        │   │   ├── (既有 api clients)
        │   │   └── trace/              # ← trace-ai 专属 HTTP 客户端
        │   │         └── observability.ts
        │   └── trace-core/             # ← trace-ai 专属内核（FSM / git / async-poll / schema / experiment-folder）
        │         ├── experiment-folder/
        │         ├── exp/
        │         ├── curate/
        │         ├── eval-set/
        │         ├── replay/
        │         ├── bundle/
        │         ├── verify/
        │         ├── schema/
        │         ├── remote-job.ts
        │         └── git-state.ts
        └── schema-mirrors.lock         # ← schema 静态契约同步源 SHA 锁
```

**单向依赖**：`commands/trace/*` → `trace-core/*` → `api/trace/*` → `auth/` + `config/`（既有基础设施）

### 1.2 命令树总览

7 个模块按 cli_conventions §2 "顶层 + 子资源 + 动作" 三段式落地：

```
kweaver trace exp       run | resume | watch | abort | list                     # M6
kweaver trace curate    scan | rules list | rules validate                       # M4
kweaver trace eval-set  build | relabel                                          # M5
kweaver trace replay    <trace_id|--experiment-id+--query>                       # M7（动作即资源）
kweaver trace bundle    submit | show | list                                     # M8
kweaver trace verify    check | scan                                             # M9
kweaver trace schema    validate | audit                                         # MX1
```

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
| **B1 ObservabilityClient** | `src/api/trace/observability.ts` | M3 HTTP 包装：`/traces/{id}` / `/traces?experiment_id=` / `/traces/diff` / 时间窗 + 分页 + 配额错误码处理 | M4 / M5 / M6(watch) / M7 / M9 / MX1(audit) |
| **B2 RemoteJobClient** | `src/trace-core/remote-job.ts` | async submit + poll 抽象（vision §6.4.4）：`submit(target, payload) → job_id` / `poll(job_id) → status\|result`；MVP 期内置 sync 降级开关 `--sync` | M5(relabel) / M6(executor/scorer/triage/generator) / M7 |
| **B3 ExperimentFolder** | `src/trace-core/experiment-folder/` | portable folder 目录契约（vision §6.4.3）：`mission.md` / `.trace-state/{events.jsonl, trial-forest.yaml, jobs.jsonl, lock.json, abort.signal, rounds/}` 读写 + lockfile 协议（hostname+pid+30s 心跳） | M6 独占；M8 read-only |
| **B4 GitStateDriver** | `src/trace-core/git-state.ts` | git CLI 包装（spawn `git` 子进程，不引 nodegit / isomorphic-git）：commit / push / pull / `git ls-tree` / `git show`；约定式 commit message；publish-registry repo clone-or-pull | M6 / M8 / M9 |
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
| `searchTraces(query)` | `GET /traces?...` 或受控 POST 查询 | M4 / M5 / M9 / MX1 | **必须** | 不暴露 OpenSearch DSL；只允许 B1 内部使用 M3 受控查询契约 |
| `searchTracesStream(query, page)` | `GET /traces?...&page_token=&page_size=` 或受控 POST 查询 | M4 / M9 大集合扫描 | **必须** | 不一次性拉全量；缺分页时返回 `TRACE_PAGINATION_API_REQUIRED` |
| `listExperimentTraces(experimentId, filters)` | `GET /traces?experiment_id=...` | M6 watch / M7 间接定位 | **必须** | 降级为 `searchTraces` 的结构化过滤，不允许命令层拼 DSL |
| `diffTraces(a, b)` | `GET /traces/diff?a=&b=` | M7 | post-MVP 可降级 | MVP 可在 B1 内部拉两条 trace 做 naive diff，并输出 `diff_engine=local-naive` |
| `sampleTraces(window, sample)` | `GET /traces?time_window=&sample=` | MX1 audit | **必须** | 报 `TRACE_SAMPLE_API_REQUIRED`；audit 不做全量扫描 |

**M3 错误码要求**：B1 需要区分 `SCHEMA_VALIDATION_FAILED` / `QUERY_TOO_LARGE` / `RATE_LIMITED` / `STORAGE_UNAVAILABLE` / `AUTH_FORBIDDEN`。如果 M3 仍返回泛化 5xx，B1 保留原始 HTTP status + response body 摘要，CLI 输出里标 `unclassified_m3_error=true`，避免误判成远端 job 或 CLI bug。

**流式读取契约**：`searchTracesStream` 是 B1 暴露给 M4 / M9 的唯一大集合入口，返回 `AsyncIterable<TraceBatch>`，每个 batch 带 `{items, next_page_token?, response_meta}`。CLI 默认 `page_size=500`，可由命令内核按内存压力下调；命令层只消费 async iterator，不直接感知 M3 分页字段。M3 必须保证同一查询在短时间窗口内的分页稳定性；若无法保证，B1 输出 `pagination_consistency=best_effort`，并要求调用方在报告里标注抽样/扫描口径。

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

### 2.3 publish-registry URL 解析

vision §7.M8 把 publish-registry repo "由组织管理" 留白，本设计钉统一解析（M4 / M8 / M9 共用）：

1. **命令覆盖值**：`--registry=<git-url>`（适用于没有 experiment-folder 上下文的命令，如 `curate scan` / `bundle show` / `bundle list` / `verify check` / `verify scan`）
2. **实验覆盖值**：实验文件夹 `mission.md` 的 frontmatter 可声明 `publish-registry: <git-url>`（适用于 `bundle submit <experiment-folder>` 这类有实验上下文的命令）
3. **默认值**：`kweaver config trace.publish-registry-url <git-url>`（per-profile，复用既有 `kweaver config` 设施）

优先级：`--registry` > mission.md > kweaver config > 报错。无配置时，所有需要读取或写入 publish-registry 的命令都拒绝执行；`curate scan --no-feed-pull` 是唯一例外，因为它显式跳过 registry feed。

### 2.4 用户配置心智

CLI 实现会读写多类文件，但用户只需要按三类配置理解：

| 用户心智 | 实际载体 | CLI 行为 |
|---|---|---|
| **环境配置：我连哪里** | `kweaver config` / env | `auth` / `config` 统一读取；trace 子命令不另建配置系统 |
| **任务配置：我要优化什么** | `mission.md` + 可选 `eval-sets/` | M6 `exp run/resume` 的主入口；M8 `bundle submit` 读取 registry override |
| **规则配置：我怎么筛 trace / 怎么验收** | `curation-policy.yaml` / `curation-rules/` / `manifest.yaml` | M4 读取筛选策略；M8/M9 读取 manifest 做发布和对账 |

`.trace-state/*`、watermark、`outputs/`、`verification.yaml` 都是运行态或生成物；CLI 可以展示和校验，但用户默认不手写。`eval-sets/` 例外：既可以由用户预置，也可以由 M5 生成。

## §3 模块详设

### 3.1 M4 Curation — `kweaver trace curate`

**命令树**（vision §7.M4）：

```
kweaver trace curate scan         [--policy=<path>] [--time-range=] [--tenant=] [--rules=<path>] [--out=<dir>] [--registry=<git-url>] [--update-watermark] [--no-feed-pull]
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
  ├── feed-pickup.ts         # 从 publish-registry 拾取 curation-feed.yaml（飞轮闭合）
  └── output.ts              # yaml 序列化（curation-output/<ts>.yaml）
```

**依赖**：

- **B1 ObservabilityClient** — 拉时间窗 / 租户范围内的 trace
- **B4 GitStateDriver** — `git clone --depth 1` publish-registry，读 `bundles/*/curation-feed.yaml` 子树
- **B5 SchemaRegistry** — 校验 rules yaml 自身合规
- **kweaver-sdk 现有 BKN api client**（`src/api/knowledge-networks.ts`）— 反查不变量直接复用，**不在 trace-core 里重写 BKN client**

**关键设计点**：

1. `scan` 是数据流命令——大 trace 集合通过 B1 `searchTracesStream` 分页 streaming 处理，不落整集到内存。`signal-probe` / `invariant` / `latent-failure` 三件以 pipeline 形式串成 transform stream，单条 trace 进、判定结果出。
2. M4 只有局部 **CurationPlanner**，不新增全局 TriageController。planner 把 `--policy`、CLI flags、watermark、registry feed 合成一次性的 `CurationPlan`，然后交给 pipeline 执行。
3. `rules` 子资源符合 cli_conventions §2 "顶层 + 子资源 + 动作"——规则文件本身 git-tracked（约定 `<repo>/curation-rules/`），CLI 不管 register API。
4. **飞轮闭合的唯一闭合点**：`scan` 起手 git pull publish-registry → 把所有 `bundles/*/curation-feed.yaml` merge 进当前规则集 → 再扫。`--no-feed-pull` 跳过此步（用于本地调试）。命令结束打印 `feed synced from <SHA>`。
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
registry_feed: true
watermark:
  enabled: true
  safety_lag: 10m
```

`CurationPlan` 是运行时派生对象，不需要用户手写。watermark key = `policy_id + scope_hash`，value = `{trace_end_time, trace_id}`；只在 output 写入成功且 schema 校验通过后推进到 `upper_bound - safety_lag`。失败不推进，下一次允许重复扫描；输出侧按 `trace_id + rule_id` 去重。

**核心流程与逻辑（以 `curate scan` 为例）**：

1. **环境准备与规则加载**：
   - `planner.ts` 读取 `--policy` 与 CLI flags，生成本次 `CurationPlan`。
   - 从默认路径 `<repo>/curation-rules/`、policy 声明的 rulesets、用户指定的 `--rules` 路径加载 YAML 规则。
   - 除非指定 `--no-feed-pull`，否则调用 B4 (GitStateDriver) 从 publish-registry 仓库拉取最新的 `curation-feed.yaml`，提取失败模式并转换为动态规则，合并到规则集中。
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

M6 是 7 个模块里**唯一真正复杂**的——带 FSM、长生命周期、跨进程重启、跨机器接力、async 远端编排。

**实验目录用户视图**：

```
my-experiment/
  ├── mission.md              # 必填：任务配置，用户主要编辑
  ├── curation-policy.yaml    # 可选：规则配置，生产 trace 周期筛选时使用
  ├── curation-rules/         # 可选：本实验规则；否则走组织默认 / registry feed
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
kweaver trace exp watch  [folder]                         # 只读：tail events.jsonl + 拉 M3 拼当前进度视图
kweaver trace exp abort  [folder]                         # 写 abort 信号；driver 在下一个 checkpoint 优雅退出
kweaver trace exp list   [path...]                        # 扫 path 下所有实验文件夹列状态
```

#### 3.3.2 代码模块布局（两层）

底层是 `experiment-folder/` 文件 IO 抽象（**所有对实验文件夹的读写必须经过它**），上层是 `exp/` 编排逻辑：

```
src/commands/trace/exp.ts                       # 参数解析 + dispatch（5 个子命令）

src/trace-core/experiment-folder/
  ├── paths.ts                # canonical 路径解析
  ├── mission-md.ts           # mission.md 解析（YAML frontmatter + body fallback queries 段）
  ├── preflight.ts            # mission / eval-set / provider capability 启动前校验
  ├── events-jsonl.ts         # append-only 写 + replay 读（FSM 真源）
  ├── trial-forest-yaml.ts    # 派生关系拓扑 yaml（快照式覆写）
  ├── jobs-jsonl.ts           # 远端 job_id 流水（append-only）
  ├── round-yaml.ts           # rounds/round-N.yaml 读写
  ├── lock.ts                 # 心跳锁协议（hostname+pid+last_heartbeat_ts，30s 超时）
  └── abort-signal.ts         # .trace-state/abort.signal 文件读写

src/trace-core/exp/
  ├── coordinator.ts          # FSM driver 主循环（元控制层）
  ├── fsm.ts                  # 6+1 状态枚举 + 转移表 + checkpoint 钩子
  ├── trial-forest-ops.ts     # Trial Forest 拓扑增删改逻辑
  ├── generator.ts            # thin wrapper：调 Agent Synthesizer provider
  ├── executor.ts             # K×M 调度（执行层；B2 async + jobs.jsonl）
  ├── scorer.ts               # thin wrapper：调 kweaver-eval + 三轴合成 + safety hard gate
  ├── triage.ts               # thin wrapper：调远端 Triage Agent + 跨轮记忆 binding
  ├── termination.ts          # 本地 Termination Decider
  ├── git-checkpoint.ts       # Round-内 local commit / Round-末 push 编排
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
| `bundle_ready` | Publishing 输出 bundle 后 | `bundle_id` / `bundle_hash` / `manifest_hash` | 供 M8 provenance 抽取 |
| `abort_requested` | abort.signal 被检测到 | `requested_at` / `requested_by` | 触发后续 `state_transition(to=Aborted)` |
| `abort_cleared` | resume 删除 abort.signal 后 | `cleared_at` / `cleared_by` | 审计用，不单独改变 FSM state |
| `lock_stolen` | 过期 lock 被接管 | `previous_actor` / `heartbeat_age_sec` | 审计用，不改变业务状态 |
| `synthesizer_capability_missing` | Generating 前置检查失败 | `provider` / `missing_capability` / `required_version?` | fail-fast，等待 operator 安装 / 升级后 resume |

**写入规则**：

1. append 必须使用单行原子追加；写失败则当前 tick 失败并重试，不允许先改快照再补 journal。
2. `event_id` 必须全局唯一；replay 遇到重复 `event_id` 只取第一条，保证崩溃重试幂等。
3. replay 遇到坏行：默认 fail-fast，提示 `kweaver trace exp doctor`（post-MVP）或人工修复；`watch/list` 可跳过坏行并标红。
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

`predicted_fixes` / `predicted_risks` 是 manifest 对账的前置字段；缺失时该 Trial 可用于探索，但不能进入 M8 publish bundle。

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

**watch 边界**：

- **不抢 lock、不写任何文件**——只读 events.jsonl + jobs.jsonl（fs.watch 或 100ms polling fallback for cross-platform）+ B1 拉对应 round 的 trace 拼当前进度
- 多人同时 watch 互不干扰
- watch 的 UI 用 ink，三栏：FSM 状态时间轴 / 当前 round 的 K×M 矩阵进度 / 最近 5 行 events

**list 边界**：

- 不抢 lock。扫 `path...` 下含 `.trace-state/` 的子目录 → 对每个读最后 N 行 events.jsonl / jobs.jsonl + lock.json → 输出表格（experiment / state / round / pending-jobs / driver-host / heartbeat-age）

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

### 3.5 M8 Publish Registry — `kweaver trace bundle`

**命令树**（vision §7.M8）：

```
kweaver trace bundle submit <experiment-folder> [--registry=<git-url>]         # 强制 manifest 校验 → commit 到 publish-registry
kweaver trace bundle show   <bundle-id>         [--registry=<git-url>]         # git show + 解析（便利包装）
kweaver trace bundle list   [--experiment-id=<id>] [--registry=<git-url>]      # git log/目录扫描（便利包装）
```

**代码模块布局**：

```
src/commands/trace/bundle.ts                # dispatch
src/trace-core/bundle/
  ├── submit.ts              # 主流程 = 读 experiment outputs/ → schema 校验 → 出处证据 → commit + push
  ├── schema-check.ts        # 用 B5 校验 bundle.yaml + manifest.yaml；缺 manifest 立刻 reject
  ├── provenance.ts          # 从 events.jsonl 抽 trace_id 列表 / round / triage 报告引用 → provenance.yaml
  ├── registry-driver.ts     # 解析 publish-registry URL（--registry + mission.md + kweaver config）→ clone-or-pull → 写 bundles/<id>/* → push
  ├── show.ts                # git show + yaml 解析
  └── list.ts                # git log --grep / 目录扫描
```

**依赖**：

- **B3 ExperimentFolder** — 读 `outputs/bundle.yaml`、`outputs/manifest.yaml`、`.trace-state/events.jsonl`
- **B4 GitStateDriver** — clone-or-pull publish-registry repo / commit / push
- **B5 SchemaRegistry** — bundle.yaml / manifest.yaml 校验；**manifest 缺失或不合规 → CLI 立刻 reject**（vision §7.M8 强制约束）
- **kweaver config**（既有）— 解析 `trace.publish-registry-url`

**关键设计点**：

1. **submit 是事务性的**：先在临时目录 build 完 bundle 全套（bundle.yaml / manifest.yaml / provenance.yaml）→ schema 校验全过 → 才进 commit + push。任何一步失败立刻退出，不留半成品。
2. **bundle-id 由 CLI 算定，不由远端分配**：`<experiment-id>-<short-sha-of-bundle.yaml>`。submit 是 idempotent——重复 submit 同样的 bundle 命中同一目录，git 自然 dedup。
3. **show / list 是 git 协议的便利壳**：内部直接 spawn `git show` / `git log`，不走任何 service。read 路径走 git 协议、submit 路径走 CLI，没有 read service。
4. **未来 binary asset（prompt 模板 / retrieval 索引）走 git LFS** —— vision §9.5 已留口子；MVP 不实现 LFS 集成，bundle.yaml 只引 reference。

**核心流程与逻辑（以 `bundle submit` 为例）**：

1. **前置校验**：
   - 读取实验目录下的 `outputs/bundle.yaml` 和 `outputs/manifest.yaml`。
   - 调用 B5 (SchemaRegistry) 进行严苛校验。如果缺少 `manifest.yaml` 或内容不合规，**立刻拒绝（Reject）**并退出。
2. **构建发布包（Bundle Building）**：
   - 在本地临时目录中，将 `bundle.yaml`、`manifest.yaml` 归档。
   - 调用 `provenance.ts`，从 `.trace-state/events.jsonl` 中抽取支撑该 Trial 的 Trace ID 列表、Round 轮次、以及 Triage 报告引用，生成 `provenance.yaml`（出处证据）。
3. **计算 ID 与事务性准备**：
   - 根据 `<experiment-id>-<short-sha-of-bundle.yaml>` 计算出全局唯一的 `bundle-id`。
   - 调用 B4 (GitStateDriver) 将 publish-registry 仓库 `clone-or-pull` 到本地。
4. **并发控制与提交**：
   - 检查 `bundles/<bundle-id>/` 目录是否已存在。若存在且内容不同，报 `BUNDLE_ID_COLLISION`。
   - 将临时目录中的文件写入 `bundles/<bundle-id>/`。
   - 执行 `git add` 和 `commit`。
   - 执行 `push`。若遇到冲突，自动进行 `pull --rebase` 并重试，默认最多 3 次（见下方并发协议）。

**registry 写入并发协议**：

1. `registry-driver` 每次写入前执行 `clone-or-pull --rebase`，写入后 `commit`，再 `push`。
2. `push` 被 remote 拒绝时，自动 `pull --rebase` 并重试，默认最多 3 次；仍冲突则退出 `REGISTRY_PUSH_CONFLICT`，不做自动 merge。
3. `bundles/<bundle-id>/bundle.yaml` / `manifest.yaml` / `provenance.yaml` 是不可变文件。若目录已存在且三者 hash 完全一致，submit 直接返回 success；若任一 hash 不同，拒绝覆盖并报 `BUNDLE_ID_COLLISION`。
4. `verifications/*.yaml` 是追加型目录，文件名必须含 timestamp + short hash，避免 M9 并发写同名文件。
5. `curation-feed.yaml` 是可合并文件，M9 写入时按 `feed_item_id` 做集合并；自动合并失败则写 `curation-feed.pending-<ts>.yaml` 并提示人工合并，避免覆盖别人刚写入的失败模式。

### 3.6 M9 Post-deploy Verify — `kweaver trace verify`

**命令树**（vision §7.M9，砍 CronJob 后纯 CLI）：

```
kweaver trace verify check <bundle-id> [--registry=<git-url>]     # 单 bundle 对账：拉生产 trace + 读 manifest + 写 verification.yaml + 抽 curation-feed
kweaver trace verify scan  [--registry=<git-url>]        # 扫整个 publish-registry，对到 cadence 时刻的 bundle 批量触发 check
```

**代码模块布局**：

```
src/commands/trace/verify.ts                # dispatch
src/trace-core/verify/
  ├── cadence.ts             # 读 publish-registry/bundles/*/deployed_at.yaml → 决定哪些 bundle 到 1h/24h/1w/1m 对账时刻
  ├── check.ts               # 单 bundle 对账主流程（被 scan 批量调，也被 check 直接调）
  ├── manifest-reconcile.ts  # 算 predicted_fixes 命中率 / predicted_risks 出现率（参考 AHE 阈值）
  ├── failure-extract.ts     # 新失败模式抽取入 publish-registry/bundles/<id>/curation-feed.yaml（飞轮闭合）
  ├── scan.ts                # 批量编排
  └── report.ts              # verification.yaml 序列化（含命中率 / 偏离指标 / 回退建议）
```

**依赖**：

- **B1 ObservabilityClient** — 拉关联生产 trace（按 bundle_id / agent_id / 时间窗）
- **B4 GitStateDriver** — clone-or-pull publish-registry / 写 verifications/<ts>.yaml + curation-feed.yaml / commit + push
- **B5 SchemaRegistry** — verification.yaml schema 校验
- **kweaver-sdk 现有 BKN api client** — 不变量校验沿用 M4 同一条路径

**外部触发：CI workflow（vision §7.M9 新形态）**：

publish-registry repo 自带 `.github/workflows/verify.yml`：

```yaml
on:
  schedule: { cron: '0 * * * *' }
  workflow_dispatch:
jobs:
  verify:
    steps:
      - uses: actions/checkout@v4
      - run: npm install -g @kweaver-ai/kweaver-sdk
      - run: kweaver trace verify scan
        env:
          KWEAVER_BASE_URL: ${{ secrets.KWEAVER_BASE_URL }}
          KWEAVER_TOKEN:    ${{ secrets.KWEAVER_TOKEN }}
```

M9 spec 提供这个 yaml 作为 reference impl，由 publish-registry 仓库管理员落地。`verify scan` 内部负责 commit + push；workflow 不再额外 `git push`，避免职责重复。

**关键设计点**：

1. **scan 与 check 共享 H2 主流程** —— `check <id>` 是直接调用；`scan` 是 cadence 过滤 + 批量编排再调。两条命令都给用户暴露：`scan` 给 CI，`check` 给操作员手动盯。
2. **AHE 阈值由 manifest.yaml 自身声明**——每个 bundle 自带阈值（不同任务 acceptable hit-rate 不同）；CLI 不硬编码全局阈值。
3. **curation-feed.yaml 是飞轮唯一闭合点**——抽出的新失败模式 commit 到 publish-registry 后，下次任意人 `kweaver trace curate scan` 时通过 §3.1 的 feed-pickup 拾取。

**核心流程与逻辑（以 `verify check` 为例）**：

1. **资源加载与环境准备**：
   - 根据输入的 `bundle-id`，调用 B4 (GitStateDriver) 从 publish-registry 仓库中读取该 Bundle 的 `manifest.yaml`。
   - 提取 Manifest 中声明的预期修复（Predicted Fixes）和预期风险（Predicted Risks）。
2. **生产数据拉取**：
   - 调用 B1 (ObservabilityClient)，根据 `bundle_id` 或 `agent_id`，以及时间窗口，拉取该 Bundle 部署后的真实生产 Trace。
3. **对账与评估（Manifest Reconcile）**：
   - 调用 `manifest-reconcile.ts`。
   - 检查真实 Trace 中，原本应该被修复的错误是否还在发生（计算命中率）。
   - 检查原本预期的风险是否出现（计算出现率）。
4. **新失败模式抽取（飞轮闭合）**：
   - 调用 `failure-extract.ts`。如果发现生产 Trace 中出现了 Manifest 未记录的新失败模式（如未预期的死循环或特定报错）：
     - 将其格式化为 Curation Rule 格式，追加到该 Bundle 目录下的 `curation-feed.yaml` 中，完成数据飞轮的闭合。
5. **报告输出与提交**：
   - 调用 `report.ts` 生成 `verification.yaml` 报告，包含偏离指标和是否需要回退的建议。
   - 调用 B4 将报告 commit 并 push 回 publish-registry 的 `verifications/` 目录下。

### 3.7 MX1 Schema — `kweaver trace schema`

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
8. **git 并发 tests**：用临时 bare repo 模拟 registry push conflict、idempotent submit、verification 并发追加、curation-feed 合并冲突
9. **安全输出 tests**：默认输出不得包含 prompt / tool result 全文；`--json` 与 pretty 路径同样遵守 redaction；`--unsafe-full` 需要显式开关

### 4.2 发布策略

- **整体跟 kweaver-sdk 现有节奏**：不独立发版；trace 子命名空间作为 kweaver-sdk 新 minor 上线
- **MVP 顺序**（建议，非强约束）：MX1 + B1/B5 → M4 → M5 → M7 → M6 → M8 → M9
  - 理由：先把 schema 静态契约 + M3 客户端 + 单文件校验立起来；再做无状态的 curate / eval-set；M7 replay 可直接吃现有真实 trace，较早验证 B1 + B2 + DA replay 契约，也能暴露 trace 上下文是否足够重放；然后进入最大头 M6；M8 在 M6 之后消费 outputs/；M9 最后依赖 M8 已有 bundle 进 registry + publish-registry CI workflow 就绪。
- **每个 minor 都跑 schema-mirrors.lock CI lint**——drift 即红
- **MVP 期 B2 RemoteJobClient 内置 `--sync` 降级开关**给 dev 用；正式实验路径强制 async（vision §6.4.4 钉死方向）

### 4.3 文档

- `docs/cli_conventions.md`：现有文档 §1–§7 不动；本设计完成后追加 §8 trace 子命名空间约定（指向本文档）
- 每个 trace 子命令的 `--help` 文本：包含命令字面值 + 一条 example + 链接到本文档对应章节
- README / `kweaver --help` 顶层命令树更新（加 `kweaver trace ...` 一行）

## §5 对 vision 的 patch 清单

> **状态**：P1–P15 已于 2026-05-09 全部应用到 `vision/trace-ai-continuous-learning-design.md`。本节作为变更记录保留，便于评审时对照"本设计触发了 vision 哪些位置的改动"。

本设计的成形过程中，发现 vision 文档以下位置需要更新。所有改动都是**措辞 / 路径 / 字面值层面 + 少量遗漏点澄清**，不动 vision 的设计判断。

| # | vision 段落 | 改动 |
|---|---|---|
| **P1** | §6.4.1 物理形态总表 + 净效果段 | 删 MX1 / M9 两行 CronJob；CronJob 列净效果改为 0；CLI 子命名空间改为 7 个 `kweaver trace`；新增 "周期性触发由相关 git 仓库各自的 CI 定时 workflow 承担" |
| **P2** | §7.M4–M9 各 "**路径**" 字段 | `trace-ai/cli/{...}/` → `kweaver-sdk/packages/typescript/src/commands/trace/{...}/` + `src/trace-core/{...}/` |
| **P3** | §7.M4–M9 "**入口契约**" 字面命令 | `trace …` → `kweaver trace …` |
| **P4** | §7.MX1 形态与路径 | 形态：从"git 化静态契约 + CronJob 校验器"改为"git 化静态契约 + CLI 校验器 + CI 定时 workflow"；删除 `trace-ai/charts/schema-guard/`；新增 `trace-ai/.github/workflows/schema-audit.yml`；CLI 路径 `kweaver-sdk/.../src/commands/trace/schema/` + `src/trace-core/schema/` |
| **P5** | §7.M9 形态 | 从 "CronJob + CLI" 改为 "CLI"；删除 `trace-ai/charts/post-deploy-verify/`；新增 `publish-registry/.github/workflows/verify.yml`（M9 spec 提供 reference impl） |
| **P6** | §7.MX1 schema mirror 同步机制 | 新增段落：schema 静态契约 SSOT 在 `trace-ai/schema/v1/`；kweaver-sdk monorepo 持有 mirror 副本于 `packages/typescript/src/trace-core/schema/v1/`，由 trace-ai 维护者在 schema PR 里同步推送，并附 `packages/typescript/schema-mirrors.lock` 记录同步源 SHA + CI lint 校验一致性 |
| **P7** | §6.4 新增 §6.4.7 | publish-registry URL 解析：默认走 `kweaver config trace.publish-registry-url`（per-profile）；mission.md frontmatter 可声明覆盖；无 experiment-folder 上下文的命令可用 `--registry` 覆盖；优先级 `--registry` > mission.md > kweaver config > 报错 |
| **P8** | §7.M6 内部子组件段末尾 | 增补："研判层 3 件（Generator / Scorer / Triage Agent）在 CLI 进程内是 thin wrapper——智能决策发生在远端智能体层，CLI 只做调用编排、结果合成、与 FSM 的 binding；元控制层 2 件（Coordinator / Termination Decider）才是真正本地逻辑" |
| **P9** | §7.M7 "重放本身产 trace 喂回 M1" 措辞 | 改写：远端 DA 在 replay 时正常打 OTel；CLI 只在 replay 请求里带 `replay_of=<原 trace_id>` attribute，由 DA 标进新 trace 的 root span |
| **P10** | 附录 A 术语迁移表 | 加注：`kweaver agent trace <conv_id>`（v0.7.4 已上线，agent 资源的 trace 动作）vs `kweaver trace …`（本设计新增的 trace-ai 子命名空间）；加 "trace-ai/cli/* → kweaver-sdk/packages/typescript/src/commands/trace/*" 仓库物件演进记录 |
| **P11** | §6.4.5 已知反方意见 | 新增 (d)："CI runner 距离 M2 与凭证管理（GitHub Secrets / 等价物）是 above-L0 周期性触发的设计内在约束；audit 需保持抽样而非全量；这是新设计的内在约束，不是回退选项" |
| **P12** | §7.4.1 服务调用矩阵 | 删 MX1 CronJob / M9 CronJob 两行；M9 CronJob 行合并到 M9 CLI；脚注说明触发源换成 git CI workflow |
| **P13** | §6.4.3 abort 协议措辞 | 当前写"`trace exp abort` 写终止意图到 lock.json"；改为：写独立文件 `.trace-state/abort.signal`；Coordinator 在 FSM checkpoint 间 poll；语义清晰、lock 协议保持纯净 |
| **P14** | §7.M6 state 真源描述 | 当前并列 "events.jsonl + trial-forest.yaml + jobs.jsonl + rounds/round-N.yaml + lock.json"；改为分主次："events.jsonl 是 FSM 真源；jobs.jsonl 只记录远端 job transport 生命周期；trial-forest.yaml 与 rounds/round-N.yaml 是派生快照；lock.json / abort.signal 是运行期控制文件" |
| **P15** | §9.3 安全与隐私 | 新增一句："redaction rules 是 trace-ai schema 之外的契约（脱敏 pattern 是企业敏感信息），由组织自建于 `<repo>/redaction-rules/`，不进 schema/v1/" |

---

## 附录 A：trace-core/ 目录全景

为方便实现期参考，集中列出 trace-core/ 下的所有目录与文件归属：

```
src/trace-core/
  ├── remote-job.ts                       # B2 共享：async submit + poll
  ├── git-state.ts                        # B4 共享：git CLI 包装

  ├── experiment-folder/                  # B3 共享：portable folder 抽象（M6 独占；M8 read-only）
  │   ├── paths.ts
  │   ├── mission-md.ts
  │   ├── events-jsonl.ts
  │   ├── trial-forest-yaml.ts
  │   ├── jobs-jsonl.ts
  │   ├── round-yaml.ts
  │   ├── lock.ts
  │   └── abort-signal.ts

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

  ├── exp/                                # M6 专属
  │   ├── coordinator.ts
  │   ├── fsm.ts
  │   ├── trial-forest-ops.ts
  │   ├── generator.ts
  │   ├── executor.ts
  │   ├── scorer.ts
  │   ├── triage.ts
  │   ├── termination.ts
  │   ├── git-checkpoint.ts
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

  ├── bundle/                             # M8 专属
  │   ├── submit.ts
  │   ├── schema-check.ts
  │   ├── provenance.ts
  │   ├── registry-driver.ts
  │   ├── show.ts
  │   └── list.ts

  └── verify/                             # M9 专属
      ├── cadence.ts
      ├── check.ts
      ├── manifest-reconcile.ts
      ├── failure-extract.ts
      ├── scan.ts
      └── report.ts

src/commands/trace/                       # 各 M 模块的命令入口（薄层；只做参数解析 + dispatch 到 trace-core）
  ├── exp.ts
  ├── curate.ts
  ├── eval-set.ts
  ├── replay.ts
  ├── bundle.ts
  ├── verify.ts
  └── schema.ts

src/api/trace/                            # B1 共享：M3 HTTP 客户端
  └── observability.ts

src/ui/trace/                             # B6 共享：trace 子命名空间专属输出格式器
  └── ...
```

## 附录 B：依赖矩阵（CLI 视角）

各 M 模块对共享层的依赖（行：M 模块；列：B 共享组件）：

| | B1 Obs | B2 RemoteJob | B3 ExpFolder | B4 GitState | B5 SchemaReg | B6 Output | BKN api（既有） | kweaver-core skill/CLI |
|---|---|---|---|---|---|---|---|---|
| **M4 curate** | ✓ 读 trace | | | ✓ 读 publish-registry | ✓ 校 rules | ✓ | ✓ 反查不变量 | |
| **M5 eval-set** | ✓ 圈 query | ✓ relabel | | | ✓ 校 eval-set | ✓ | | |
| **M6 exp** | ✓ watch 拉 trace | ✓ Agent Synthesizer / Executor / Scorer / Triage async | ✓ 独占写 | ✓ commit/push | ✓ 校 Trial proposal | ✓ | | ✓ synthesis 写入 / 校验平台资产 |
| **M7 replay** | ✓ 拉新旧 trace | ✓ DA replay | | | | ✓ | | |
| **M8 bundle** | | | ✓ 读 outputs/events.jsonl | ✓ commit/push registry | ✓ 强制 manifest 校验 | ✓ | | |
| **M9 verify** | ✓ 拉生产 trace | | | ✓ 读写 registry | ✓ 校 verification | ✓ | ✓ 不变量 | |
| **MX1 schema** | ✓ audit 抽样 | | | | self | ✓ | | |
