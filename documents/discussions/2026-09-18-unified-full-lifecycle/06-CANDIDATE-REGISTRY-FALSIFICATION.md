# 06 候选登记簿、零假设与证伪流水线

> **2026-09-18 当前讨论稿｜按 UNIFIED-PLAN v1 细化｜未批准实施。** [索引](README.md) · [总架构](ARCHITECTURE.md) · [MVP](MVP.md) · 上游：[03 会计核心](03-ACCOUNTING-CORE-RECONCILIATION.md)、[05 算子与特征](05-OPERATORS-FEATURES.md) · 下游：[07 融合](07-FUSION-SELECTION.md)、[08 学习](08-LEARNING.md)、[10 验证](10-VALIDATION-GATES.md)、[12 生命周期](12-LIFECYCLE-CAPITAL-OPS.md)

## 1. 目的与边界

让机会发现可重复、可审计、可淘汰。每一个想做的策略、特征族或机制利用，在进入昂贵的训练与评价之前，必须先成为一张登记过的 `Candidate`：写清机制、可证伪预期、零假设、证伪脚本、看数据前锁定的阈值、允许的试验次数、环境适用范围与数据预算。缺任何一项，程序拒收，不进入 07 及之后的环节。

这是把机制派登记簿与生命周期派 `HypothesisCard` 合并后的对象。它同时管住三件事：Agent 幻觉、研究者的 p-hacking、外部不可信内容对系统的影响。**唯一的授权者是验收通过，不是提议者的置信度。**

本环节输出假设，不输出策略，也不输出 alpha。"接缝存在"是公开信息，"接缝可利用"需要具体状态、流动性与成本；后者只能由后续环节的仿真、回测与前向证明。

## 2. 对象与契约

`Candidate` 字段级定义：

| 字段 | 必填 | 内容与限制 |
|---|---|---|
| `candidate_id`、`track_id` | 是 | 唯一标识；轨道 A/B/C |
| `mechanism_ref` | 是 | 指向 `AccountingSpec` 字段（如 `hyperliquid.hip3.discovery_bounds`）或一段经济机制陈述；不能是"数据挖掘发现" |
| `falsifiable_expectation` | 是 | 若机制成立，预计观察到什么中间量与最终量的变化 |
| `null_hypothesis` | 是 | 若机制不成立，最合理的替代解释；写成可检验的陈述 |
| `falsify_script` | 是 | 可运行脚本路径；输入数据版本、输出判定；不含策略逻辑 |
| `threshold_hash` | 是 | 判定阈值文件的 SHA-256，在 `falsify_script` 首次运行**之前**提交；运行时校验一致 |
| `max_trials` | 是 | 允许的试验次数上限，含参数扫描与变体；超出即自动标记为多重检验风险 |
| `env_scope` | 是 | 声称适用的 `EnvState` 桶集合；桶外不得声称有效 |
| `data_budget` | 是 | 允许消耗的开发窗口与独立事件簇数；引用数据预算账本 |
| `seam_type` | 否 | 六类接缝之一，见 §3 |
| `state` | 系统 | 候选 → 证伪中 → 已确认 → 冻结 → 独立评价 → 影子 → 小额实盘 → 放量 → 退役；任意可回退 |
| `evidence[]` | 系统 | `run_id`、`EvaluationBundle` 引用 |
| `proposer` | 系统 | `agent_id` 或人 |
| `trials_used` | 系统 | 已用试验数，由程序累计 |
| `created_at`、`updated_at` | 系统 | — |

**入库校验由程序强制**：八项必填缺一拒收；`threshold_hash` 对应文件必须在任何相关运行之前存在于版本库；`env_scope` 中的桶必须是 [02](02-CLEANING-ALIGNMENT-ENVSTATE.md) 已定义的桶；`data_budget` 必须能在数据预算账本中扣减。

**状态机**（与 [12 生命周期](12-LIFECYCLE-CAPITAL-OPS.md) 的 `LifecycleState` 共用）：

```
候选 ──► 证伪中 ──┬──► 已否决（零假设未被证伪，或试验数超限）
                  └──► 已确认 ──► 冻结 ──► 独立评价 ──► 影子 ──► 小额实盘 ──► 放量 ──► 退役
任意状态 ◄── 回退（机制漂移、对账红灯、偏差不可归因、场所规则变更）
```

转移由代码判定，条件是 [10 验证](10-VALIDATION-GATES.md) 的九关；`已上线` 的候选被 03 的漂移分诊持续盯，对应 `spec_version` 变化时自动回退到"证伪中"。

## 3. 具体过程

1. **假设生成**：来源有三类。慢速推理 Agent 读场所文档、公告、论文后提议；人提出主观机制；接缝扫描器对 `AccountingSpec` 做静态查询。接缝分类学六类作为假设生成器，不作为 alpha 来源：clamp/floor/cap；控制量与计价量失配；冻结或陈旧状态；强制流；阈值离散跳变；更新频率失配。每类自带一条强制零假设。
2. **八项填写**：提议者填全八项；`falsify_script` 只做归因或检验，不建策略；阈值写入独立文件并提交。
3. **入库校验**：程序校验必填、哈希存在、桶合法、预算可扣；通过则 `state=候选`。
4. **排序**：按 `priority = 1 / (证伪所需时间 × 证伪所需新基建)` 排序，不按预期收益排序。吞吐量来自快速淘汰，不来自押中。
5. **证伪执行**：`state=证伪中`；运行 `falsify_script`，运行前校验 `threshold_hash`；结果写 `evidence[]`；`trials_used` 累加。两种结论都算达成：零假设被证伪进入"已确认"，未被证伪进入"已否决"。
6. **试验数管理**：`trials_used` 超过 `max_trials` 时自动标记并停止新运行；要继续必须以新 `candidate_id` 登记并说明与旧候选的关系，旧试验数计入 DSR。
7. **进入下游**：已确认候选的机制与 `env_scope` 交 [07](07-FUSION-SELECTION.md) 与 [08](08-LEARNING.md)；之后的状态转移由 [10](10-VALIDATION-GATES.md) 与 [12](12-LIFECYCLE-CAPITAL-OPS.md) 判定。
8. **回退与退役**：03 的 `ReconciliationReport` 显示相关 `spec_version` 漂移，或 12 的偏差不可归因，候选自动回退；退役候选保留全部记录，失败记录不删除。

## 4. 双轨示例

**轨道 A：卡片 H-A（按 [MVP](MVP.md) §2，参数为假设）。** `mechanism_ref`："订单流与资金费状态携带一小时期限内未被 HL 价格完全反映的信息；类型化宏观语义在特定流动性状态下提供条件增量"。`falsifiable_expectation`：加入语义字段后，"高波动、新事件"桶内的成本后净增量为正且换手可控。`null_hypothesis`：HL 是 Binance 的跟随者，本地特征只是滞后复述；语义无增量。`falsify_script`：`falsify/track_a_orderflow_semantics.py`，输入 `dataset_version`，输出三命题各自的判定。`threshold_hash`：`sha256(thresholds/track_a_v0.json)`。`max_trials=24`。`env_scope`：A 轨全部桶，声称有效的桶随消融更新。`data_budget`：开发窗口 D1，独立事件簇 ≥ N 个（N 待定）。

**轨道 B：卡片 H-B。** `mechanism_ref`：`hyperliquid.hip3.discovery_bounds` 与会话日历。`falsifiable_expectation`：休市内内部价相对重开参考价存在按跳幅分层的可预期收敛。`null_hypothesis` 三条并列（隔夜风险定价；费率后无残差；已被做市商吃掉）。`falsify_script`：`falsify/track_b_reopen_convergence.py`，分别对三条零假设输出判定。`threshold_hash` 在看任何 `xyz:` 数据前提交。`max_trials=12`。`env_scope`：internal 会话、外部参考源健康、跳幅分桶。`data_budget`：受 `xyz:` 历史覆盖限制，前三个工程日核清后填入。B 轨工程工时 ≤ 共享预算 20% 也登记在 `data_budget` 的旁注中。

**接缝扫描示例。** 对 `AccountingSpec` 做静态查询发现 `hip3.discovery_bounds.reset_count_max` 是阈值离散跳变；生成候选草稿"重锚定次数用尽前后 mark 行为不同"，附零假设"跳变点已有人守"；`falsify_script` 只统计用尽前后价差与深度分布差异，不建策略。

## 5. 失败观测与处置

| 观测 | 候选原因 | 区分方法 | 返回环节 |
|---|---|---|---|
| 登记簿全是假阳性 | 零假设写得太弱；阈值事后调整 | 检查 `threshold_hash` 提交时间；重写零假设 | 06 步骤 2 |
| 同一机制以多个 `candidate_id` 反复登记 | 规避 `max_trials` | 按 `mechanism_ref` 聚合试验数计 DSR | 06 步骤 6、10 |
| 证伪脚本悄悄包含策略逻辑 | 越界 | 代码审查；脚本只输出判定 | 06 步骤 2 |
| 候选在桶外被声称有效 | `env_scope` 被忽略 | 10 的评价按桶报告 | 07/10 |
| 已上线候选在场所改参后仍运行 | 漂移未触发回退 | 03 的 `ReconciliationReport` 与 `spec_version` 关联 | 03/06/12 |
| 排序按预期收益而非证伪成本 | 流程漂移 | 检查 `priority` 计算 | 06 步骤 4 |
| 外部文本直接改了候选参数 | 不可信内容越权 | 变更必须经提议与校验 | 04/06 |

## 6. 两速 AI 在本环节的权限

| 环节动作 | 慢速推理 Agent | 快速类型化模型 | 数值/确定性程序 | 裁决者 |
|---|---|---|---|---|
| 提议候选与八项内容 | 提议 | 可做初筛路由（如判断文本是否含规则变更） | schema 校验、哈希存在性校验、预算扣减 | 程序拒收缺项 |
| 接缝扫描 | 解释扫描结果 | — | 对 `AccountingSpec` 静态查询 | 强制零假设附带 |
| 证伪脚本编写 | 提议代码 | — | 运行前校验哈希；只读数据 | 阈值在看数据前锁定 |
| 判定证伪结果 | 不可 | 不可 | 按阈值文件判定 | CI |
| 修改阈值或 `max_trials` | 不可 | 不可 | 只能以新候选登记 | 版本库 |
| 状态转移 | 不可 | 不可 | 按九关结果转移 | 10/12 |

Agent 的全部产出是提议；它不是任何验收的裁判；它不能选择或修改阈值；不能以"另一个模型认可了"作为通过依据。

## 7. 分阶段能力与证据

| 阶段 | 本环节实质内容 | 验证与推进条件 |
|---|---|---|
| 最小 MVP（对应 10 月 15 日改约交付） | 登记簿与状态机有测试；八项必填由程序强制；H-A 与 H-B 两张卡片入库；两轨证伪脚本与阈值哈希在看数据前提交；至少 H-B 的三条零假设走完"证伪中"到"已确认/已否决" | 缺项确被拒收；哈希校验在 CI 中触发；两种结论都算达成且如实记录 |
| 第二步：针对失败补强 | 针对已见的假阳性模式改善零假设模板；接缝扫描扩展到更多 `AccountingSpec` 字段；对相关候选的试验数聚合计 DSR | 等预算下更快淘汰；无证据不新增候选类别 |
| 第三步：跨状态与运行验证 | 候选按 `env_scope` 分桶评价与回退；03 漂移信号自动回退已上线候选；候选间相关性与共同暴露进入登记 | 回退在真实漂移事件上触发过；候选全清单、失败尝试、相关性、搜索预算完整保留 |
| 完整能力：可复用研究产品 | 跨轨道、跨场所的可审查假设资产；有需要时训练研究选择策略，其动作是实验选择与预算分配 | 研究效率与经济增量分别与固定流程比较；自动化程度不是目标 |

## 8. 依赖与待定

[03](03-ACCOUNTING-CORE-RECONCILIATION.md) 提供 `AccountingSpec` 供接缝扫描与 `mechanism_ref` 引用，并通过 `ReconciliationReport` 触发回退；[02](02-CLEANING-ALIGNMENT-ENVSTATE.md) 定义 `env_scope` 可用的桶；[10](10-VALIDATION-GATES.md) 维护数据预算账本与九关；[12](12-LIFECYCLE-CAPITAL-OPS.md) 与本环节共用状态机；[11](11-RESEARCH-FEEDBACK-AGENT-LOOP.md) 的 `ResearchDecision` 产生新候选或修改候选。

待定：两张卡片的具体阈值数字与 `max_trials`；独立事件簇的最低数量；接缝扫描器的实现范围（首期可以是对 spec 字段的人工查询清单）；候选相关性的计算方式；B 轨预算旁注如何进入预算账本。
