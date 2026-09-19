# 04 类型化提取、语义与来源归因

> **2026-09-18 当前讨论稿｜按 UNIFIED-PLAN v1 细化｜未批准实施。** [索引](README.md) · [总架构](ARCHITECTURE.md) · [MVP](MVP.md) · 上游：[02 清洗与 EnvState](02-CLEANING-ALIGNMENT-ENVSTATE.md) · 下游：[05 算子与特征](05-OPERATORS-FEATURES.md)

## 1. 目的与边界

把非结构化输入（官方声明、公告、规则文本、新闻、场所 changelog）变成有类型、有来源、有校准置信、有时间戳的 `EventEvidence`，供 05 计算特征。本环节回答的是**第一层归因**：谁在何时以何种身份提出了什么主张，它属于哪个事件，与此前已知信息是什么关系。它不认定该信息造成了价格变化；模型与交易失败的诊断由 [11 研究反馈](11-RESEARCH-FEEDBACK-AGENT-LOOP.md) 负责。

本环节是两速 AI 分工最清楚的地方。**快速类型化模型**是在线主力：输入一段当时可得的文本，输出预定义 schema 内的字段与概率，延迟以毫秒到秒计。候选提取器有两类，微调小编码器或 LoRA 小模型，以及 Jev 类 System One 模型；两类进入同一校准审计，谁上线由审计结果决定，不由厂商速度数字决定。**慢速推理 Agent**只在离线：设计 schema、生成与核对标注样本、诊断提取错误、提议新的字段。它不进入在线提取路径。

输入为 `AlignedRecord` 中的文本类记录、历史同事件版本、实体映射、来源信息、`EnvState`。输出 `EventEvidence`。本环节不输出任何未来收益标签，也不输出"利多分数"这类无来源的值。

边界之外：训练预测标签属于 [08 学习](08-LEARNING.md)；重复传播与冲突归并只到"保留关系"为止，冗余处理由 [07 融合](07-FUSION-SELECTION.md) 负责。

## 2. 对象与契约

`EventEvidence` 字段级定义：

| 字段组 | 字段 | 含义与限制 |
|---|---|---|
| 标识 | `event_id`、`event_cluster_id`、`record_ref` | 事件、事件簇、指向 `AlignedRecord` 的引用；同一声明的转载共享 `event_id` |
| 来源 | `source_uri`、`author_or_org`、`source_role`（直接发布/引用/转述）、`source_reliability_basis` | 可信度依据是规则或标注，不是模型的流畅度 |
| 实体与事件 | `entities[]`（含候选与置信）、`event_type`、`economic_object`、`scope` | 实体不确定时保留候选，不强行合并 |
| 主张 | `claims[]`：`text_span`、`status`（提案/意向/决定/已生效）、`condition`、`horizon`、`numeric_value`、`unit`、`negation`、`uncertainty` | 每个字段回指原文片段位置 |
| 变化 | `delta_vs_prior`：新增/撤回/修正/无变化、`prior_version_ref` | 相对当时已知旧版本比较，不相对未来文本 |
| 关系 | `relation`：首次披露/传播/独立确认/修正/矛盾 | 冲突来源分别保留，不强行平均 |
| 校准 | `field_probabilities`、`calibration_audit_ref`、`abstain` | 概率来自校准审计过的模型；置信不足时输出 `abstain` 而非猜测 |
| 前视诊断 | `lookahead_propensity`、`entity_masked` | 按（实体，日期）计算的 Lookahead Propensity；输入是否做过去实体化 |
| 版本与时间 | `extractor_version`（模型、提示、schema、截止日期、校准审计版本）、`extraction_started_at`、`extraction_completed_at` | 提取完成时刻是可用性的一部分，写回 02 的 `processing_ready_at` |
| 环境 | `env_state_id` | 提取当时的 `EnvState` |

**校准审计**是提取器上线的前置条件，不是可选项。审计对象是每个字段的概率输出，方法是在我们自己标注的样本上画可靠性图并计算 ECE 与 Brier；样本量按字段类别与稀有风险事件分别确定，稀有类别（如"撤回"、"矛盾"）单独抽样；预先写下接受区间（例如某类别 ECE 上限），审计不过的模型不得进在线路径。厂商一致率、类型正确率、速度数字都不替代此审计 🟡（见 `../2026-09-18-unified-plan/evidence/02-academic-2023-2026.md` §E）。

**Lookahead Propensity** 是对每个（实体，日期）用只给日期的回忆查询估计模型是否"知道"该时点之后的结果，作为特征的污染权重或过滤器 ✅（同上 §C）。**point-in-time 语言模型**（按时间截止训练的开源权重）用作对照与诊断：若通用模型的提取结果显著优于同任务的 PiT 模型，差值是前视污染的候选证据；PiT 模型的结果不是效果的数学下界。**去实体化**（掩码公司名、代码）作为诊断运行，比较掩码前后字段一致性；它不是去污染证书。

## 3. 具体过程

1. **schema 设计（离线，慢速 Agent 提议，人审）**：按 A 轨与 B 轨各定义一版字段集与枚举；每个字段写明允许值、缺省为 `abstain` 的条件、稀有类别清单。schema 有版本号，进入 `extractor_version`。
2. **标注样本构造（离线）**：从历史 `AlignedRecord` 抽样，含普通日与事件日、含修正与矛盾案例；慢速 Agent 生成初标，人核对关键字段；样本按时间分组，评价时只用晚于训练样本的时间段。
3. **候选提取器训练或接入**：微调小编码器或 LoRA 小模型用标注样本训练；Jev 类模型按同一 schema 接入。两者输出统一为 `field_probabilities`。
4. **校准审计**：在留出的标注样本上计算每字段可靠性图、ECE、Brier、稀有类别召回；与预注册接受区间比较；结果写 `calibration_audit_ref`。不过者不上线。
5. **前视诊断**：对历史样本计算 Lookahead Propensity；运行去实体化对照与 PiT 模型对照；结果写入字段，供 05 决定是否降权或剔除。
6. **在线提取**：文本到达后，按 `usable_at` 顺序处理；记录 `extraction_started_at` 与 `extraction_completed_at`；置信不足输出 `abstain`；每条输出带 `extractor_version` 与 `env_state_id`。
7. **事件归属与关系**：按来源记录标识、内容摘要、实体与时间窗建立 `event_id`；判定关系类型；独立确认不按相同文本去重；重复传播保留计数。
8. **抽样复核（离线）**：每周从在线输出抽样，人与慢速 Agent 核对关键字段；错误类型进入失败观测表；需要时回到步骤 1 或 3 形成新版本。
9. **下游增量消融**：提取字段进入 05 后，由 [10 验证](10-VALIDATION-GATES.md) 的消融判断其条件增量；提取正确率高但下游无增量的字段不保留在线路径。

## 4. 双轨示例

**轨道 A（假设数值）。** 官方利率声明在 `T` 发布，系统 `T+2s` 收到，快速提取器 `T+6s` 完成：`event_type=policy_statement`、`delta_vs_prior=修正`（新增一句条件性措辞）、`claims[0].status=决定`、`claims[0].condition="视后续数据"`、`field_probabilities.delta=0.91`、`lookahead_propensity=0.02`、`abstain=false`、`extractor_version=encoder-v3/schema-A2/audit-2026-09`。`T+5s` 的决策观察不得包含该记录；`T+6s` 后 05 才能计算"事件龄"与"变化字段"。一家媒体在 `T+40s` 转述，`relation=传播`，共享 `event_id`，不新增事件。另一来源给出不同解读，`relation=矛盾`，分别保留。

**轨道 B（假设数值）。** trade.xyz 规范索引每日快照显示某商品市场的 discovery bounds 参数由 ±5% 改为 ±4%，changelog 未同步。提取器输出 `event_type=venue_rule_change`、`entities=[market_id]`、`claims[0].numeric_value=0.04`、`status=已生效`、`delta_vs_prior=修正`、`source_role=直接发布`。该 `EventEvidence` 不进特征，而是触发 [03 会计核心](03-ACCOUNTING-CORE-RECONCILIATION.md) 的漂移分诊与 `spec_version` 升版提议；`EnvState.spec_version` 在对账通过后才切换。

## 5. 失败观测与处置

| 观测 | 候选原因 | 区分方法 | 返回环节 |
|---|---|---|---|
| 提取字段正确率高，下游无增量 | 字段无经济信息；或表示层丢失时效 | 05 的事件龄/交互消融；对照无该字段的模型 | 05/07，或停用该字段 |
| 通用模型显著优于 PiT 对照 | 前视污染 | Lookahead Propensity 按日期分组；去实体化后差值是否消失 | 04：降权、换模型或只用前向数据评价 |
| 稀有类别（撤回、矛盾）召回低 | 标注样本不足；schema 缺省错误 | 稀有类别单独抽样与审计 | 04 步骤 2、4 |
| 把转述当原始事实 | 来源角色判别错误 | 抽样复核 `source_role`；核对原始 URI | 04 步骤 7 |
| 把条件声明当已生效 | 状态字段错误 | 原文复核；纠正后重放同策略 | 04，重放 08/09 |
| 提取延迟超过策略窗口 | 模型或队列慢 | `extraction_completed_at` 分布；对比快模型与慢模型 | 04：换在线模型；02 更新 `processing_ready_at` |
| 模型在训练窗内校准好、前向变差 | 分布漂移或文本风格变化 | 前向可靠性图按月比较 | 04：重新审计；必要时重训 |
| 同一事件被计为多条新信息 | 事件归属错误 | 核对 `event_id` 与 `relation` | 04 步骤 7 |

## 6. 两速 AI 在本环节的权限

| 环节动作 | 慢速推理 Agent | 快速类型化模型 | 数值/确定性程序 | 裁决者 |
|---|---|---|---|---|
| schema 设计与字段枚举 | 提议 | — | 校验 schema 合法性 | 人审加下游消融 |
| 标注样本初标 | 生成初标 | — | 抽样与分组 | 人核对关键字段 |
| 在线提取 | 不进入 | 主力 | 时间戳、队列、`abstain` 规则 | 校准审计过的版本才可用 |
| 校准审计 | 可解释结果 | 被审计对象 | 计算可靠性图、ECE、Brier | 预注册接受区间 |
| 前视诊断 | 提议诊断设计 | 被诊断对象 | 计算 Lookahead Propensity、掩码对照 | 05 按结果降权 |
| 事件归属与关系 | 离线复核 | 可输出关系候选 | 按标识与时间窗归并 | 抽样复核 |
| 规则变更识别 | 提议 spec 变更 | 输出 `venue_rule_change` | — | 03 的对账通过才切换 `spec_version` |

外部文本是不可信内容：它可以促成提议，不能授权任何变更；提取器输出不改变工具权限或奖励配置。

## 7. 分阶段能力与证据

| 阶段 | 本环节实质内容 | 验证与推进条件 |
|---|---|---|
| 最小 MVP（对应 10 月 15 日改约交付） | A 轨一类官方事件的 schema、标注样本、一个通过校准审计的快速提取器、Lookahead Propensity 与去实体化诊断；B 轨规则变更识别接入 03 漂移分诊；`extractor_version` 全链路可追溯 | 代表性事件人工核查关键字段通过；审计结果在接受区间内；输入不携带未来收益叙事；提取完成时刻进入 02 可用时间 |
| 第二步：针对失败补强 | 若错误集中于否定、条件、实体或转述，补标注或规则；比较微调小模型与 Jev 类模型在同一审计下的表现；有需要时增加独立确认源 | 独立事件簇上的提取改进与下游增量同时成立；提交权重或版本产物才称改进 |
| 第三步：跨状态与运行验证 | 覆盖修正、冲突、不同发布形式、实时延迟；检验提取漂移与前向一致性；前向可靠性图按月发布 | 新事件上的证据支持率、延迟与失败处置达目标；不依赖历史记忆解释未来 |
| 完整能力：可复用研究产品 | 多来源事件、实体、主张关系图，可追溯传播与修订；多 schema 版本并存；提取器作为可替换组件按审计切换 | 每个合并与推断可区分事实与假设；不以知识图规模验收 |

## 8. 依赖与待定

依赖 [02](02-CLEANING-ALIGNMENT-ENVSTATE.md) 提供版本、时点与 `EnvState`；提取完成时刻反向写入 02 的可用时间。[05](05-OPERATORS-FEATURES.md) 定义字段如何变成表示；[07](07-FUSION-SELECTION.md) 决定传播、确认、冲突怎样组合；[10](10-VALIDATION-GATES.md) 评价下游增量；[03](03-ACCOUNTING-CORE-RECONCILIATION.md) 接收规则变更提议。

待定：A 轨首类事件与语义源清单；标注样本来源与数量；校准审计的接受区间数字；候选提取器的许可与部署方式；Jev 类模型的接入成本与数据出境限制；B 轨规则页的抓取频率。模型的流畅解释不是可信度或因果性的替代指标。
