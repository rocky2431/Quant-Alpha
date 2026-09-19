# 05 算子、特征与三态有效域

> **2026-09-18 当前讨论稿｜按 UNIFIED-PLAN v1 细化｜未批准实施。** [索引](README.md) · [总架构](ARCHITECTURE.md) · [MVP](MVP.md) · 上游：[02 清洗与 EnvState](02-CLEANING-ALIGNMENT-ENVSTATE.md)、[04 提取](04-EXTRACTION-SEMANTICS.md) · 下游：[07 融合](07-FUSION-SELECTION.md)、[08 学习](08-LEARNING.md)

## 1. 目的与边界

把可用信息变成能重算、组合、评价的数值或结构化表示，并让每个表示自带"在哪种环境下被什么探针检验过"的证据。这是所有者所说抽象层的核心：特征不是一列数，而是一份带契约的对象，契约里写明它的输入、计算、拟合范围、可用时间、有效域、探针结果、因果声明与提取器版本。

输入为 `AlignedRecord`、`EventEvidence`、`EnvState` 与 `Candidate` 中的机制假设；输出 `FeatureSpec` 与带时间和版本的 `FeatureFrame`。因子既可以是人工表达式，也可以是学习得到的表征；表达式不等于只能使用价量。

边界：本环节不判定特征的条件增量，那是 [07 融合](07-FUSION-SELECTION.md) 与 [10 验证](10-VALIDATION-GATES.md) 的事；本环节负责把判定结果写回 `FeatureSpec`。初期用普通函数和现成算子，不开发专用 DSL。

## 2. 对象与契约

`FeatureSpec` 字段级定义。前六组沿用生命周期派 04 文档，后四组是本套新增。

| 字段组 | 字段 | 内容与限制 |
|---|---|---|
| 输入 | `inputs[]`：字段、单位、来源、实体键、所需历史长度、质量状态要求 | 引用 `AlignedRecord` 或 `EventEvidence` 字段 |
| 计算 | `formula_or_model`、`params`、`window_direction`、`aggregation`、`missing_rule` | 确切公式或模型引用；窗口只向后 |
| 拟合 | `requires_fit`、`fit_scope`、`fit_artifact_version` | 需拟合的缩放、编码器、降维只使用允许的训练分区 |
| 时间 | `available_at_rule`、`compute_latency`、`depends_on_completed_bucket`、`reset_rule` | 输出可用时刻由输入可用时刻加计算延迟决定 |
| 输出 | `name`、`dtype`、`unit`、`range`、`missing_flag`、`confidence_field` | 缺失是状态，不填零 |
| 证据 | `record_refs`、`impl_version`、`numeric_checks` | 与小样本手算或来源数据核对 |
| **有效域** | `validity_env{env_bucket: 有效 / 反证成立 / 证据不足, evidence_ref}` | 按 `EnvState` 桶记录三态之一；部署时可保守拒用后两态，但不把"证据不足"写成"已失效" |
| **探针结果** | `probe_results{known_signal_recovery, negative_control, point_in_time_test, dsr, seed_stability}` 各含版本与日期 | 任一项未跑标记为未检验，不默认通过 |
| **因果声明** | `causal_declaration{mechanism, controls_allowed[], controls_forbidden[], collider_risk_note}` | 显式写明控制变量与禁止的控制项 ✅（`../2026-09-18-unified-plan/evidence/02-academic-2023-2026.md` §A） |
| **提取器版本** | `extractor_version`、`lookahead_propensity_summary` | 由 04 产生的字段必填；用于污染加权或剔除 |
| 制约关系 | `redundancy_group_id`、`shared_exposure_group_id`、`conditional_complement_of[]` | 由 07 的消融写回 |

`FeatureFrame` 是按 `env_state_id`、`feature_version`、`dataset_version` 切片的数值快照，每行带 `available_at` 与 `missing_flag`。

**三种制约关系的定义**：冗余组是高相关且无时效差异的特征集合，组内只保留一个进模型或合并；共同暴露是在同一风险上重复下注的特征集合，组合时按暴露而非按特征数计权；条件互补是"一个特征只在另一个特征定义的环境桶里有价值"，记录为 `conditional_complement_of` 并体现在 `validity_env` 的桶划分上。

## 3. 具体过程

1. 从 `Candidate` 的机制陈述提出表示需求，例如"事件内容相同但流动性状态不同，反应和可成交性不同"。
2. 先建立少量有明确含义的字段，写全契约前六组；不以自动生成数量为目标。
3. 写 `causal_declaration`：机制、允许的控制项、禁止的控制项；禁止项来自"过度控制 collider 会系统性奖励结构错误因子"的规则。
4. 计算无拟合算子；需拟合的缩放、编码器、降维只在训练分区拟合，`fit_scope` 写明；应用到验证或前向时冻结拟合结果。
5. 生成 `FeatureFrame`，每行带 `available_at` 与 `env_state_id`；记录计算成本、缺失比例与分布。
6. 跑探针：已知信号恢复（在该特征参与的学习路径上注入未来收益，必须被学会）、阴性对照（零预测能力合成数据上不得稳定"有效"）、point-in-time 测试（未来数据变化不改过去特征值）、DSR、跨 seed 稳定性；结果写 `probe_results`。
7. 检查常数、除零、量纲、未来依赖、训练与实时差异；与小样本手算或来源数据核对。
8. 交 [07](07-FUSION-SELECTION.md) 做组合与增量验证；不因单独 IC 低直接删除。07 的消融结果写回 `validity_env` 三态与制约关系字段。
9. 同一计算代码用于历史回放与前向；不另写第二份公式。任何实现变更升 `feature_version`。

## 4. 双轨示例

**轨道 A 最小表示表（假设参数）。**

| 表示 | 定义建议 | 有效域与边界 |
|---|---|---|
| 短时收益与波动 | 对决策前已完成的价格窗口计算，窗口 5 到 60 分钟 | 不混入当前未完成窗口 |
| 买卖价差 | `(ask - bid) / mid` 转基点 | 缺报价时输出缺失，不默认零成本 |
| 深度失衡 | 指定价格范围内 `(bid_qty - ask_qty) / (bid_qty + ask_qty)` | 分母为零缺失；范围与方向一致 |
| 主动成交方向统计 | 可识别的成交方向滚动聚合 | 无法可靠识别时不冒充 |
| 订单流特征 | 跨场所净买卖量，Binance 参考加 HL 本地 | 研究证据来自 CEX 数据 ✅🟡（`../2026-09-18-unified-plan/evidence/02-academic-2023-2026.md` §F）；HL 侧是跟随者，`validity_env` 先标证据不足 |
| 资金费状态 | 当前 premium、距 clamp 边界余量、累积资金费 | 由 03 的 `AccountingSpec` 公式计算 |
| 事件变化与条件字段 | 04 的 `EventEvidence` 结构化字段编码 | 带 `extractor_version` 与污染权重；不附未来收益 |
| 事件龄与数据龄 | 当前时刻减对应可用时刻 | 区分原文发布龄、语义可用龄、盘口龄 |
| 语义与流动性交互 | 事件字段与价差、波动、深度状态的组合 | 交互假设先写进 `causal_declaration` |
| 账户状态 | 持仓、可用资金、未完成订单 | 由 09 当时账户生成 |

假设一例：`event_age_minutes` 在 `EnvState` 桶"高波动、新事件"里被 07 的消融判为有效，在"低波动、无事件"桶里判为证据不足；`validity_env` 分别记录，策略层在后一桶里不得使用该特征。

**轨道 B 的 m 通道表示表（假设参数）。**

| 表示 | 定义建议 | 有效域与边界 |
|---|---|---|
| 会话状态 | external/internal，来自 02 的 `EnvState` | 决定其他 m 通道特征是否有定义 |
| 距重开时长 | 下一外部开市时间减当前时刻 | 依赖版本化的会话日历 |
| bounds 剩余空间 | `(bound_upper - mark) / mark` 与下侧对称 | 由 03 的 bounds v2 公式与 `spec_version` 计算 |
| 重锚定计数 | 本会话已用与剩余 reset 次数 | 用尽即硬顶，状态跳变 |
| 冻结价与 oracle 差 | `(oracle_internal - frozen_external) / frozen_external` | 外部参考源不健康时缺失 |
| 内部簿深度与价差 | 同 A 轨定义，作用于 `xyz:` 市场 | 休市内深度通常薄，缺失比例高 |
| 跳幅分层 | 冻结价相对上次外部收盘的变动分桶 | 用于骨架策略的分层入场；来源见 MVP §2 |
| 部署者与抵押品状态 | DEX 级保证金、抵押品、oracle 健康 | 来自 `EnvState.spec_version` |

假设一例：`bounds_remaining` 在"internal、跳幅 >100bp"桶里有效，在"external"桶里无定义，`validity_env` 对 external 桶记录"无定义"而非"反证成立"。

## 5. 失败观测与处置

| 观测 | 候选原因 | 区分方法 | 返回环节 |
|---|---|---|---|
| 表达式大量不同但输出几乎相同 | 重复搜索、冗余 | 相关矩阵与冗余组归并 | 05：合并进 `redundancy_group_id` |
| 跨年符号翻转 | 机制或状态问题、时间问题 | 按 `EnvState` 桶分组检验 | 05/07：`validity_env` 分桶 |
| 训练特征正常、实时大量缺失 | 契约不一致、实时字段缺 | 比较训练与前向 `missing_flag` 分布 | 01/02/05 |
| 去噪后收益提升只因抹除尾部 | 清洗污染 | 保留原始通道对照 | 02 |
| 复杂特征仅在特定事件成立 | 过拟合单事件 | 按事件簇留一法 | 07/10 |
| point-in-time 测试失败 | 使用了未来数据或全样本拟合 | 未来数据扰动后过去值是否变化 | 05 步骤 4、6 |
| 已知信号恢复失败 | 该特征参与的学习路径坏了 | 换路径注入同一信号 | 08：先修学习路径 |
| 提取字段污染权重高 | 前视 | 04 的 Lookahead Propensity 分组 | 04/05：降权或只前向评价 |
| 共同暴露未识别，组合放大回撤 | 多特征押同一风险 | 持仓共同暴露分析 | 07 |

## 6. 两速 AI 在本环节的权限

| 环节动作 | 慢速推理 Agent | 快速类型化模型 | 数值/确定性程序 | 裁决者 |
|---|---|---|---|---|
| 提出表示需求与表达式 | 提议，附 `causal_declaration` 草稿 | — | schema 校验 | 探针与 07 消融 |
| 实现算子代码 | 提议代码 | — | 只通过白名单特征工具访问数据 | point-in-time 测试、数值核对 |
| 拟合缩放、编码器 | — | — | 在 `fit_scope` 内拟合 | 分区规则 |
| 跑探针 | 解释结果 | — | 执行注入、阴性对照、DSR | CI 报告 |
| 写回 `validity_env` | 提议解释 | — | 按 07 消融结果写回 | 10 的分区规则 |
| 提取字段进特征 | — | 提供字段 | 编码、污染加权 | 04 校准审计过的版本 |

Agent 生成的代码只能通过白名单特征工具访问数据，不能直接读原始表或未来分区 🟡（`../2026-09-18-unified-plan/evidence/04-engineering-platforms.md` Q6）。

## 7. 分阶段能力与证据

| 阶段 | 本环节实质内容 | 验证与推进条件 |
|---|---|---|
| 最小 MVP（对应 10 月 15 日改约交付） | A 轨与 B 轨上述少量特征，全部带完整契约与 `causal_declaration`；探针在 CI 中运行并写 `probe_results`；`validity_env` 按 `EnvState` 桶初始化为证据不足，随 07 消融更新；同一计算用于回放与前向 | 代表性事件可重算；未来输入变化不改过去结果；单位与缺失规则一致；已知信号恢复通过；每个特征的三态有效域可查询 |
| 第二步：针对失败补强 | 若语义或状态丢失，补针对性算子、历史窗口、条件交互或编码器；若冗余，合并表达式；若共同暴露，改按暴露计权 | 修改的预期效果与消融一致；新增代价可接受；无增量候选淘汰 |
| 第三步：跨状态与运行验证 | 检验不同波动、流动性、会话与事件簇状态的表示稳定性、漂移、计算时效与实时一致性 | 特征不只对某段历史有效；在缺失与极端状态仍有明确语义；`validity_env` 覆盖的桶随独立事件簇增加 |
| 完整能力：可复用研究产品 | 带血缘、用途、有效域与制约关系的算子目录，支持组合、版本比较与必要的自动生成 | 每个候选有可执行契约与增量证据；更丰富的表示依据实际问题加入 |

## 8. 依赖与待定

[02](02-CLEANING-ALIGNMENT-ENVSTATE.md) 保证时间与 `EnvState`；[03](03-ACCOUNTING-CORE-RECONCILIATION.md) 提供资金费、bounds 等机制量的公式；[04](04-EXTRACTION-SEMANTICS.md) 保证语义证据与提取器版本；[07](07-FUSION-SELECTION.md) 负责条件价值与选择并写回制约关系；[08](08-LEARNING.md) 负责学习；[10](10-VALIDATION-GATES.md) 控制整个搜索的选择偏差。

待定：窗口、深度范围、事件编码、拟合范围、实时预算；`EnvState` 桶的划分粒度（桶太细则每桶样本不足，太粗则有效域无意义）；探针接受阈值；订单流特征在 HL 侧的可得性。这些在 [MVP](MVP.md) 实例化后冻结，不以论文默认参数代替业务定义。
