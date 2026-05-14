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
- Synthesizer（生成 next_change）和 Triage（诊断 round 结果）provider 接口（MVP-C 仅 ship claude-code）
- EvalRunner 复用 MVP-B eval-set test 流程
- 三轴打分（Outcome / Trajectory / Guardrail）
- 实验目录 README 自动生成

**不在范围内（post-MVP）：**
- Trial Forest / 多路径并行
- GitCheckpointDriver（自动 git commit）
- `exp watch / list` 命令
- decision-agent provider 实现（MVP-C 只有 claude-code + provider 接口 stub）
- Dolphin DSL patch（DSL 结构待专项研究）
- BKN patch（平台 API 写操作，需 rollback 设计）
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
| Generating | `exp resume` / 自动 | PatchApplier 应用 next_change → candidate-vN |
| Executing | 自动 | EvalRunner 跑 eval-set，收集轨迹摘要 |
| Scoring | 自动 | 本地计算三轴分数 |
| Triaging | 自动 | TriageClient 诊断 + SynthesizerClient 生成下一轮建议 |
| Deciding | 自动 | 暂停等用户决策；verdict=publish 或达到 max_rounds 则跳 Publishing |
| Publishing | `exp resume`（最终轮）/ verdict 自动触发 | BundleWriter 写出产物 |

### Provider 配置层级

```yaml
# ~/.kweaver/config.yaml（全局默认）
exp:
  default_provider: claude-code   # MVP-C 仅支持 claude-code；decision-agent 为 post-MVP

# mission.md（覆盖全局，post-MVP 生效）
# provider: decision-agent
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
Init（exp run 触发）
  acquireLock() → replayState()
  若 events.jsonl 存在且状态非终态（non Published/Aborted）
    → 报错退出，提示用户用 exp resume
  若终态（Published/Aborted）
    → 拒绝覆盖，提示用 --new-run 另起实验
      --new-run：将 .trace-state/ 归档为 .trace-state-archived-<timestamp>/，再重新开始
  若空（全新实验）
    → initDir()，创建目录骨架 + README.md
      │
      ▼ round = 1

────────────────── 每轮起点（exp resume 进入此处）──────────────────

Generating（Apply Phase）
  读 mission.md.next_change（首轮：用户手写；后续轮：上轮 Synthesizer 写入）
  PatchApplier.apply(current_candidate, next_change.patch)
    → 写出 candidates/candidate-v{N}.yaml
    → 更新 current_candidate 指针
  appendLineage({version: N, candidate_path, next_change, status: "running"})
      │
      ▼
Executing
  EvalRunner.run(eval_sets, candidates/candidate-v{N}.yaml)
  复用 MVP-B eval-set test 流程，每条 query 执行后：
    - 收集 kweaver trace（原始 span 仅本地存档，不传 LLM）
    - 提取轨迹摘要：tool_call_sequence / retry_count / latency_ms / error_codes
  → per-query: {assertion_results, trajectory_summary, raw_trace_id}
      │
      ▼
Scoring
  四层输入 → 三轴分数：
    [轨迹摘要]  tool_call_sequence / retry_count / latency_ms   → Trajectory 轴
    [断言结果]  assertion_results (contains/regex/semantic…)    → Outcome 轴
    [候选配置]  candidate-v{N}.yaml 快照                        → Guardrail 轴检查
    [声明不变量] mission.md guardrails 字段                     → Guardrail hard gate

  Guardrail hard gate 违反
    → updateLineage(N, {status: "guardrail_failed"})
    → writeRound(n, {guardrail_failed: true, scores})
    → 跳 Deciding（用户修改 candidate 或 next_change 后可 resume）
  正常 → updateLineage(N, {status: "scored"})
       → writeRound(n, {scores, per_query_results, trajectory_summaries})
      │
      ▼
Triaging
  输入（四层）：
    - round_n: {scores, per_query_results, trajectory_summaries}
    - prev_rounds: [{scores, triage_conclusion}]（历史轮摘要，不含原始 trace）
    - candidate_config: candidate-v{N}.yaml 快照
    - cross_round_memory_ref: 上轮 memory token（首轮为空）

  TriageClient.triage(上述四层)
  → {diagnoses, hints, verdict: continue | publish, new_memory_token}
  → writeRound(n, {triage_conclusion, cross_round_memory_ref: new_memory_token})

  若 verdict == continue：
    SynthesizerClient.generate({mission, candidate_config, round_n, prev_rounds, new_memory_token})
    → next_change: {target, hypothesis, patch}
    → writeSuggestedChange()  覆写 mission.md 的 next_change 字段（供用户 review）
      │
      ▼
Deciding
  verdict == publish  → 跳 Publishing
  round == max_rounds → 跳 Publishing
  否则：releaseLock()，打印 exp show 摘要（含 next_change 建议），暂停等 exp resume
      │
      │ exp resume → acquireLock() → 回到 Generating 开始下一轮
      ▼
Publishing
  BundleWriter.write(candidate_lineage, all_rounds)
  → outputs/bundle.yaml
  → outputs/manifest.yaml（predicted_fixes / risks）
  → outputs/provenance.yaml
  releaseLock()，打印产物路径
```

**abort 检测**：每个状态开始前 `isAborted()` 检查一次，若 abort.signal 存在则立即退出、释放锁，并写 `{type:"aborted"}` 事件。

### 分析与迭代的信息层汇总

| 信息层 | 来源 | 用于 | 备注 |
|--------|------|------|------|
| **执行轨迹摘要** | EvalRunner 从 kweaver trace 提取 | Trajectory 轴打分 / Triage 诊断 | 原始 span 仅本地存档；MVP-C provider 仅 claude-code |
| **断言结果** | eval-set assertion 本地计算 | Outcome 轴打分 / Triage 诊断 | 结构化，token 成本低 |
| **候选配置快照** | candidate YAML（prompt / skill / BKN） | Synthesizer 生成 patch / Guardrail 检查 | 每轮写入 lineage |
| **跨轮记忆 token** | 上轮 Triage 返回 new_memory_token | Synthesizer + Triage 跨轮上下文 | 避免重复探索同一方向 |
| **历史轮摘要** | rounds/round-N.yaml（scores + hints） | Triage 趋势判断 | 不含原始 trace，仅聚合分数 |
| **mission.md** | 用户编写 | goal / guardrails / next_change 建议 | Synthesizer 覆写 next_change 字段 |

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
  target: agent.system_prompt           # 见下方"迭代目标类型"
  hypothesis: "加 stop condition"
  patch: |
    ...
```

### 迭代目标类型（PatchApplier 类型派发）

MVP-C 的 `next_change.target` 限定为以下两类，PatchApplier 按 target 前缀派发到对应 handler：

| 类型 | target 前缀 | target 示例 | patch 语义 | MVP-C |
|------|------------|------------|-----------|-------|
| **Agent Config** | `agent.*` | `agent.system_prompt` / `agent.temperature` / `agent.model` | 本地 candidate YAML 字段替换/追加 | ✅ AgentConfigPatcher |
| **Skill** | `skill.*` | `skill.add` / `skill.remove` / `skill.swap` | 本地 candidate YAML 的 skills 列表修改 | ✅ SkillPatcher |
| Dolphin DSL | `agent.dsl.*` | `agent.dsl.flow` | DSL 结构未研究清楚，需专项设计 | post-MVP stub |
| BKN | `bkn.*` | `bkn.entity.stop_rule` | 平台 API 写操作，需 rollback 设计 | post-MVP stub |

**candidate YAML 结构（MVP-C 范围）：**
```yaml
candidate_version: v1
agent:
  model: claude-sonnet-4-6
  temperature: 0.3
  system_prompt: |
    ...
skills:
  - name: retrieval_v1
  - name: summarize_v2
```

PatchApplier 读取当前 candidate YAML，应用 next_change.patch（JSON Merge Patch 语义），写出 `candidates/candidate-v{N}.yaml`。

---

## 6. 错误处理与恢复

### 错误分类

| 类型 | 触发场景 | 处理 |
|------|---------|------|
| 可重试 | 网络超时、远端 provider 临时失败 | 指数退避重试 3 次（1s/2s/4s）；超限后写 `step_failed(retryable:true)`，**释放锁**退出；`exp resume` 重新获锁，从 `step_failed` 事件恢复，重入同一状态 |
| 不可恢复 | schema 校验失败 | 写 `step_failed(retryable:false)`，释放锁，打印原因；需用户修复后 `exp resume` 重入 |
| 用户中止 | `exp abort` | 写 abort.signal；Coordinator 步前检测 → 写 `aborted` 事件 → 释放锁退出 |

### `exp resume` 幂等性

`replayState()` fold events.jsonl 得出当前状态：
- 最后事件为 `state_transition` → 从该状态重入（含 Deciding 后恢复）
- 最后事件为 `step_failed(retryable:true)` → 从 failed 所在状态重入（重试）
- 最后事件为 `step_failed(retryable:false)` → 同上，重入后由用户确认修复

每个状态开始前检查 events.jsonl 是否已有该步骤的完成事件，避免重做（幂等保证）。

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
    synthesizer-client.ts   # SynthesizerClient 接口 + claude-code 实现（DA stub）
    triage-client.ts        # TriageClient 接口 + claude-code 实现（DA stub）
  patch/
    index.ts                # PatchApplier 类型派发入口
    agent-config.ts         # AgentConfigPatcher（agent.* target）
    skill.ts                # SkillPatcher（skill.* target）
    dsl-stub.ts             # Dolphin DSL stub（post-MVP）
    bkn-stub.ts             # BKN stub（post-MVP）
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
