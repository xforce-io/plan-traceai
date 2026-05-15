---
title: trace exp 多层优化目标 — KN / Skill / Agent 三层 patch target
status: draft
date: 2026-05-15
依赖: vision/trace-cli-detailed-design.md §3.3（M6 Experiment Engine）
增强范围: MVP-C 之后（与 FSM / ExpStore 契约兼容，扩展 PatchApplier）
---

## §0 背景与问题

当前 `trace exp` 的优化循环只能在 **Agent 层**打 patch：

```yaml
# mission.md next_change（现状）
next_change:
  target: agent.system_prompt
  patch: '{"agent": {"system_prompt": "..."}}'
```

Triage 能诊断出 round 失败的症状，但所有建议都被迫挤进 `system_prompt`——无论根因在哪一层。

### 实际案例：HuaTai 汽车产业链图谱问答实验

`eval-set-quick`（10 题）Round 1 结果：`outcome=0.29, trajectory=1.00`

| 失败题 | 根因层 | 具体根因 | system_prompt 能修？ |
|--------|--------|---------|---------------------|
| Q36：销量表覆盖哪些月份 | **KN** | `vehicle_sales` 对象类型未绑入 KN，数据存在于 dataview 但不可查 | ✗ |
| Q42：激光雷达配置记录数 | **KN/Skill** | Agent 查了错误对象类型（config_supplier 而非正确类型） | 部分 |
| Q52：激光雷达供应商价格区间 | **Skill** | `query_object_instance` 未翻页，只拿了 top-k，漏掉 7 条记录 | ✗ |
| Q54：域控芯片最高价记录 | **Skill** | 未使用 ORDER BY + LIMIT 1，取到错误记录 | ✗ |

当前框架的 Synthesizer 将所有这些诊断转化为 `agent.system_prompt` 的改动建议，导致**优化循环在错误的层上空转**。

---

## §1 三层优化框架

```
┌─────────────────────────────────────────────────┐
│  Agent 层（理解与编排）                           │
│  system_prompt / skills 绑定 / 模型选择           │
├─────────────────────────────────────────────────┤
│  Skill 层（查询能力封装）                         │
│  SOP 文档 / 查询模板 / 排序翻页用法               │
├─────────────────────────────────────────────────┤
│  KN 层（数据完整性）                              │
│  对象类型定义 / dataview 绑定 / 关系类型           │
└─────────────────────────────────────────────────┘
         依赖方向：Agent → Skill → KN
```

**核心原则**：KN 层缺陷不能被 Agent 层 prompt 绕过；Skill 层缺陷不能靠堆示例弥补。Triage 必须先归因到根因层，再在对应层打 patch。

---

## §2 扩展 `next_change` schema

### 2.1 新增 patch target 枚举

```
agent.system_prompt    现有，保持不变
agent.skills           绑定 / 解绑 skill_id 列表
kn.object_type         新增对象类型（绑定 dataview）
kn.relation_type       新增关系类型
skill.content          更新 skill SKILL.md / SOP 文档
```

### 2.2 `candidate.yaml` schema 扩展

```yaml
# 现有（agent 层）
agent:
  description: "..."
  system_prompt: "..."
  skills: []

# 新增（KN 层）
kn:
  id: "d820qu7a2s1et30t7ckg"        # 必填，操作哪个知识网络
  object_types: []                   # patch 追加的对象类型定义列表
  relation_types: []                 # patch 追加的关系类型定义列表

# 新增（Skill 层）
skills:
  - id: "xxxx"
    content_patch: "..."             # 覆盖 SKILL.md 中的特定段落
```

### 2.3 `next_change` 格式扩展

```yaml
# KN 层 patch 示例（Q36 根因修复）
next_change:
  target: kn.object_type
  hypothesis: >
    Q36 失败因为 vehicle_sales 对象类型未绑入 KN。
    将 dataview ht_data_513_05141232vehicle_sales 绑入后，
    agent 可通过 query_object_instance 直接查询月份覆盖范围。
  patch:
    kn_id: "d820qu7a2s1et30t7ckg"
    add_object_types:
      - concept_name: "vehicle_sales"
        dataview_id: "c2b94354c4fb4b9aba200d480aea8539"
        primary_keys: ["vehicle_sales_id"]
        data_properties:
          - name: vehicle_id
            type: string
          - name: sales_label
            type: string
          - name: sales
            type: decimal
          - name: month
            type: string
    add_relation_types:
      - concept_name: "vehicle_has_sales"
        source_object_type: "vehicle"
        target_object_type: "vehicle_sales"
        join_key: "VEHICLEID → vehicle_id"

# Skill 层 patch 示例（Q52/Q54 根因修复）
next_change:
  target: skill.content
  hypothesis: >
    Q52/Q54 失败因为 query_object_instance 缺少排序和全量扫描指导。
    在 SOP skill 中补充：取极值用 sort_by + limit=1；
    获取全量记录需循环 search_after 直到 total_count 耗尽。
  patch:
    skill_id: "xxxx-component-price-sop"
    append_section: |
      ## 查询完整性要求
      - 取最大/最小值：传 sort_by=[{"field":"UNITPRICEHIGH","order":"desc"}] + limit=1
      - 获取全量记录：检查 total_count，若 > limit 则用 search_after 循环翻页
```

---

## §3 Triage 输出扩展：失败层归因

Triage 当前输出只有症状描述（`semantic_match 失败`）。需要增加 `failure_layer` 字段：

```yaml
# 现有 triage 输出
triage:
  verdict: continue
  summary: "outcome=0.29..."

# 扩展后
triage:
  verdict: continue
  summary: "..."
  failure_attribution:
    - layer: kn
      evidence: "kn_search 返回的 object_types 中无 vehicle_sales；Q36 失败"
      affected_queries: [Q36]
      suggested_target: kn.object_type

    - layer: skill
      evidence: "query_object_instance 调用未传 sort_by；Q54 取到错误记录"
      affected_queries: [Q52, Q54]
      suggested_target: skill.content

    - layer: agent
      evidence: "Q42 查了 config_supplier 而非正确类型，属理解问题"
      affected_queries: [Q42]
      suggested_target: agent.system_prompt
```

Synthesizer 根据 `failure_attribution` 中影响最大的层决定 `next_change.target`，而不是默认写 `agent.system_prompt`。

---

## §4 PatchApplier 扩展

`PatchApplier` 是 Generating 状态下执行 `next_change` 的组件。扩展后需支持三条执行路径：

```
PatchApplier.apply(next_change)
  ├── target == agent.*    → 现有逻辑（写 candidate.yaml，agent 字段）
  ├── target == kn.*       → 调用 kweaver bkn / context-loader API
  │     kn.object_type     →   创建 object_type + 绑定 dataview
  │     kn.relation_type   →   创建 relation_type
  └── target == skill.*    → 调用 kweaver skill register / update
        skill.content      →   更新 SKILL.md 内容，发布新版本
```

每条路径执行后需写 `candidate-lineage.yaml` 追加版本记录，并可回滚（Rollback 接口）。

### KN patch 执行细节

KN 写操作有副作用（影响生产知识网络），需额外保护：

1. **沙箱优先**：若平台支持 KN 分支/快照，在快照上操作
2. **`--dry-run` 验证**：写前用 `bkn validate` 校验 patch 合法性
3. **回滚记录**：写 `rollback.yaml` 记录操作前的状态，支持 `exp abort` 时还原

---

## §5 验证计划

使用 HuaTai 实验（`~/lab/ht/exp`）作为端到端验证场景：

| Round | patch target | 预期结果 |
|-------|-------------|---------|
| Round 2 | `kn.object_type`（补 vehicle_sales） | Q36 通过，outcome 提升 |
| Round 3 | `skill.content`（补排序/翻页 SOP） | Q52/Q54 通过，outcome 进一步提升 |
| Round 4 | `agent.system_prompt`（细化答案格式） | 剩余语义偏差修复 |

**验收条件**：三轮迭代后 `outcome >= 0.7`，每轮 Triage 正确归因失败层。

---

## §6 范围与限制

**在范围内（本 spec）：**
- `next_change` schema 扩展（三个 target 类型）
- Triage `failure_attribution` 字段
- PatchApplier 三条执行路径接口定义
- KN patch 的 dry-run + rollback 机制

**不在范围内（post-MVP）：**
- 多层 patch 在同一 round 内并行执行（当前一个 round 只打一个 target）
- Skill 版本管理与自动回滚
- KN 分支/快照支持（依赖平台能力）
- Synthesizer 自动决策 patch target（MVP 阶段 Triage 输出归因，人工确认后 resume）
