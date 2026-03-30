# PROGRESS — 额度异动排障 Agent

## 当前阶段：P0-P2 场景与评估

**状态**：已通过（临时降标）
**最后更新**：2026-03-28

### 已完成

- [x] 目录结构修正（拼写纠正：00-discovery, cleaned-samples）
- [x] `00-discovery/harness_score.md` — Harness 四象限评估 v2.0，Q1 定位（清晰度 4/5，验证自动化 4/5）
- [x] `00-discovery/cleaned-samples/` — 1 条完整 SOP trace（专享消费分期卡溢缴款场景）
- [x] `00-discovery/raw-corpus/` — 原始场景描述（3 类典型场景 + 业务痛点）
- [x] 语料缺口清单已写入 harness_score.md（6 类场景，后续需持续补充）

### 降标记录

**原标准**：cleaned-samples ≥30 条。**临时标准**：cleaned-samples ≥1 条。人工确认放行，语料补充延后至后续阶段并行推进。

### P0-P2 退出契约检查

| 条件 | 状态 |
|------|------|
| harness_score.md 产出 | 已完成 |
| Q1 定位（清晰度 ≥3 且 验证自动化 ≥3） | 通过（4, 4） |
| cleaned-samples ≥1 条（临时降标） | 通过（1/1） |
| **P0-P2 退出判定** | **通过** |

### 归档说明

`88-archieve/` 存放前一轮迭代产出（P0 场景采集 v1、P1 综合调研、Harness 评估 v1、Agent PRD 草稿），已整合至当前 v2 交付物中。

---

## 当前阶段：P3a 架构与 Schema

**状态**：进行中
**最后更新**：2026-03-30

### 已完成

- [x] `01-design/architecture.md` v3.3 — 三层混合架构：控制层（LangGraph）+ 业务抽象层（5 Skill，对齐 PRD 7 步 Core Workflow）+ 执行层（Claude Agent SDK）+ MCP 工具协议。v3.2 核心变更：Skill 分解从技术视角修正为业务工作流视角。v3.3 核心变更：Step 4 Skill I/O Contract 显式声明 + SKILL_ROUTING 动态路由机制 + TECH-DEBT 重构触发信号表
- [x] `01-design/tool_schema.json` v1.0 — 4 MCP Tool Schema（query_limit_usage_detail, query_cca_transaction_detail, query_limit_view, query_actual_available_limit）+ 统一 Error Envelope（6 类错误枚举），通过 ACI 11/11 项检查。[ASSUMPTION] tx_code/limit_node_type 枚举值待补充，后续写入 Skill 或 memory
- [x] `01-design/trace_schema.json` v1.0 — Eval 消费为主的 Trace Schema：17 节点全覆盖、4 HITL 检查点、4 MCP Tool 对齐、eval_metadata 后处理字段。校验：节点覆盖 17/17、HITL CP 4/4、Tool 名称 4/4 对齐 tool_schema.json

### P3a 退出契约检查

| 条件 | 状态 |
|------|------|
| 架构选型走决策树并记录决策路径 | ✅ 已完成（v3.3 三层混合架构，5 Skill 对齐 PRD Core Workflow + OCP 防腐设计） |
| Tool Schema 通过 ACI 11 项检查 | ✅ 已完成（v1.0，11/11 通过） |
| 上下文结构覆盖四层（Skill/运行时/记忆/系统） | ✅ 已完成 |
| trace_schema.json 产出 | ✅ 已完成（v1.0，节点 17/17 + HITL 4/4 + Tool 4/4） |
| **P3a 退出判定** | **✅ 通过** |

---

## 下一阶段：P3b PRD & Eval 骨架

**状态**：未启动
**前置**：P3a ✅ 已通过

### 待产出

- [ ] Eval 骨架：基于 trace_schema.json 定义 eval case 格式、gold dataset 结构、Pass@K 评分标准
- [ ] PRD 更新：将 architecture.md v3.3 的变更（Skill Contract、SKILL_ROUTING、TECH-DEBT）同步至 PRD
- [ ] Skill 文件 v0：至少产出 skill_limit_rule_match.md 的可执行初稿（P0 SOP trace 覆盖）

---

## 后续阶段（未启动）

- P4+ — Claude Code 执行
