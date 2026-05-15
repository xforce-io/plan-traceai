# trace exp Agent Discovery Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add exp-registry + `trace exp list` + `trace exp info` so agents can discover and understand experiment state without knowing the workspace path.

**Architecture:** Three layered additions: (1) `exp-registry.ts` — a tiny read/write module for `~/.kweaver/exp-registry.json`; (2) `info.ts` — builds `ExpSnapshot` from live `.trace-state/` and formats output; (3) `index.ts` changes — dispatch list/info + upsert side-effect on run/resume.

**Tech Stack:** TypeScript, node:fs/promises, js-yaml, node:test (tests)

---

## File Structure

```
src/trace-ai/exp/
  exp-store/
    exp-registry.ts          # NEW: upsertRegistry() / listRegistry()
  info.ts                    # NEW: ExpSnapshot, buildExpSnapshot(), runInfo(), runList(), getHealthChecks()
  index.ts                   # MODIFY: add list/info dispatch, upsert on run/resume, reuse getHealthChecks in doctor

test/
  exp-registry.test.ts       # NEW: upsert dedup, sort, swallow-error, missing file
  exp-info.test.ts           # NEW: buildExpSnapshot, runInfo, runList missing-path, empty registry
  exp-list-info-dispatch.test.ts  # NEW: parseExpArgs extensions
```

**Key interfaces:**
- `RegistryEntry { path: string; last_active_ts: string }`
- `Registry { schema_version: "exp-registry/v1"; entries: RegistryEntry[] }`
- `ExpSnapshot { workspace, state, round, scores, triage_summary, suggested_next, lineage_versions, health }`
- `HealthChecks { mission_valid, eval_set_valid, candidate_readable, provider_available, no_step_failed }`

**Codebase reference:**
- Source root: `/Users/xupeng/dev/github/kweaver-sdk/packages/typescript/`
- Config dir utility: `src/config/store.ts` → `getConfigDir(): string`
- Test runner: `node --import tsx --test test/*.test.ts`
- Build: `npm run build`

---

## Task 1: exp-registry.ts — Read/Write Module

**Files:**
- Create: `src/trace-ai/exp/exp-store/exp-registry.ts`
- Create: `test/exp-registry.test.ts`

- [ ] **Step 1.1: Write the failing tests**

```typescript
// test/exp-registry.test.ts
import test from "node:test";
import assert from "node:assert/strict";
import fs from "node:fs/promises";
import path from "node:path";
import os from "node:os";

async function makeTmpDir() {
  return fs.mkdtemp(path.join(os.tmpdir(), "exp-registry-"));
}

function withConfigDir<T>(dir: string, fn: () => Promise<T>): Promise<T> {
  const prev = process.env["KWEAVERC_CONFIG_DIR"];
  process.env["KWEAVERC_CONFIG_DIR"] = dir;
  return fn().finally(() => {
    if (prev === undefined) delete process.env["KWEAVERC_CONFIG_DIR"];
    else process.env["KWEAVERC_CONFIG_DIR"] = prev;
  });
}

test("listRegistry: returns [] when file missing", async () => {
  const dir = await makeTmpDir();
  const { listRegistry } = await import("../src/trace-ai/exp/exp-store/exp-registry.js");
  const entries = await withConfigDir(dir, () => listRegistry());
  assert.deepEqual(entries, []);
});

test("upsertRegistry: creates file and adds entry", async () => {
  const dir = await makeTmpDir();
  const { upsertRegistry, listRegistry } = await import("../src/trace-ai/exp/exp-store/exp-registry.js");
  await withConfigDir(dir, async () => {
    await upsertRegistry("/some/exp/path", "2026-05-15T10:00:00.000Z");
    const entries = await listRegistry();
    assert.equal(entries.length, 1);
    assert.equal(entries[0].path, "/some/exp/path");
    assert.equal(entries[0].last_active_ts, "2026-05-15T10:00:00.000Z");
  });
});

test("upsertRegistry: deduplicates by path, updates timestamp", async () => {
  const dir = await makeTmpDir();
  const { upsertRegistry, listRegistry } = await import("../src/trace-ai/exp/exp-store/exp-registry.js");
  await withConfigDir(dir, async () => {
    await upsertRegistry("/same/path", "2026-05-15T09:00:00.000Z");
    await upsertRegistry("/same/path", "2026-05-15T10:00:00.000Z");
    const entries = await listRegistry();
    assert.equal(entries.length, 1);
    assert.equal(entries[0].last_active_ts, "2026-05-15T10:00:00.000Z");
  });
});

test("listRegistry: sorted by last_active_ts descending", async () => {
  const dir = await makeTmpDir();
  const { upsertRegistry, listRegistry } = await import("../src/trace-ai/exp/exp-store/exp-registry.js");
  await withConfigDir(dir, async () => {
    await upsertRegistry("/old/path", "2026-05-14T10:00:00.000Z");
    await upsertRegistry("/new/path", "2026-05-15T10:00:00.000Z");
    const entries = await listRegistry();
    assert.equal(entries[0].path, "/new/path");
    assert.equal(entries[1].path, "/old/path");
  });
});

test("upsertRegistry: swallows errors silently (read-only dir)", async () => {
  const dir = await makeTmpDir();
  const { upsertRegistry } = await import("../src/trace-ai/exp/exp-store/exp-registry.js");
  const badDir = path.join(dir, "not-a-dir");
  await fs.writeFile(badDir, "blocking file");
  await assert.doesNotReject(() =>
    withConfigDir(badDir, () => upsertRegistry("/exp/path", "2026-05-15T10:00:00.000Z"))
  );
});

test("listRegistry: returns [] on malformed JSON", async () => {
  const dir = await makeTmpDir();
  const { listRegistry } = await import("../src/trace-ai/exp/exp-store/exp-registry.js");
  await fs.writeFile(path.join(dir, "exp-registry.json"), "{ not json }", "utf8");
  const entries = await withConfigDir(dir, () => listRegistry());
  assert.deepEqual(entries, []);
});
```

- [ ] **Step 1.2: Run tests to confirm they fail**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript && node --import tsx --test test/exp-registry.test.ts 2>&1 | tail -20
```

Expected: all 6 tests fail with "Cannot find module".

- [ ] **Step 1.3: Implement exp-registry.ts**

```typescript
// src/trace-ai/exp/exp-store/exp-registry.ts
import fs from "node:fs/promises";
import path from "node:path";
import { getConfigDir } from "../../../config/store.js";

export interface RegistryEntry {
  path: string;
  last_active_ts: string;
}

interface Registry {
  schema_version: "exp-registry/v1";
  entries: RegistryEntry[];
}

function registryFilePath(): string {
  return path.join(getConfigDir(), "exp-registry.json");
}

async function readRegistry(): Promise<Registry> {
  try {
    const raw = await fs.readFile(registryFilePath(), "utf8");
    const parsed = JSON.parse(raw) as Registry;
    if (!Array.isArray(parsed.entries)) return { schema_version: "exp-registry/v1", entries: [] };
    return parsed;
  } catch {
    return { schema_version: "exp-registry/v1", entries: [] };
  }
}

export async function upsertRegistry(absPath: string, ts: string): Promise<void> {
  try {
    const reg = await readRegistry();
    const idx = reg.entries.findIndex((e) => e.path === absPath);
    if (idx >= 0) {
      reg.entries[idx].last_active_ts = ts;
    } else {
      reg.entries.push({ path: absPath, last_active_ts: ts });
    }
    const filePath = registryFilePath();
    await fs.mkdir(path.dirname(filePath), { recursive: true });
    await fs.writeFile(filePath, JSON.stringify(reg, null, 2) + "\n", "utf8");
  } catch (e) {
    process.stderr.write(`warn: exp-registry write failed: ${(e as Error).message}\n`);
  }
}

export async function listRegistry(): Promise<RegistryEntry[]> {
  const reg = await readRegistry();
  return [...reg.entries].sort((a, b) =>
    b.last_active_ts.localeCompare(a.last_active_ts)
  );
}
```

- [ ] **Step 1.4: Run tests to confirm they pass**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript && node --import tsx --test test/exp-registry.test.ts 2>&1
```

Expected: `# tests 6`, `# pass 6`, `# fail 0`.

- [ ] **Step 1.5: Build check**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript && npm run build 2>&1 | tail -5
```

Expected: no errors.

- [ ] **Step 1.6: Commit**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript && git add src/trace-ai/exp/exp-store/exp-registry.ts test/exp-registry.test.ts && git commit -m "feat(exp): add exp-registry read/write module for global experiment discovery"
```

---

## Task 2: info.ts — getHealthChecks + buildExpSnapshot + format

**Files:**
- Create: `src/trace-ai/exp/info.ts`
- Create: `test/exp-info.test.ts`

- [ ] **Step 2.1: Write the failing tests**

```typescript
// test/exp-info.test.ts
import test from "node:test";
import assert from "node:assert/strict";
import fs from "node:fs/promises";
import path from "node:path";
import os from "node:os";

const MISSION_CONTENT = `---
schema_version: trace-mission/v1
goal: reduce retries
eval_sets:
  - path: eval-sets/v1
    role: seed
current_candidate:
  path: candidates/baseline.yaml
---
`;

async function makeExpDir(): Promise<string> {
  const dir = await fs.mkdtemp(path.join(os.tmpdir(), "exp-info-"));
  await fs.mkdir(path.join(dir, ".trace-state", "rounds"), { recursive: true });
  await fs.mkdir(path.join(dir, "candidates"), { recursive: true });
  await fs.mkdir(path.join(dir, "eval-sets", "v1"), { recursive: true });
  await fs.writeFile(path.join(dir, ".trace-state", "events.jsonl"), "");
  await fs.writeFile(path.join(dir, "mission.md"), MISSION_CONTENT);
  await fs.writeFile(path.join(dir, "candidates", "baseline.yaml"), "agent_id: test\n");
  await fs.writeFile(path.join(dir, "eval-sets", "v1", "index.yaml"), "schema_version: trace-eval-set-index/v1\neval_set_id: test\nshards: []\n");
  return dir;
}

test("getHealthChecks: all pass on valid experiment dir", async () => {
  const dir = await makeExpDir();
  const { getHealthChecks } = await import("../src/trace-ai/exp/info.js");
  const health = await getHealthChecks(dir);
  assert.equal(health.mission_valid, true);
  assert.equal(health.eval_set_valid, true);
  assert.equal(health.candidate_readable, true);
  assert.equal(health.no_step_failed, true);
  assert.equal(typeof health.provider_available, "boolean");
});

test("getHealthChecks: mission_valid=false when mission.md missing", async () => {
  const dir = await fs.mkdtemp(path.join(os.tmpdir(), "exp-info-nomission-"));
  await fs.mkdir(path.join(dir, ".trace-state"), { recursive: true });
  await fs.writeFile(path.join(dir, ".trace-state", "events.jsonl"), "");
  const { getHealthChecks } = await import("../src/trace-ai/exp/info.js");
  const health = await getHealthChecks(dir);
  assert.equal(health.mission_valid, false);
  assert.equal(health.eval_set_valid, false);
  assert.equal(health.candidate_readable, false);
});

test("getHealthChecks: no_step_failed=false when step_failed in events", async () => {
  const dir = await makeExpDir();
  const eventsPath = path.join(dir, ".trace-state", "events.jsonl");
  await fs.appendFile(eventsPath, JSON.stringify({ ts: new Date().toISOString(), type: "step_failed", state: "Generating", error: "timeout", retryable: true }) + "\n");
  const { getHealthChecks } = await import("../src/trace-ai/exp/info.js");
  const health = await getHealthChecks(dir);
  assert.equal(health.no_step_failed, false);
});

test("buildExpSnapshot: returns correct shape for fresh experiment", async () => {
  const dir = await makeExpDir();
  const { buildExpSnapshot } = await import("../src/trace-ai/exp/info.js");
  const snap = await buildExpSnapshot(dir);
  assert.equal(snap.workspace, dir);
  assert.equal(snap.state, "Init");
  assert.equal(snap.round, 0);
  assert.equal(snap.scores, null);
  assert.equal(snap.triage_summary, null);
  assert.equal(snap.suggested_next, null);
  assert.equal(snap.lineage_versions, 0);
  assert.equal(typeof snap.health.mission_valid, "boolean");
});

test("buildExpSnapshot: picks up scores from last round", async () => {
  const dir = await makeExpDir();
  const { ExpStore } = await import("../src/trace-ai/exp/exp-store/index.js");
  const store = new ExpStore(dir);
  await store.writeRound(1, {
    round: 1,
    trial_version: 1,
    scores: { outcome: 0.85, trajectory: 0.9, guardrail: 1.0, guardrail_hard_fail: false },
    triage_conclusion: { diagnoses: ["retry too high"], hints: [], verdict: "continue" },
  });
  const { buildExpSnapshot } = await import("../src/trace-ai/exp/info.js");
  const snap = await buildExpSnapshot(dir);
  assert.ok(snap.scores !== null);
  assert.equal(snap.scores!.outcome, 0.85);
  assert.equal(snap.triage_summary, "retry too high");
});

test("buildExpSnapshot: picks up suggested_next from mission next_change", async () => {
  const dir = await makeExpDir();
  const missionWithChange = `---
schema_version: trace-mission/v1
goal: reduce retries
eval_sets:
  - path: eval-sets/v1
    role: seed
current_candidate:
  path: candidates/baseline.yaml
next_change:
  target: agent.system_prompt
  hypothesis: try shorter prompt
  patch: '{"agent":{"system_prompt":"short"}}'
---
`;
  await fs.writeFile(path.join(dir, "mission.md"), missionWithChange);
  const { buildExpSnapshot } = await import("../src/trace-ai/exp/info.js");
  const snap = await buildExpSnapshot(dir);
  assert.ok(snap.suggested_next !== null);
  assert.equal(snap.suggested_next!.target, "agent.system_prompt");
  assert.equal(snap.suggested_next!.hypothesis, "try shorter prompt");
});

test("formatSnapshotYaml: output contains key fields", async () => {
  const dir = await makeExpDir();
  const { buildExpSnapshot, formatSnapshotYaml } = await import("../src/trace-ai/exp/info.js");
  const snap = await buildExpSnapshot(dir);
  const out = formatSnapshotYaml(snap);
  assert.ok(out.includes("workspace:"));
  assert.ok(out.includes("state:"));
  assert.ok(out.includes("round:"));
  assert.ok(out.includes("health:"));
});

test("runList: prints (missing) row for nonexistent path", async () => {
  const { runList } = await import("../src/trace-ai/exp/info.js");
  const chunks: string[] = [];
  const orig = process.stdout.write.bind(process.stdout);
  (process.stdout as any).write = (chunk: string | Uint8Array): boolean => { chunks.push(String(chunk)); return true; };
  try {
    await runList([{ path: "/definitely/does/not/exist", last_active_ts: "2026-05-15T10:00:00.000Z" }]);
  } finally {
    (process.stdout as any).write = orig;
  }
  const output = chunks.join("");
  assert.ok(output.includes("(missing)"), `Expected (missing) in output: ${output}`);
});

test("runInfo: outputs yaml for a valid experiment dir", async () => {
  const dir = await makeExpDir();
  const { runInfo } = await import("../src/trace-ai/exp/info.js");
  const chunks: string[] = [];
  const orig = process.stdout.write.bind(process.stdout);
  (process.stdout as any).write = (chunk: string | Uint8Array): boolean => { chunks.push(String(chunk)); return true; };
  try {
    await runInfo(dir);
  } finally {
    (process.stdout as any).write = orig;
  }
  const output = chunks.join("");
  assert.ok(output.includes("workspace:"), `Expected workspace: in output`);
  assert.ok(output.includes("state:"), `Expected state: in output`);
});

test("runInfo: outputs JSON when opts.json=true", async () => {
  const dir = await makeExpDir();
  const { runInfo } = await import("../src/trace-ai/exp/info.js");
  const chunks: string[] = [];
  const orig = process.stdout.write.bind(process.stdout);
  (process.stdout as any).write = (chunk: string | Uint8Array): boolean => { chunks.push(String(chunk)); return true; };
  try {
    await runInfo(dir, { json: true });
  } finally {
    (process.stdout as any).write = orig;
  }
  const parsed = JSON.parse(chunks.join(""));
  assert.equal(parsed.state, "Init");
  assert.equal(parsed.workspace, dir);
});
```

- [ ] **Step 2.2: Run tests to confirm they fail**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript && node --import tsx --test test/exp-info.test.ts 2>&1 | tail -20
```

Expected: all 10 tests fail with "Cannot find module".

- [ ] **Step 2.3: Implement info.ts**

```typescript
// src/trace-ai/exp/info.ts
import path from "node:path";
import fs from "node:fs/promises";
import yaml from "js-yaml";
import { ExpStore } from "./exp-store/index.js";
import { defaultRegistry } from "../../agent-providers/registry.js";
import type { ThreeAxisScores } from "./schemas.js";

export interface HealthChecks {
  mission_valid: boolean;
  eval_set_valid: boolean;
  candidate_readable: boolean;
  provider_available: boolean;
  no_step_failed: boolean;
}

export interface ExpSnapshot {
  workspace: string;
  state: string;
  round: number;
  scores: ThreeAxisScores | null;
  triage_summary: string | null;
  suggested_next: { target: string; hypothesis: string } | null;
  lineage_versions: number;
  health: HealthChecks;
}

export async function getHealthChecks(expDir: string): Promise<HealthChecks> {
  const store = new ExpStore(expDir);
  let mission_valid = false;
  let eval_set_valid = false;
  let candidate_readable = false;

  try {
    const mission = await store.readMission();
    mission_valid = true;
    let allEvalSetsOk = true;
    for (const es of mission.eval_sets) {
      try { await fs.access(path.join(expDir, es.path)); }
      catch { allEvalSetsOk = false; }
    }
    eval_set_valid = allEvalSetsOk;
    try {
      await fs.access(path.join(expDir, mission.current_candidate.path));
      candidate_readable = true;
    } catch {
      candidate_readable = false;
    }
  } catch { /* mission_valid stays false */ }

  let provider_available = false;
  try { provider_available = defaultRegistry.resolve({ preferred: "claude-code" }) !== null; }
  catch { provider_available = false; }

  const replayed = await store.replayState();
  const no_step_failed = replayed.lastFailure === null;

  return { mission_valid, eval_set_valid, candidate_readable, provider_available, no_step_failed };
}

export async function buildExpSnapshot(expDir: string): Promise<ExpSnapshot> {
  const store = new ExpStore(expDir);
  const replayed = await store.replayState();
  const rounds = await store.readAllRounds();
  const lineage = await store.readLineage();
  const mission = await store.readMission().catch(() => null);
  const health = await getHealthChecks(expDir);

  const lastRound = rounds.length > 0 ? rounds[rounds.length - 1] : null;
  const scores = lastRound?.scores ?? null;
  const triage_summary = lastRound?.triage_conclusion?.diagnoses.join("; ") ?? null;
  const suggested_next = mission?.next_change
    ? { target: mission.next_change.target, hypothesis: mission.next_change.hypothesis }
    : null;

  return {
    workspace: expDir,
    state: replayed.currentState,
    round: replayed.currentRound,
    scores: scores ?? null,
    triage_summary,
    suggested_next,
    lineage_versions: lineage.length,
    health,
  };
}

export function formatSnapshotYaml(snap: ExpSnapshot): string {
  return yaml.dump(snap, { lineWidth: -1 });
}

export function formatSnapshotTableRow(
  entry: { path: string; last_active_ts: string },
  snap: ExpSnapshot | null
): string {
  if (snap === null) {
    return [entry.path.padEnd(50), "(missing)"].join("  ");
  }
  const outcome = snap.scores?.outcome.toFixed(2) ?? "-";
  const trajectory = snap.scores?.trajectory.toFixed(2) ?? "-";
  const lastActive = entry.last_active_ts.replace("T", " ").slice(0, 19);
  return [
    entry.path.padEnd(50),
    snap.state.padEnd(12),
    String(snap.round).padEnd(6),
    outcome.padEnd(8),
    trajectory.padEnd(10),
    lastActive,
  ].join("  ");
}

export async function runInfo(expDir: string, opts: { json?: boolean } = {}): Promise<void> {
  const snap = await buildExpSnapshot(expDir);
  if (opts.json) {
    process.stdout.write(JSON.stringify(snap, null, 2) + "\n");
  } else {
    process.stdout.write(formatSnapshotYaml(snap));
  }
}

export async function runList(
  registryEntries: Array<{ path: string; last_active_ts: string }>
): Promise<void> {
  const header = [
    "PATH".padEnd(50),
    "STATE".padEnd(12),
    "ROUND".padEnd(6),
    "OUTCOME".padEnd(8),
    "TRAJECTORY".padEnd(10),
    "LAST_ACTIVE",
  ].join("  ");
  process.stdout.write(header + "\n");
  process.stdout.write("-".repeat(header.length) + "\n");
  for (const entry of registryEntries) {
    let snap: ExpSnapshot | null = null;
    try { snap = await buildExpSnapshot(entry.path); } catch { /* missing path */ }
    process.stdout.write(formatSnapshotTableRow(entry, snap) + "\n");
  }
}
```

- [ ] **Step 2.4: Run tests to confirm they pass**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript && node --import tsx --test test/exp-info.test.ts 2>&1
```

Expected: `# tests 10`, `# pass 10`, `# fail 0`.

- [ ] **Step 2.5: Build check**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript && npm run build 2>&1 | tail -5
```

Expected: no errors.

- [ ] **Step 2.6: Commit**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript && git add src/trace-ai/exp/info.ts test/exp-info.test.ts && git commit -m "feat(exp): add info.ts — getHealthChecks, buildExpSnapshot, runInfo, runList"
```

---

## Task 3: index.ts — Add list/info Dispatch + Upsert on run/resume

**Files:**
- Modify: `src/trace-ai/exp/index.ts`
- Create: `test/exp-list-info-dispatch.test.ts`

- [ ] **Step 3.1: Write dispatch tests**

```typescript
// test/exp-list-info-dispatch.test.ts
import test from "node:test";
import assert from "node:assert/strict";
import { parseExpArgs } from "../src/trace-ai/exp/index.js";

test("parseExpArgs: parses 'list' with no dir → expDir is empty string", () => {
  const result = parseExpArgs(["list"]);
  assert.equal(result.subcommand, "list");
  assert.equal(result.expDir, "");
});

test("parseExpArgs: parses 'list' with explicit path → expDir is resolved", () => {
  const result = parseExpArgs(["list", "/some/path"]);
  assert.equal(result.subcommand, "list");
  assert.equal(result.expDir, "/some/path");
});

test("parseExpArgs: parses 'info' with no dir → expDir is empty string", () => {
  const result = parseExpArgs(["info"]);
  assert.equal(result.subcommand, "info");
  assert.equal(result.expDir, "");
});

test("parseExpArgs: parses 'info' with explicit path → expDir is resolved", () => {
  const result = parseExpArgs(["info", "/some/path"]);
  assert.equal(result.subcommand, "info");
  assert.equal(result.expDir, "/some/path");
});

test("parseExpArgs: error message includes list and info", () => {
  assert.throws(
    () => parseExpArgs(["bogus"]),
    /list.*info|info.*list/i,
  );
});

test("parseExpArgs: still accepts all legacy subcommands", () => {
  for (const sub of ["run", "resume", "show", "status", "abort", "doctor"]) {
    const result = parseExpArgs([sub]);
    assert.equal(result.subcommand, sub);
  }
});
```

- [ ] **Step 3.2: Run dispatch tests to confirm they fail**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript && node --import tsx --test test/exp-list-info-dispatch.test.ts 2>&1 | tail -20
```

Expected: list/info tests fail (unknown subcommand), error-message test fails.

- [ ] **Step 3.3: Update index.ts**

Replace the entire contents of `src/trace-ai/exp/index.ts` with:

```typescript
// src/trace-ai/exp/index.ts
import path from "node:path";
import fs from "node:fs/promises";
import { fileURLToPath } from "node:url";
import { ExpStore } from "./exp-store/index.js";
import { ExperimentCoordinator } from "./coordinator.js";
import { ClaudeCodeSynthesizer } from "./providers/synthesizer-client.js";
import { ClaudeCodeTriageClient } from "./providers/triage-client.js";
import { runEval } from "./eval-runner.js";
import { defaultRegistry } from "../../agent-providers/registry.js";
import { ClaudeCodeSubprocessProvider } from "../../agent-providers/providers/claude-code-subprocess.js";
import { PromptTemplateRegistry } from "../../agent-providers/prompt-template.js";
import { createBuiltinSemanticMatchProvider } from "../eval-set/semantic-match-provider.js";
import type { SemanticMatchProvider } from "../eval-set/assertion-evaluator.js";
import { ensureValidToken } from "../../auth/oauth.js";
import { fetchAgentInfo, sendChatRequest } from "../../api/agent-chat.js";
import { getTracesByConversation } from "../../api/conversations.js";
import { upsertRegistry, listRegistry } from "./exp-store/exp-registry.js";
import { runInfo, runList, getHealthChecks } from "./info.js";

const __expIndexDir = path.dirname(fileURLToPath(import.meta.url));
const EVAL_SET_RUBRIC_DIR = path.join(__expIndexDir, "..", "eval-set", "rubric-templates");

function ensureProvider() {
  if (!defaultRegistry.has("claude-code")) {
    defaultRegistry.register(new ClaudeCodeSubprocessProvider(), { setAsDefault: true });
  }
}

export interface ParsedExpArgs {
  subcommand: "run" | "resume" | "show" | "status" | "abort" | "doctor" | "list" | "info";
  expDir: string;
  newRun?: boolean;
}

export function parseExpArgs(argv: string[]): ParsedExpArgs {
  const [sub, dir, ...flags] = argv;
  const validSubs = ["run", "resume", "show", "status", "abort", "doctor", "list", "info"] as const;
  if (!validSubs.includes(sub as never)) {
    throw new Error(`Unknown exp subcommand: ${sub}. Use: ${validSubs.join(", ")}`);
  }
  const isDiscoveryCmd = sub === "list" || sub === "info";
  const expDir = isDiscoveryCmd
    ? (dir ? path.resolve(dir) : "")
    : path.resolve(dir ?? ".");
  return { subcommand: sub as ParsedExpArgs["subcommand"], expDir, newRun: flags.includes("--new-run") };
}

export async function runExpCommand(argv: string[]): Promise<number> {
  const args = parseExpArgs(argv);

  switch (args.subcommand) {
    case "list": {
      if (args.expDir) {
        await runList([{ path: args.expDir, last_active_ts: new Date().toISOString() }]);
      } else {
        const entries = await listRegistry();
        await runList(entries);
      }
      return 0;
    }

    case "info": {
      let expDir = args.expDir;
      if (!expDir) {
        const entries = await listRegistry();
        if (entries.length === 0) {
          process.stderr.write("Error: no experiments in registry. Run 'trace exp run <dir>' first, or provide a path: trace exp info <dir>\n");
          return 1;
        }
        expDir = entries[0].path;
        process.stderr.write(`Using most recent: ${expDir}\n`);
      }
      await runInfo(expDir);
      return 0;
    }

    case "run": {
      ensureProvider();
      const store = new ExpStore(args.expDir);
      const replayed = await store.replayState();
      if (!replayed.isTerminal && replayed.currentRound > 0 && !replayed.lastFailure) {
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
      await upsertRegistry(args.expDir, new Date().toISOString());
      const coord = await makeCoordinator(args.expDir);
      await coord.run();
      return 0;
    }

    case "resume": {
      ensureProvider();
      const store = new ExpStore(args.expDir);
      const replayed = await store.replayState();
      if (replayed.currentState !== "Deciding") {
        process.stderr.write(`Error: cannot resume — experiment is in state ${replayed.currentState}. Only Deciding state supports resume.\n`);
        return 2;
      }
      await upsertRegistry(args.expDir, new Date().toISOString());
      const coord = await makeCoordinator(args.expDir);
      await coord.resume();
      return 0;
    }

    case "show": {
      const store = new ExpStore(args.expDir);
      const replayed = await store.replayState();
      const rounds = await store.readAllRounds();
      const lineage = await store.readLineage();
      const mission = await store.readMission().catch(() => null);
      process.stdout.write(`State: ${replayed.currentState}  Round: ${replayed.currentRound}\n`);
      if (mission?.next_change) {
        process.stdout.write(`Suggested next change:\n  target: ${mission.next_change.target}\n  hypothesis: ${mission.next_change.hypothesis}\n`);
      }
      if (rounds.length > 0) {
        const last = rounds[rounds.length - 1];
        process.stdout.write(`Last round scores: outcome=${last.scores?.outcome.toFixed(2) ?? "?"}, trajectory=${last.scores?.trajectory.toFixed(2) ?? "?"}\n`);
        if (last.triage_conclusion) {
          process.stdout.write(`Triage: ${last.triage_conclusion.diagnoses.join("; ")}\n`);
        }
      }
      process.stdout.write(`Lineage: ${lineage.length} versions\n`);
      return 0;
    }

    case "status": {
      const store = new ExpStore(args.expDir);
      const replayed = await store.replayState();
      process.stdout.write(`${args.expDir}: ${replayed.currentState} (round ${replayed.currentRound})\n`);
      return 0;
    }

    case "abort": {
      const store = new ExpStore(args.expDir);
      await store.writeAbortSignal();
      process.stdout.write(`Abort signal written. Running process will stop at next checkpoint.\n`);
      return 0;
    }

    case "doctor": {
      const store = new ExpStore(args.expDir);
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
      const esPath = path.join(expDir, es.path);
      try {
        await fs.access(esPath);
        check(`eval_set ${es.path}`, true, "");
      } catch {
        check(`eval_set ${es.path}`, false, `not found: ${esPath}`);
      }
    }
    const candPath = path.join(expDir, mission.current_candidate.path);
    try {
      await fs.access(candPath);
      check("current_candidate readable", true, "");
    } catch {
      check("current_candidate readable", false, `not found: ${candPath}`);
    }
  } catch (e) {
    check("mission.md valid", false, String(e));
  }

  const health = await getHealthChecks(expDir);
  check("claude-code provider available", health.provider_available, "run: npx @anthropic-ai/claude-code --version");
  check("no step_failed in events", health.no_step_failed, "step_failed found in events.jsonl");

  return ok ? 0 : 1;
}

async function makeCoordinator(expDir: string): Promise<ExperimentCoordinator> {
  let baseUrl = process.env["KWEAVER_BASE_URL"] ?? "";
  let token = process.env["KWEAVER_TOKEN"] ?? "";
  const bd = process.env["KWEAVER_BUSINESS_DOMAIN"] ?? "bd_public";
  if (!baseUrl || !token) {
    const t = await ensureValidToken();
    if (!baseUrl) baseUrl = t.baseUrl;
    if (!token) token = t.accessToken;
  }

  let semanticMatchProvider: SemanticMatchProvider | undefined;
  try {
    const provider = defaultRegistry.resolve({ requiredCapabilities: ["structured_output"] });
    if (provider && (await provider.isAvailable())) {
      const promptRegistry = new PromptTemplateRegistry();
      await promptRegistry.loadBuiltinDir(EVAL_SET_RUBRIC_DIR);
      semanticMatchProvider = createBuiltinSemanticMatchProvider({ provider, promptRegistry, lang: "zh" });
    }
  } catch {
    process.stderr.write("warn: could not create semantic-match provider — semantic_match assertions will be skipped\n");
  }

  return new ExperimentCoordinator({
    expDir,
    synthesizer: new ClaudeCodeSynthesizer(),
    triage: new ClaudeCodeTriageClient(),
    runEval: ({ evalSetPaths, candidatePath, round }) => runEval({
      evalSetPaths,
      candidatePath,
      expDir,
      round,
      maxParallel: 2,
      deps: {
        fetchAgent: async (agentId) =>
          fetchAgentInfo({ baseUrl, accessToken: token, agentId, version: "latest", businessDomain: bd }),
        sendChat: async ({ agentInfo, query }) => {
          const result = await sendChatRequest({
            baseUrl,
            accessToken: token,
            agentId: agentInfo.id,
            agentKey: agentInfo.key,
            agentVersion: agentInfo.version,
            query,
            stream: true,
            businessDomain: bd,
          });
          return { text: result.text, conversationId: result.conversationId };
        },
        fetchTrace: async (conversationId) => {
          const r = await getTracesByConversation({ baseUrl, accessToken: token, conversationId, businessDomain: bd });
          return { spans: r.spans };
        },
        semanticMatchProvider,
      },
    }),
  });
}
```

- [ ] **Step 3.4: Run dispatch tests**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript && node --import tsx --test test/exp-list-info-dispatch.test.ts 2>&1
```

Expected: `# tests 6`, `# pass 6`, `# fail 0`.

- [ ] **Step 3.5: Run all exp tests**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript && node --import tsx --test test/exp-registry.test.ts test/exp-info.test.ts test/exp-list-info-dispatch.test.ts test/exp-store-events.test.ts test/exp-coordinator.test.ts 2>&1 | tail -10
```

Expected: all pass.

- [ ] **Step 3.6: Build check**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript && npm run build 2>&1 | tail -5
```

Expected: no errors.

- [ ] **Step 3.7: Commit**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript && git add src/trace-ai/exp/index.ts test/exp-list-info-dispatch.test.ts && git commit -m "feat(exp): add list/info subcommands and upsert registry on run/resume"
```

---

## Task 4: Edge-Case Tests — Empty Registry + info/list boundary behavior

**Files:**
- Modify: `test/exp-info.test.ts`

- [ ] **Step 4.1: Append edge-case tests**

Add to the bottom of `test/exp-info.test.ts`:

```typescript
async function makeTmpDir() {
  return fs.mkdtemp(path.join(os.tmpdir(), "exp-edge-"));
}

test("runExpCommand info: exits 1 when registry is empty and no path given", async () => {
  const configDir = await makeTmpDir();
  const prev = process.env["KWEAVERC_CONFIG_DIR"];
  process.env["KWEAVERC_CONFIG_DIR"] = configDir;
  try {
    const { runExpCommand } = await import("../src/trace-ai/exp/index.js");
    const code = await runExpCommand(["info"]);
    assert.equal(code, 1);
  } finally {
    if (prev === undefined) delete process.env["KWEAVERC_CONFIG_DIR"];
    else process.env["KWEAVERC_CONFIG_DIR"] = prev;
  }
});

test("runExpCommand list: prints header even when registry is empty", async () => {
  const configDir = await makeTmpDir();
  const prev = process.env["KWEAVERC_CONFIG_DIR"];
  process.env["KWEAVERC_CONFIG_DIR"] = configDir;
  const chunks: string[] = [];
  const orig = process.stdout.write.bind(process.stdout);
  (process.stdout as any).write = (chunk: string | Uint8Array): boolean => { chunks.push(String(chunk)); return true; };
  try {
    const { runExpCommand } = await import("../src/trace-ai/exp/index.js");
    await runExpCommand(["list"]);
  } finally {
    (process.stdout as any).write = orig;
    if (prev === undefined) delete process.env["KWEAVERC_CONFIG_DIR"];
    else process.env["KWEAVERC_CONFIG_DIR"] = prev;
  }
  const output = chunks.join("");
  assert.ok(output.includes("PATH"), `Expected PATH header in: ${output}`);
  assert.ok(output.includes("STATE"), `Expected STATE header in: ${output}`);
});
```

- [ ] **Step 4.2: Run all exp-info tests**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript && node --import tsx --test test/exp-info.test.ts 2>&1
```

Expected: `# tests 12`, `# pass 12`, `# fail 0`.

- [ ] **Step 4.3: Full regression**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript && node --import tsx --test test/exp-registry.test.ts test/exp-info.test.ts test/exp-list-info-dispatch.test.ts test/exp-store-events.test.ts test/exp-coordinator.test.ts 2>&1 | tail -10
```

Expected: all pass.

- [ ] **Step 4.4: Commit**

```bash
cd /Users/xupeng/dev/github/kweaver-sdk/packages/typescript && git add test/exp-info.test.ts && git commit -m "test(exp): edge cases — empty registry exit-1, list header always printed"
```
