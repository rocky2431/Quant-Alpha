# 02 清洗、时点对齐与 EnvState

> **2026-09-18 当前讨论稿｜按 UNIFIED-PLAN v1 细化｜未批准实施。** [索引](README.md) · [总架构](ARCHITECTURE.md) · [MVP](MVP.md) · 上游：[01 录制与接入](01-DATA-RECORDING-INGESTION.md) · 下游：[03 会计核心](03-ACCOUNTING-CORE-RECONCILIATION.md)、[04 提取](04-EXTRACTION-SEMANTICS.md)、[05 算子](05-OPERATORS-FEATURES.md)

## 1. 目的与边界

把 `RawRecord` 转成在每次决策时确实可使用的观测，保留不确定性与历史版本，并给每个决策时点生成一个 `EnvState`。环境是所有者三层抽象中"数据所处的环境"的落点：它只能由决策时可观测量定义，事后的牛熊或"事件后"分组只能作诊断标签，不能进入观察。

输入为 `RawRecord`、决策时间、场所规则版本、数据分区。输出为 `AlignedRecord`、`EnvState`、修复记录、缺失与新鲜度字段、版本映射。

清洗包含规则性错误处理；去噪是依赖任务的候选变换，不能默认删除异常行情。文本的重复传播与冲突归并由 04 处理；特征冗余由 07 处理。

## 2. 对象与契约

### 2.1 六时间字段与使用规则

| 字段 | 含义与限制 |
|---|---|
| `event_time` | 事件实际发生时刻，不自动代表系统知道它 |
| `published_at` | 来源对外发布时刻；修订另建版本 |
| `first_observed_at` | 本系统实际首次收到该版本的时刻；回填为空 |
| `processing_ready_at` | 解码、提取完成、可供策略使用的时刻 |
| `usable_at` | 本实验确定的可用时刻，须满足前述因果先后；回填时附延迟假设并标注 |
| `retrieved_at` | 本次取得数据的时刻，只用于血缘 |

决策时点 `t` 只读 `usable_at <= t` 的记录，按实体与字段取当时适用的最新版本，并输出数据龄与过期状态。已知未来的**日程**可提前使用，未来的**发布内容**不能。

### 2.2 `AlignedRecord`

| 字段 | 含义 |
|---|---|
| `raw_record_ids` | 来源 `RawRecord` 引用，合并时保留全部 |
| `entity` / `field` | 实体与字段键，如 `BTC.mark_px`、`xyz:CL.external_px` |
| `value` / `unit` | 值与单位；单位来自场所规则 |
| `version` | 该实体字段的版本序号 |
| `usable_at` | 见上 |
| `age_at_decision` | 决策时点减 `usable_at` |
| `staleness` | 是否超过该来源的最大数据龄 |
| `missing_reason` | 缺失原因：断流、来源未发布、隔离、超期 |
| `repair_log` | 修复动作与版本；无法确定的记录标隔离而不补造 |
| `latency_assumption` | 回填记录的延迟假设与来源 |

### 2.3 `EnvState`

| 字段组 | 字段 | 生成规则 |
|---|---|---|
| 规则版本 | `spec_version`、`dex`、`deployer`、`collateral`、`max_leverage`、`fee_tier` | 来自 03 的 `AccountingSpec` 版本与 01 的规则快照；变化即新版本 |
| 市场状态桶 | `vol_bucket`、`depth_bucket`、`funding_bucket`、`spread_bucket` | 用决策前已完成窗口的滚动分位数，分位数边界在训练分区拟合并冻结 |
| 会话状态 | `session_phase`（external/internal）、`time_to_reopen`、`bounds_remaining`、`reanchor_count`、`freeze_px_minus_oracle`、`external_source_health` | 来自 `activeAssetCtx`、oracle 更新流与规范索引；A 轨原生市场恒为 external 且会话字段为空 |
| 数据质量 | `missing_flags`、`stale_flags`、`latency_regime`、`gap_recent` | 来自 `AlignedRecord` 聚合 |
| 制度断点 | `regime_break_id` | 来自 01 事件台账，仅作诊断分组，不入策略观察 |

`env_state_id` 是这些字段的哈希。观察、特征快照、`StepTrace`、`EvaluationBundle` 都带它。

## 3. 具体过程

1. 校验单位、编码、时区、标识与交易字段关系；HIP-3 市场检查 `dex` 与部署者标识存在。
2. 以 `record_id` 与 `content_hash` 判定原始重复，保留被合并记录的引用与接收信息；来源不同的相同内容不在此层删除。
3. 处理乱序、缺段与重复事件；可修复的错误生成新处理版本，无法确定的记录隔离，不补造价格或成交。
4. 为公告、规则页、宏观值保留修订序列，建立当时可见版本视图；规则页版本交 03 生成 `spec_version`。
5. 按真实可用时刻做 backward as-of 对齐；不同频率分别聚合，窗口只含当时已完成的数据；未完成 K 线用不同字段名。Polars 带分组的 as-of 不检查组内排序，先显式 sort ✅ [E4 Q2]。
6. 把缺失、新鲜度、数据龄与更新次数作为状态输出，不只输出补齐后的数值。
7. 生成 `EnvState`：规则版本取自 03；市场状态桶的分位数边界只用训练分区拟合，应用到验证与前向时冻结；会话状态从 `activeAssetCtx` 的 `external` 是否更新、oracle 更新流与会话日历推出；外部参考源健康按更新间隔与跨源偏离判定。
8. 插补、缩放、滤波若需拟合，只用训练分区；应用到验证与未来时冻结拟合结果或遵守预定更新规则。
9. 为代表性决策生成"为什么这条信息此时可见、此时处于哪个环境"的可追溯证据。
10. 制度断点只写 `regime_break_id`，供 10 与 11 分组诊断。

## 4. 双轨示例

**轨道 A（示意时间，非已测延迟）。** 官方公告在 `T` 发布，系统 `T+2s` 接收，类型化提取 `T+8s` 完成。`T+5s` 的观察只能使用此前内容与"新文档待处理"状态；`T+10s` 才能用新语义。若 BTC 簿在 `T+10s` 前已超过最大数据龄，`stale_flags` 同时置位。当时 `EnvState` 为 `spec_version = hl-perp-2026-09-a`、`vol_bucket = 3/5`、`depth_bucket = 2/5`、`session_phase = external`、`latency_regime = normal`。

**轨道 B（示意）。** 某商品 `xyz:` 市场周五 `21:00 UTC` 外部收盘。`activeAssetCtx.external` 自此不再更新，oracle 更新流改为内部 EWMA。02 在下一决策时点生成 `session_phase = internal`、`time_to_reopen = 2d 3h`、`bounds_remaining = 0.9`（距上界剩余占比）、`reanchor_count = 0`、`freeze_px_minus_oracle = +0.6%`、`external_source_health = frozen_by_schedule`。周日夜间若 oracle 更新间隔超过阈值，健康字段转 `degraded`，这是决策时可观测量。事后知道重开跳幅是 +1.4%，只能写进 `regime_break_id` 或诊断标签，不能进任何观察。

## 5. 失败观测与处置

| 观测 | 候选原因 | 区分方法 | 返回环节 |
|---|---|---|---|
| 全样本缩放、居中滤波改变过去特征 | 拟合范围越界 | prefix-invariance 测试：未来数据变化不改过去输出 | 02 限制拟合范围 |
| 删除极端跳变后回撤变小 | 真实清算或 oracle 事件被当噪声 | 回查 03 清算事件与 01 事故台账 | 保留尾部，附质量标记 |
| 断流后指标仍平稳 | 无限前向填充 | 检查 `staleness` 与 `missing_reason` | 02 加过期条件；09 禁止新开仓 |
| 使用最终修订值 | 版本视图缺失 | 修订链复原 | 02 或标明区间不可用 |
| 会话状态与实际不符 | 会话日历过期、oracle 流缺失 | 对照规范索引快照与 oracle 更新间隔 | 01 补快照；02 修推断规则 |
| 环境桶在前向漂移 | 分位数边界在验证段重拟合 | 检查边界版本与冻结记录 | 02；10 判定泄漏 |
| 事后标签混入观察 | `regime_break_id` 被特征引用 | 静态 lint 加字段白名单 | 05 拒绝该特征 |

## 6. 两速 AI 在本环节的权限

| 环节动作 | 慢速推理 Agent | 快速类型化模型 | 数值/确定性程序 | 裁决者 |
|---|---|---|---|---|
| 提出清洗规则与修复方案 | 提议规则与原因解释 | — | 执行规则、生成 `repair_log` | prefix-invariance 与隔离测试 |
| 版本视图与 as-of 对齐 | — | — | 确定性执行 | 逐笔追溯测试 |
| `EnvState` 字段定义 | 提议新字段与机制理由 | 可对公告分类为"规则变更/事故/其他"作初筛 | 生成与哈希 | 03 的 `spec_version` 加 10 的分组稽核 |
| 分位数边界拟合 | — | — | 只在训练分区拟合并冻结 | 拟合范围检查 |
| 制度断点标注 | 提议断点候选 | 可初筛 | 落表 | 只作诊断，不入观察 |

## 7. 分阶段能力与证据

| 阶段 | 本环节实质内容 | 验证与推进条件 |
|---|---|---|
| 最小 MVP（对应 10 月 15 日改约交付） | 六时间字段统一、原始去重、版本保留、backward as-of、缺失与过期标记、`EnvState` 五组字段生成、分位数边界冻结 | 选定事件簇前后逐笔可追溯；未来数据变化不影响已冻结的过去输入；回填的延迟假设显式标注；两轨会话字段语义正确 |
| 第二步：针对失败补强 | 对已发现的乱序、延迟偏差、修订或缺失补处理；必要时比较因果滤波与原始通道；补 `EnvState` 字段 | 修复消除具体错误并检验是否改善下游；不能只看曲线更平滑 |
| 第三步：跨状态与运行验证 | 断流、来源改版、夏令时、极端行情、oracle 降级下的一致性；环境桶在前向的稳定性 | 历史与实时输入语义一致；故障恢复不引入未来信息或重复动作 |
| 完整能力：可复用研究产品 | 按任一研究版本恢复当时可见数据与 `EnvState`；比较处理变更影响；支持多频率任务 | 可解释每项修复及适用边界；复杂去噪仅在有下游证据时保留 |

## 8. 依赖与待定

01 提供原始字段、覆盖与规则快照；03 提供 `spec_version` 并消费对齐后的 asset contexts 与 fills；04 的提取耗时反向影响 `usable_at`；05 与 07 的拟合范围必须遵守本环节时间规则；10 负责整条流程的隔离验证与按 `env_state_id` 分组评价。待定为真实延迟分布、各来源最大数据龄、市场状态桶的分位数个数、外部参考源健康的阈值、缺失时的操作策略；不能用一个通用超时覆盖所有来源。
