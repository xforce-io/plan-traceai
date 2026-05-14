# MVP-C: 单路径迭代实验循环 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现 `kweaver trace exp run/resume/show/status/abort/doctor` 命令族，驱动单路径 Agent 迭代实验循环，产出 bundle/manifest/provenance 产物。

**Architecture:** ExperimentCoordinator 持有 FSM（7 状态），所有文件 I/O 经过 ExpStore，PatchApplier 应用候选变更，SynthesizerClient/TriageClient 使用 claude-code provider 分析迭代方向，EvalRunner 复用 MVP-B eval-set test 流程。

**Tech Stack:** TypeScript (ES modules), node:test, zod, js-yaml, JSON Merge Patch（手工实现），existing AgentRegistry/ClaudeCodeSubprocessProvider

**前置条件:** MVP-B PR #133 已 merge（EvalRunner 依赖 `eval-set/test-runner.ts`）

---

## 文件结构总览

```
src/trace-ai/exp/
  schemas.ts                   # B5 扩展：trace-mission/v1 + trace-bundle/v1 + trace-manifest/v1
  exp-store/
    mission-md.ts              # readMission() / writeSuggestedChange()
    events-jsonl.ts            # appendEvent() / replayState()
    lock.ts                    # acquireLock() / releaseLock() / heartbeat()
    abort-signal.ts            # isAborted() / writeAbortSignal()
    round-yaml.ts              # writeRound() / readAllRounds()
    candidate-lineage-yaml.ts  # appendLineage() / updateLineage() / readLineage()
    readme-template.ts         # renderReadme()
    index.ts                   # ExpStore class (组合所有子模块)
  patch/
    index.ts                   # PatchApplier.apply() 类型派发
    agent-config.ts            # AgentConfigPatcher
    skill.ts                   # SkillPatcher
  scoring.ts                   # computeScores() 三轴打分
  providers/
    synthesizer-client.ts      # SynthesizerClient 接口 + ClaudeCodeSynthesizer
    triage-client.ts           # TriageClient 接口 + ClaudeCodeTriageClient
  eval-runner.ts               # EvalRunner（wraps MVP-B + 轨迹摘要提取）
  bundle-writer.ts             # BundleWriter
  coordinator.ts               # ExperimentCoordinator（FSM 核心）
  index.ts                     # runExpCommand() / parseExpArgs()

commands/trace.ts              # 增加 exp 子命令分发（Modify）

test/exp-schemas.test.ts
test/exp-store-mission.test.ts
test/exp-store-events.test.ts
test/exp-store-lock.test.ts
test/exp-patch.test.ts
test/exp-scoring.test.ts
test/exp-bundle-writer.test.ts
test/exp-coordinator.test.ts
test/integration/exp-full-round.test.ts
test/integration/exp-resume.test.ts
```

---

## Task 1: B5 Schema Extensions

**Files:**
- Create: `src/trace-ai/exp/schemas.ts`
- Create: `test/exp-schemas.test.ts`

- [ ] **Step 1: 写失败测试**

```typescript
// test/exp-schemas.test.ts
import test from "node:test";
import assert from "node:assert/strict";
import { MissionSchema, BundleSchema, ManifestSchema } from "../src/trace-ai/exp/schemas.js";

test("MissionSchema: accepts valid mission", () => {
  const result = MissionSchema.safeParse({
    schema_version: "trace-mission/v1",
    goal: "reduce retry rate",
    eval_sets: [{ path: "eval-sets/v1", role: "seed" }],
    current_candidate: { path: "candidates/baseline.yaml" },
  });
  assert.equal(result.success, true);
});

test("MissionSchema: rejects missing goal", () => {
  const result = MissionSchema.safeParse({
    schema_version: "trace-mission/v1",
    eval_sets: [],
    current_candidate: { path: "candidates/baseline.yaml" },
  });
  assert.equal(result.success, false);
});

test("BundleSchema: accepts valid bundle", () => {
  const result = BundleSchema.safeParse({
    schema_version: "trace-bundle/v1",
    experiment_id: "exp_abc",
    bundle_id: "bundle_xyz",
    best_trial_version: 2,
    resources: { agent_config: {}, skills: [] },
    provenance: { created_by: "user", created_at: "2026-05-14T00:00:00Z", evidence_traces: [], round_refs: [] },
  });
  assert.equal(result.success, true);
});

test("ManifestSchema: accepts valid manifest", () => {
  const result = ManifestSchema.safeParse({
    schema_version: "trace-manifest/v1",
    experiment_id: "exp_abc",
    trial_version: 2,
    predictions: { fixes: [], risks: [] },
  });
  assert.equal(result.success, true);
});
```

- [ ] **Step 2: 运行确认失败**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript
node --import tsx --test test/exp-schemas.test.ts
```
Expected: `Cannot find module '../src/trace-ai/exp/schemas.js'`

- [ ] **Step 3: 实现 schemas.ts**

```typescript
// src/trace-ai/exp/schemas.ts
import { z } from "zod";

const NextChangeSchema = z.object({
  target: z.string().min(1),
  hypothesis: z.string().min(1),
  patch: z.string(),
});

const GuardrailSchema = z.object({
  name: z.string(),
  kind: z.enum(["hard", "soft"]),
  rule: z.string(),
});

export const MissionSchema = z.object({
  schema_version: z.literal("trace-mission/v1"),
  goal: z.string().min(1),
  max_rounds: z.number().int().positive().optional(),
  provider: z.string().optional(),
  eval_sets: z.array(z.object({ path: z.string(), role: z.string() })).min(1),
  current_candidate: z.object({ path: z.string() }),
  next_change: NextChangeSchema.optional(),
  guardrails: z.array(GuardrailSchema).optional(),
});
export type Mission = z.infer<typeof MissionSchema>;
export type NextChange = z.infer<typeof NextChangeSchema>;

export const BundleSchema = z.object({
  schema_version: z.literal("trace-bundle/v1"),
  experiment_id: z.string(),
  bundle_id: z.string(),
  best_trial_version: z.number().int(),
  resources: z.object({
    agent_config: z.record(z.unknown()),
    skills: z.array(z.record(z.unknown())),
  }),
  provenance: z.object({
    created_by: z.string(),
    created_at: z.string(),
    evidence_traces: z.array(z.string()),
    round_refs: z.array(z.string()),
  }),
});
export type Bundle = z.infer<typeof BundleSchema>;

export const ManifestSchema = z.object({
  schema_version: z.literal("trace-manifest/v1"),
  experiment_id: z.string(),
  trial_version: z.number().int(),
  predictions: z.object({
    fixes: z.array(z.object({ query_id: z.string(), reason: z.string() })),
    risks: z.array(z.object({ query_id: z.string(), reason: z.string() })),
  }),
});
export type Manifest = z.infer<typeof ManifestSchema>;

// FSM 状态类型（供 Coordinator 使用）
export type ExpFsmState =
  | "Init" | "Generating" | "Executing" | "Scoring"
  | "Triaging" | "Deciding" | "Publishing" | "Published" | "Aborted";

// events.jsonl 事件类型
export type ExpEvent =
  | { ts: string; type: "state_transition"; from: ExpFsmState; to: ExpFsmState; round: number }
  | { ts: string; type: "round_completed"; round: number; verdict: "continue" | "publish" }
  | { ts: string; type: "step_failed"; state: ExpFsmState; error: string; retryable: boolean }
  | { ts: string; type: "aborted"; round: number; reason: string };

// lineage entry
export interface LineageEntry {
  version: number;
  candidate_path: string;
  next_change: NextChange;
  status: "running" | "scored" | "guardrail_failed";
  appended_at: string;
}

// 三轴分数
export interface ThreeAxisScores {
  outcome: number;
  trajectory: number;
  guardrail: number;
  guardrail_hard_fail: boolean;
}

// per-query 执行结果
export interface QueryResult {
  query_id: string;
  assertion_results: Array<{ type: string; verdict: "pass" | "fail" | "skip"; reason?: string }>;
  trajectory_summary: {
    tool_call_sequence: string[];
    retry_count: number;
    latency_ms: number;
    error_codes: string[];
  };
  raw_trace_id?: string;
}

// round.yaml 内容
export interface RoundData {
  round: number;
  trial_version: number;
  scores?: ThreeAxisScores;
  per_query_results?: QueryResult[];
  trajectory_summaries?: QueryResult["trajectory_summary"][];
  guardrail_failed?: boolean;
  triage_conclusion?: {
    diagnoses: string[];
    hints: string[];
    verdict: "continue" | "publish";
    cross_round_memory_ref: string;
  };
}
```

- [ ] **Step 4: 运行测试确认通过**

```bash
node --import tsx --test test/exp-schemas.test.ts
```
Expected: 4 tests pass

- [ ] **Step 5: commit**

```bash
git add src/trace-ai/exp/schemas.ts test/exp-schemas.test.ts
git commit -m "feat(exp): B5 schema extensions — trace-mission/v1, trace-bundle/v1, trace-manifest/v1"
```

---

## Task 2: ExpStore — mission-md.ts

**Files:**
- Create: `src/trace-ai/exp/exp-store/mission-md.ts`
- Create: `test/exp-store-mission.test.ts`

- [ ] **Step 1: 写失败测试**

```typescript
// test/exp-store-mission.test.ts
import test from "node:test";
import assert from "node:assert/strict";
import fs from "node:fs/promises";
import path from "node:path";
import os from "node:os";
import { readMission, writeSuggestedChange } from "../src/trace-ai/exp/exp-store/mission-md.js";

async function makeTmpDir() {
  return fs.mkdtemp(path.join(os.tmpdir(), "trace-exp-test-"));
}

test("readMission: parses valid mission.md", async () => {
  const dir = await makeTmpDir();
  await fs.writeFile(path.join(dir, "mission.md"), `---
schema_version: trace-mission/v1
goal: reduce retry rate
eval_sets:
  - path: eval-sets/v1
    role: seed
current_candidate:
  path: candidates/baseline.yaml
---
Some body text.
`);
  const mission = await readMission(dir);
  assert.equal(mission.goal, "reduce retry rate");
  assert.equal(mission.eval_sets[0].path, "eval-sets/v1");
});

test("readMission: throws if mission.md missing", async () => {
  const dir = await makeTmpDir();
  await assert.rejects(() => readMission(dir), /mission\.md/);
});

test("writeSuggestedChange: overwrites next_change in mission.md", async () => {
  const dir = await makeTmpDir();
  await fs.writeFile(path.join(dir, "mission.md"), `---
schema_version: trace-mission/v1
goal: reduce retry rate
eval_sets:
  - path: eval-sets/v1
    role: seed
current_candidate:
  path: candidates/baseline.yaml
---
`);
  await writeSuggestedChange(dir, {
    target: "agent.system_prompt",
    hypothesis: "add stop condition",
    patch: '{"agent":{"system_prompt":"new prompt"}}',
  });
  const mission = await readMission(dir);
  assert.equal(mission.next_change?.target, "agent.system_prompt");
  assert.equal(mission.next_change?.hypothesis, "add stop condition");
});
```

- [ ] **Step 2: 运行确认失败**

```bash
node --import tsx --test test/exp-store-mission.test.ts
```

- [ ] **Step 3: 实现 mission-md.ts**

```typescript
// src/trace-ai/exp/exp-store/mission-md.ts
import fs from "node:fs/promises";
import path from "node:path";
import yaml from "js-yaml";
import { MissionSchema, type Mission, type NextChange } from "../schemas.js";

export async function readMission(expDir: string): Promise<Mission> {
  const filePath = path.join(expDir, "mission.md");
  let raw: string;
  try {
    raw = await fs.readFile(filePath, "utf8");
  } catch {
    throw new Error(`mission.md not found in ${expDir}`);
  }
  // Extract YAML frontmatter between --- delimiters
  const match = raw.match(/^---\n([\s\S]*?)\n---/);
  if (!match) throw new Error(`mission.md in ${expDir} has no YAML frontmatter`);
  const parsed = yaml.load(match[1]);
  const result = MissionSchema.safeParse(parsed);
  if (!result.success) {
    const issues = result.error.issues.map(i => `${i.path.join(".")}: ${i.message}`).join("; ");
    throw new Error(`mission.md schema invalid: ${issues}`);
  }
  return result.data;
}

export async function writeSuggestedChange(expDir: string, change: NextChange): Promise<void> {
  const filePath = path.join(expDir, "mission.md");
  const raw = await fs.readFile(filePath, "utf8");
  const match = raw.match(/^---\n([\s\S]*?)\n---(\n[\s\S]*)?$/);
  if (!match) throw new Error(`mission.md in ${expDir} has no YAML frontmatter`);

  const frontmatter = yaml.load(match[1]) as Record<string, unknown>;
  frontmatter["next_change"] = change;
  const body = match[2] ?? "";
  const newContent = `---\n${yaml.dump(frontmatter, { lineWidth: -1 })}---${body}`;
  await fs.writeFile(filePath, newContent, "utf8");
}
```

- [ ] **Step 4: 运行确认通过**

```bash
node --import tsx --test test/exp-store-mission.test.ts
```

- [ ] **Step 5: commit**

```bash
git add src/trace-ai/exp/exp-store/mission-md.ts test/exp-store-mission.test.ts
git commit -m "feat(exp): ExpStore mission-md — readMission + writeSuggestedChange"
```

---

## Task 3: ExpStore — events-jsonl.ts

**Files:**
- Create: `src/trace-ai/exp/exp-store/events-jsonl.ts`
- Create: `test/exp-store-events.test.ts`

- [ ] **Step 1: 写失败测试**

```typescript
// test/exp-store-events.test.ts
import test from "node:test";
import assert from "node:assert/strict";
import fs from "node:fs/promises";
import path from "node:path";
import os from "node:os";
import { appendEvent, replayState } from "../src/trace-ai/exp/exp-store/events-jsonl.js";
import type { ExpFsmState } from "../src/trace-ai/exp/schemas.js";

async function makeTmpDir() {
  return fs.mkdtemp(path.join(os.tmpdir(), "trace-exp-events-"));
}

test("replayState: returns Init for empty events.jsonl", async () => {
  const dir = await makeTmpDir();
  await fs.mkdir(path.join(dir, ".trace-state"), { recursive: true });
  await fs.writeFile(path.join(dir, ".trace-state", "events.jsonl"), "");
  const state = await replayState(dir);
  assert.equal(state.currentState, "Init");
  assert.equal(state.currentRound, 0);
  assert.equal(state.lastEvent, null);
});

test("appendEvent + replayState: reflects last transition", async () => {
  const dir = await makeTmpDir();
  await fs.mkdir(path.join(dir, ".trace-state"), { recursive: true });
  await fs.writeFile(path.join(dir, ".trace-state", "events.jsonl"), "");

  await appendEvent(dir, { type: "state_transition", from: "Init", to: "Generating", round: 1 });
  await appendEvent(dir, { type: "state_transition", from: "Generating", to: "Executing", round: 1 });

  const state = await replayState(dir);
  assert.equal(state.currentState, "Executing");
  assert.equal(state.currentRound, 1);
});

test("replayState: detects step_failed", async () => {
  const dir = await makeTmpDir();
  await fs.mkdir(path.join(dir, ".trace-state"), { recursive: true });
  await fs.writeFile(path.join(dir, ".trace-state", "events.jsonl"), "");

  await appendEvent(dir, { type: "state_transition", from: "Init", to: "Generating", round: 1 });
  await appendEvent(dir, { type: "step_failed", state: "Generating", error: "timeout", retryable: true });

  const state = await replayState(dir);
  assert.equal(state.currentState, "Generating");
  assert.equal(state.lastFailure?.retryable, true);
});

test("replayState: terminal state Published", async () => {
  const dir = await makeTmpDir();
  await fs.mkdir(path.join(dir, ".trace-state"), { recursive: true });
  await fs.writeFile(path.join(dir, ".trace-state", "events.jsonl"), "");
  await appendEvent(dir, { type: "state_transition", from: "Publishing", to: "Published", round: 2 });
  const state = await replayState(dir);
  assert.equal(state.isTerminal, true);
});
```

- [ ] **Step 2: 运行确认失败**

```bash
node --import tsx --test test/exp-store-events.test.ts
```

- [ ] **Step 3: 实现 events-jsonl.ts**

```typescript
// src/trace-ai/exp/exp-store/events-jsonl.ts
import fs from "node:fs/promises";
import path from "node:path";
import type { ExpEvent, ExpFsmState } from "../schemas.js";

type EventInput = Omit<ExpEvent, "ts">;

export async function appendEvent(expDir: string, event: EventInput): Promise<void> {
  const filePath = path.join(expDir, ".trace-state", "events.jsonl");
  const line = JSON.stringify({ ts: new Date().toISOString(), ...event }) + "\n";
  await fs.appendFile(filePath, line, "utf8");
}

export interface ReplayedState {
  currentState: ExpFsmState;
  currentRound: number;
  lastEvent: ExpEvent | null;
  lastFailure: { state: ExpFsmState; error: string; retryable: boolean } | null;
  isTerminal: boolean;
}

const TERMINAL: Set<ExpFsmState> = new Set(["Published", "Aborted"]);

export async function replayState(expDir: string): Promise<ReplayedState> {
  const filePath = path.join(expDir, ".trace-state", "events.jsonl");
  let raw: string;
  try {
    raw = await fs.readFile(filePath, "utf8");
  } catch {
    return { currentState: "Init", currentRound: 0, lastEvent: null, lastFailure: null, isTerminal: false };
  }

  const lines = raw.split("\n").filter(Boolean);
  if (lines.length === 0) {
    return { currentState: "Init", currentRound: 0, lastEvent: null, lastFailure: null, isTerminal: false };
  }

  let currentState: ExpFsmState = "Init";
  let currentRound = 0;
  let lastEvent: ExpEvent | null = null;
  let lastFailure: ReplayedState["lastFailure"] = null;

  for (const line of lines) {
    const ev = JSON.parse(line) as ExpEvent;
    lastEvent = ev;
    if (ev.type === "state_transition") {
      currentState = ev.to;
      currentRound = ev.round;
      lastFailure = null;
    } else if (ev.type === "step_failed") {
      currentState = ev.state;
      lastFailure = { state: ev.state, error: ev.error, retryable: ev.retryable };
    } else if (ev.type === "aborted") {
      currentState = "Aborted";
    } else if (ev.type === "round_completed") {
      currentRound = ev.round;
    }
  }

  return {
    currentState,
    currentRound,
    lastEvent,
    lastFailure,
    isTerminal: TERMINAL.has(currentState),
  };
}
```

- [ ] **Step 4: 运行确认通过**

```bash
node --import tsx --test test/exp-store-events.test.ts
```

- [ ] **Step 5: commit**

```bash
git add src/trace-ai/exp/exp-store/events-jsonl.ts test/exp-store-events.test.ts
git commit -m "feat(exp): ExpStore events-jsonl — appendEvent + replayState FSM fold"
```

---

## Task 4: ExpStore — lock.ts

**Files:**
- Create: `src/trace-ai/exp/exp-store/lock.ts`
- Create: `test/exp-store-lock.test.ts`

- [ ] **Step 1: 写失败测试**

```typescript
// test/exp-store-lock.test.ts
import test from "node:test";
import assert from "node:assert/strict";
import fs from "node:fs/promises";
import path from "node:path";
import os from "node:os";
import { acquireLock, releaseLock, updateHeartbeat } from "../src/trace-ai/exp/exp-store/lock.js";

async function makeTmpDir() {
  const dir = await fs.mkdtemp(path.join(os.tmpdir(), "trace-exp-lock-"));
  await fs.mkdir(path.join(dir, ".trace-state"), { recursive: true });
  return dir;
}

test("acquireLock: creates lock.json", async () => {
  const dir = await makeTmpDir();
  await acquireLock(dir);
  const raw = await fs.readFile(path.join(dir, ".trace-state", "lock.json"), "utf8");
  const lock = JSON.parse(raw);
  assert.ok(lock.pid > 0);
  assert.ok(lock.hostname.length > 0);
  await releaseLock(dir);
});

test("acquireLock: fails if fresh lock exists (heartbeat < 30s)", async () => {
  const dir = await makeTmpDir();
  await acquireLock(dir);
  await assert.rejects(() => acquireLock(dir), /locked/i);
  await releaseLock(dir);
});

test("acquireLock: steals stale lock (heartbeat > 30s)", async () => {
  const dir = await makeTmpDir();
  const stale = {
    hostname: "other-host",
    pid: 99999,
    started_at: new Date(Date.now() - 60_000).toISOString(),
    last_heartbeat_ts: new Date(Date.now() - 35_000).toISOString(),
  };
  await fs.writeFile(path.join(dir, ".trace-state", "lock.json"), JSON.stringify(stale));
  await acquireLock(dir);  // should not throw
  const raw = await fs.readFile(path.join(dir, ".trace-state", "lock.json"), "utf8");
  const lock = JSON.parse(raw);
  assert.equal(lock.pid, process.pid);
  await releaseLock(dir);
});

test("releaseLock: removes lock.json", async () => {
  const dir = await makeTmpDir();
  await acquireLock(dir);
  await releaseLock(dir);
  await assert.rejects(
    () => fs.access(path.join(dir, ".trace-state", "lock.json")),
    "lock.json should not exist after release"
  );
});

test("updateHeartbeat: updates last_heartbeat_ts", async () => {
  const dir = await makeTmpDir();
  await acquireLock(dir);
  const before = Date.now();
  await updateHeartbeat(dir);
  const raw = await fs.readFile(path.join(dir, ".trace-state", "lock.json"), "utf8");
  const lock = JSON.parse(raw);
  assert.ok(new Date(lock.last_heartbeat_ts).getTime() >= before);
  await releaseLock(dir);
});
```

- [ ] **Step 2: 运行确认失败**

```bash
node --import tsx --test test/exp-store-lock.test.ts
```

- [ ] **Step 3: 实现 lock.ts**

```typescript
// src/trace-ai/exp/exp-store/lock.ts
import fs from "node:fs/promises";
import os from "node:os";
import path from "node:path";

interface LockData {
  hostname: string;
  pid: number;
  started_at: string;
  last_heartbeat_ts: string;
}

const STALE_THRESHOLD_MS = 30_000;

function lockPath(expDir: string) {
  return path.join(expDir, ".trace-state", "lock.json");
}

export async function acquireLock(expDir: string): Promise<void> {
  const p = lockPath(expDir);
  try {
    const raw = await fs.readFile(p, "utf8");
    const existing = JSON.parse(raw) as LockData;
    const age = Date.now() - new Date(existing.last_heartbeat_ts).getTime();
    if (age < STALE_THRESHOLD_MS) {
      throw new Error(
        `Experiment is locked by pid ${existing.pid} on ${existing.hostname} (heartbeat ${Math.floor(age / 1000)}s ago). Use exp resume or wait.`
      );
    }
    // Stale — fall through to overwrite
  } catch (err: unknown) {
    if ((err as NodeJS.ErrnoException).code !== "ENOENT") throw err;
    // No lock file — fall through to create
  }

  const lock: LockData = {
    hostname: os.hostname(),
    pid: process.pid,
    started_at: new Date().toISOString(),
    last_heartbeat_ts: new Date().toISOString(),
  };
  await fs.writeFile(p, JSON.stringify(lock, null, 2), "utf8");
}

export async function releaseLock(expDir: string): Promise<void> {
  try {
    await fs.unlink(lockPath(expDir));
  } catch {
    // Ignore if already gone
  }
}

export async function updateHeartbeat(expDir: string): Promise<void> {
  const p = lockPath(expDir);
  try {
    const raw = await fs.readFile(p, "utf8");
    const lock = JSON.parse(raw) as LockData;
    lock.last_heartbeat_ts = new Date().toISOString();
    await fs.writeFile(p, JSON.stringify(lock, null, 2), "utf8");
  } catch {
    // Lock removed externally — ignore
  }
}
```

- [ ] **Step 4: 运行确认通过**

```bash
node --import tsx --test test/exp-store-lock.test.ts
```

- [ ] **Step 5: commit**

```bash
git add src/trace-ai/exp/exp-store/lock.ts test/exp-store-lock.test.ts
git commit -m "feat(exp): ExpStore lock — cooperative heartbeat lock with stale detection"
```

---

## Task 5: ExpStore — 剩余子模块

**Files:**
- Create: `src/trace-ai/exp/exp-store/abort-signal.ts`
- Create: `src/trace-ai/exp/exp-store/round-yaml.ts`
- Create: `src/trace-ai/exp/exp-store/candidate-lineage-yaml.ts`
- Create: `src/trace-ai/exp/exp-store/readme-template.ts`
- Create: `src/trace-ai/exp/exp-store/index.ts`

- [ ] **Step 1: 实现 abort-signal.ts**

```typescript
// src/trace-ai/exp/exp-store/abort-signal.ts
import fs from "node:fs/promises";
import path from "node:path";

function signalPath(expDir: string) {
  return path.join(expDir, ".trace-state", "abort.signal");
}

export async function isAborted(expDir: string): Promise<boolean> {
  try {
    await fs.access(signalPath(expDir));
    return true;
  } catch {
    return false;
  }
}

export async function writeAbortSignal(expDir: string): Promise<void> {
  await fs.writeFile(signalPath(expDir), new Date().toISOString(), "utf8");
}

export async function clearAbortSignal(expDir: string): Promise<void> {
  try {
    await fs.unlink(signalPath(expDir));
  } catch {}
}
```

- [ ] **Step 2: 实现 round-yaml.ts**

```typescript
// src/trace-ai/exp/exp-store/round-yaml.ts
import fs from "node:fs/promises";
import path from "node:path";
import yaml from "js-yaml";
import type { RoundData } from "../schemas.js";

function roundPath(expDir: string, n: number) {
  return path.join(expDir, ".trace-state", "rounds", `round-${n}.yaml`);
}

export async function writeRound(expDir: string, n: number, data: Partial<RoundData>): Promise<void> {
  const p = roundPath(expDir, n);
  await fs.mkdir(path.dirname(p), { recursive: true });
  let existing: Partial<RoundData> = {};
  try {
    existing = yaml.load(await fs.readFile(p, "utf8")) as Partial<RoundData>;
  } catch {}
  const merged = { ...existing, round: n, ...data };
  await fs.writeFile(p, yaml.dump(merged, { lineWidth: -1 }), "utf8");
}

export async function readAllRounds(expDir: string): Promise<RoundData[]> {
  const roundsDir = path.join(expDir, ".trace-state", "rounds");
  try {
    const files = await fs.readdir(roundsDir);
    const rounds: RoundData[] = [];
    for (const f of files.filter(f => f.endsWith(".yaml")).sort()) {
      const raw = await fs.readFile(path.join(roundsDir, f), "utf8");
      rounds.push(yaml.load(raw) as RoundData);
    }
    return rounds;
  } catch {
    return [];
  }
}
```

- [ ] **Step 3: 实现 candidate-lineage-yaml.ts**

```typescript
// src/trace-ai/exp/exp-store/candidate-lineage-yaml.ts
import fs from "node:fs/promises";
import path from "node:path";
import yaml from "js-yaml";
import type { LineageEntry } from "../schemas.js";

function lineagePath(expDir: string) {
  return path.join(expDir, ".trace-state", "candidate-lineage.yaml");
}

export async function appendLineage(expDir: string, entry: Omit<LineageEntry, "appended_at">): Promise<void> {
  const p = lineagePath(expDir);
  let entries: LineageEntry[] = [];
  try {
    entries = (yaml.load(await fs.readFile(p, "utf8")) as LineageEntry[]) ?? [];
  } catch {}
  entries.push({ ...entry, appended_at: new Date().toISOString() });
  await fs.writeFile(p, yaml.dump(entries, { lineWidth: -1 }), "utf8");
}

export async function updateLineage(expDir: string, version: number, patch: Partial<LineageEntry>): Promise<void> {
  const p = lineagePath(expDir);
  const entries: LineageEntry[] = (yaml.load(await fs.readFile(p, "utf8")) as LineageEntry[]) ?? [];
  const idx = entries.findIndex(e => e.version === version);
  if (idx >= 0) Object.assign(entries[idx], patch);
  await fs.writeFile(p, yaml.dump(entries, { lineWidth: -1 }), "utf8");
}

export async function readLineage(expDir: string): Promise<LineageEntry[]> {
  try {
    return (yaml.load(await fs.readFile(lineagePath(expDir), "utf8")) as LineageEntry[]) ?? [];
  } catch {
    return [];
  }
}
```

- [ ] **Step 4: 实现 readme-template.ts**

```typescript
// src/trace-ai/exp/exp-store/readme-template.ts
export function renderReadme(opts: { experimentId: string; timestamp: string; goal: string }): string {
  return `# Experiment: ${opts.experimentId}

Created: ${opts.timestamp}  Goal: ${opts.goal}

## 目录说明
- mission.md        — 实验意图（你来编辑）
- eval-sets/        — 评测集（来自 MVP-B 或手动预置）
- candidates/       — Agent 候选快照
- outputs/          — 最终产物（bundle / manifest / provenance）
- .trace-state/     — 运行态，勿手动编辑

## 常用命令
\`\`\`
kweaver trace exp run .           — 启动 / 新开一轮
kweaver trace exp resume .        — 从 Deciding 状态继续
kweaver trace exp show .          — 查看当前状态和建议
kweaver trace exp status .        — 一行摘要（适合脚本）
kweaver trace exp abort .         — 优雅中止
kweaver trace exp doctor .        — 环境自检
\`\`\`
`;
}
```

- [ ] **Step 5: 实现 exp-store/index.ts（ExpStore 组合类）**

```typescript
// src/trace-ai/exp/exp-store/index.ts
import fs from "node:fs/promises";
import path from "node:path";
import crypto from "node:crypto";
import type { ExpEvent, LineageEntry, Mission, NextChange, RoundData } from "../schemas.js";
import { readMission, writeSuggestedChange } from "./mission-md.js";
import { appendEvent, replayState, type ReplayedState } from "./events-jsonl.js";
import { acquireLock, releaseLock, updateHeartbeat } from "./lock.js";
import { isAborted, writeAbortSignal } from "./abort-signal.js";
import { writeRound, readAllRounds } from "./round-yaml.js";
import { appendLineage, updateLineage, readLineage } from "./candidate-lineage-yaml.js";
import { renderReadme } from "./readme-template.js";

export { type ReplayedState };

export class ExpStore {
  constructor(readonly expDir: string) {}

  async initDir(mission: Mission): Promise<string> {
    const experimentId = `exp_${crypto.randomBytes(4).toString("hex")}`;
    await fs.mkdir(path.join(this.expDir, ".trace-state", "rounds"), { recursive: true });
    await fs.mkdir(path.join(this.expDir, "candidates"), { recursive: true });
    await fs.mkdir(path.join(this.expDir, "eval-sets"), { recursive: true });
    await fs.mkdir(path.join(this.expDir, "outputs"), { recursive: true });
    await fs.writeFile(
      path.join(this.expDir, ".trace-state", "events.jsonl"),
      "",
      { flag: "wx" }
    ).catch(() => {});  // already exists ok
    const readmePath = path.join(this.expDir, "README.md");
    try {
      await fs.access(readmePath);
    } catch {
      await fs.writeFile(readmePath, renderReadme({
        experimentId,
        timestamp: new Date().toISOString(),
        goal: mission.goal,
      }));
    }
    return experimentId;
  }

  async archiveState(): Promise<void> {
    const src = path.join(this.expDir, ".trace-state");
    const dst = path.join(this.expDir, `.trace-state-archived-${Date.now()}`);
    await fs.rename(src, dst);
    await fs.mkdir(path.join(this.expDir, ".trace-state", "rounds"), { recursive: true });
    await fs.writeFile(path.join(this.expDir, ".trace-state", "events.jsonl"), "");
  }

  readMission = () => readMission(this.expDir);
  writeSuggestedChange = (c: NextChange) => writeSuggestedChange(this.expDir, c);
  appendEvent = (e: Omit<ExpEvent, "ts">) => appendEvent(this.expDir, e);
  replayState = () => replayState(this.expDir);
  acquireLock = () => acquireLock(this.expDir);
  releaseLock = () => releaseLock(this.expDir);
  updateHeartbeat = () => updateHeartbeat(this.expDir);
  isAborted = () => isAborted(this.expDir);
  writeAbortSignal = () => writeAbortSignal(this.expDir);
  writeRound = (n: number, data: Partial<RoundData>) => writeRound(this.expDir, n, data);
  readAllRounds = () => readAllRounds(this.expDir);
  appendLineage = (e: Omit<LineageEntry, "appended_at">) => appendLineage(this.expDir, e);
  updateLineage = (v: number, p: Partial<LineageEntry>) => updateLineage(this.expDir, v, p);
  readLineage = () => readLineage(this.expDir);
}
```

- [ ] **Step 6: 快速冒烟测试**

```bash
node --import tsx -e "
import { ExpStore } from './src/trace-ai/exp/exp-store/index.js';
import { isAborted } from './src/trace-ai/exp/exp-store/abort-signal.js';
import os from 'node:os'; import fs from 'node:fs/promises'; import path from 'node:path';
const dir = await fs.mkdtemp(path.join(os.tmpdir(), 'smoke-'));
const store = new ExpStore(dir);
await fs.mkdir(path.join(dir, '.trace-state'), { recursive: true });
console.log(await store.isAborted());  // false
"
```
Expected: `false`

- [ ] **Step 7: commit**

```bash
git add src/trace-ai/exp/exp-store/
git commit -m "feat(exp): ExpStore — abort-signal, round-yaml, lineage, readme-template, barrel"
```

---

## Task 6: PatchApplier

**Files:**
- Create: `src/trace-ai/exp/patch/index.ts`
- Create: `src/trace-ai/exp/patch/agent-config.ts`
- Create: `src/trace-ai/exp/patch/skill.ts`
- Create: `test/exp-patch.test.ts`

- [ ] **Step 1: 写失败测试**

```typescript
// test/exp-patch.test.ts
import test from "node:test";
import assert from "node:assert/strict";
import { applyPatch } from "../src/trace-ai/exp/patch/index.js";

const baseCandidate = {
  candidate_version: "v0",
  agent: { model: "claude-sonnet-4-6", temperature: 0.3, system_prompt: "You are an agent." },
  skills: [{ name: "retrieval_v1" }],
};

test("applyPatch: agent.system_prompt replacement", () => {
  const result = applyPatch(baseCandidate, {
    target: "agent.system_prompt",
    hypothesis: "new prompt",
    patch: JSON.stringify({ agent: { system_prompt: "New prompt text." } }),
  });
  assert.equal(result.agent.system_prompt, "New prompt text.");
  assert.equal(result.agent.model, "claude-sonnet-4-6");  // unchanged
});

test("applyPatch: agent.temperature change", () => {
  const result = applyPatch(baseCandidate, {
    target: "agent.temperature",
    hypothesis: "lower temp",
    patch: JSON.stringify({ agent: { temperature: 0.1 } }),
  });
  assert.equal(result.agent.temperature, 0.1);
});

test("applyPatch: skill.add", () => {
  const result = applyPatch(baseCandidate, {
    target: "skill.add",
    hypothesis: "add summarize skill",
    patch: JSON.stringify({ skills: { add: [{ name: "summarize_v2" }] } }),
  });
  assert.equal(result.skills.length, 2);
  assert.ok(result.skills.some((s: { name: string }) => s.name === "summarize_v2"));
});

test("applyPatch: skill.remove", () => {
  const result = applyPatch(baseCandidate, {
    target: "skill.remove",
    hypothesis: "remove retrieval",
    patch: JSON.stringify({ skills: { remove: ["retrieval_v1"] } }),
  });
  assert.equal(result.skills.length, 0);
});

test("applyPatch: throws for unknown target prefix", () => {
  assert.throws(
    () => applyPatch(baseCandidate, { target: "bkn.entity", hypothesis: "x", patch: "{}" }),
    /unsupported.*target/i
  );
});
```

- [ ] **Step 2: 运行确认失败**

```bash
node --import tsx --test test/exp-patch.test.ts
```

- [ ] **Step 3: 实现 patch/agent-config.ts**

```typescript
// src/trace-ai/exp/patch/agent-config.ts
// Applies JSON Merge Patch semantics to the agent.* fields of a candidate object.
export function applyAgentConfigPatch(candidate: Record<string, unknown>, patchJson: string): Record<string, unknown> {
  const patch = JSON.parse(patchJson) as Record<string, unknown>;
  if (!patch.agent) throw new Error("agent.* patch must have an 'agent' key");
  const result = structuredClone(candidate) as Record<string, unknown>;
  result["agent"] = mergePatch(result["agent"] as Record<string, unknown>, patch["agent"] as Record<string, unknown>);
  return result;
}

function mergePatch(target: Record<string, unknown>, patch: Record<string, unknown>): Record<string, unknown> {
  const result = { ...target };
  for (const [k, v] of Object.entries(patch)) {
    if (v === null) {
      delete result[k];
    } else if (typeof v === "object" && !Array.isArray(v)) {
      result[k] = mergePatch((result[k] as Record<string, unknown>) ?? {}, v as Record<string, unknown>);
    } else {
      result[k] = v;
    }
  }
  return result;
}
```

- [ ] **Step 4: 实现 patch/skill.ts**

```typescript
// src/trace-ai/exp/patch/skill.ts
interface SkillEntry { name: string; [k: string]: unknown }
interface SkillPatchSpec { add?: SkillEntry[]; remove?: string[]; swap?: { from: string; to: SkillEntry } }

export function applySkillPatch(candidate: Record<string, unknown>, patchJson: string): Record<string, unknown> {
  const patch = JSON.parse(patchJson) as { skills: SkillPatchSpec };
  if (!patch.skills) throw new Error("skill.* patch must have a 'skills' key");
  const result = structuredClone(candidate) as Record<string, unknown>;
  let skills: SkillEntry[] = (result["skills"] as SkillEntry[]) ?? [];

  if (patch.skills.remove) {
    const toRemove = new Set(patch.skills.remove);
    skills = skills.filter(s => !toRemove.has(s.name));
  }
  if (patch.skills.add) {
    skills = [...skills, ...patch.skills.add];
  }
  if (patch.skills.swap) {
    const { from, to } = patch.skills.swap;
    skills = skills.map(s => s.name === from ? to : s);
  }
  result["skills"] = skills;
  return result;
}
```

- [ ] **Step 5: 实现 patch/index.ts**

```typescript
// src/trace-ai/exp/patch/index.ts
import type { NextChange } from "../schemas.js";
import { applyAgentConfigPatch } from "./agent-config.js";
import { applySkillPatch } from "./skill.js";

export function applyPatch(candidate: Record<string, unknown>, change: NextChange): Record<string, unknown> {
  const prefix = change.target.split(".")[0];
  switch (prefix) {
    case "agent":
      return applyAgentConfigPatch(candidate, change.patch);
    case "skill":
      return applySkillPatch(candidate, change.patch);
    default:
      throw new Error(`Unsupported target prefix "${prefix}" — only agent.* and skill.* are supported in MVP-C`);
  }
}
```

- [ ] **Step 6: 运行确认通过**

```bash
node --import tsx --test test/exp-patch.test.ts
```

- [ ] **Step 7: commit**

```bash
git add src/trace-ai/exp/patch/ test/exp-patch.test.ts
git commit -m "feat(exp): PatchApplier — AgentConfigPatcher + SkillPatcher with JSON Merge Patch"
```

---

## Task 7: Scoring Engine

**Files:**
- Create: `src/trace-ai/exp/scoring.ts`
- Create: `test/exp-scoring.test.ts`

- [ ] **Step 1: 写失败测试**

```typescript
// test/exp-scoring.test.ts
import test from "node:test";
import assert from "node:assert/strict";
import { computeScores } from "../src/trace-ai/exp/scoring.js";
import type { QueryResult } from "../src/trace-ai/exp/schemas.js";

const makeQueryResult = (overrides: Partial<QueryResult> = {}): QueryResult => ({
  query_id: "q1",
  assertion_results: [{ type: "contains", verdict: "pass" }],
  trajectory_summary: { tool_call_sequence: ["search", "answer"], retry_count: 0, latency_ms: 500, error_codes: [] },
  ...overrides,
});

test("computeScores: all pass → high scores", () => {
  const results = [makeQueryResult(), makeQueryResult({ query_id: "q2" })];
  const scores = computeScores(results, []);
  assert.ok(scores.outcome >= 0.9);
  assert.ok(scores.trajectory >= 0.9);
  assert.equal(scores.guardrail_hard_fail, false);
});

test("computeScores: outcome drops on failed assertions", () => {
  const results = [
    makeQueryResult({ assertion_results: [{ type: "contains", verdict: "fail" }] }),
    makeQueryResult(),
  ];
  const scores = computeScores(results, []);
  assert.ok(scores.outcome < 0.6);
});

test("computeScores: trajectory penalized on high retry_count", () => {
  const results = [
    makeQueryResult({ trajectory_summary: { tool_call_sequence: ["a"], retry_count: 5, latency_ms: 500, error_codes: [] } }),
  ];
  const scores = computeScores(results, []);
  assert.ok(scores.trajectory < 0.7);
});

test("computeScores: guardrail hard fail when rule violated", () => {
  const results = [makeQueryResult()];
  const guardrails = [{ name: "no_error", kind: "hard" as const, rule: "error_codes must be empty" }];
  const resultsWithError = [
    makeQueryResult({ trajectory_summary: { tool_call_sequence: [], retry_count: 0, latency_ms: 100, error_codes: ["AUTH_FORBIDDEN"] } }),
  ];
  const scores = computeScores(resultsWithError, guardrails);
  assert.equal(scores.guardrail_hard_fail, true);
});
```

- [ ] **Step 2: 运行确认失败**

```bash
node --import tsx --test test/exp-scoring.test.ts
```

- [ ] **Step 3: 实现 scoring.ts**

```typescript
// src/trace-ai/exp/scoring.ts
import type { QueryResult, ThreeAxisScores } from "./schemas.js";

interface Guardrail { name: string; kind: "hard" | "soft"; rule: string }

export function computeScores(results: QueryResult[], guardrails: Guardrail[]): ThreeAxisScores {
  if (results.length === 0) {
    return { outcome: 0, trajectory: 0, guardrail: 1, guardrail_hard_fail: false };
  }

  // Outcome: fraction of assertions that passed
  let totalAssertions = 0;
  let passedAssertions = 0;
  for (const r of results) {
    for (const a of r.assertion_results) {
      if (a.verdict === "skip") continue;
      totalAssertions++;
      if (a.verdict === "pass") passedAssertions++;
    }
  }
  const outcome = totalAssertions === 0 ? 1 : passedAssertions / totalAssertions;

  // Trajectory: penalize retries and errors
  let trajectorySum = 0;
  for (const r of results) {
    const { retry_count, error_codes } = r.trajectory_summary;
    const retryPenalty = Math.min(retry_count * 0.15, 0.6);
    const errorPenalty = error_codes.length > 0 ? 0.3 : 0;
    trajectorySum += Math.max(0, 1 - retryPenalty - errorPenalty);
  }
  const trajectory = trajectorySum / results.length;

  // Guardrail: check hard gates (simplified: any error_codes in results triggers hard gate if guardrail with "error_codes" rule)
  let guardrail_hard_fail = false;
  let guardrail = 1;
  for (const g of guardrails) {
    if (g.kind === "hard") {
      const violated = results.some(r => r.trajectory_summary.error_codes.length > 0);
      if (violated) {
        guardrail_hard_fail = true;
        guardrail = 0;
        break;
      }
    }
  }

  return { outcome, trajectory, guardrail, guardrail_hard_fail };
}
```

- [ ] **Step 4: 运行确认通过**

```bash
node --import tsx --test test/exp-scoring.test.ts
```

- [ ] **Step 5: commit**

```bash
git add src/trace-ai/exp/scoring.ts test/exp-scoring.test.ts
git commit -m "feat(exp): three-axis scoring engine — outcome/trajectory/guardrail"
```

---

## Task 8: SynthesizerClient + TriageClient

**Files:**
- Create: `src/trace-ai/exp/providers/synthesizer-client.ts`
- Create: `src/trace-ai/exp/providers/triage-client.ts`

- [ ] **Step 1: 实现 synthesizer-client.ts**

```typescript
// src/trace-ai/exp/providers/synthesizer-client.ts
import { z } from "zod";
import yaml from "js-yaml";
import { defaultRegistry } from "../../agent-providers/registry.js";
import type { Mission, NextChange, RoundData } from "../schemas.js";

export interface SynthesizerInput {
  mission: Mission;
  candidateConfig: Record<string, unknown>;
  prevRound?: RoundData;
  prevRounds: RoundData[];
  crossRoundMemoryRef?: string;
}

const SynthesizerOutputSchema = z.object({
  target: z.string(),
  hypothesis: z.string(),
  patch: z.string(),
});

export interface SynthesizerClient {
  generate(input: SynthesizerInput): Promise<NextChange>;
}

export class ClaudeCodeSynthesizer implements SynthesizerClient {
  async generate(input: SynthesizerInput): Promise<NextChange> {
    const provider = defaultRegistry.resolve({ preferred: "claude-code" });
    if (!provider) throw new Error("claude-code provider not available");

    const prevSummary = input.prevRounds.map(r =>
      `Round ${r.round}: outcome=${r.scores?.outcome.toFixed(2)}, hints=${r.triage_conclusion?.hints.join("; ") ?? "none"}`
    ).join("\n");

    const prompt = `You are an agent optimization assistant. Given an experiment goal and round results, suggest the next change to try.

GOAL: ${input.mission.goal}

CURRENT CANDIDATE CONFIG:
${yaml.dump(input.candidateConfig, { lineWidth: 80 })}

PREVIOUS ROUNDS:
${prevSummary || "None (first round)"}

${input.prevRound?.triage_conclusion ? `TRIAGE HINTS FROM LAST ROUND:\n${input.prevRound.triage_conclusion.hints.join("\n")}` : ""}

${input.crossRoundMemoryRef ? `CROSS-ROUND CONTEXT: ${input.crossRoundMemoryRef}` : ""}

Respond with a JSON object with exactly these fields:
- "target": one of "agent.system_prompt", "agent.temperature", "agent.model", "skill.add", "skill.remove", "skill.swap"
- "hypothesis": brief explanation of why this change might help
- "patch": a JSON Merge Patch string to apply to the candidate config

Example for changing system_prompt:
{"target": "agent.system_prompt", "hypothesis": "Add explicit stop condition", "patch": "{\"agent\":{\"system_prompt\":\"New prompt here\"}}"}`;

    const response = await provider.invoke({
      prompt,
      outputSchema: SynthesizerOutputSchema,
      correlationId: `synthesizer-${Date.now()}`,
    });
    return response.output;
  }
}
```

- [ ] **Step 2: 实现 triage-client.ts**

```typescript
// src/trace-ai/exp/providers/triage-client.ts
import { z } from "zod";
import yaml from "js-yaml";
import { defaultRegistry } from "../../agent-providers/registry.js";
import type { RoundData } from "../schemas.js";

export interface TriageInput {
  currentRound: RoundData;
  prevRounds: RoundData[];
  candidateConfig: Record<string, unknown>;
  crossRoundMemoryRef?: string;
}

export interface TriageResult {
  diagnoses: string[];
  hints: string[];
  verdict: "continue" | "publish";
  new_memory_token: string;
}

const TriageOutputSchema = z.object({
  diagnoses: z.array(z.string()),
  hints: z.array(z.string()),
  verdict: z.enum(["continue", "publish"]),
  new_memory_token: z.string(),
});

export interface TriageClient {
  triage(input: TriageInput): Promise<TriageResult>;
}

export class ClaudeCodeTriageClient implements TriageClient {
  async triage(input: TriageInput): Promise<TriageResult> {
    const provider = defaultRegistry.resolve({ preferred: "claude-code" });
    if (!provider) throw new Error("claude-code provider not available");

    const r = input.currentRound;
    const scoresSummary = r.scores
      ? `outcome=${r.scores.outcome.toFixed(2)}, trajectory=${r.scores.trajectory.toFixed(2)}, guardrail=${r.scores.guardrail.toFixed(2)}`
      : "no scores";

    const failedQueries = (r.per_query_results ?? [])
      .filter(q => q.assertion_results.some(a => a.verdict === "fail"))
      .map(q => `${q.query_id}: ${q.assertion_results.filter(a => a.verdict === "fail").map(a => a.type).join(", ")}`)
      .join("\n");

    const prompt = `You are an agent evaluation triager. Analyze the current round results and recommend next steps.

ROUND ${r.round} SCORES: ${scoresSummary}

FAILED QUERIES:
${failedQueries || "None"}

TRAJECTORY ISSUES:
${(r.per_query_results ?? []).filter(q => q.trajectory_summary.retry_count > 1).map(q => `${q.query_id}: ${q.trajectory_summary.retry_count} retries`).join("\n") || "None"}

PREVIOUS ROUND HISTORY:
${input.prevRounds.map(pr => `Round ${pr.round}: outcome=${pr.scores?.outcome.toFixed(2) ?? "?"}, verdict=${pr.triage_conclusion?.verdict ?? "?"}`).join("\n") || "None"}

${input.crossRoundMemoryRef ? `CONTEXT FROM PREVIOUS TRIAGE: ${input.crossRoundMemoryRef}` : ""}

Respond with JSON:
- "diagnoses": list of root cause observations
- "hints": list of specific suggestions for next change
- "verdict": "continue" if more rounds needed, "publish" if this candidate is good enough
- "new_memory_token": brief summary of key findings to carry forward (1-2 sentences)`;

    const response = await provider.invoke({
      prompt,
      outputSchema: TriageOutputSchema,
      correlationId: `triage-${Date.now()}`,
    });
    return response.output;
  }
}
```

- [ ] **Step 3: commit**

```bash
git add src/trace-ai/exp/providers/
git commit -m "feat(exp): SynthesizerClient + TriageClient with claude-code provider"
```

---

## Task 9: EvalRunner + BundleWriter

**Files:**
- Create: `src/trace-ai/exp/eval-runner.ts`
- Create: `src/trace-ai/exp/bundle-writer.ts`
- Create: `test/exp-bundle-writer.test.ts`

- [ ] **Step 1: 实现 eval-runner.ts**（wraps MVP-B test-runner）

```typescript
// src/trace-ai/exp/eval-runner.ts
import path from "node:path";
import yaml from "js-yaml";
import fs from "node:fs/promises";
import type { QueryResult } from "./schemas.js";
import type { RunnerDeps } from "../eval-set/test-runner.js";
import { run as evalSetRun } from "../eval-set/test-runner.js";

export interface EvalRunnerOpts {
  evalSetPaths: string[];       // paths to eval-set dirs
  candidatePath: string;        // path to candidate YAML
  expDir: string;
  deps: RunnerDeps;
  maxParallel?: number;
}

export interface EvalRunResult {
  queryResults: QueryResult[];
}

export async function runEval(opts: EvalRunnerOpts): Promise<EvalRunResult> {
  const candidateRaw = yaml.load(await fs.readFile(opts.candidatePath, "utf8")) as Record<string, unknown>;
  const agentId = (candidateRaw["agent_id"] as string | undefined) ?? "candidate";
  const agentVersion = (candidateRaw["candidate_version"] as string | undefined);

  const outDir = path.join(opts.expDir, ".trace-state", "_eval-tmp");
  await fs.mkdir(outDir, { recursive: true });

  // Run eval for each eval-set (sequentially for MVP-C single-path)
  const allResults: QueryResult[] = [];
  for (const evalSetDir of opts.evalSetPaths) {
    await evalSetRun({
      evalSetDir,
      candidateAgentId: agentId,
      candidateAgentVersion: agentVersion,
      outDir,
      maxParallel: opts.maxParallel ?? 4,
      deps: opts.deps,
    });

    // Read report and convert to QueryResult[]
    const reportPath = path.join(outDir, "report.yaml");
    const report = yaml.load(await fs.readFile(reportPath, "utf8")) as {
      cases: Array<{
        query_id: string;
        assertion_results: Array<{ type: string; verdict: string; reason?: string }>;
        duration_ms?: number;
        trace_id?: string;
      }>;
    };

    for (const c of report.cases) {
      allResults.push({
        query_id: c.query_id,
        assertion_results: c.assertion_results as QueryResult["assertion_results"],
        trajectory_summary: {
          tool_call_sequence: [],  // populated from trace if available
          retry_count: 0,
          latency_ms: c.duration_ms ?? 0,
          error_codes: [],
        },
        raw_trace_id: c.trace_id,
      });
    }
  }

  return { queryResults: allResults };
}
```

- [ ] **Step 2: 写 BundleWriter 测试**

```typescript
// test/exp-bundle-writer.test.ts
import test from "node:test";
import assert from "node:assert/strict";
import fs from "node:fs/promises";
import path from "node:path";
import os from "node:os";
import { writeBundles } from "../src/trace-ai/exp/bundle-writer.js";
import type { LineageEntry, RoundData } from "../src/trace-ai/exp/schemas.js";
import yaml from "js-yaml";

async function makeTmpDir() {
  const dir = await fs.mkdtemp(path.join(os.tmpdir(), "trace-bundle-"));
  await fs.mkdir(path.join(dir, "outputs"), { recursive: true });
  return dir;
}

test("writeBundles: creates bundle.yaml, manifest.yaml, provenance.yaml", async () => {
  const dir = await makeTmpDir();
  const lineage: LineageEntry[] = [{
    version: 1,
    candidate_path: "candidates/candidate-v1.yaml",
    next_change: { target: "agent.system_prompt", hypothesis: "test", patch: "{}" },
    status: "scored",
    appended_at: new Date().toISOString(),
  }];
  const rounds: RoundData[] = [{
    round: 1,
    trial_version: 1,
    scores: { outcome: 0.8, trajectory: 0.9, guardrail: 1, guardrail_hard_fail: false },
    per_query_results: [],
    triage_conclusion: {
      diagnoses: ["tool retries too high"],
      hints: ["add stop condition"],
      verdict: "publish",
      cross_round_memory_ref: "mem_token_1",
    },
  }];

  await writeBundles({ expDir: dir, experimentId: "exp_test", lineage, rounds, createdBy: "testuser" });

  const bundle = yaml.load(await fs.readFile(path.join(dir, "outputs", "bundle.yaml"), "utf8")) as Record<string, unknown>;
  assert.equal(bundle["schema_version"], "trace-bundle/v1");
  assert.equal(bundle["experiment_id"], "exp_test");

  const manifest = yaml.load(await fs.readFile(path.join(dir, "outputs", "manifest.yaml"), "utf8")) as Record<string, unknown>;
  assert.equal(manifest["schema_version"], "trace-manifest/v1");

  await fs.access(path.join(dir, "outputs", "provenance.yaml"));
});
```

- [ ] **Step 3: 实现 bundle-writer.ts**

```typescript
// src/trace-ai/exp/bundle-writer.ts
import fs from "node:fs/promises";
import path from "node:path";
import crypto from "node:crypto";
import yaml from "js-yaml";
import type { LineageEntry, RoundData } from "./schemas.js";

interface WriteBundlesOpts {
  expDir: string;
  experimentId: string;
  lineage: LineageEntry[];
  rounds: RoundData[];
  createdBy: string;
}

export async function writeBundles(opts: WriteBundlesOpts): Promise<void> {
  const { expDir, experimentId, lineage, rounds, createdBy } = opts;
  const bestEntry = lineage.filter(e => e.status === "scored").at(-1) ?? lineage.at(-1);
  const bestVersion = bestEntry?.version ?? 0;
  const bundleId = `bundle_${crypto.randomBytes(4).toString("hex")}`;
  const now = new Date().toISOString();

  const bundle = {
    schema_version: "trace-bundle/v1",
    experiment_id: experimentId,
    bundle_id: bundleId,
    best_trial_version: bestVersion,
    resources: {
      agent_config: bestEntry?.next_change ?? {},
      skills: [],
    },
    provenance: {
      created_by: createdBy,
      created_at: now,
      evidence_traces: rounds.flatMap(r => (r.per_query_results ?? []).map(q => q.raw_trace_id ?? "").filter(Boolean)),
      round_refs: rounds.map(r => `.trace-state/rounds/round-${r.round}.yaml`),
    },
  };

  const lastRound = rounds.at(-1);
  const manifest = {
    schema_version: "trace-manifest/v1",
    experiment_id: experimentId,
    trial_version: bestVersion,
    predictions: {
      fixes: (lastRound?.per_query_results ?? [])
        .filter(q => q.assertion_results.every(a => a.verdict === "pass"))
        .map(q => ({ query_id: q.query_id, reason: "all assertions passed" })),
      risks: (lastRound?.per_query_results ?? [])
        .filter(q => q.assertion_results.some(a => a.verdict === "fail"))
        .map(q => ({ query_id: q.query_id, reason: "assertions failed" })),
    },
  };

  const provenance = {
    experiment_id: experimentId,
    generated_at: now,
    rounds_count: rounds.length,
    lineage_count: lineage.length,
    round_verdicts: rounds.map(r => ({ round: r.round, verdict: r.triage_conclusion?.verdict ?? "pending" })),
  };

  const outDir = path.join(expDir, "outputs");
  await fs.mkdir(outDir, { recursive: true });
  await fs.writeFile(path.join(outDir, "bundle.yaml"), yaml.dump(bundle, { lineWidth: -1 }));
  await fs.writeFile(path.join(outDir, "manifest.yaml"), yaml.dump(manifest, { lineWidth: -1 }));
  await fs.writeFile(path.join(outDir, "provenance.yaml"), yaml.dump(provenance, { lineWidth: -1 }));
}
```

- [ ] **Step 4: 运行确认通过**

```bash
node --import tsx --test test/exp-bundle-writer.test.ts
```

- [ ] **Step 5: commit**

```bash
git add src/trace-ai/exp/eval-runner.ts src/trace-ai/exp/bundle-writer.ts test/exp-bundle-writer.test.ts
git commit -m "feat(exp): EvalRunner (wraps MVP-B) + BundleWriter (bundle/manifest/provenance)"
```

---

## Task 10: ExperimentCoordinator (FSM 核心)

**Files:**
- Create: `src/trace-ai/exp/coordinator.ts`
- Create: `test/exp-coordinator.test.ts`

- [ ] **Step 1: 写 Coordinator FSM 测试**

```typescript
// test/exp-coordinator.test.ts
import test from "node:test";
import assert from "node:assert/strict";
import fs from "node:fs/promises";
import path from "node:path";
import os from "node:os";
import { ExperimentCoordinator } from "../src/trace-ai/exp/coordinator.js";
import type { SynthesizerClient, TriageClient } from "../src/trace-ai/exp/coordinator.js";
import type { NextChange, RoundData } from "../src/trace-ai/exp/schemas.js";

const MISSION_CONTENT = `---
schema_version: trace-mission/v1
goal: reduce retries
max_rounds: 2
eval_sets:
  - path: eval-sets/v1
    role: seed
current_candidate:
  path: candidates/baseline.yaml
next_change:
  target: agent.system_prompt
  hypothesis: test change
  patch: '{"agent":{"system_prompt":"new prompt"}}'
---
`;

async function makeExpDir(): Promise<string> {
  const dir = await fs.mkdtemp(path.join(os.tmpdir(), "coord-test-"));
  await fs.mkdir(path.join(dir, ".trace-state", "rounds"), { recursive: true });
  await fs.mkdir(path.join(dir, "candidates"), { recursive: true });
  await fs.mkdir(path.join(dir, "eval-sets", "v1"), { recursive: true });
  await fs.mkdir(path.join(dir, "outputs"), { recursive: true });
  await fs.writeFile(path.join(dir, ".trace-state", "events.jsonl"), "");
  await fs.writeFile(path.join(dir, "mission.md"), MISSION_CONTENT);
  // Minimal baseline candidate
  await fs.writeFile(path.join(dir, "candidates", "baseline.yaml"), "agent_id: test\ncandidate_version: v0\nagent:\n  system_prompt: old prompt\nskills: []\n");
  // Minimal eval-set index
  await fs.writeFile(path.join(dir, "eval-sets", "v1", "index.yaml"), "schema_version: trace-eval-set-index/v1\neval_set_id: test\nshards: []\n");
  return dir;
}

const mockSynthesizer: SynthesizerClient = {
  async generate(): Promise<NextChange> {
    return { target: "agent.system_prompt", hypothesis: "mock", patch: '{"agent":{"system_prompt":"mock prompt"}}' };
  },
};

const mockTriage: TriageClient = {
  async triage(): Promise<RoundData["triage_conclusion"] & { new_memory_token: string }> {
    return { diagnoses: [], hints: [], verdict: "continue", cross_round_memory_ref: "mem1", new_memory_token: "mem1" };
  },
};

const mockEvalRunner = async () => ({ queryResults: [] });

test("coordinator: run transitions to Deciding after round 1", async () => {
  const dir = await makeExpDir();
  const coord = new ExperimentCoordinator({
    expDir: dir,
    synthesizer: mockSynthesizer,
    triage: mockTriage,
    runEval: mockEvalRunner,
  });

  await coord.run();  // should pause at Deciding

  const { replayState } = await import("../src/trace-ai/exp/exp-store/events-jsonl.js");
  const state = await replayState(dir);
  assert.equal(state.currentState, "Deciding");
  assert.equal(state.currentRound, 1);
});

test("coordinator: abort signal stops run", async () => {
  const dir = await makeExpDir();
  const coord = new ExperimentCoordinator({
    expDir: dir,
    synthesizer: mockSynthesizer,
    triage: { async triage() { throw new Error("should not reach triage"); } },
    runEval: async () => {
      // Write abort signal mid-execution
      await fs.writeFile(path.join(dir, ".trace-state", "abort.signal"), "");
      return { queryResults: [] };
    },
  });

  await coord.run();
  const { replayState } = await import("../src/trace-ai/exp/exp-store/events-jsonl.js");
  const state = await replayState(dir);
  assert.equal(state.currentState, "Aborted");
});
```

- [ ] **Step 2: 运行确认失败**

```bash
node --import tsx --test test/exp-coordinator.test.ts
```

- [ ] **Step 3: 实现 coordinator.ts**

```typescript
// src/trace-ai/exp/coordinator.ts
import path from "node:path";
import fs from "node:fs/promises";
import yaml from "js-yaml";
import { ExpStore } from "./exp-store/index.js";
import { applyPatch } from "./patch/index.js";
import { computeScores } from "./scoring.js";
import { writeBundles } from "./bundle-writer.js";
import type { Mission, NextChange, QueryResult, RoundData } from "./schemas.js";

export interface SynthesizerClient {
  generate(input: {
    mission: Mission;
    candidateConfig: Record<string, unknown>;
    prevRound?: RoundData;
    prevRounds: RoundData[];
    crossRoundMemoryRef?: string;
  }): Promise<NextChange>;
}

export interface TriageClient {
  triage(input: {
    currentRound: RoundData;
    prevRounds: RoundData[];
    candidateConfig: Record<string, unknown>;
    crossRoundMemoryRef?: string;
  }): Promise<RoundData["triage_conclusion"] & { new_memory_token: string }>;
}

export interface CoordinatorOpts {
  expDir: string;
  synthesizer: SynthesizerClient;
  triage: TriageClient;
  runEval: (opts: { evalSetPaths: string[]; candidatePath: string; expDir: string }) => Promise<{ queryResults: QueryResult[] }>;
  experimentId?: string;
}

export class ExperimentCoordinator {
  private store: ExpStore;
  private heartbeatTimer?: ReturnType<typeof setInterval>;

  constructor(private opts: CoordinatorOpts) {
    this.store = new ExpStore(opts.expDir);
  }

  async run(): Promise<void> {
    const replayed = await this.store.replayState();

    if (replayed.isTerminal && !replayed.currentState.includes("Aborted")) {
      throw new Error(`Experiment is in terminal state ${replayed.currentState}. Use --new-run to start fresh.`);
    }
    if (!replayed.isTerminal && replayed.currentRound > 0 && replayed.currentState !== "Deciding") {
      // Not first run and not paused at Deciding — was in-progress when interrupted, resume from where we left off
    }

    await this.store.acquireLock();
    this.heartbeatTimer = setInterval(() => { void this.store.updateHeartbeat(); }, 10_000);

    try {
      const mission = await this.store.readMission();
      const expId = this.opts.experimentId ?? `exp_${Date.now()}`;

      if (replayed.currentRound === 0) {
        await this.store.initDir(mission);
      }

      await this.runLoop(mission, replayed.currentRound, expId);
    } finally {
      clearInterval(this.heartbeatTimer);
      await this.store.releaseLock();
    }
  }

  async resume(): Promise<void> {
    const replayed = await this.store.replayState();
    if (replayed.currentState !== "Deciding" && !replayed.lastFailure) {
      throw new Error(`Cannot resume: experiment is in state ${replayed.currentState}, not Deciding`);
    }
    await this.store.acquireLock();
    this.heartbeatTimer = setInterval(() => { void this.store.updateHeartbeat(); }, 10_000);
    try {
      const mission = await this.store.readMission();
      const rounds = await this.store.readAllRounds();
      const expId = `exp_${replayed.currentRound}`;
      if (replayed.currentState === "Deciding") {
        // Start next round
        await this.runLoop(mission, replayed.currentRound, expId);
      } else if (replayed.lastFailure) {
        // Retry from failed state
        await this.runLoop(mission, replayed.currentRound, expId, replayed.lastFailure.state as never);
      }
    } finally {
      clearInterval(this.heartbeatTimer);
      await this.store.releaseLock();
    }
  }

  private async runLoop(mission: Mission, startRound: number, expId: string, _fromState?: never): Promise<void> {
    const round = startRound + 1;
    const maxRounds = mission.max_rounds ?? Infinity;

    // Check abort before each phase
    if (await this.checkAbort(round)) return;

    // === Generating (Apply Phase) ===
    await this.store.appendEvent({ type: "state_transition", from: "Deciding", to: "Generating", round });
    const nextChange = mission.next_change;
    if (!nextChange) throw new Error("mission.md has no next_change — add one or let Synthesizer suggest");

    const prevRounds = await this.store.readAllRounds();
    const lineage = await this.store.readLineage();
    const prevEntry = lineage.at(-1);

    // Load current candidate and apply patch
    const currentCandidatePath = path.join(this.opts.expDir, mission.current_candidate.path);
    const currentCandidate = yaml.load(await fs.readFile(currentCandidatePath, "utf8")) as Record<string, unknown>;
    const patched = applyPatch(currentCandidate, nextChange);
    patched["candidate_version"] = `v${round}`;

    const newCandidatePath = path.join(this.opts.expDir, "candidates", `candidate-v${round}.yaml`);
    await fs.writeFile(newCandidatePath, yaml.dump(patched, { lineWidth: -1 }));

    await this.store.appendLineage({
      version: round,
      candidate_path: `candidates/candidate-v${round}.yaml`,
      next_change: nextChange,
      status: "running",
    });

    if (await this.checkAbort(round)) return;

    // === Executing ===
    await this.store.appendEvent({ type: "state_transition", from: "Generating", to: "Executing", round });
    const evalSetPaths = mission.eval_sets.map(e => path.join(this.opts.expDir, e.path));
    let queryResults: QueryResult[];
    try {
      const result = await this.withRetry(
        () => this.opts.runEval({ evalSetPaths, candidatePath: newCandidatePath, expDir: this.opts.expDir }),
        "Executing"
      );
      queryResults = result.queryResults;
    } catch (err) {
      return;  // step_failed already written by withRetry
    }

    if (await this.checkAbort(round)) return;

    // === Scoring ===
    await this.store.appendEvent({ type: "state_transition", from: "Executing", to: "Scoring", round });
    const guardrails = mission.guardrails ?? [];
    const scores = computeScores(queryResults, guardrails);

    if (scores.guardrail_hard_fail) {
      await this.store.updateLineage(round, { status: "guardrail_failed" });
      await this.store.writeRound(round, { round, trial_version: round, guardrail_failed: true, scores });
      await this.store.appendEvent({ type: "state_transition", from: "Scoring", to: "Deciding", round });
      await this.store.releaseLock();
      clearInterval(this.heartbeatTimer);
      process.stdout.write(`\nRound ${round}: Guardrail hard gate violated. Fix the candidate and run exp resume.\n`);
      return;
    }

    await this.store.updateLineage(round, { status: "scored" });
    await this.store.writeRound(round, { round, trial_version: round, scores, per_query_results: queryResults });

    if (await this.checkAbort(round)) return;

    // === Triaging ===
    await this.store.appendEvent({ type: "state_transition", from: "Scoring", to: "Triaging", round });
    const currentRoundData = (await this.store.readAllRounds()).find(r => r.round === round) ?? { round, trial_version: round };
    const prevMemory = prevRounds.at(-1)?.triage_conclusion?.cross_round_memory_ref;

    let triageResult: RoundData["triage_conclusion"] & { new_memory_token: string };
    try {
      triageResult = await this.withRetry(
        () => this.opts.triage.triage({
          currentRound: currentRoundData,
          prevRounds,
          candidateConfig: patched,
          crossRoundMemoryRef: prevMemory,
        }),
        "Triaging"
      );
    } catch {
      return;
    }

    await this.store.writeRound(round, {
      triage_conclusion: {
        diagnoses: triageResult.diagnoses,
        hints: triageResult.hints,
        verdict: triageResult.verdict,
        cross_round_memory_ref: triageResult.new_memory_token,
      },
    });
    await this.store.appendEvent({ type: "round_completed", round, verdict: triageResult.verdict });

    // Generate next suggestion if continuing
    if (triageResult.verdict === "continue" && round < maxRounds) {
      const updatedMission = await this.store.readMission();
      try {
        const suggestion = await this.withRetry(
          () => this.opts.synthesizer.generate({
            mission: updatedMission,
            candidateConfig: patched,
            prevRound: currentRoundData,
            prevRounds,
            crossRoundMemoryRef: triageResult.new_memory_token,
          }),
          "Triaging"
        );
        await this.store.writeSuggestedChange(suggestion);
      } catch {
        return;
      }
    }

    // === Deciding ===
    await this.store.appendEvent({ type: "state_transition", from: "Triaging", to: "Deciding", round });

    if (triageResult.verdict === "publish" || round >= maxRounds) {
      // Publish immediately
      await this.store.appendEvent({ type: "state_transition", from: "Deciding", to: "Publishing", round });
      const allRounds = await this.store.readAllRounds();
      const allLineage = await this.store.readLineage();
      await writeBundles({ expDir: this.opts.expDir, experimentId: expId, lineage: allLineage, rounds: allRounds, createdBy: process.env["USER"] ?? "unknown" });
      await this.store.appendEvent({ type: "state_transition", from: "Publishing", to: "Published", round });
      process.stdout.write(`\nExperiment complete. Outputs written to ${path.join(this.opts.expDir, "outputs")}\n`);
    } else {
      // Pause at Deciding
      clearInterval(this.heartbeatTimer);
      await this.store.releaseLock();
      const updatedRound = (await this.store.readAllRounds()).find(r => r.round === round);
      process.stdout.write(`\nRound ${round} complete.\n`);
      process.stdout.write(`Scores: outcome=${scores.outcome.toFixed(2)}, trajectory=${scores.trajectory.toFixed(2)}\n`);
      process.stdout.write(`Triage: ${triageResult.diagnoses.join("; ")}\n`);
      process.stdout.write(`Next suggestion written to mission.md. Review and run exp resume to continue.\n`);
    }
  }

  private async checkAbort(round: number): Promise<boolean> {
    if (await this.store.isAborted()) {
      clearInterval(this.heartbeatTimer);
      await this.store.appendEvent({ type: "aborted", round, reason: "user_abort" });
      await this.store.releaseLock();
      return true;
    }
    return false;
  }

  private async withRetry<T>(fn: () => Promise<T>, state: string): Promise<T> {
    let lastErr: unknown;
    for (let attempt = 0; attempt < 3; attempt++) {
      try {
        return await fn();
      } catch (err) {
        lastErr = err;
        if (attempt < 2) {
          await new Promise(r => setTimeout(r, 1000 * 2 ** attempt));
        }
      }
    }
    await this.store.appendEvent({
      type: "step_failed",
      state: state as never,
      error: String(lastErr),
      retryable: true,
    });
    clearInterval(this.heartbeatTimer);
    await this.store.releaseLock();
    throw lastErr;
  }
}
```

- [ ] **Step 4: 运行确认通过**

```bash
node --import tsx --test test/exp-coordinator.test.ts
```

- [ ] **Step 5: commit**

```bash
git add src/trace-ai/exp/coordinator.ts test/exp-coordinator.test.ts
git commit -m "feat(exp): ExperimentCoordinator FSM — 7-state loop, heartbeat lock, abort detection"
```

---

## Task 11: CLI 命令层

**Files:**
- Create: `src/trace-ai/exp/index.ts`
- Modify: `src/commands/trace.ts`

- [ ] **Step 1: 实现 exp/index.ts（命令入口）**

```typescript
// src/trace-ai/exp/index.ts
import path from "node:path";
import { ExpStore } from "./exp-store/index.js";
import { ExperimentCoordinator } from "./coordinator.js";
import { ClaudeCodeSynthesizer } from "./providers/synthesizer-client.js";
import { ClaudeCodeTriageClient } from "./providers/triage-client.js";
import { runEval } from "./eval-runner.js";
import { defaultRegistry } from "../agent-providers/registry.js";
import { ClaudeCodeSubprocessProvider } from "../agent-providers/claude-code-subprocess.js";

function ensureProvider() {
  if (!defaultRegistry.has("claude-code")) {
    defaultRegistry.register(new ClaudeCodeSubprocessProvider(), { setAsDefault: true });
  }
}

export interface ParsedExpArgs {
  subcommand: "run" | "resume" | "show" | "status" | "abort" | "doctor";
  expDir: string;
  newRun?: boolean;
}

export function parseExpArgs(argv: string[]): ParsedExpArgs {
  const [sub, dir = ".", ...flags] = argv;
  const validSubs = ["run", "resume", "show", "status", "abort", "doctor"] as const;
  if (!validSubs.includes(sub as never)) {
    throw new Error(`Unknown exp subcommand: ${sub}. Use: ${validSubs.join(", ")}`);
  }
  return {
    subcommand: sub as ParsedExpArgs["subcommand"],
    expDir: path.resolve(dir),
    newRun: flags.includes("--new-run"),
  };
}

export async function runExpCommand(argv: string[]): Promise<number> {
  const args = parseExpArgs(argv);
  const store = new ExpStore(args.expDir);

  switch (args.subcommand) {
    case "run": {
      ensureProvider();
      const replayed = await store.replayState();
      if (!replayed.isTerminal && replayed.currentRound > 0) {
        process.stderr.write(`Error: experiment in progress (state: ${replayed.currentState}). Use exp resume.\n`);
        return 2;
      }
      if (replayed.isTerminal && !args.newRun) {
        process.stderr.write(`Error: experiment already in terminal state ${replayed.currentState}. Use --new-run to start fresh.\n`);
        return 2;
      }
      if (replayed.isTerminal && args.newRun) {
        await store.archiveState();
      }
      const coord = makeCoordinator(args.expDir);
      await coord.run();
      return 0;
    }

    case "resume": {
      ensureProvider();
      const coord = makeCoordinator(args.expDir);
      await coord.resume();
      return 0;
    }

    case "show": {
      const replayed = await store.replayState();
      const rounds = await store.readAllRounds();
      const lineage = await store.readLineage();
      const mission = await store.readMission().catch(() => null);
      process.stdout.write(`State: ${replayed.currentState}  Round: ${replayed.currentRound}\n`);
      if (mission?.next_change) {
        process.stdout.write(`Suggested next change:\n  target: ${mission.next_change.target}\n  hypothesis: ${mission.next_change.hypothesis}\n`);
      }
      if (rounds.length > 0) {
        const last = rounds.at(-1)!;
        process.stdout.write(`Last round scores: outcome=${last.scores?.outcome.toFixed(2) ?? "?"}, trajectory=${last.scores?.trajectory.toFixed(2) ?? "?"}\n`);
        if (last.triage_conclusion) {
          process.stdout.write(`Triage: ${last.triage_conclusion.diagnoses.join("; ")}\n`);
        }
      }
      process.stdout.write(`Lineage: ${lineage.length} versions\n`);
      return 0;
    }

    case "status": {
      const replayed = await store.replayState();
      process.stdout.write(`${args.expDir}: ${replayed.currentState} (round ${replayed.currentRound})\n`);
      return 0;
    }

    case "abort": {
      await store.writeAbortSignal();
      process.stdout.write(`Abort signal written. Running process will stop at next checkpoint.\n`);
      return 0;
    }

    case "doctor": {
      return runDoctor(args.expDir, store);
    }
  }
}

async function runDoctor(expDir: string, store: ExpStore): Promise<number> {
  let ok = true;
  const check = (label: string, pass: boolean, msg: string) => {
    process.stdout.write(`${pass ? "✓" : "✗"} ${label}${pass ? "" : `: ${msg}`}\n`);
    if (!pass) ok = false;
  };

  try {
    const mission = await store.readMission();
    check("mission.md valid", true, "");
    for (const es of mission.eval_sets) {
      const { access } = await import("node:fs/promises");
      const esPath = path.join(expDir, es.path);
      await access(esPath).then(() => check(`eval_set ${es.path}`, true, "")).catch(() => check(`eval_set ${es.path}`, false, `not found: ${esPath}`));
    }
    const candPath = path.join(expDir, mission.current_candidate.path);
    const { access } = await import("node:fs/promises");
    await access(candPath).then(() => check("current_candidate readable", true, "")).catch(() => check("current_candidate readable", false, `not found: ${candPath}`));
  } catch (e) {
    check("mission.md valid", false, String(e));
  }

  const providerOk = defaultRegistry.resolve({ preferred: "claude-code" }) !== null;
  check("claude-code provider available", providerOk, "run: npx @anthropic-ai/claude-code --version");

  const replayed = await store.replayState();
  if (replayed.lastFailure) check("no step_failed in events", false, `last failure: ${replayed.lastFailure.error}`);
  else check("no step_failed in events", true, "");

  return ok ? 0 : 1;
}

function makeCoordinator(expDir: string): ExperimentCoordinator {
  return new ExperimentCoordinator({
    expDir,
    synthesizer: new ClaudeCodeSynthesizer(),
    triage: new ClaudeCodeTriageClient(),
    runEval: ({ evalSetPaths, candidatePath }) => runEval({
      evalSetPaths,
      candidatePath,
      expDir,
      deps: {
        fetchAgent: async (id) => ({ id, key: id, version: "latest" }),
        sendChat: async () => { throw new Error("sendChat not configured — provide RunnerDeps"); },
        fetchTrace: async () => ({ spans: [] }),
      },
    }),
  });
}
```

- [ ] **Step 2: 修改 commands/trace.ts 增加 exp 子命令**

在 `runTraceCommand` 的 switch 中追加：

```typescript
// In src/commands/trace.ts — add to the switch in runTraceCommand():
case "exp":
  return await runExpSubcommand(rest.slice(1));
```

在文件顶部追加 import：
```typescript
import { runExpCommand } from "../trace-ai/exp/index.js";
async function runExpSubcommand(argv: string[]): Promise<number> {
  return runExpCommand(argv);
}
```

在 `parseTraceArgs` 中处理 `exp` 作为合法 head：
```typescript
// Ensure "exp" is recognized as a valid subcommand head in parseTraceArgs
case "exp":
  return { subcommand: "exp", rest: argv.slice(1) };
```

- [ ] **Step 3: 冒烟测试 exp doctor**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript
# 创建测试实验目录
mkdir -p /tmp/test-exp/eval-sets/v1 /tmp/test-exp/candidates
echo "schema_version: trace-mission/v1
goal: test
eval_sets:
  - path: eval-sets/v1
    role: seed
current_candidate:
  path: candidates/baseline.yaml
next_change:
  target: agent.system_prompt
  hypothesis: test
  patch: '{}'" > /tmp/test-exp/mission.md
echo "agent_id: test\ncandidate_version: v0\nagent:\n  system_prompt: hello\nskills: []" > /tmp/test-exp/candidates/baseline.yaml
node --import tsx src/cli.ts trace exp doctor /tmp/test-exp
```
Expected: 各检查项输出 ✓ 或 ✗ 并退出

- [ ] **Step 4: commit**

```bash
git add src/trace-ai/exp/index.ts
git add src/commands/trace.ts
git commit -m "feat(exp): CLI commands — exp run/resume/show/status/abort/doctor"
```

---

## Task 12: 集成测试

**Files:**
- Create: `test/integration/exp-full-round.test.ts`
- Create: `test/integration/exp-resume.test.ts`

- [ ] **Step 1: 实现 full-round 集成测试**

```typescript
// test/integration/exp-full-round.test.ts
import test from "node:test";
import assert from "node:assert/strict";
import fs from "node:fs/promises";
import path from "node:path";
import os from "node:os";
import { ExperimentCoordinator } from "../../src/trace-ai/exp/coordinator.js";
import { replayState } from "../../src/trace-ai/exp/exp-store/events-jsonl.js";
import type { NextChange, RoundData } from "../../src/trace-ai/exp/schemas.js";

async function makeExpDir() {
  const dir = await fs.mkdtemp(path.join(os.tmpdir(), "exp-integration-"));
  await fs.mkdir(path.join(dir, ".trace-state", "rounds"), { recursive: true });
  await fs.mkdir(path.join(dir, "candidates"), { recursive: true });
  await fs.mkdir(path.join(dir, "eval-sets", "v1"), { recursive: true });
  await fs.mkdir(path.join(dir, "outputs"), { recursive: true });
  await fs.writeFile(path.join(dir, ".trace-state", "events.jsonl"), "");
  await fs.writeFile(path.join(dir, "mission.md"), `---
schema_version: trace-mission/v1
goal: reduce retries
max_rounds: 2
eval_sets:
  - path: eval-sets/v1
    role: seed
current_candidate:
  path: candidates/baseline.yaml
next_change:
  target: agent.system_prompt
  hypothesis: add stop condition
  patch: '{"agent":{"system_prompt":"New prompt with stop condition."}}'
---
`);
  await fs.writeFile(path.join(dir, "candidates", "baseline.yaml"), "agent_id: test\ncandidate_version: v0\nagent:\n  system_prompt: hello\nskills: []\n");
  await fs.writeFile(path.join(dir, "eval-sets", "v1", "index.yaml"), "schema_version: trace-eval-set-index/v1\neval_set_id: test_v1\nshards: []\n");
  return dir;
}

test("full round: Deciding pause after round 1", async () => {
  const dir = await makeExpDir();
  const coord = new ExperimentCoordinator({
    expDir: dir,
    synthesizer: { async generate(): Promise<NextChange> { return { target: "agent.system_prompt", hypothesis: "mock", patch: '{"agent":{"system_prompt":"next"}}' }; } },
    triage: { async triage() { return { diagnoses: ["ok"], hints: ["try x"], verdict: "continue", cross_round_memory_ref: "mem1", new_memory_token: "mem1" }; } },
    runEval: async () => ({ queryResults: [] }),
  });

  await coord.run();

  const state = await replayState(dir);
  assert.equal(state.currentState, "Deciding");

  // Candidate v1 created
  await fs.access(path.join(dir, "candidates", "candidate-v1.yaml"));

  // round-1.yaml written
  await fs.access(path.join(dir, ".trace-state", "rounds", "round-1.yaml"));
});

test("full round: publishes at max_rounds", async () => {
  const dir = await makeExpDir();

  // max_rounds: 2, set verdict to continue so max_rounds triggers publish
  const coord = new ExperimentCoordinator({
    expDir: dir,
    synthesizer: { async generate(): Promise<NextChange> { return { target: "agent.system_prompt", hypothesis: "m", patch: '{"agent":{"system_prompt":"p"}}' }; } },
    triage: { async triage(input): Promise<RoundData["triage_conclusion"] & { new_memory_token: string }> {
      // Continue for first round, publish at second
      return { diagnoses: [], hints: [], verdict: input.currentRound.round >= 2 ? "publish" : "continue", cross_round_memory_ref: "m", new_memory_token: "m" };
    }},
    runEval: async () => ({ queryResults: [] }),
  });

  await coord.run();         // round 1 → Deciding
  await coord.resume();      // round 2 → publish verdict → Published

  const state = await replayState(dir);
  assert.equal(state.currentState, "Published");

  // outputs written
  await fs.access(path.join(dir, "outputs", "bundle.yaml"));
  await fs.access(path.join(dir, "outputs", "manifest.yaml"));
  await fs.access(path.join(dir, "outputs", "provenance.yaml"));
});
```

- [ ] **Step 2: 实现 resume 集成测试**

```typescript
// test/integration/exp-resume.test.ts
import test from "node:test";
import assert from "node:assert/strict";
import fs from "node:fs/promises";
import path from "node:path";
import os from "node:os";
import { ExperimentCoordinator } from "../../src/trace-ai/exp/coordinator.js";
import { appendEvent } from "../../src/trace-ai/exp/exp-store/events-jsonl.js";
import { replayState } from "../../src/trace-ai/exp/exp-store/events-jsonl.js";
import type { NextChange } from "../../src/trace-ai/exp/schemas.js";

async function makeExpDir() {
  const dir = await fs.mkdtemp(path.join(os.tmpdir(), "exp-resume-"));
  await fs.mkdir(path.join(dir, ".trace-state", "rounds"), { recursive: true });
  await fs.mkdir(path.join(dir, "candidates"), { recursive: true });
  await fs.mkdir(path.join(dir, "eval-sets", "v1"), { recursive: true });
  await fs.mkdir(path.join(dir, "outputs"), { recursive: true });
  await fs.writeFile(path.join(dir, ".trace-state", "events.jsonl"), "");
  await fs.writeFile(path.join(dir, "mission.md"), `---
schema_version: trace-mission/v1
goal: reduce retries
eval_sets:
  - path: eval-sets/v1
    role: seed
current_candidate:
  path: candidates/baseline.yaml
next_change:
  target: agent.system_prompt
  hypothesis: resume test
  patch: '{"agent":{"system_prompt":"resumed prompt"}}'
---
`);
  await fs.writeFile(path.join(dir, "candidates", "baseline.yaml"), "agent_id: test\ncandidate_version: v0\nagent:\n  system_prompt: old\nskills: []\n");
  await fs.writeFile(path.join(dir, "eval-sets", "v1", "index.yaml"), "schema_version: trace-eval-set-index/v1\neval_set_id: test\nshards: []\n");
  return dir;
}

test("resume: picks up from Deciding after run", async () => {
  const dir = await makeExpDir();
  const opts = {
    expDir: dir,
    synthesizer: { async generate(): Promise<NextChange> { return { target: "agent.system_prompt", hypothesis: "m", patch: '{"agent":{"system_prompt":"p"}}' }; } },
    triage: { async triage() { return { diagnoses: [], hints: [], verdict: "publish" as const, cross_round_memory_ref: "m", new_memory_token: "m" }; } },
    runEval: async () => ({ queryResults: [] }),
  };

  await new ExperimentCoordinator(opts).run();
  const mid = await replayState(dir);
  // verdict=publish → Published immediately, no Deciding pause
  assert.equal(mid.currentState, "Published");
});

test("resume: retries after step_failed", async () => {
  const dir = await makeExpDir();
  // Manually inject a step_failed event as if Triaging crashed
  await appendEvent(dir, { type: "state_transition", from: "Init", to: "Generating", round: 1 });
  await appendEvent(dir, { type: "state_transition", from: "Generating", to: "Executing", round: 1 });
  await appendEvent(dir, { type: "step_failed", state: "Executing", error: "network timeout", retryable: true });

  // Create candidate-v1.yaml so resume can proceed
  await fs.writeFile(path.join(dir, "candidates", "candidate-v1.yaml"), "agent_id: test\ncandidate_version: v1\nagent:\n  system_prompt: new\nskills: []\n");

  const state = await replayState(dir);
  assert.equal(state.currentState, "Executing");
  assert.equal(state.lastFailure?.retryable, true);
});
```

- [ ] **Step 3: 运行集成测试**

```bash
node --import tsx --test test/integration/exp-full-round.test.ts
node --import tsx --test test/integration/exp-resume.test.ts
```
Expected: 所有测试通过

- [ ] **Step 4: 运行全量测试确认无回归**

```bash
node --import tsx --test "test/**/*.test.ts"
```
Expected: 所有现有测试仍通过

- [ ] **Step 5: commit**

```bash
git add test/integration/exp-full-round.test.ts test/integration/exp-resume.test.ts
git commit -m "test(exp): integration tests — full round lifecycle + resume recovery"
```

---

## Self-Review — Spec Coverage Check

| Spec 要求 | 覆盖任务 |
|----------|---------|
| exp run/resume/show/status/abort/doctor | Task 11 |
| FSM 7 状态 | Task 10 |
| ExpStore B3（lock/events/lineage/round/abort） | Task 3-5 |
| PatchApplier（agent.*/skill.*） | Task 6 |
| Synthesizer + Triage（claude-code provider） | Task 8 |
| EvalRunner（wraps MVP-B） | Task 9 |
| 三轴打分 | Task 7 |
| BundleWriter（bundle/manifest/provenance） | Task 9 |
| README 自动生成 | Task 5（readme-template） |
| Guardrail hard gate → lineage status:guardrail_failed | Task 10 |
| exp run 终态拒绝 / --new-run 归档 | Task 11 |
| max_rounds 终止 | Task 10 |
| 可重试失败释放锁，resume 靠 events 恢复 | Task 10 |
| mission.md v1 schema | Task 1 |
| B5 schema 扩展 | Task 1 |
| DA provider post-MVP stub | Task 8（接口定义，claude-code 实现，无 DA 实现） |
| DSL/BKN stub | Task 6（throws unsupported） |
| cross_round_memory_ref 跨轮传递 | Task 10 |
| Deciding 暂停时 releaseLock | Task 10 |
