---
title: trace exp 智能体发现增强 — 注册表 + list + info
status: draft
date: 2026-05-15
依赖: vision/trace-cli-detailed-design.md §3.3（M6 Experiment Engine）
增强范围: MVP-C（与 §3.3.1 命令树并列，不修改 FSM / ExpStore 契约）
---

## §0 背景与问题

`trace exp status/show/doctor` 都要求调用方已知工作区路径（`[folder]` 参数）。  
当一个智能体（或新开的终端）需要了解"现在的实验情况"时，必须经历以下探索过程：

```
1. kweaver trace exp status .              # 当前目录，可能不是实验目录
   → /Users/xupeng/lab/plan-traceai: Init (round 0)   # 错的

2. find ~ -name "exp-state.yaml" ...      # 扫文件系统，依赖文件名约定
   → 找不到（状态文件名不是 exp-state.yaml）

3. find ~ -newer /tmp -name "*.yaml"      # 更宽泛的扫描
   → 偶然命中 /Users/xupeng/lab/ht/exp/candidate.yaml

4. kweaver trace exp status /Users/xupeng/lab/ht/exp
   → Deciding (round 1)                   # 终于找到
```

这对人工操作已经很痛苦，对智能体更不可接受：路径依赖文件系统布局、探索步骤不确定、无法保证找到全部实验。

**根因**：CLI 没有对实验工作区的全局感知。每次 `run/resume` 只操作本地文件夹，不留任何注册记录。

---

## §1 解决方案概述

引入两个正交增强，覆盖同一痛点的不同侧面：

| 增强 | 解决什么 | 代价 |
|------|---------|------|
| **全局注册表**（§2） | `run/resume` 时自动记录工作区路径，建立"已知实验"集合 | `run/resume` 额外写一次 `~/.kweaver/exp-registry.json` |
| **`trace exp list`**（§3） | 不需要知道路径，列出所有已知实验及状态 | 注册表 reader + 批量 status fold |
| **`trace exp info [<dir>]`**（§4） | 一条命令输出"智能体需要的全部上下文"，路径可选 | 合并 status + show + doctor，加机器友好格式 |

三个增强一起落地，互相配合：`info` 依赖注册表来实现"无路径"调用；`list` 是注册表的表格视图；注册表是 `run/resume` 的副作用。

---

## §2 全局注册表

### 2.1 文件位置与格式

**位置**：`~/.kweaver/exp-registry.json`（与 `~/.kweaver/platforms/` 并列，属 CLI 全局状态）

```json
{
  "schema_version": "exp-registry/v1",
  "entries": [
    {
      "path": "/Users/xupeng/lab/ht/exp",
      "last_active_ts": "2026-05-14T19:40:22+08:00"
    },
    {
      "path": "/Users/xupeng/lab/foo/exp",
      "last_active_ts": "2026-05-10T11:00:00+08:00"
    }
  ]
}
```

字段约定：

| 字段 | 类型 | 说明 |
|------|------|------|
| `path` | string | 绝对路径，注册表的唯一键 |
| `last_active_ts` | ISO-8601 | `run/resume` 最后一次成功抢到 lock 的时间；注册表唯一的独家信息，不能从 `.trace-state/` 廉价算出 |

state / round / scores 等运行状态不缓存在注册表，`list` / `info` 始终从 `.trace-state/` 读 live 值，避免注册表内容腐烂。

### 2.2 写入时机与规则

**何时写**：`trace exp run` / `trace exp resume` 在成功抢到 lock（`lock.ts` 返回后）、进入 preflight 之前写入注册表。不在 preflight 失败时回滚——路径已存在即值得记录（下次 doctor 能找到它）。

**写入幂等**：按 `path` 去重，已存在则 upsert（更新 `last_active_ts`）；不存在则 append。

**写入失败不阻塞主流程**：注册表写入失败（磁盘满、权限问题）仅打印 warning，不中断 `run/resume`。注册表是可重建的辅助索引，不是实验真源。

**不写入的场景**：`show` / `status` / `doctor` / `abort` / `list` / `info` 均不修改注册表。

### 2.3 代码位置

```
~/.kweaver/exp-registry.json          # 运行时产物（不进 git）

src/trace-ai/exp-store/
  └── exp-registry.ts                 # 新增：注册表读写（upsert / list / prune）
```

`exp-registry.ts` 只依赖 `fs/promises`，不依赖 ExpStore 的其他模块。`run/resume` 的 dispatch 层（`src/commands/trace/exp.ts`）在 lock 成功后调用 `ExpRegistry.upsert(path, lastActiveTs)`。

---

## §3 `trace exp list`

### 3.1 调用形式

```bash
kweaver trace exp list                  # 列出注册表内所有实验（推荐，无需路径）
kweaver trace exp list ~/lab/ht/exp     # 指定单个路径（不查注册表，直接读该目录）
kweaver trace exp list ~/lab/           # 扫指定目录下的所有直接子目录（浅扫，非递归）
```

**参数缺省行为**：无参数时读注册表；有参数时跳过注册表，直接处理指定路径。两种来源的输出格式完全相同。

### 3.2 输出格式

```
WORKSPACE                          STATE      ROUND  OUTCOME  TRAJECTORY  LAST ACTIVE
/Users/xupeng/lab/ht/exp           Deciding       1     0.45        1.00  2026-05-14 19:40
/Users/xupeng/lab/foo/exp          Completed      3     0.82        0.95  2026-05-10 11:00
/Users/xupeng/lab/bar/exp          Aborted        2      —            —   2026-05-08 09:15
```

- `OUTCOME` / `TRAJECTORY` 取自最后一轮 `round-N.yaml` 的 `scores` 字段；未完成轮次显示 `—`。
- `LAST ACTIVE` 优先读注册表的 `last_active_ts`；若路径来自参数而非注册表，则读 `.trace-state/events.jsonl` 最后一行的 `ts`。
- 路径不存在或 `.trace-state/` 缺失时，该行显示 `(missing)` 而不是让整条命令失败。

**`--json` flag**：输出 JSON array，每项包含上述所有字段，供脚本消费。

### 3.3 与原有 post-MVP `list` 的关系

§3.3.1 命令树注释中 `list [path...]` 标注为 post-MVP，原计划扫 `path` 下所有实验文件夹。  
本 spec 将 `list` 提前到 MVP-C，功能调整为：**默认读注册表（无参），可选接受单路径参数**。扫多路径、深度扫描等功能保持 post-MVP。

---

## §4 `trace exp info [<dir>]`

### 4.1 设计目标

`info` 是专为智能体（和人工快速诊断）设计的**单入口上下文命令**。它把需要分别跑三条命令才能拼出的信息合并成一次输出：

| 信息类别 | 现在需要的命令 | `info` 是否覆盖 |
|---------|-------------|---------------|
| 当前 FSM 状态、round | `exp status` | ✓ |
| 分数、triage 摘要、下一步建议 | `exp show` | ✓ |
| 健康检查（mission/eval-set/provider） | `exp doctor` | ✓ 简化版 |
| 工作区路径 | 调用方必须已知 | ✓ 可从注册表自动解析 |

### 4.2 调用形式

```bash
kweaver trace exp info                  # 无路径：从注册表取最近活跃实验
kweaver trace exp info ~/lab/ht/exp    # 指定路径
```

**无路径时的解析规则**：

1. 读注册表，按 `last_active_ts` 降序取第一条。
2. 若注册表为空或路径不存在，exit 1 并提示用户先运行 `kweaver trace exp run <dir>`。
3. 若注册表有多条（>1），打印 `Using most recent: <path>` 到 stderr，正常输出到 stdout（不中断）。

### 4.3 输出格式

默认输出 YAML，结构如下：

```yaml
# kweaver trace exp info — /Users/xupeng/lab/ht/exp
workspace: /Users/xupeng/lab/ht/exp
state: Deciding
round: 1

scores:
  outcome: 0.45
  trajectory: 1.00
  guardrail: 1.00

suggested_next:
  target: agent.system_prompt
  hypothesis: >
    Q36-Q56的失败原因是输出表述与参考答案不一致，而非推理错误（trajectory=1.00确认
    推理链正确）。添加明确的输出格式规范和行业术语标准……

triage_summary: >
  All 16 failures are 'semantic_match' type — the agent is taking correct actions in
  the right order (trajectory=1.00) but producing outputs whose meaning diverges from
  expected answers. Failures cluster heavily in Q36–Q56 range.

lineage_versions: 2

health:
  mission_valid: true
  eval_set_valid: true
  candidate_readable: true
  provider_available: false   # claude-code not on PATH — LLM steps will be skipped
  no_step_failed: true
```

**`--json` flag**：等价结构的 JSON 输出，便于程序解析。

### 4.4 与 `show` / `status` / `doctor` 的关系

`info` 不替代这三条命令，它们各自仍保持完整语义：

- `status`：机器可读的状态快照，适合脚本轮询
- `show`：完整的配置+诊断视图，适合人工深阅读
- `doctor`：健康检查，适合 preflight 排障
- `info`：**智能体冷启动的第一条命令**，快速建立上下文后再决定是否需要 `show` 的完整输出

### 4.5 代码位置

```
src/commands/trace/exp.ts              # 新增 info 分支的参数解析 + dispatch

src/trace-ai/exp-engine/
  └── info.ts                          # 新增：组合 read-model + doctor health + format
```

`info.ts` 复用已有模块：

- `exp-store/read-model.ts` → 读 `ExperimentSnapshot`（state / round / scores / suggested_next / triage_summary / lineage）
- `exp-engine/doctor.ts` → 运行 health checks，提取 `{mission_valid, eval_set_valid, candidate_readable, provider_available, no_step_failed}`
- `exp-store/exp-registry.ts` → 无路径时的 workspace 解析

`info.ts` 不引入新的远端调用，完全本地只读，调用耗时应 < 200ms。

---

## §5 命令树变更（对 §3.3.1 的局部修订）

```
# MVP-C（原有）
kweaver trace exp run    [folder] --mode=single-path
kweaver trace exp resume [folder]
kweaver trace exp show   [folder]
kweaver trace exp status [folder]
kweaver trace exp abort  [folder]
kweaver trace exp doctor [folder]

# MVP-C（本 spec 新增）
kweaver trace exp list   [path]          # 无参数时读注册表；有参数时扫指定路径
kweaver trace exp info   [folder]        # 无参数时取注册表最近实验；智能体冷启动首选

# post-MVP（不变）
kweaver trace exp watch  [folder]
```

---

## §6 代码模块变更汇总

```diff
 src/commands/trace/exp.ts
+  info 分支（路径解析 + dispatch to info.ts）
+  list 分支（注册表 / 路径参数 + dispatch to list logic）

 src/trace-ai/exp-store/
+  exp-registry.ts               # 注册表读写（upsert / list / prune-stale）

 src/trace-ai/exp-engine/
+  info.ts                       # 组合 read-model + doctor + registry + format

 src/commands/trace/exp.ts（run/resume 分支）
+  在 lock 成功后调用 ExpRegistry.upsert(absPath, now())
```

**新文件数**：3（`exp-registry.ts` / `info.ts` / 对 `exp.ts` 的两处扩展可合并）  
**改动已有文件**：`exp.ts`（dispatch 扩展）、`run/resume` 路径（注册表写入 side-effect）

---

## §7 验收口径

| 场景 | 验收命令 | 期望结果 |
|------|---------|---------|
| 注册表写入 | `kweaver trace exp run ~/lab/ht/exp --mode=single-path`（任意状态） | `~/.kweaver/exp-registry.json` 出现该路径条目 |
| list 无参 | `kweaver trace exp list` | 列出注册表内所有实验，含 state / round / scores |
| list 有参 | `kweaver trace exp list ~/lab/ht/exp` | 同上，仅一行，不查注册表 |
| info 无参（有注册表） | `kweaver trace exp info` | 输出最近实验的 state + scores + suggested_next + health |
| info 有参 | `kweaver trace exp info ~/lab/ht/exp` | 同上，不查注册表 |
| info 无参（注册表空） | `kweaver trace exp info` | exit 1，提示先 run |
| 注册表写入失败 | 磁盘满时运行 `run` | warning 到 stderr，`run` 继续执行 |
| 路径已删除 | 注册表有条目但目录不存在 | `list` 显示 `(missing)`，`info` exit 1 with 明确错误 |
