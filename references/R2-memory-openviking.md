# 基于 OpenViking 的 Agent Skill 规则引擎架构最佳实践

**核心洞察 (First-Principles)**：编写复杂业务 Skill 时的最大痛点是规则的碎片化、硬编码与 Token 溢出。OpenViking 通过 **VFS（虚拟文件系统）**与 **L0/L1/L2 分层机制**，成功将大模型的“被动记忆压力（Context Burden）”转化为“按需寻址的知识资产（On-demand Assets）”。

---

## 🛠 核心架构路径 (Core Solutions)

### 1. 规则资源化：将 Prompt 解耦至 `viking://`

禁止在 Skill 的 `system_prompt` 中硬编码成百上千行的业务规则。

* **重构方案**：将业务规则按逻辑域（权限、审批流、计算公式）拆解为独立的 Markdown 文件。
* **存储映射**：统一挂载至 `viking://agent/instructions/business_rules/` 目录。
* **架构收益**：Skill 的初始 Prompt 极度精简，降低首字延迟（TTFT）。Agent 仅在触发特定路由时，才执行按需读取（Lazy Load）。

### 2. 规则寻址：利用 L1 Layer 构建“规则地图”

当规则文件呈指数级增长时，防御 Agent 的检索迷失（Lost in Retrieval）。

* **重构方案**：在 `business_rules` 目录下，利用 OpenViking 的 L1 层生成自动摘要文件（`.overview`）。
* **上下文示例**：

    ```text
    该目录下包含三类规则：
    1. 财务审批 (finance.md)
    2. 考勤计算 (attendance.md)
    3. 报销标准 (reimbursement.md)
    ```

* **架构收益**：Agent 执行 Skill 时，优先扫描 L1 概览，精准下钻至目标 `.md` 文件，彻底规避全量加载导致的 Token 爆炸。

### 3. 确定性检索：以目录树 (`ls`) 降维 RAG 模糊匹配

传统的 RAG（向量检索）在处理强逻辑的“业务规则”时，极易因语义相近而命中错误条款。

* **重构方案**：利用 OpenViking 的 Directory Recursive Retrieval（目录递归检索）。
* **执行逻辑**：Skill 接收任务后，Agent 强制发起 `ov ls viking://agent/instructions/rules/`。
* **架构收益**：将非确定性的向量检索，降维成程序员查阅 API 文档般的“确定性路径寻址”。目录定位的准确度远高于相似度计算。

### 4. 业务逻辑热更新：与 Skill 发布生命周期解耦

业务规则变更频繁（如：下周起报销额度从 200 上调至 300），若与代码或 Prompt 耦合，将导致高频的带险重度发布。

* **重构方案**：业务人员直接通过 `ov add-resource` 或后台编辑 `viking://` 下的映射文件。
* **架构收益**：Skill 的控制流逻辑完全解耦。Agent 永远读取最新态的规则文件，实现业务规则的**零代码热更新**。

### 5. 可观测性：检索轨迹可视化 (Retrieval Trajectory)

解决 Skill 执行不符预期时，“不知道哪条规则生效了”的黑盒痛点。

* **重构方案**：依托 OpenViking 的可视化检索轨迹记录。
* **链路追踪 (Trace)**：

    ```text
    [Trigger] Skill 
     └── [Read L1] viking://agent/rules/ 
          └──[Target L2] viking://agent/rules/region_shanghai.md -> 命中执行
    ```

* **架构收益**：发现 Agent 找错文件时，无需修改 Agent 核心逻辑，直接重构文件目录拓扑或优化 L0/L1 的摘要描述即可完成修复。

---

## 🏗 实战架构蓝图 (Architecture Blueprint)

### A. OpenViking VFS 拓扑设计

```text
viking://agent/rules/
├── .overview                 # L1 层：全局规则路由地图
├── global.md                 # L2 层：通用基线规则
└── specific/                 # L2 层：垂直场景规则簇
    ├── .overview             # L1 层：垂直场景路由地图
    ├── finance_v2.md
    └── hr_compliance.md
```

### B. Skill 契约规范 (System Prompt Definition)

Skill 本身只保留极其克制的调度指令：

```markdown
# Role
你是一个财务合规审核 Agent。

# Workflow
1. 接收用户的报销请求。
2. 强制查阅 `viking://agent/rules/.overview` 定位当前请求适用的业务规则文件。
3. 读取对应的 L2 规则文件，严格按照文件内的约束条件进行审批推演。
```

### C. 运行时状态机 (Runtime State Machine)

1. `User Request` -> `Skill Triggered`
2. `Agent Action`: `Read L1 Overview` -> 锁定目标域
3. `Agent Action`: `Read L2 Detail` -> 加载硬约束规则
4. `Agent Execution` -> 输出具备规则依据的结构化结果
