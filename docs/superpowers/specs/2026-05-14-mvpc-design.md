# MVP-C 设计文档：单路径迭代实验循环

**日期：** 2026-05-14  
**状态：** 已审阅  
**前置：** MVP-A（trace diagnose）、MVP-B（eval-set build/test）

---

## 1. 目标与范围

MVP-C 实现 `kweaver trace exp` 命令族，让用户沿单条修改路径（baseline → v1 → v2 → …）对 Agent 候选进行持续迭代评估，最终产出可追溯的 bundle/manifest/provenance 产物。

**在范围内：**
- `exp run / resume / show / status / abort / doctor` 六条命令
- FSM 驱动的单路径迭代循环（7 状态）
- ExpStore（B3）：实验文件夹 + lockfile 协议 + 状态文件管理
- Synthesizer（生成 next_change）和 Triage（诊断 round 结果）双 provider 支持
- EvalRunner 复用 MVP-B eval-set test 流程
- 三轴打分（Outcome / Trajectory / Guardrail）
- 实验目录 README 自动生成

**不在范围内（post-MVP）：**
- Trial Forest / 多路径并行
- GitCheckpointDriver（自动 git commit）
- `exp watch / list` 命令
- 真实 DA 接口的 E2E 测试
- 分布式锁

---

## 2. 整体架构

```
CLI 命令层（薄壳）
  exp run / resume / show / status / abort / doctor
        │
        ▼
ExperimentCoordinator（核心）
  ├── 持有 FSM 当前状态
  ├── 驱动状态推进（每步 → 写 events.jsonl）
  ├── 管理心跳锁（每 10s 更新 lock.json）
  └── 检测 abort.signal（每步前检查）
        │
        ├── ExpStore（B3）—— 所有文件 I/O
        ├── SynthesizerClient —— 生成 next_change
        ├── EvalRunner —— 复用 MVP-B eval-set test 流程
        ├── TriageClient —— 诊断 round 结果
        └── BundleWriter —— 生成最终产物
```

### FSM 状态（7 个）

```
Init → Generating → Executing → Scoring → Triaging → Deciding → Publishing
                                                          ↑
                                          每轮完成后暂停，等 exp resume
```

| 状态 | 入口命令 | 说明 |
|------|---------|------|
| Init | `exp run`（冷启） | acquireLock → initDir → replayState |
| Generating | `exp run`（新轮）/ `exp resume` | SynthesizerClient 生成 next_change |
| Executing | 自动 | EvalRunner 跑 eval-set |
| Scoring | 自动 | 本地计算三轴分数 |
| Triaging | 自动 | TriageClient 诊断 round 结果 |
| Deciding | 自动 | 暂停等用户决策；verdict=publish 或达到 max_rounds 则跳 Publishing |
| Publishing | `exp resume`（最终轮）/ verdict 自动触发 | BundleWriter 写出产物 |

### Provider 配置层级

```yaml
# ~/.kweaver/config.yaml（全局默认）
exp:
  default_provider: claude-code   # claude-code | decision-agent

# mission.md（覆盖全局）
provider: decision-agent
```

---

## 3. ExpStore（B3）与状态文件管理

所有对 `.trace-state/` 的读写必须经过 ExpStore，Coordinator 和命令层不直接操作文件系统。

### 实验目录结构

```
my-experiment/
  ├── README.md                     # 自动生成，目录地图 + 命令速查
  ├── mission.md                    # 用户编辑，Synthesizer 覆写 next_change
  ├── eval-sets/                    # 预置或 MVP-B 生成
  ├── candidates/
  │   └── baseline.yaml
  ├── outputs/                      # 最终产物（Publishing 阶段写出）
  │   ├── bundle.yaml
  │   ├── manifest.yaml
  │   └── provenance.yaml
  └── .trace-state/
      ├── events.jsonl              # 状态转换日志（append-only，真源）
      ├── jobs.jsonl                # 远端 job_id 流水（async poll 用）
      ├── candidate-lineage.yaml    # baseline → v1 → v2 快照链
      ├── lock.json                 # cooperative 心跳锁
      ├── abort.signal              # 优雅中止信号（存在即中止）
      └── rounds/
          ├── round-1.yaml
          └── round-2.yaml
```

### README.md 模板（Init 时自动生成）

```markdown
# Experiment: exp_<id>

Created: <timestamp>  Goal: <mission.goal>

## 目录说明
- mission.md        — 实验意图（你来编辑）
- eval-sets/        — 评测集（来自 MVP-B 或手动预置）
- candidates/       — Agent 候选快照
- outputs/          — 最终产物（bundle / manifest / provenance）
- .trace-state/     — 运行态，勿手动编辑

## 常用命令
exp run .           — 启动 / 新开一轮
exp resume .        — 从 Deciding 状态继续
exp show .          — 查看当前状态和建议
exp status .        — 一行摘要（适合脚本）
exp abort .         — 优雅中止
exp doctor .        — 环境自检
```

### ExpStore 关键方法

| 方法 | 读/写 | 说明 |
|------|-------|------|
| `initDir(mission)` | 写 | 创建目录骨架 + 写 README.md（已存在则跳过） |
| `appendEvent(event)` | 写 | append events.jsonl，唯一状态推进入口 |
| `replayState()` | 读 | fold events.jsonl → 当前 FSM 状态快照 |
| `acquireLock()` | 写 | 写 lock.json；心跳超时 >30s 可强夺 |
| `releaseLock()` | 写 | 删除 lock.json |
| `heartbeat()` | 写 | 每 10s 更新 lock.json 的 `last_heartbeat_ts` |
| `isAborted()` | 读 | 检查 abort.signal 是否存在 |
| `writeRound(n, data)` | 写 | 写 rounds/round-N.yaml |
| `readMission()` | 读 | 解析 mission.md frontmatter |
| `writeSuggestedChange(change)` | 写 | 覆写 mission.md 的 `next_change` 字段 |
| `appendLineage(snapshot)` | 写 | candidate-lineage.yaml 追加新快照 |

### Lock 协议

- `exp run` 启动时 `acquireLock()`，进程退出前 `releaseLock()`
- Coordinator 主循环每 10s 调 `heartbeat()`
- lock.json 已存在且 `now - last_heartbeat_ts < 30s` → 报错退出
- `>= 30s` → 强夺锁（视上一个进程为僵死）
- `exp show / status / doctor` 不获取锁（只读）
- Deciding 暂停时 `releaseLock()`，`exp resume` 重新 `acquireLock()`

### events.jsonl 事件类型

```jsonl
{"ts":"...","type":"state_transition","from":"Init","to":"Generating","round":1}
{"ts":"...","type":"state_transition","from":"Generating","to":"Executing","round":1}
{"ts":"...","type":"round_completed","round":1,"verdict":"continue"}
{"ts":"...","type":"state_transition","from":"Deciding","to":"Generating","round":2}
{"ts":"...","type":"step_failed","state":"Generating","error":"...","retryable":true}
{"ts":"...","type":"aborted","round":2,"reason":"user_abort"}
```

---

## 4. 数据流：单轮迭代循环

```
Init
  acquireLock() → initDir() → replayState()
  若 events.jsonl 已有记录 → 跳转到当前状态（resume 路径）
      │
      ▼ round = last_round + 1
Generating
  SynthesizerClient.generate(mission, prev_round?)
  → next_change: {target, hypothesis, patch}
  → writeSuggestedChange()  覆写 mission.md
  → appendLineage(candidate_snapshot)
      │
      ▼
Executing
  EvalRunner.run(eval_sets, candidate)
  复用 MVP-B eval-set test 流程
  → per-query execution results + raw scores
      │
      ▼
Scoring
  本地计算三轴分数（Outcome / Trajectory / Guardrail）
  Guardrail hard gate 违反 → Trial 淘汰，跳 Deciding
  → writeRound(n, {scores, per_query_results})
      │
      ▼
Triaging
  TriageClient.triage(round_n_scores, prev_rounds)
  → {diagnoses, hints, verdict: continue | publish}
  → writeRound(n, {triage_conclusion})
      │
      ▼
Deciding
  verdict == publish  → 跳 Publishing
  round == max_rounds → 跳 Publishing
  否则：releaseLock()，打印 exp show 摘要，暂停等 exp resume
      │
      │ exp resume
      ▼
Publishing
  BundleWriter.write(candidate_lineage, all_rounds)
  → outputs/bundle.yaml
  → outputs/manifest.yaml（predicted_fixes / risks）
  → outputs/provenance.yaml
  releaseLock()，打印产物路径
```

**abort 检测**：每个状态开始前 `isAborted()` 检查一次，若 abort.signal 存在则立即退出并写 `{type:"aborted"}` 事件。

---

## 5. mission.md v1 Schema

```yaml
schema_version: trace-mission/v1
goal: "降低重复调用 tool 的概率"
max_rounds: 5                           # 可选，默认不限
provider: claude-code                   # 可选，覆盖全局默认

eval_sets:
  - path: eval-sets/customer-v1/
    role: seed

current_candidate:
  path: candidates/baseline.yaml

next_change:                            # Synthesizer 覆写此字段
  target: decision_agent.prompt
  hypothesis: "加 stop condition"
  patch: |
    ...
```

---

## 6. 错误处理与恢复

### 错误分类

| 类型 | 触发场景 | 处理 |
|------|---------|------|
| 可重试 | 网络超时、远端 provider 临时失败 | 指数退避重试 3 次（1s/2s/4s）；超限后写 `step_failed(retryable:true)` 退出，**不释放锁**；`exp resume` 从该状态重入 |
| 不可恢复 | schema 校验失败、Guardrail hard gate | 写 `step_failed(retryable:false)`，释放锁，打印原因；需用户修复后重新 `exp run` |
| 用户中止 | `exp abort` | 写 abort.signal；Coordinator 步前检测 → 写 `aborted` 事件 → 释放锁退出 |

### `exp resume` 幂等性

`replayState()` 重建状态时，若当前状态不是 Deciding（上次中途崩溃），从该状态重入；每个状态开始前检查 events.jsonl 是否已有该步骤的完成事件，避免重做。

### `exp doctor` 检查项

| 检查项 | 失败时提示 |
|--------|-----------|
| mission.md 存在且 schema 合法 | 指出缺失字段 |
| eval_sets 路径可读 | 列出找不到的路径 |
| candidate 路径可读 | 同上 |
| provider 可达（ping） | 给出配置建议 |
| lock.json 是否僵死（>30s 无心跳） | 提示可强夺或清理 |
| .trace-state/ 权限可写 | 提示 chmod |

---

## 7. 测试策略

### 单元测试

| 测试对象 | 策略 |
|---------|------|
| `ExpStore` | 真实临时目录，不 mock 文件系统 |
| `ExperimentCoordinator` FSM | mock ExpStore 接口，验证状态转换序列 |
| Lock 协议 | 注入假时钟，验证心跳超时强夺逻辑 |
| abort 检测 | Generating 中途写 abort.signal，验证正确退出 |
| `BundleWriter` | 固定输入，快照测试产物结构 |

### 集成测试

| 场景 | 做法 |
|------|------|
| 完整单轮循环 | mock Synthesizer + Triage，真实 ExpStore + EvalRunner（MVP-B fixture） |
| resume 恢复 | Executing 阶段人为杀进程，验证 resume 从正确状态重入 |
| max_rounds 终止 | 设 `max_rounds: 2`，验证第 2 轮后跳 Publishing |
| Guardrail hard gate | 注入违反 guardrail 的 score，验证 Trial 淘汰跳 Deciding |

### 测试文件布局

```
src/trace-ai/exp/
  __tests__/
    exp-store.test.ts
    coordinator.test.ts
    lock.test.ts
    bundle-writer.test.ts
    integration/
      full-round.test.ts
      resume.test.ts
```

---

## 8. 文件模块布局

```
src/trace-ai/exp/
  index.ts                  # 命令入口导出
  coordinator.ts            # ExperimentCoordinator（FSM 核心）
  exp-store/
    index.ts
    mission-md.ts           # mission.md 解析与写回
    events-jsonl.ts         # append + replay
    candidate-lineage-yaml.ts
    jobs-jsonl.ts
    round-yaml.ts
    lock.ts
    abort-signal.ts
    readme-template.ts      # README.md 模板渲染
  providers/
    synthesizer-client.ts   # SynthesizerClient 接口 + 两个 provider 实现
    triage-client.ts        # TriageClient 接口 + 两个 provider 实现
  eval-runner.ts            # 包装 MVP-B eval-set test
  bundle-writer.ts          # BundleWriter
  schemas.ts                # B5 扩展：trace-mission/v1, trace-bundle/v1, trace-manifest/v1
```

---

## 9. 前置依赖

| 前置 | 状态 |
|-----|------|
| MVP-A：agent-providers/claude-code provider | ✅ 已完成 |
| MVP-B：eval-set test 流程 + B5 schema | 🟡 PR #133 待 merge |
| B5 扩展：trace-mission/v1 等新 schema | 本 MVP-C 新增 |
| kweaver DA chat/completion API | ✅ 已通（可选 provider） |
