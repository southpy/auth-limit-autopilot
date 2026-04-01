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

## 当前阶段：P3b PRD & Eval 骨架

**状态**：已通过
**最后更新**：2026-03-30

### 已完成

- [x] `01-design/prd.md` v1.0 — 按 agent-prd-template 填写全部 12 节。Harness 四要素完整（AC/执行边界/反馈信号/回退手段）；AC-1~AC-5 与 trace_schema.json + tool_schema.json 双向映射完整；工具清单 9/10 ACI 合规
- [x] `01-design/eval_criteria.py` v1.0 — 11 条 case 骨架（7 critical 64% + 4 edge），覆盖 Happy Path / CP-1 / CP-2 / CP-3 / CP-4 / NoAnomaly / 超范围 / API 失败 / 分币容差 / 空交易 / 多笔交易场景。骨架模式可运行，P4 接入 Agent 后替换 agent_fn

### P3b 退出契约检查

| 条件 | 状态 |
|------|------|
| prd.md 按 agent-prd-template 填写 | ✅ 已完成（12 节全填，含 Harness 四要素） |
| AC 与 Schema 双向映射完整 | ✅ 已完成（AC-1~AC-5 映射表，见 prd.md §8） |
| Harness 四要素全部填写 | ✅ 已完成（验收标准/执行边界/反馈信号/回退手段） |
| eval case ≥10 条，critical ≥ 50% | ✅ 已完成（11 条，critical 64%） |
| **P3b 退出判定** | **✅ 通过** |

---

---

## 下一阶段：P4 Tool/MCP 封装

**状态**：未启动（移交 Claude Code 执行）
**前置**：P3a ✅ + P3b ✅
**输入文件**：`01-design/tool_schema.json`（锁定 4 MCP Tool）、`01-design/architecture.md` v3.2

### 待产出

- [ ] Tool Wrapper 代码 + 单元测试
- [ ] 错误码覆盖 ≥3 种 failure mode，遵循 Error Envelope（见 tool_schema.json）
- [ ] 写操作 Mock 拦截（本项目全 Read，验证 Mock 策略）

**移交说明**：P4 起在 Claude Code 中执行，Cowork 不再产出代码文件。
