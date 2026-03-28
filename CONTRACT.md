# Agentic Engineering 全流程契约表 v2 (SOP Contract Matrix)

*版本: v2.0 | 引入插件*

## 插件参考文档索引

以下文件从 agentic-architect-plugin 提取，放入项目 `references/` 目录，在对应 Phase 的 task prompt 中按需引用：

| 文件 | 来源 Skill | 用于 Phase | 内容 |
| :--- | :--- | :--- | :--- |
| `architecture-decision-tree.md` | design-agent | P3a | 四种架构模式选型决策树 + 好坏案例 |
| `state-machine-patterns.md` | design-agent | P3a | 状态机模板库 + 异常路径设计 + 质量检查清单 |
| `aci-principles.md` | tool-design | P3a/P4 | ACI 工具设计原则 + 11项质量门控 + 错误响应模板 |
| `context-layers.md` | context-design | P3a | 五层上下文分层规范 + Context Rot 防治 |
| `agent-prd-template.md` | write-agent-prd | P3b | PRD 完整模板（含 Harness 四要素、HITL 表、指标定义） |
| `anti-patterns-detail.md` | anti-patterns | P7/P9 | 8条反模式完整分析 + 诊断信号 + 处置方法 |

---

## 契约表

| 阶段 | 前置输入 | 交付物 & 退出契约 | 场景举例（额度异动分析 Agent） | 参考文档 & 工具 |
| :--- | :--- | :--- | :--- | :--- |
| **P0-P2 场景与评估** | 业务线原始痛点；未清洗的业务数据。 | **交付**：`harness_score.md` + `cleaned-samples/`（≥30条代表性语料）。**契约**：落在 Q1（清晰度≥3 且 验证自动化≥3），否则项目熔断。 | 抽取 50 条"额度异常变动"工单。判定：异动规则有明确的业务文档（清晰），异动结果可通过金额/时间规则自动断言（可验证）。进入 Q1。 | **Cowork** 执行。引用 `harness-quadrant` 评估框架。 |
| **P3a 架构与Schema** | Q1 评估报告；业务 SOP 文档。 | **交付**：`architecture.md`（架构模式选型 + 状态机 + 上下文分层）、`tool_schema.json`（含 Error Envelope）、`trace_schema.json`。**契约**：架构选型必须走决策树并记录决策路径；Tool Schema 通过 ACI 11 项检查；上下文结构覆盖五层。 | 走决策树：步骤可设计时确定 → DAG 候选；无需多领域并行 → 单 Agent；无高风险操作 → 无需 HITL。选型 Workflow DAG。设计 `check_quota_change`、`get_transaction_history` 等工具，统一 Error Envelope。 | **Cowork** 执行。引用 `architecture-decision-tree.md`、`state-machine-patterns.md`、`aci-principles.md`、`context-layers.md`。 |
| **P3b PRD & Eval 骨架** | P3a 的架构文档和 Tool Schema。 | **交付**：`prd.md`（按 agent-prd-template 填写）、`eval_criteria.py`（≥10 条 case 骨架）。**契约**：AC 与 Schema 双向映射完整；Harness 四要素（验收标准、执行边界、反馈信号、回退手段）全部填写；eval case 标注 critical/edge，critical ≥ 50%。 | PRD 填写：验收标准="异动分类准确率≥95%"；执行边界="单次分析≤30s，重试≤3次"；反馈信号="与人工标注对比"；回退="降级为规则引擎"。 | **Cowork** 执行。引用 `agent-prd-template.md`。PRD 中 HITL 表和指标定义按模板填写。 |
| **P4 Tool/MCP 封装** | P3a 锁定的 `tool_schema.json`。 | **交付**：Tool Wrapper 代码 + 单元测试。**契约**：写操作 Mock 拦截；错误码覆盖≥3种 failure mode；所有错误遵循 Error Envelope；通过 ACI 11 项门控。 | 封装额度查询 API。Mock `adjust_quota` 的写操作。错误码覆盖：ACCOUNT_NOT_FOUND、TIMEOUT、PERMISSION_DENIED。 | **Claude Code** 执行。引用 `aci-principles.md` 中的质量门控清单做自检。 |
| **V0 & SanityCheck** | P4 的 Tools；P0 的≥30条语料。 | **交付**：`v0_traces.jsonl`（≥30条）。**契约**：Eval vs 人工一致率≥85%（n≥20）。不达标退回 P3b。 | V0 用真语料跑出轨迹。发现 Trace #8 中 Agent 对"历史额度回溯"场景选错了工具，但 Eval 判了 Pass。修正 eval_criteria.py 中的工具选择断言。 | **Claude Code** 构建 V0。人工标注在 Cowork 中辅助分析。 |
| **P5 Eval 基建** | 通过 SanityCheck 的 Eval 脚本 + 标注 Trace。 | **交付**：`gold_dataset.jsonl`（≥50条，人工审核）、`run_eval.py`。**契约**：一键 `python run_eval.py` 跑通，无环境报错。 | 从 30 条标注泛化至 80 条，覆盖并发查询、账户冻结、跨境额度等边缘场景。每条泛化 case 经人工 review。 | **Claude Code** 执行。 |
| **P6 生产级编码** | P3a 架构；P5 Eval 基建。 | **交付**：Agent 主干代码 + System Prompt（版本化）。**契约**：所有 LLM 调用挂载 trace_id；System Prompt 遵循五层分层（常驻层≤500 tokens）。 | 编写额度异动分析状态机。System Prompt 常驻层只放角色定义+约束+输出格式，领域规则放按需加载层。 | **Claude Code** 执行。引用 `context-layers.md` 校验 prompt 分层。 |
| **P7 Offline Eval 调优** | V1 Agent + Eval 基建。 | **交付**：Pass@K 报告。**契约**：critical Pass@K≥95%，edge≥85%。未达标时先做反模式预检再定回退目标。 | Pass 率 82%。反模式预检发现 AP-1（system prompt 超 1500 tokens，领域规则未 skill 化）。回退 P3a 重构上下文分层后重跑。 | **Claude Code** 跑 eval。引用 `anti-patterns-detail.md` 做失败归因。 |
| **P8 渐进上线** | P7 达标版本。 | **交付**：流量分发配置 + HITL 操作台。**契约**：Shadow 最终决策一致率>95%；Canary 错误率<SLA。 | Shadow 期将 10% 额度异动工单镜像给 Agent，对比人工分析结论。HITL 阶段人工确认"高风险额度调整建议"。 | **Cowork** 辅助分析 Shadow 数据。 |
| **P9 持续运营** | GA 生产监控数据。 | **交付**：Bad Case 归档 + 回注路径。**契约**：新场景→追加 gold_dataset；标准盲区→回退 P3b。 | 发现新 failure mode：联名账户的额度共享导致分析逻辑出错。归为"标准盲区"，回流 P3b 补充 AC。反模式复检确认无 AP-5（记忆不整合）。 | **Cowork** 批量分析 Bad Case。引用 `anti-patterns-detail.md` 做周期性健康检查。 |
