# Agent PRD: 额度异动排障 Agent (Limit Troubleshooting Agent)

**Status**: In Review
**Owner**: 额度产品团队
**Last Updated**: 2026-03-30
**Harness Quadrant**: Supervised Autonomy（MVP）→ Full Autonomy（目标态）
**Architecture Ref**: architecture.md v3.2

---

## 1. Problem & Business Context

### Problem Statement

额度产品业务方处理「可用额度变化不连续」类客诉时，需跨多个系统（额度使用明细、CCA 运营管理、额度视图等）手动查询数据，人工匹配分散的业务规则（识别交易码→额度恢复逻辑→溢缴款判定），再手动计算验证。单次排障涉及多个操作，其中 33% 为「检索知识库匹配规则」，强依赖个人经验，同类问题知识复用率低。

### Success Definition (Business Level)

3 个月内：额度产品业务方处理「额度变化不连续」类工单的平均排障时间从 [TBD: 当前基线，需业务方提供] 分钟降至 5 分钟以内（Agent 自动输出根因分析 + 话术草稿，业务人员仅需审核确认）。

### Scope

**In scope（MVP）**:
- 指定账户、额度节点（消费额度、非循环专享消费分期额度等）的可用额度变化不连续排障
- 覆盖场景：客户指定时间区间内的交易→额度变动链路还原与解释
- 支持的额度变动因素：消费授权、消费入账、分期借方入账（4029）、分期贷方入账（4306）、溢缴款等交易对不同类型额度节点的影响
- 输出：根因分析报告 + 客户解释话术草稿

**Out of scope（MVP）**:
- 多账户联动额度分析
- 境外交易缴款购汇、国际卡组织退货、无授权直接入账等特殊事件
- 临额/固额生效失效引起的额度变动
- 银行汇率参数调整引起的额度变动
- 授权交易过期引起的额度变动
- e招贷占用场景
- 一线客服直接使用

---

## 2. Harness Quadrant Assessment

| Axis | Rating | Evidence |
|------|--------|----------|
| Task Specification Clarity | Medium-High (7/10) | SOP 7 步流程明确，但 33% 步骤依赖隐性业务知识（交易码→业务含义→额度影响规则），需结构化为 Skill 文件后才能被 Agent 稳定执行 |
| Verification Automation | High (9/10) | 额度变化是纯数学逻辑：`可用额度变化 = f(交易金额, 业务规则, 系统参数)`，通过 skill_limit_calc_verify 对账公式可完全自动验证 |

**Quadrant Position**: Q1 Autopilot 区（High Spec × High Verify）
**Recommended Autonomy Level**: MVP 阶段 → Supervised Autonomy；v2 → Full Autonomy

**Justification**: 验证环节的极高确定性（数学对账公式）抵消了业务规则匹配时的语义复杂性。Task Clarity 7 分反映业务规则文档分散不完整、依赖经验，这一风险通过 Skill 文件结构化（业务方共建）来缓解。MVP 锁定 Supervised Autonomy，CP-3 强制人工审核所有输出。

### Prerequisites（升级至 Full Autonomy 的前置条件）

- Gold Dataset ≥ 30 条且 Pass@10 ≥ 90%
- skill_limit_rule_match 覆盖交易码场景 ≥ 95%
- Human Intervention Rate ≤ 20% 持续 4 周
- Hallucination Rate ≤ 3%

---

## 3. Architecture Mode

**Selected Mode**: Hybrid（三层：控制层 + 业务抽象层 + 执行层）

| Option | Pros | Cons | Selected? |
|--------|------|------|-----------|
| 纯 Workflow DAG | 最高确定性，完全可审计 | RuleMatching 内语义推理无法穷举 if-else | No |
| 纯 Claude Agent SDK + Skill | 灵活，扩展性强 | HITL 触发依赖 Skill 遵从，非物理保证 | No |
| Hybrid（三层混合） | 控制流 100% 确定 + 执行层语义灵活 + Skill 业务可共建 | 协作模式复杂度较高 | **Yes** |
| Multi-Agent | 支持并行 | 单一领域无并行收益，引入协调开销 | No |

**Decision rationale**: 排障流程的 7 步 SOP 必须确定性执行（LangGraph 控制层保证），但 Step 1/3/4/6/7 含不可穷举的业务语义（Agent SDK 执行层处理）。业务规则和推理指引需业务方参与共建（Skill 业务抽象层承载）。三层分工确保：流程可审计（控制层）+ 规则可维护（Skill 层）+ 推理可泛化（执行层）。

---

## 4. Core Workflow

### Happy Path

```
客户问题输入（账户号 + 额度节点 + 问题描述 + 时间范围）
  ↓
Step 1: 意图识别与问题拆分 [Skill: skill_intent_parsing]
  → 识别问题类型：额度变化不连续 / 单笔交易差异 / 超范围
  → 提取关键参数：账户号、额度节点、时间范围、交易类型
  ↓
Step 2: 数据检索 [Tool: query_limit_usage_detail]
  → 入参：账户号、额度节点类型、时间范围（扩大 ±3 天）
  → 出参：逐笔交易记录（交易时间、交易码、交易金额、可用额度前值、可用额度后值）
  ↓
Step 3: 定位问题交易 [Skill: skill_tx_continuity_check]
  → 逐笔校验：diff = 可用额度后值 - 可用额度前值 - 交易金额
  → 标记差异交易（diff ≠ 0）和不连续点（前序后值 ≠ 后序前值）
  ↓
Step 4: 业务归因分析 [Skill: skill_limit_rule_match]
  → 对每笔差异交易：识别交易码→业务含义→额度影响规则→归因推理链
  → 若需补充数据 → Step 5
  → 若规则可直接解释 → Step 6
  ↓
Step 5: 补充数据检索（条件触发）[Tool: query_cca_transaction_detail]
  → 入参：客户号、交易时间、交易金额、唯一参考号
  → 出参：单笔交易 TCL 节点、冲抵金额等深层信息
  ↓
Step 6: 计算验证 [Skill: skill_limit_calc_verify + 控制层代码二次校验]
  → 根据匹配规则构造验证公式：expected = f(交易金额, 规则参数)
  → 对比：expected vs actual（容差 ±0.01 元）
  → 验证通过 → Step 7
  → 验证不通过 → [CP-2 HITL: 标记为系统异常，转业务人员深度排查]
  ↓
Step 7: 生成根因分析 + 话术草稿 [Skill: skill_report_generation]
  → 输出结构化根因报告 + 客户解释话术
  ↓
[CP-3 HITL: 业务人员审核确认（MVP 强制触发）]
  → 审核通过 → 交付
  → 审核修正 → 记录修正点写入评测集 (FeedbackCapture)
```

### Key Branches

| Condition | Branch | Recovery Path |
|-----------|--------|---------------|
| 意图无法解析（缺少账户号或时间范围） | 返回 incomplete，控制层路由至 OutOfScope (CP-4) | 转人工 |
| 超范围场景（临额、汇率、e招贷等） | IntentParsing 识别后直接触发 CP-4 | 转人工 |
| API 返回失败 | 指数退避重试 ≤2 次 | retry>2 → Degraded，输出部分报告 + 转人工 |
| 无差异交易（Step 3 anomalies 为空） | 跳过 Step 4-6，直接 Step 7 输出"无异常"报告 | — |
| 遇到未匹配交易码（Step 4） | 放入 unmatched 数组 → 触发 CP-1 HITL | 业务人员补充规则后恢复 Step 4 |
| 计算验证失败（Step 6） | verified=false → 触发 CP-2 HITL | 业务人员确认系统异常 |
| 总耗时 > 120s | 强制终止 → 转人工 | — |

---

## 5. Human-in-the-Loop (HITL) Design

### HITL Policy

**Policy level**: Strict（MVP 阶段，所有输出强制审核）
**Default escalation channel**: [TBD: 二线客服工作台通知 / Slack 频道]

### HITL Checkpoint Table

| Checkpoint | 触发条件 | 触发层 | 人工决策 | 超时/默认处理 |
|------------|---------|--------|---------|-------------|
| CP-1: 未知交易码 | RuleMatching 输出的 unmatched 数组非空 | 控制层代码（interrupt） | 补充该交易码的业务规则，写入 skill_limit_rule_match.md 后恢复 | 4h → 标记为「待补充规则」，记录至 gap log |
| CP-2: 计算验证失败 | CalcVerification 返回 verified=false | 控制层代码（interrupt） | 确认是系统异常还是规则错误；决定是否关闭工单或升级 | 4h → 标记为「系统异常疑似」，升级至技术团队 |
| CP-3: 输出审核 | ReportGeneration 完成（MVP 强制触发） | 控制层代码（interrupt） | 审核根因分析和话术草稿；可直接通过或修正后通过 | 8h → 标记为「超时未审核」，不自动交付 |
| CP-4: 超范围 | IntentParsing 识别为超出 MVP 支持场景 | 控制层代码（interrupt） | 决定是否人工处理或记录为新场景需求 | 即时 → 直接转人工，无等待 |

### Escalation Flow

```
触发条件满足（代码层检测，非 LLM 判断）
  → 控制层 interrupt_before/after 暂停流程
  → 格式化上下文消息：{排障任务 ID, 当前步骤, 触发原因, 已有分析结果, 所需人工决策}
  → 推送至工作台（[TBD: 接口待确认]）
  → 等待人工响应（各 CP 超时见上表）
  → 若响应: 按决策继续或终止
  → 若超时: 按默认处理行动
```

---

## 6. Toolchain Design

### Tool Inventory

| Tool Name | Category | Action Type | Description (ACI compliant) |
|-----------|----------|-------------|------------------------------|
| `query_limit_usage_detail` | MCP Tool | Read | **When**: 需要获取指定账户、额度节点在时间范围内的逐笔交易与额度变动记录时调用。**Not when**: 已有交易数据在上下文中。**Returns**: `{transactions: [{time, tx_code, amount, limit_before, limit_after}]}` |
| `query_cca_transaction_detail` | MCP Tool | Read | **When**: 需要补充单笔交易的 TCL 节点、冲抵金额等深层信息（Step 5 条件触发）时调用。**Not when**: `query_limit_usage_detail` 返回的数据已足够解释差异。**Returns**: `{tx_detail: {tcl_node, pst_amt, offset_amount}}` |
| `query_limit_view` | MCP Tool | Read | **When**: 需要确认客户当前各额度节点配置和额度值时调用。**Not when**: 排障仅涉及历史交易链路分析。**Returns**: `{limit_nodes: [{node_type, total, used, available}]}` |
| `query_actual_available_limit` | MCP Tool | Read | **When**: 需要确认客户当前实际可用额度（含所有影响因素）时调用。**Not when**: 仅分析历史额度变化。**Returns**: `{actual_available: number, factors: [...]}` |
| `skill_intent_parsing` | Skill | — | Step 1 内置能力。场景分类、参数提取、超范围判定。**Returns**: `{account_id, account_type, limit_node, time_range, problem_type}` 或 `{status: "incomplete", missing: [...]}` |
| `skill_tx_continuity_check` | Skill | — | Step 3 连续性校验。**When**: 拿到交易列表后逐笔检查连续性。**Returns**: `{anomalies: [{tx_index, type: "gap"\|"diff", expected, actual, delta}]}` |
| `skill_limit_rule_match` | Skill | — | Step 4 业务归因。**When**: 对差异交易进行业务归因分析。**Returns**: `{matched_rules: [{tx_code, biz_meaning, limit_recovery_rule, explanation}], unmatched: [...], supplement_needed: [...]}` |
| `skill_limit_calc_verify` | Skill | — | Step 6 计算验证（公式构造部分）。**When**: 规则匹配完成后构造验证公式。**Returns**: `{verified: boolean, formula: string, expected: number, actual: number, diff: number}` |
| `skill_report_generation` | Skill | — | Step 7 报告生成。**When**: 计算验证通过或无异常后生成根因分析和话术。**Returns**: `{report: {根因分析 JSON}, explanation: {客户话术文本}}` |

**Total tool count**: 4 MCP Tool + 5 Skill = 9 / 10 ✓

### ACI Compliance Check

- [x] 所有 MCP 工具名称描述业务操作（非技术实现）
- [x] 每个 MCP 工具描述包含"When"和"Not when"
- [x] 所有工具返回结构化 JSON（非原始 API 响应）
- [x] Error 响应需包含：type, message, suggestion, retryable（实现时强制，见 tool_schema.json）
- [x] 无功能重叠工具（4 个 MCP 工具查询不同维度数据，5 个 Skill 对应不同 Step）
- [x] 工具返回中不含框架元数据

---

## 7. Context Structure Design

### Permanent Layer（system prompt，≤500 tokens）

```
你是信用卡额度异动排障专家 Agent。

职责：按照 7 步 SOP 对「可用额度变化不连续」问题进行系统性排障，输出根因分析报告和客户话术草稿。

绝对约束：
- 所有归因必须基于已获取的数据和 Skill 中定义的规则，禁止编造
- 遇到未匹配交易码，必须标记为 unmatched，不允许猜测业务含义
- 所有 HITL 触发由控制层代码决定，不由 Agent 自行判断是否需要人工介入
- 计算验证由控制层代码执行，Agent 负责构造公式，不负责最终判定

成功标准：每笔差异交易都有明确的业务归因，且归因通过数学公式验证
```

### On-Demand Skills

| Skill | 加载时机 | 预估 tokens |
|-------|---------|------------|
| `skill_intent_parsing.md` | Step 1 IntentParsing 节点唤醒时 | ~300 |
| `skill_tx_continuity_check.md` | Step 3 TransactionLocating 节点唤醒时 | ~300 |
| `skill_limit_rule_match.md` | Step 4 RuleMatching 节点唤醒时 | ~800 |
| `skill_limit_calc_verify.md` | Step 6 CalcVerification 节点唤醒时 | ~400 |
| `skill_report_generation.md` | Step 7 ReportGeneration 节点唤醒时 | ~500 |

各 Skill 由控制层在唤醒对应 Agent SDK 节点时注入为 system_prompt。不同节点加载不同 Skill，避免 Context Rot。单个 Skill 上限 2000 tokens，超限触发 VFS 拆分评估。

### Runtime Injection（≤200 tokens/轮）

```xml
<runtime>
  <task_id>{排障任务 ID}</task_id>
  <customer_id>{客户号}</customer_id>
  <account_id>{账户号}</account_id>
  <account_type>{专享消费分期卡}</account_type>
  <limit_node>{消费额度/非循环专享消费分期额度}</limit_node>
  <time_range>{2025-12-20 ~ 2025-12-30}</time_range>
  <current_step>{当前排障步骤}</current_step>
</runtime>
```

### Memory Strategy

| Memory Type | Storage | When Written | When Read |
|-------------|---------|--------------|-----------|
| Working memory（任务状态） | LangGraph StateGraph checkpoint | 每个节点执行后自动持久化 | HITL 恢复时自动加载 |
| Procedural memory（业务规则） | Skill 文件（skills/*.md） | CP-1 业务人员补充新规则后编辑 | 控制层唤醒节点时注入 |
| Error memory（失败案例） | 评测集文件（gold_dataset.jsonl） | CP-2 或 FeedbackCapture 触发后 | 定期 eval 时批量加载 |

---

## 8. Harness Definition

### Acceptance Criteria（验收标准）

| # | Criterion | Verification Method | Owner |
|---|-----------|---------------------|-------|
| AC-1 | 差异交易归因准确率 ≥ 90%（每笔差异交易有正确的 biz_meaning 和 limit_recovery_rule） | 对比 gold_dataset.jsonl 中标注的 matched_rules | 产品团队 |
| AC-2 | 计算验证准确率 ≥ 95%（verified 字段与人工对账结论一致） | 对比 gold_dataset.jsonl 中标注的 verified 字段 | 产品团队 |
| AC-3 | 报告完整率 ≥ 95%（输出包含 report + explanation，无空字段） | trace_schema.json 中 report_generation 节点输出字段校验 | 工程团队 |
| AC-4 | 流程不越界：HITL 触发条件全部由控制层代码执行，零 prompt-only 触发 | 代码审查 + trace 中 hitl_trigger 字段来源校验 | 工程团队 |
| AC-5 | 端到端 P95 延迟 ≤ 60s（不含 HITL 等待时间） | trace_schema.json 中 duration_ms 字段统计 | 工程团队 |

**AC ↔ Schema 双向映射**：

| AC | trace_schema.json 字段 | tool_schema.json 字段 |
|----|----------------------|----------------------|
| AC-1 | `nodes[rule_matching].output.matched_rules` | `skill_limit_rule_match.returns.matched_rules` |
| AC-2 | `nodes[calc_verification].output.verified` | `skill_limit_calc_verify.returns.verified` |
| AC-3 | `nodes[report_generation].output.report` | `skill_report_generation.returns.report` |
| AC-4 | `hitl_checkpoints[*].trigger_source` | — |
| AC-5 | `metadata.duration_ms` | — |

### Execution Boundary（执行边界）

- **Max retries per API call**: 2 次（指数退避），超出 → Degraded 状态
- **Max end-to-end duration**: 120s（不含 HITL 等待），超出 → 强制终止并转人工
- **Permitted operations**: Read only（query_* 工具均为只读查询）
- **Forbidden operations**: 任何写操作、任何直接向客户发送信息的操作、一线客服使用

### Feedback Signal（反馈信号）

- **Primary signal**: CP-3 人工审核结论（通过/修正）——来自工作台审核记录，非 Agent 自评
- **Secondary signal**: CalcVerification 的 verified 字段——纯数学对账，客观可验证
- **Monitoring**: [TBD: 日志平台] 中 trace_id 关联的 eval_metadata 字段；Pass@10 定期 eval 报告

### Rollback Procedure（回退手段）

- **触发条件**: 任意 AC 连续 3 次未通过，或 Human Intervention Rate > 40% 持续 1 周
- **Steps**:
  1. 暂停新工单分配给 Agent，切回人工排障
  2. 从 FeedbackCapture 收集修正记录，分析失败根因（rule gap / skill 质量 / data issue）
  3. 针对根因：补充 Skill 规则（rule gap）或回退 P3b 重新设计 eval criteria（系统性失败）
- **Recovery time objective**: 24h 内完成失败根因定位，72h 内完成修复并重新验证

---

## 9. Anti-Pattern Pre-Check

| Anti-Pattern | Risk Level | Status | Mitigation |
|---|---|---|---|
| System prompt 作为知识库 | Medium | ✅ Mitigated | 业务规则在 Skill 文件中（≤800 tokens/Skill），system prompt 只含角色定义和绝对约束（≤500 tokens） |
| 工具数量超限（>10） | Low | ✅ Mitigated | 4 MCP Tool + 5 Skill = 9/10，在限制内 |
| 缺少验证循环 | Low | ✅ Mitigated | CalcVerification 节点 + 控制层代码二次校验（双重保障） |
| Multi-Agent 无边界 | Low | ✅ Mitigated | MVP 单 Agent，场景扩展时走明确升级信号（>5 场景或 >15 状态） |
| 记忆不整合（Context Rot） | Medium | 🔶 At Risk | Skill 文件上限 2000 tokens 已设；但 rule_matching_skill 随规则补充可能膨胀，需监控 |
| 无评测基线 | Medium | 🔶 At Risk | eval_criteria.py 在 P3b 产出，Gold Dataset ≥ 30 条为 P5 目标；MVP 前必须完成 Sanity Check |
| 过早 Multi-Agent | Low | ✅ Mitigated | 明确的升级信号和路径 B 触发条件（见 architecture.md §7） |
| 约束靠期望不靠机制（AP-8） | Low | ✅ Mitigated | HITL 触发、计算验证、输出格式校验均由代码/Hook 执行，非 prompt 遵从 |

---

## 10. Success Metrics

### Product-Level Metrics（业务影响）

| Metric | Baseline | Target | Measurement Method |
|--------|----------|--------|--------------------|
| 平均排障时长 | [TBD: 当前基线，需业务方提供] | ≤5 分钟 | 工作台工单处理时长记录 |
| 人工深度排查率 | [TBD] | ≤20%（CP-2 + CP-4 触发率） | trace_schema.json hitl_checkpoints 统计 |
| 知识复用率 | ~0%（全靠个人经验） | ≥80%（问题通过 Skill 规则匹配解决） | matched_rules 非空率 |

### Agent-Level Metrics（系统健康）

| Metric | Definition | Target | Alert Threshold |
|--------|------------|--------|-----------------|
| Task Success Rate | % 任务完成所有 AC 且无人工干预 | ≥ 80%（MVP） | < 60% → review |
| Latency P95 | 端到端任务耗时 P95（不含 HITL） | ≤ 60s | > 90s → page |
| Human Intervention Rate（HIR） | % 任务触发至少一次 HITL | ≤ 40%（MVP） | > 60% → freeze |
| Hallucination Rate | % 任务中出现未经验证的归因（CalcVerify 失败且无 unmatched） | ≤ 5% | > 10% → freeze |
| Token Cost per Task | 每次排障平均 token 消耗 | ≤ 8k tokens | > 15k → review context design |

---

## 11. Connector Dependencies

| Category | Required? | Connected Tool | Fallback If Absent |
|----------|-----------|----------------|-------------------|
| 额度使用明细 API | Required | query_limit_usage_detail（MCP） | Cannot proceed — blocker |
| CCA 交易详情 API | Required（条件触发） | query_cca_transaction_detail（MCP） | Step 5 跳过，依赖现有数据归因（准确率下降） |
| 额度视图 API | Optional | query_limit_view（MCP） | 仅影响 v2 扩展场景 |
| 实际可用额度 API | Optional | query_actual_available_limit（MCP） | 仅影响 v2 扩展场景 |
| 业务规则知识库 | Required | Skill 文件（skills/*.md） | Cannot proceed — blocker |
| 工单工作台（HITL 通知） | Required | [TBD: 待集成] | MVP 可用邮件/Slack 临时替代 |

---

## 12. Implementation Path

### MVP（Supervised Autonomy）

- **Scope**: 单一场景（额度变化不连续），二线客服使用，人工审核所有输出（CP-3 强制触发）
- **Autonomy level**: Supervised（4 个 HITL checkpoint 全部启用）
- **Harness**: AC-1/AC-2/AC-3 验收；AC-4/AC-5 作为健康监控
- **Skill 建设**: 至少完成 skill_limit_rule_match.md v0（覆盖 4029/4306 等高频交易码）
- **Launch gate**: Gold Dataset ≥ 30 条，Pass@10 ≥ 80%（critical case Pass@10 ≥ 90%）

### v2（Expanded Autonomy）

- **Scope additions**: 新增 2-3 个场景类型（如临额过期、授权交易过期）；新增 query_temp_limit_history、query_auth_expiry 工具
- **Autonomy upgrade**: CP-3 由强制触发改为抽检（10%），HIR ≤ 20% 持续 4 周后触发
- **Gate**: HIR ≤ 20% 持续 4 周，Hallucination Rate ≤ 3%，Gold Dataset ≥ 80 条

### v3（Full Autonomy Target）

- **条件**: Pass@10 ≥ 95% 持续 8 周，HIR ≤ 5%，一线客服可用评估通过
- **Scope**: 多场景覆盖；如需继续扩展 → architecture.md §7 路径 B（多 workflow 编排）
