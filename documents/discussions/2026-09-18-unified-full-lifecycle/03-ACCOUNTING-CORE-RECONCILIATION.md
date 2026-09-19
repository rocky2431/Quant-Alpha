# 03 会计核心与三方对账

> **2026-09-18 当前讨论稿｜按 UNIFIED-PLAN v1 细化｜未批准实施。** [索引](README.md) · [总架构](ARCHITECTURE.md) · [MVP](MVP.md) · 上游：[01 录制](01-DATA-RECORDING-INGESTION.md)、[02 清洗与 EnvState](02-CLEANING-ALIGNMENT-ENVSTATE.md) · 下游：[09 仿真与执行](09-SIMULATION-EXECUTION.md)、[10 验证](10-VALIDATION-GATES.md)、[11 反馈](11-RESEARCH-FEEDBACK-AGENT-LOOP.md)

## 1. 目的与边界

把场所机制中 PnL 真正依赖的部分写成可执行代码，每日用链上真值自动校验，并用已知变更注入证明校验能力本身。这是不变量"账本语义先于任何学习"的落点，也是所有者三层抽象中"数据写入规则"的落点。

输入为 `AlignedRecord`（asset contexts、fills、清算与 ADL 事件、账户状态）、01 的规则快照、`EnvState` 的规则字段。输出为按版本管理的 `AccountingSpec`、每日 `ReconciliationReport`、`spec_version` 供 02 写入 `EnvState`、漂移信号供 11。

**写进 spec 的**：funding、margin、liquidation、ADL、约束、HIP-3 discovery bounds v2，全部按 DEX、部署者、抵押品分版本。**不写进 spec 的**：撮合、队列位置、部分成交顺序。理由是 HL 不公开 L3，校验不了的东西不 spec；撮合交给 09 的执行内核加明确标注的假设。

**闸门语义**：对账失败阻断依赖该账本的上线，不否定整个目标。三方一致不能排除共同输入错误，所以必须配独立规则案例集与已知变更注入。

## 2. 对象与契约

### 2.1 `AccountingSpec`

| 字段组 | 内容 | 版本键 |
|---|---|---|
| `market_class` | `perp` 或 `hip3`；决定 premium 公式与费率比例 | 每市场 |
| `funding` | 原生：`F = avg_premium + clamp(0.0001 − premium, −0.0005, +0.0005)`，premium 每 5 秒采样按小时平均、按 1/8 小时支付、封顶 4%/小时、impact notional 分档、结算用 spot oracle 换算名义值；HIP-3：`premium = 0.5*(impact_bid+impact_ask)/oracle − 1`，无 clamp | `spec_version` |
| `margin` | 维持保证金为最大杠杆下初始保证金的一半；分档 `mm_rate(n)` 与 `mm_deduction(n)`；cross 与 isolated 差异；HIP-3 每个 DEX 独立保证金与抵押品 | `dex`、`collateral` |
| `liquidation` | `liq_price = price − side*margin_available/position_size/(1 − l*side)`，`l = 1/MAINT_LEV`；大仓位 20% 分步与 30 秒冷却；权益低于维持保证金 2/3 转 backstop | `spec_version` |
| `adl` | 排序索引 `(mark/entry) × (notional/account_value)`；按上一 mark 成交 | `spec_version` |
| `bounds` | HIP-3 discovery bounds v2：mark 限于参考价 ±(1/最大杠杆)，触发阈值后棘轮重锚，每方向有限次；外部恢复时参考价回到外部价、计数归零；界外爆仓价不清算 | `dex`、`deployer` |
| `constraints` | `max_market_order_ntl` 分档、`max_limit_order_ntl`、OI cap | `spec_version` |
| `fees` | 原生基础档 taker 0.045%、maker 0.015%；HIP-3 部署者比例 0 到 300%，growth mode 减免 ≥90% ✅（官方费用页） | `fee_tier`、`deployer` |
| `known_residuals` | 不作对账项的已知不确定源，如 mark price 精确构成 | — |

字段以官方文档为准；任何数字进 spec 前记录来源 URL、快照哈希与日期。

### 2.2 `ReconciliationReport`

| 字段 | 内容 |
|---|---|
| `window` | 对账区间；必须是有同期账户状态与事件状态的区间 |
| `spec_version` / `engine_version` | 会计核心与执行内核版本 |
| `items` | 每小时 funding rate、每笔 funding payment、`liquidationPx`、清算触发区块、`markPx`/`premium`/`openInterest`、账户权益 |
| `tolerances` | 逐项容差：最小计价单位、≤1 tick、≤1 block |
| `three_way` | spec vs 引擎、引擎 vs 链上、spec vs 链上的差异清单 |
| `injection_results` | 已知变更注入清单与是否检出 |
| `independent_cases` | 独立规则案例集通过情况 |
| `verdict` | 通过、失败、覆盖不足 |
| `blocked_tracks` | 失败时阻断的轨道与候选 |

## 3. 具体过程

1. 从 01 规则快照与官方文档抽取公式与参数，写成 `AccountingSpec`；HIP-3 每个 DEX 一个版本，`market_class` 判别必须存在，否则静默错算。
2. 属性测试：账本恒等式、单调性、边界，用 proptest 类工具。
3. 独立规则案例集：手算若干已知历史清算、资金费结算、ADL 排序样例，作为不依赖三方的真值。
4. 三方对账：会计核心从 `AlignedRecord` 独立算账，与执行内核账户状态、链上真值逐字段比对。三种不一致分别指向 spec 错或场所改参数、适配器或撮合建模不符、口径差异。
5. 已知变更注入：人为改 impact notional、维持保证金率、clamp 边界、bounds 参数，对账必须报警；注入清单与结果写进 `ReconciliationReport`。检测能力由注入证明，不依赖场所恰好改参数。
6. 每日 CI 运行；失败非零退出，写 `blocked_tracks`。共享账本错误阻断两轨；B 轨特有 spec 错误只阻断 B。
7. 对账区间限定在有同期账户与事件状态的覆盖内；20 个历史清算事件的触发价与区块对齐验证的是清算逻辑，不是全账本。
8. 检出真实差异后交 11 做漂移分诊；补丁重跑对账通过才可合入；期间依赖该字段的策略降级为只减仓。
9. `spec_version` 变化写回 02 的 `EnvState`，历史观察不改写。
10. 差异样例保留为回归测试；三个月无红灯时评估覆盖是否太浅。

## 4. 双轨示例

**轨道 A（假设数值）。** 某小时 BTC 原生 premium 平均 +0.00003，`clamp(0.0001 − 0.00003, ±0.0005) = +0.00007`，`F = 0.0001`，即处于利率地板。会计核心算出某账户该小时 funding payment 为 −0.0125 USDC；执行内核账户状态显示 −0.0125；链上 `cumFunding` 增量 −0.0125。三方一致。注入测试把 clamp 上界改为 +0.0004，会计核心输出 `F = 0.00043`，与链上不一致，CI 报警，注入被检出。

**轨道 B（假设数值）。** 某商品 `xyz:` 市场休市中，impact bid 100.70、impact ask 100.72、oracle 100.00，HIP-3 premium 为 `0.5*(100.70+100.72)/100.00 − 1 = +0.0071`，无 clamp。bounds 上界为参考价 ×(1+1/20) = 105.00，mark 100.80 在界内，`reanchor_count = 0`。若 spec 误把该市场按 `perp` 处理，会得到 clamp 后的 funding 与链上 `fundingHistory` 不一致，三方对账中 spec vs 链上失败、引擎 vs 链上通过，定位为 spec 的 `market_class` 判别错误。

## 5. 失败观测与处置

| 观测 | 候选原因 | 区分方法 | 返回环节 |
|---|---|---|---|
| spec ≠ 链上，引擎 = 链上 | spec 错或场所改参数 | 查 01 规则快照哈希是否变化；查 `market_class` | 11 漂移分诊；03 补丁 |
| 引擎 ≠ 链上，spec = 链上 | 适配器或撮合建模不符 | 对照原始 venue 报告 | 09 修适配器 |
| spec ≠ 引擎，两者 = 链上 | 口径差异：舍入、时点 | 逐项容差检查 | 03 对齐口径 |
| 三方一致但独立案例失败 | 共同输入错误，如 asset contexts 解析偏移 | 独立案例集 | 02 修解析；03 重跑 |
| 注入未被检出 | 对账项未覆盖该字段、容差过宽 | 注入清单逐项核对 | 03 扩对账项 |
| 清算触发区块差 >1 | 事件时间对齐或部分清算规则 | 对照 `misc_events` 与 fills 的 `liquidation` 标记 | 03；02 时点 |
| 三个月从未红 | 覆盖太浅或场所未变 | 注入频率与覆盖审计 | 重新评估对账价值 |

## 6. 两速 AI 在本环节的权限

| 环节动作 | 慢速推理 Agent | 快速类型化模型 | 数值/确定性程序 | 裁决者 |
|---|---|---|---|---|
| 读文档起草 spec 代码 | 提议草稿，写分支不能合并 | 可对公告做"参数变更/机制变更/无关"分类初筛 | 属性测试、独立案例 | 三方对账 CI 加注入全检出 |
| 漂移分诊与补丁 | 提议定位字段与补丁 | — | 重跑对账 | 对账通过才合入 |
| 选择容差与对账项 | 提议 | — | — | 预注册，人工确认，不由 Agent 改 |
| 注入清单设计 | 提议注入项 | — | 执行注入 | 全部检出为通过条件 |
| 账户或密钥 | 不可触及 | 不可触及 | 研究区无密钥；对账读链上公开状态 | 权限分区 |

## 7. 分阶段能力与证据

| 阶段 | 本环节实质内容 | 验证与推进条件 |
|---|---|---|
| 最小 MVP（对应 10 月 15 日改约交付） | 两套 funding、margin 分档、liquidation、ADL、bounds v2 按 DEX 分版本；属性测试；独立案例集；三方对账在覆盖区间运行；已知变更注入；每日 CI | 注入全部检出；覆盖区间内逐笔对齐；20 个清算事件触发价与区块对齐；失败写 `blocked_tracks` |
| 第二步：针对失败补强 | 针对真实差异修 spec 或适配器；扩对账项到新字段；补独立案例 | 补丁重跑对账通过；回归测试收录差异样例 |
| 第三步：跨状态与运行验证 | 高波动期、部署者改参数、新市场上线、抵押品变化下的对账；对账与实盘回执的比对 | 真实漂移被当日检出并归因；实盘与影子偏差可归因到 spec 字段 |
| 完整能力：可复用研究产品 | 多场所多 DEX 的 `AccountingSpec` 库；对账作为新场所接入的准入条件 | 每种工具独立核实规则与适配；新增场所先过对账再进研究 |

## 8. 依赖与待定

01 提供 asset contexts、fills、清算与 ADL 事件、规则快照；02 提供对齐与 `EnvState` 规则字段；09 的执行内核提供账户状态与原始 venue 报告；10 把 G0 定义为本环节的通过；11 消费漂移信号。待定项为对账项清单与容差数字、注入清单、HIP-3 目标 DEX 的具体参数版本、mark price 构成的残差处理、S3 对 `xyz:` 覆盖决定的可对账区间。这些不阻止 spec 起草，但对账通过的声明必须限定在实际覆盖区间内。
