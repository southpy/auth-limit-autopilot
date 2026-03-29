# Auth Limit Autopilot (授权限额自动导航)

本项目用于构建客户额度异动分析Agent

## 项目目录结构

* **`00-discovery/`**: 存放需求发现和调研相关的内容，包括原始语料 (`raw-corpus/`)、清洗后的样本 (`cleaned-samples/`) 以及相关的评分标准或发现文档 (`harness_score.md`)。
* **`01-design/`**: 架构设计文档目录。其中 `architecture.md` 包含系统整体架构设计、MCP 工具设计和多技能协同机制等。
* **`88-deprecated/`**: 废弃文件存档目录。存放历史遗留或已被替代的设计文档与思考记录，供查阅参考，不再维护。
* **`99-patterns/`**: 设计模式与工作流契约目录。包含 AI Agent 开发过程中总结的范式、全流程契约表 (`D1-Agentic Engineering 全流程契约表.md`) 和项目协作规范 (`I1-cowork-project-instructions.md`) 等。
* **`references/`**: 参考资料目录。存放外部或团队内的参考文档、技术标准指南、混合编排 (`R1-hybrid-orchestration.md`) 和 OpenViking 记忆机制 (`R2-memory-openviking.md`) 的分析资料。

## 关键文件

* **`CONTRACT.md`**: 全流程契约的主文件（如果与 `99-patterns` 中的内容相呼应），用于记录 Agent 开发的范式和契约。
* **`PROGRESS.md`**: 项目开发进度记录文件。
* **`README.md`**: 项目的主说明文件，即本文档。
