这组 TODO 触及了 Skill 架构的核心问题：**封装粒度应该按工作流步骤切（水平）还是按业务事件切（垂直）？** 逐个分析。

---

## TODO 1：Step 6 与 Step 4 是否耦合？是否按「额度事件」垂直整合？

**耦合是事实。** Step 6 的公式选择 100% 依赖 Step 4 的归因结果。新增一种额度事件（如临额过期），必须同时改两个 Skill。这违反了"变更收敛到单一文件"的目标。

但直接合并有代价：**架构文档 §2.4 的 AP-8（约束靠机制不靠期望）要求控制层代码在 Step 4 和 Step 6 之间做物理隔断。** 如果把归因和验证塞进一个 Skill 让 Agent SDK 一次跑完，控制层就失去了对 Step 5（补充数据）和代码二次校验的控制权。

**决策：不合并 Skill 执行，但合并 Skill 知识来源。**

```
当前设计（水平切分，按步骤）：
  skill_limit_rule_match.md    → Step 4 加载
  skill_limit_calc_verify.md   → Step 6 加载
  问题：新增事件类型需改两个文件

目标设计（垂直切分，按事件）：
  skill_event_installment_posting.md  → 包含「归因」和「验证」两个 section
  skill_event_temp_limit_expiry.md    → 同上
  skill_event_auth_transaction.md     → 同上
  
  控制层在 Step 4 加载 ## Attribution section
  控制层在 Step 6 加载 ## Verification section
  新增事件 = 新增一个文件，不改控制层
```

这样做的收益：业务知识的闭合性（一个事件的全部规则在一个文件里），同时保留了控制层在 Step 4 和 Step 6 之间的物理隔断。

**但这不是 MVP 做的事。** 原因：只有 1 个事件类型（分期卡溢缴款），没有足够样本验证 section 划分是否合理。MVP 保持现有两个文件，在 TECH-DEBT 表中记录垂直整合方案。

---

## TODO 2：溢缴款是否独立封装为子 Skill？封装原则是什么？

先回答原则问题。

**Skill 封装原则 = 业务决策的最小闭合单元。**

判定方法：给定输入，这个 Skill 能否**独立**产出输出，不需要运行时咨询另一个 Skill？

用这个原则检验现有 5 个 Skill：

| Skill | 闭合性 | 判断 |
|-------|--------|------|
| `skill_intent_parsing` | 给定客户问题 → 输出结构化意图 | ✅ 闭合 |
| `skill_tx_continuity_check` | 给定交易列表 → 输出 anomalies | ✅ 闭合 |
| `skill_limit_rule_match` | 给定 anomalies → 输出 matched_rules | ⚠️ 部分闭合——「怎么匹配规则」和「怎么验证规则」是同一个业务知识的两面 |
| `skill_limit_calc_verify` | 给定 matched_rules → 输出 verified | ❌ 不闭合——完全依赖 Step 4 的输出选公式 |
| `skill_report_generation` | 给定验证结果 → 输出报告 | ✅ 闭合 |

这验证了你的直觉：Step 4 和 Step 6 存在闭合性问题。

**回到溢缴款问题：** 溢缴款不是独立的额度事件——它是还款交易（4306 贷方入账）的**推理分支**，不是独立触发的。它没有自己的交易码，而是通过分析 TCL 冲抵明细推断出来的。

```
4306（贷方入账）
  → 查 TCL 冲抵明细
  → IF total_offset < tx_amount
      → 溢缴款 = tx_amount - total_offset
      → 溢缴款冲抵其他欠款 → 该部分不恢复额度
  → ELSE
      → 正常冲抵 → 恢复额度 = offset_amount
```

**决策：MVP 阶段溢缴款逻辑内嵌在 `skill_limit_rule_match` 的 4306 推理链中，不独立封装。**

提取为子 Skill 的触发信号：溢缴款的路由规则因账户类型不同而分化（如循环卡 vs 分期卡的溢缴款处理不同），此时推理分支膨胀到 >500 tokens，就该拆。

---

## TODO 3：Skill ≈ 业务逻辑编码？能否复用为代码审查基线？双重维护问题？

**这个思路方向对，但定位要精确。**

Skill 和代码的本质区别：

| 维度 | Skill | 代码 |
|------|-------|------|
| 消费者 | LLM（需要推理指引和自然语言上下文） | Runtime（需要精确语法和异常处理） |
| 表达方式 | 声明式（what rules + how to reason） | 命令式（how to execute） |
| 变更驱动 | 业务规则变更 | 技术实现变更 |
| 精度要求 | 允许模糊推理（"推断溢缴款"） | 必须精确（`overpayment = tx_amount - total_offset`） |

**Skill 不是代码的替代品，而是代码的上游规格说明。** 维护模型：

```
业务规则变更
  → 先改 Skill（业务方 + 工程师共建）
  → Skill 是 source of truth
  → 代码据此调整（如果需要）
```

**双重维护问题在当前架构中是受控的：**

真正同时存在于 Skill 和代码中的逻辑只有一处——Step 6 的数值校验。但这是**有意设计**的双重校验（AP-8）：Skill 告诉 Agent SDK 怎么构造公式，控制层代码做 `expected == actual ± 0.01` 的二次校验。两者的复杂度完全不对等——代码侧只有一行比较，不存在实质性的维护负担。

**Skill 作为代码审查基线是可行的**，但不是直接复用，而是作为 review checklist：

- 代码 review 时对照 Skill 检查：代码实现的业务规则是否与 Skill 中声明的一致？
- Skill review 时对照 trace：Agent 的实际推理链是否遵循了 Skill 中定义的推理模板？

这是两条互补的验证链路，不是维护两份代码。

---

## TODO 4：渐进式加载 / @import

你提到的"借助 Claude 的 @import"实现渐进式加载——这对应架构文档 §4 的 **VFS 演进选项**。

当前不需要，触发信号已在架构中定义：单 Skill > 2000 tokens 或规则 > 50 条。

但你的二级 Skill 思路可以预埋为 VFS 的目录结构：

```
未来目录结构（非 MVP）：
skills/
├── events/                              # 按事件垂直切分
│   ├── installment_posting.md           # 分期入账（含归因 + 验证）
│   ├── temp_limit_expiry.md             # 临额过期
│   └── auth_transaction.md              # 授权交易
├── mechanisms/                          # 跨事件共享的底层机制
│   ├── overpayment_routing.md           # 溢缴款路由规则（当分化时提取）
│   ├── limit_recovery_rules.md          # 额度恢复通用规则
│   └── tolerance_thresholds.md          # 容差标准
├── skill_intent_parsing.md              # Step 1（不变）
├── skill_tx_continuity_check.md         # Step 3（不变）
└── skill_report_generation.md           # Step 7（不变）
```

---

## 总决策

| 问题 | MVP 决策 | 触发重构信号 |
|------|---------|-------------|
| Step 4 + Step 6 合并 | 不合并。保持两个文件 + 控制层物理隔断 | 事件类型 ≥ 3 时，迁移为按事件垂直切分（单文件含 Attribution + Verification 两个 section） |
| 溢缴款独立 Skill | 不独立。内嵌在 4306 推理链中 | 溢缴款路由规则因账户类型分化时提取 |
| Skill 与代码双重维护 | 受控。Skill = source of truth，代码二次校验仅一行 | 如果出现代码层大量复制 Skill 逻辑的情况，重新评估边界 |
| 渐进式加载 | 不实现。单 Skill < 2000 tokens | 单 Skill > 2000 tokens 或规则 > 50 条 |
