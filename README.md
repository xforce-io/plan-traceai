# plan-traceai

> **这是什么** — KWeaver 子模块 `trace-ai` 的**研究与规划工作区**。这里**不放代码**，只放：现状阅读报告、trace 样本解读、未来形态设计 spec、以及推进中的活清单。
>
> **真实代码** 位于 [`kweaver-ai/kweaver-core` 仓库的 `trace-ai/` 子目录](https://github.com/kweaver-ai/kweaver-core/tree/main/trace-ai)（本机 clone 在 `~/dev/github/kweaver/trace-ai/`）。本目录的产出最终会回流到那里（spec / docs / issues）。
>
> **相关研究** — `triage agent` 这个 topic 的研究目前在 [`kweaver-ai/kweaver-core-triage`](https://github.com/kweaver-ai/kweaver-core-triage/tree/main)。

---

## 一句话定位

`trace-ai` 是 KWeaver 平台为 LLM 应用 / Agent 系统打造的**可观测 + 持续学习**双形态基础设施。一期已落地 OpenTelemetry 采集 → OpenSearch 存储 → `agent-observability` Go 查询服务的**最小骨架**；本工作区在规划如何把它从"事后排障凭证"升级为驱动 Agent 系统持续学习的**飞轮燃料**（覆盖 post-deployment agent engineering 的 L0 + L1 + L2 三层）。

## 目录布局

```
plan-traceai/
├── README.md                      ← 你正在看的入口
├── status_quo/                    ← 现状：trace-ai 一期已经长成什么样
│   └── 附录-完整trace样本/         ← 一条真实 trace 的三视图（OpenSearch raw / by-conversation / 拓扑树解读）
├── vision/                        ← 未来：trace-ai 重新立形的设计 spec
│   └── 2026-05-07-trace-ai-continuous-learning-design.md   ← 顶层 Vision Spec（vision-level，不锁 MVP）
└── plan/                          ← 执行计划：vision 落地的 issue 切分与进度（living 文档，跟随实现演化）
```

## 关键外部路径

| 路径 | 作用 |
|------|------|
| [`kweaver-ai/kweaver-core` · `trace-ai/`](https://github.com/kweaver-ai/kweaver-core/tree/main/trace-ai) | 真实代码（trace-ai 是 kweaver-core 仓库下的子目录） |
| [`trace-ai/READING_REPORT.md`](https://github.com/kweaver-ai/kweaver-core/blob/main/trace-ai/READING_REPORT.md) | v0.2.2 仓库阅读报告（本工作区现状研究的主要参照物） |
| [`trace-ai/agent-observability/`](https://github.com/kweaver-ai/kweaver-core/tree/main/trace-ai/agent-observability) | Go 实现的 Trace 查询服务（OpenSearch 代理 + 两个查询接口） |
| [`trace-ai/otelcol-contribute-chart/`](https://github.com/kweaver-ai/kweaver-core/tree/main/trace-ai/otelcol-contribute-chart) | OTel Collector Contrib 的 Helm Chart |
| [`agent-observability/docs/design/`](https://github.com/kweaver-ai/kweaver-core/tree/main/trace-ai/agent-observability/docs/design) | 一期 HLD/LLD 设计文档 |
| [`agent-observability/docs/prd/`](https://github.com/kweaver-ai/kweaver-core/tree/main/trace-ai/agent-observability/docs/prd) | 一期 PRD |
| [`kweaver-ai/kweaver-core-triage`](https://github.com/kweaver-ai/kweaver-core-triage/tree/main) | **triage agent** topic 的独立研究仓库（vision §7.2 L1 信号分诊的上游研究） |

## 推荐阅读顺序

新进来的 agent 按以下顺序读，可在 ~15 分钟内建立完整上下文：

1. **本文档** — 知道工作区是干什么的
2. [`trace-ai/READING_REPORT.md`](https://github.com/kweaver-ai/kweaver-core/blob/main/trace-ai/READING_REPORT.md) §1–§3 — 一期能力边界与术语
3. `status_quo/附录-完整trace样本/03_tree_and_summary.md` — 一条真实 trace 长什么样、字段稀疏到什么程度（理解"schema 不充分"的直观证据）
4. `vision/2026-05-07-trace-ai-continuous-learning-design.md` §1 §3 §6 — 未来形态的背景、目标、总体架构
5. 需要细节时再回查 vision spec 的 §4 §7（能力 / 模块拆分）；triage agent 相关的研究底盘见 [`kweaver-core-triage`](https://github.com/kweaver-ai/kweaver-core-triage/tree/main)

## 当前阶段

- **现状研究**：基本完成。READING_REPORT 已沉淀 v0.2.2 视角的能力清单、Schema 现状、痛点诊断
- **未来设计**：Vision Spec 已出 Draft（2026-05-07），覆盖 L0+L1+L2 顶层形态、9 个章节齐备
- **下一阶段**：从 vision-level 落到二级子模块 spec / MVP 选型

## 核心概念速查

读 vision spec 之前预热：

- **L0 / L1 / L2** — post-deployment agent engineering 工程栈三层：L0 可观测基础设施、L1 信号分诊（Signal Triage）、L2 数据重构 / 偏好数据生成。L3 模型对齐**不在 trace-ai 范围**
- **Trace / Span / Resource Spans** — OTel 三层数据结构：一次执行 → Span 集合 → 按 service+scope 分组
- **ss4o** — OpenSearch Simple Schema for Observability，trace-ai 一期默认落库 schema
- **agent.trace.type** — 一期把 OTel 通用语义"AI 化"的关键扩展字段，把 Span 分为四类（model / tool / retrieval / reasoning）
- **KWeaver 平台栈** — BKN（知识网络）/ Vega（数据虚拟化）/ Decision Agent（推理编排）/ Execution Factory（动作执行）/ Skill（被 Execution Factory 管理的资源）
- **Falsifiable Manifest** — Vision spec 提出的发布契约：每次 Agent 配置变更必须自带可证伪的 predicted_fixes / risk_tasks / 文件级回滚清单

## 工作约定

- 所有 spec / 报告日期写绝对日期（`YYYY-MM-DD`），不写"上周 / 下周"
- vision/ 下文件命名：`YYYY-MM-DD-<slug>.md`
- 现状类产出放 `status_quo/`；未来形态设计放 `vision/`；不在两者之一的临时产出先不要落盘
- 引用 trace-ai 内部文档时优先使用 GitHub 链接（`https://github.com/kweaver-ai/kweaver-core/blob/main/trace-ai/...`），避免与本工作区相对路径混淆；本机 clone 路径 `~/dev/github/kweaver/trace-ai/` 仅在需要本地操作时使用

## 活清单（Living TODO，随研究推进随时改）

> 不是承诺，只是当前能想到的下一步候选。完成 / 改方向时直接编辑此清单。

- [ ] **Schema 治理 spec** — 把 vision §4.1 / §7.1 关于 SchemaGuard、tool call 完整字段（name / args / return / scope / source）的要求落到独立子模块 spec
- [ ] **L1 Signal Triage MVP 选型** — 在 Signals / AgentSeer / Trajectory Guard / Sentinel / Near-Miss 几条路线里选一条作为 KWeaver 首发，产出选型对比文档
- [ ] **采样策略 spec** — vision §9.1 提到的拓扑感知采样 / 动态阈值触发，需要单独立稿（避免观测税失控）
- [ ] **现状 schema 缺口量化** — 基于 `status_quo/附录-完整trace样本` 的 37-span 样本，写一份"对照 L1 方法准入要求，当前缺哪些字段"的差距表
- [ ] **AgentHER 落点 spec** — 失败 trace → DPO 偏好数据的具体 schema 与流程，对接 trace-ai 而非平台外
- [ ] **Falsifiable Manifest 数据结构草案** — vision §8 亮点 2，需要先把字段表打出来才能进入工程
- [ ] 与 [`kweaver-core-triage`](https://github.com/kweaver-ai/kweaver-core-triage/tree/main) 对齐：把那边的研究产出映射到 vision §7.2 中的 L1 子模块槽位

## 更新本文档的时机

- 工作区目录结构变化（新增 `experiments/` `notes/` 等子目录）
- vision spec 进入下一版本（如 Draft → v1.0），更新"当前阶段"段落
- 活清单完成 / 失效项的清理
- 新发现的关键外部路径（如新增 issue tracker、benchmark 数据集等）
