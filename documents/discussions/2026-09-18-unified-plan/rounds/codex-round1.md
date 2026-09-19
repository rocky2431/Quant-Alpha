# 统一方案：Codex 第一轮对抗性评议

日期：2026-09-18。本文是研究判断及方案建议，不是实施批准，也不宣称任何候选已经通过实盘验证。

核验标记：✅ 本轮直接阅读原始材料的相关正文；🟡 二手报道、摘要或有限检索片段；❌ 本轮未找到所需证据。机构官网的 ✅ 只证明机构作过该表述，不等于业绩经过独立审计。无发布日期的网页注明本轮核验日。下文“建议”“推论”均是我的判断，不冒充来源结论。

**结论：以完整生命周期承载机制模型，首选 BTC 单场所实例，ETH 留作迁移检验；Polymarket 逻辑组合第二，HIP-3 会话第三。** 排序依据是所有者目标、闭环覆盖和首期执行复杂度，不是宣称 BTC 的 alpha 已获证明。保留自主研究、真实 SFT/RL、自动交易与反馈；将评估、风险和资金权限放在研究 Agent 不能自行改写的边界之外。

## A. 统一抽象：候选决策系统及其生产过程

机制派的 `p/m` 分解可以放进生命周期：`p` 描述未知的市场演化，`m` 根据**实际发生的事件和当时规则**更新账户。但成交、滑点和对手反应并不因会计可计算而变成确定性。HRT 明确指出交易缺少棋类的完全状态、确定转移及无限真实自博弈数据。✅ [HRT，2021-09-15](https://www.hudsonrivertrading.com/hrtbeat/ai-trading/)。因此，“机制正确”与“预测有效”分别验证，再由同一条交易链路检验其组合。

先纠正历史约束：基金工作稿明确写明所有者已扩展方法范围，旧文档“禁止 LLM 决策、RL 只能调参数”等不再自动构成新约束。✅ [THOUGHTS，2026-09-17 更新，第 9、39 行](../../../../thinking/onchain-rl-trading/fund-delivery-2026-10/THOUGHTS.md)。这不是必须由 D1 实验裁定的偏好冲突；实验裁定技术效果，不能撤销所有者已明确的目标。

| 层 | 统一内容与制约 | 历史教训／创新的落点 |
|---|---|---|
| 数据与环境 | BTC/ETH 链上转账、交易所地址流量及标签版本；目标场所逐笔、L2、订单状态、持仓、资金费、标记价、清算；跨场所参考价格；宏观公告原文与发布版本。每条记录区分事件时间、公开时间、系统实际可用时间，另存来源、修订、序号及缺口。 | Two Sigma 将原始数据与整理后数据并行提供，并把变换视为代码管理；我们也允许研究先开始，但不能把后来修订的资料当成当时可用数据。✅ [Two Sigma，2025-10-23](https://www.twosigma.com/articles/platform-thinking-three-views-from-two-sigma-leaders/) |
| 特征／表征与验证 | 环境至少由资产、场所规则、可观测市场状态、预测期限、信息集、费用与成交假设定义。特征须声明依赖、时间窗口、缺失行为和目标。检验其在相同信息与风险预算下，相对价格基线的**增量价值**；区分收益预测、波动／风险预测、成交预测。组合还要检验冗余、相互作用及共同失效。 | AlphaGen 按组合增益寻找因子，而非逐个追高 IC；AQuA 的时间泄漏事故表明必须检查实际读取的数据范围。相关原文及深度见 F。 |
| 执行与反馈 | 信号或动作经过固定风险约束形成订单；保存成交、拒单、撤单、费用、资金费、现金流与账户变化。将预测失败、执行失败、会计失败分开归因，反馈到下一候选。 | Two Sigma 未授权改参数事件要求部署参数也受版本约束；FTX 与 Wintermute 说明收益模型不能代替资产隔离和密钥控制，见 E。 |

“环境”不能事后用牛熊标签切出漂亮子样本；只能用决策时可观测量定义状态，或把事后分组明确作为诊断。链上地址标签若后来才补全，其历史回填不能直接进入严格回测。**链上业务不等于只允许链上特征**：订单流文献中的中心化交易所数据必须如实标注，不能改称链上证据。

两种进化分别管理：

- **Agent 研究反馈**：更新假设、研究优先级、开发代码、提示、候选组合与实验记忆；受固定工具、开发数据和试验预算约束。不得读取封存标签，也不得自行修改验收器、扩大资金权限或发布自己。
- **算法反馈**：更新模型权重、校准器、组合权重或 RL 策略。数据截止、奖励、训练规则、更新频率与晋级规则一起版本化。仅改提示／记忆不算参数 RL；数值 PPO 也不等于完成 LLM Agent RL。

**建议冻结的是完整候选**：数据清单、特征／提取器、模型／提示、训练及更新规则、选择器、风险与执行策略、成本模型、评价程序。最终留出数据不进入任何一层的搜索。研究方向选择器自身也在被评估的系统内，而不是可以免费反复看答案的裁判。固定验收程序可以在所有者预授权边界内自动晋级；禁止的是 Agent 自改标准或豁免失败，不是要求人手工批准每轮实验。

## B. 首个实例与链上预测证据

| 排序 | 为什么选／暂缓 | 仍须解决的关键问题 |
|---|---|---|
| **1．BTC 单场所普通永续；ETH 后续迁移** | 最接近所有者明确方向，可用同一资产贯穿发现、训练、风控、成交与反馈；单腿便于归因。Hyperliquid 是待资格验证的执行候选，不把它与 HIP-3 混为一谈。 | 链上信号未证明能覆盖该场所当前成本；ETH 与 BTC 也不是天然独立样本。纯现货／不同场所要按实际账户条件调整。 |
| **2．Polymarket 逻辑组合** | 支付结构可形式化，适合验证语义提取、规则一致性及组合执行；确有历史套利证据。 | 条件需对应同一事件定义、结算时点及预言机规则；多腿成交、深度、争议和资金占用会吞掉账面优势。 |
| **3．HIP-3 会话结构** | 可研究闭市、重开和预言机状态变化，机制清楚时有价值。 | HIP-3 的预言机、合约与运行由部署者定义；协议允许部署者结算，不代表每次重开都会强制按某价支付。首期又增加标的、日历、预言机与抵押品规则维度。✅ [官方 HIP-3，核验 2026-09-18](https://hyperliquid.gitbook.io/hyperliquid-docs/hyperliquid-improvement-proposals-hips/hip-3-builder-deployed-perpetuals) |

对“2020 年后有没有可信样本外价值”的回答是：**有局部正证据；不足以证明我们当前的 BTC 链上策略有净 alpha，更不足以证明没有价值。**

| 数据／作者 | 实际找到的证据 | 能迁移什么，不能推到哪里 |
|---|---|---|
| 链上流量：Chi、Chu、Hao | ✅ [Return and Volatility Forecasting…，2025-09-01 修订](https://arxiv.org/pdf/2411.06327)，样本 2017–2023：ETH 净流入与后续收益负相关，USDT 流入对 BTC/ETH 短期限收益有信息；BTC 自身净流入大多无收益预测力，4 小时例外。 | 正文自称做了样本外检验，但本轮未核清足以复现的滚动划分、历史地址标签可用时间及策略选择流程。不能将预测回归／挑选的期权场景升级成严格、当前可交易的 BTC 净收益证据。 |
| 订单流：Anastasopoulos 等 | 🟡 [Order flow and cryptocurrency returns，Journal of Financial Markets，2026-06](https://doi.org/10.1016/j.finmar.2026.101047)：出版方摘要报告国际订单流在跨币样本外预测中优于经济基本面，非线性 ML 有经济价值。 | 这是目前较强的正线索，但使用 CryptoCompare 交易所买卖量，非纯链上指标；本轮未读全篇验证成本与划分，不能直接外推单 BTC／Hyperliquid。 |
| 永续资金费：Emre Inan | 🟡 [Predictability of Funding Rates，2025-10-07，摘要](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=5576424)：Binance／Bybit BTC 资金费的 DAR 模型，在样本外预测误差和方向准确率上优于不变预测，预测性随时间变化。 | 支持将资金费作为可预测现金流单独建模；预测资金费不等于预测 BTC 收益，也未证明扣除建仓与对冲成本后的套利利润。 |
| 基差／资金压力：Schmeling、Schrimpf、Todorov | 🟡 [BIS Crypto carry，2023-04-04；正文后有 2025 修订](https://www.bis.org/publications/working-paper-1087-crypto-carry)：作者概要报告高 carry 预示价格崩跌，并强调套利中的保证金和清算风险。 | 支持风险状态及资金占用建模；期货基差不等于永续资金费，也不是已验证的短期方向交易信号。 |
| 清算、OI 与多源组合 | ✅ [AQuA，2026-08-17，附录 A](https://arxiv.org/html/2608.12841v2) 给出 OI 冲击、成交方向和基差组合的研究轨迹；部分表达式未披露。 | 可借研究设计；❌ 本轮未找到可据以确认“单独清算指标在目标场所、严格样本外、成本后稳定获利”的证据。 |

**Polymarket 的强正证据也必须正面承认。** ✅ [Saguillo 等，2025-08-05](https://arxiv.org/html/2508.03474) 估计历史各类套利提取利润合计约 3,959 万美元；这个总数不能全算成跨事件逻辑组合套利。研究识别的依赖事件组合范围有限，不能据此预测我们每天的机会量。

反面证据同样具体：✅ [Cheng 等，2026-04-22](https://arxiv.org/html/2605.00864) 在 173 场 NBA、约 7,509 万订单簿快照中找到 7 次单市场机会、290 次组合机会；组合中 76.9% 所属浅深度类别平均可执行量仅 14.8 股。这是快照重建，非作者实盘成交；采样还可能高估机会持续时间。**它不否定小资金参与，却足以否定“事件多，所以容易取得大量可兑现独立样本”。**

FOMC 问题必须单列：✅ [美联储日历，核验 2026-09-18](https://www.federalreserve.gov/monetarypolicy/fomccalendars.htm) 列明 9/15–16 后下一次例会为 10/27–28。因此 10/15 前不能验证新的例会决议效果。我建议 BTC 研究保留宏观语义，同时评估订单流、资金与市场状态；这是对 FOMC 专用 MVP 的明确调整建议。若维持专用范围，就必须承认期限内该机制前向验证缺失，不能用普通 BTC 成交替代。

## C. 两速 AI：按权限与可验证任务分工

| 主体 | 可以做什么 | 不授予什么 |
|---|---|---|
| 慢推理 Agent | 发现假设、调用开发实验、解释失败、提议下一版本；在限定预算内自主循环。 | 修改封存评价、吞入测试结果后仍称未见、放宽账户风险、接触提款密钥、绕过发布流程。 |
| 快模型／自训策略 | 文本提取、事件判别、路由、校准；也可以成为受约束的动作选择器，与固定规则／数值模型比较。 | 仅凭输出合法就获得订单或资金权限；自行修改自己的风险边界。 |
| 确定性执行层 | 检查数据新鲜度、仓位、名义敞口、价格／数量边界、账户与场所状态，执行获准动作并对账。 | 让模型文本覆盖检查结果，或把 LLM 风险评论当作硬限制。 |

最小示意（非 Jev API 原始响应）：模型输出 `{"action_probabilities":{"hold":0.8,"long":0.1,"short":0.1}}`，表示该任务中候选答案的模型概率分配；**不是交易盈利概率 80%，也不是仓位应为 80%。** ✅ [TypeSafe 原文，2026-09-15](https://typesafe.ai/blog/introducing-system-one-models-and-jev) 将 Jev 定位为类型化概率决策，类型边界不证明金融校准和收益。本轮 ❌ 未找到 Jev 的独立交易验证。故先给它提取／分流任务是合理工程顺序，永久禁止进入交易决策则证据不足。

正面研究中，FinCon 的语言反馈、TradingAgents 的分工、Trading-R1／FLAG-Trader 的参数训练代表不同层次，不能统称“RL”；负面研究也不能一概推成 LLM 无用。Glasserman–Lin 发现匿名化新闻有时更好，显示既有知识可能造成干扰；Profit Mirage 的跨训练截止期表现落差提示泄漏风险，但不同市场时期也混杂其中；FINSABER 显示短样本赢家扩展评价后优势衰退。来源与核验边界见 F。**建议评价模型实际增加的信息及决策价值，不按“LLM／非 LLM”身份预先判输赢。**

## D. 工程、并行与 Full Picture／MVP

**建议继续以 NautilusTrader 为底座候选，先验证版本和接缝；采用模块化胶水工程，按权限隔离进程，不先造微服务平台。**

| 当前原始证据 | 对实施选择的含义 |
|---|---|
| ✅ [发布页，2026-09-15](https://github.com/nautechsystems/nautilus_trader/releases)：最新是 **2.0.0rc5，Pre-release**；最新非 prerelease 条目是 **1.231.0 Beta，2026-08-02**，并被列为计划中的最后一个 1.x 版本。 | “最新”不等于稳定版，1.x 也不是长期维护承诺。锁版本、提交与文档；不能用 2.x 的 `latest` 文档证明 1.x 能力。选型验证先于生产承诺。 |
| ✅ [Hyperliquid 适配文档，核验 2026-09-18](https://nautilustrader.io/docs/latest/integrations/hyperliquid/)：已有行情和执行接入，覆盖普通现货／永续及 HIP-3 发现；agent wallet 的账户订阅需正确配置主账户地址。 | “有适配器”成立；账户流、重连、漏单恢复和我们账户的可用性仍须运行验证。本轮未安装或运行。 |
| ✅ [Polymarket 适配文档，核验 2026-09-18](https://nautilustrader.io/docs/latest/integrations/polymarket/)：不支持 `reduce_only`，批量订单按最多 15 笔分块，不能当组合原子成交；小余额有 dust 限制。rc5 发布说明继续增加钱包及拆分／合并／赎回等能力。 | 多腿剩余敞口、残值及结算必须进入适配验收，不能由通用回测器自动担保。 |

最小工程边界：研究／训练任务与实时执行进程隔离；执行进程拥有有限交易凭证，提款／管理密钥不进入 Agent；前端经现有 API 读取候选、风险、订单、余额和研究状态。复用现有数据库、Parquet 与任务调度，不为统一外观重写交易引擎或 RL 框架。隔离依据是资金权限、故障与负载，出现独立扩缩容需要后再拆服务。

保留可重放的原始数据清单、决策输入及版本、订单意图、场所回执、成交和账户快照；“事件溯源”首期就是这些耐久记录与恢复顺序，不等于先部署 Kafka。会计规范、引擎与场所记录须在相同时间边界对齐；三方使用同一错误输入时，一致也不能证明正确。

机构借鉴应落到能力而非规模：Jane Street 描述 Python／notebook、标准数据格式和共享模型库，同时区别研究便利与生产可靠性；HRT 强调模拟的局限；Man AHL 将 Agent 接入本公司的数据、成本与模拟库；Two Sigma 允许原始数据研究与整理工作并行。✅ [Jane Street，2025-03-10](https://signalsandthreads.com/finding-signal-in-the-noise/)、[HRT，2021-09-15](https://www.hudsonrivertrading.com/hrtbeat/ai-trading/)、[Man AHL，核验 2026-09-18](https://www.man.com/insights/ai-agents-trend)、[Two Sigma，2025-10-23](https://www.twosigma.com/articles/platform-thinking-three-views-from-two-sigma-leaders/)。这些没有证明我们的体量需要复制机构微服务拓扑。

**并行方式**：生产候选 P 运行，候选 C 做封存评价，候选 B 训练，Agent 为 A 找方向，数据与 UI 工作同时推进。跨候选流水线正确，但同一候选不必永远冻结权重：预先固定更新算法 `U`，按“先预测、标签成熟后学习”运行，评价对象仍可是一条固定的自适应策略。研究者看过评价后修改 `U`，才产生新的候选；新版本不能继承旧版本的全部实盘天数。

多个预先登记候选可以共用同一前向窗口比较，但选择结果须计入多重试验；市场多也不自动独立。候选全清单、失败尝试、相关性、重叠标签以及搜索预算都要保留。这个限制来自选择偏差，不是“同一未来只能被一个模型读取”的物理定律。✅ [DSR，2014-07-31](https://www.davidhbailey.com/dhbpapers/deflated-sharpe.pdf)、[Spurious Predictability，2026-04-16，§3](https://arxiv.org/html/2604.15531)。

| 交付层次 | 必须跑通 | 不应冒充完成的东西 |
|---|---|---|
| **Full Picture** | 三层数据与环境 → 假设／表征 → 训练 → 独立评价 → 晋级 → 风险配置 → 自动执行 → 账户与经济归因 → 漂移与下一轮研究；多候选并行；统一 UI 呈现证据及运行状态。 | 只做因子 IC、模型 API 或策略聊天界面。 |
| **MVP** | 一资产、一场所、一类明确期限任务，保留上述整圈；价格基线、增加信息后的监督模型、数值 RL、SFT-only、同底模 SFT+RL 做分阶段对照；先让较少候选进入昂贵前向评价。 | 用数值 PPO 替代已要求的 LLM 参数 RL；用纸面收益替代实际订单；用没有机会的监控天数充当目标策略实盘证据。 |

具体切片建议：BTC 普通永续，以一小时预测／持有期限起步，输入当前可用的价格、流量、资金与语义状态，输出受限的空仓／多／空目标；按真实时间推进账户，反馈成本后净值、风险和成交结果。SFT 学习可核验的状态处理与工具轨迹，RL 在同一环境更新策略参数；监督模型加固定交易规则作为强对照。期限和假设在开发集确定后冻结，不能在测试结果出来后挑最好的一档。这是待验证设计，不是已有交易优势。

先行验证至少包含三类：诊断支路中的已知信号／故意泄漏阳性对照；正式特征入口拒绝未来读取及零预测环境的假发现率检查；部分成交、拒单、重启、资金费、陈旧价格与账户对账的可回放检查。三者分别覆盖学习能力、证据可信度和执行正确性，不能互相替代。

日期不能由架构讨论消除：✅ [基金 THOUGHTS，2026-09-17](../../../../thinking/onchain-rl-trading/fund-delivery-2026-10/THOUGHTS.md) 以 10/13 00:00 封存为假设，20／15 个完整实盘日对应 9/23／9/28 启动；距 9/18 仅 5／10 日。此处是日期算术，不是工期承诺。完整 UI／后端、真实训练、实盘时长都应独立报告达成情况；旧“四周影子期”与该日程的冲突要显式解决。本轮没有运行检查，不能把“9/13 未安装”的历史记录当成今天状态。

同意尽快保存目标场所数据；跨场所录制只有在不挤占首实例、且能保存快照／增量一致性、时间与缺口时才有价值。本轮授权只读，因此这里只提出建议，未启动采集。

## E. 机构路径与踩坑：事实、限制、对我们的含义

| 机构／事件 | 证据与日期 | 可采纳的教训，不作因果过推 |
|---|---|---|
| **Point72／Cubist／AI 基金** | ✅ [Cubist 官网，核验 2026-09-18](https://point72.com/cubist/) 描述系统化、多来源数据研究。🟡 [Reuters，2025-01-16](https://www.investing.com/news/stock-market-news/point72s-new-ai-fund-near-15-billion-after-doubledigit-returns-sources-say-3816245)：Turion 于 2024-10 开始交易，首季约 14.2%，关注 AI 产业链赢家和输家。 | **AI 主题投资基金不是 AI 自主操作基金。** 不能用 Turion 收益替 LLM 决策背书；借鉴数据与研究平台，非产品名称。 |
| **Jane Street 增长与 ML** | 🟡 [Reuters，2025-07-04](https://www.marketscreener.com/news/latest/What-is-Jane-Street-the-US-trading-firm-facing-heat-in-India-50430080/) 报道 2024 年收入 205 亿美元及跨国扩张。✅ [官方访谈，2025-03-10](https://signalsandthreads.com/finding-signal-in-the-noise/) 描述从简单回归走向树与深度网络，同时强调小数据、高噪声和状态变化。 | 支持认真研究复杂模型；没有公开的收益归因实验能把公司增长全部归因于 ML，更不能直接外推 LLM。 |
| **Jane Street／SEBI** | ✅ [SEBI 临时令，2025-07-03](https://www.sebi.gov.in/enforcement/orders/jul-2025/interim-order-in-the-matter-of-index-manipulation-by-jane-street-group_95040.html)；✅ [NSE 通知，2025-07-21](https://nsearchives.nseindia.com/content/circulars/INVG69234.pdf)：缴存约 484.36 亿卢比 escrow 后，指定限制停止适用，监控及有关行为限制保留。 | 是被指控的指数操纵及临时措施，不应写成最终定罪或永久禁入。❌ 本轮未找到终局裁定。我们的奖励和订单限制须排除不允许的交易行为，盈利不证明机制可接受。 |
| **Renaissance／Simons 之后** | 🟡 [Institutional Investor，2021-04-19](https://www.institutionalinvestor.com/article/2bswp26sjjpvgp7r5upz4/portfolio/the-medallion-fund-is-still-outperforming-other-renaissance-funds-still-arent)：报道 2020 年 Medallion +76%，外部 RIEF 约 −20%。✅ [Simons Foundation，事件 2024-05-10／核验 2026-09-18](https://www.simonsfoundation.org/about/our-history/) 确认 Simons 逝世。 | 同一机构产品也可严重分化，不能用招牌业绩替代策略、容量与账户条件。2020 分化早于其逝世；❌ 未找到可审计、同口径的 2024–2026 策略级证据来归因“Simons 之后”的变化。 |
| **Two Sigma 未授权模型参数** | ✅ [SEC，2025-01-16](https://www.sec.gov/newsroom/press-releases/2025-15)：已知权限弱点长期未修复；未授权参数修改造成损失，机构自愿补偿逾 1.65 亿美元，和解罚款 9,000 万美元。 | 模型代码、组合参数和生产配置都要锁定并核验实际部署内容；拥有研究权限不应等于拥有上线参数写权限。此案和解不等同承认全部指控。 |
| **Man Group／AHL** | ✅ [Alpha Assistant 原文，核验 2026-09-18，页面未明确发布日期](https://www.man.com/insights/ai-agents-trend)：通用 LLM 忽略内部数据、成本与模拟约定；专用 Agent 接入内部工具，主要承接实现环节，研究者定义目标并评价。 | 这是可操作的接入经验，不是公开验证的全自动盈利系统。我们借共享工具和一致评价，保留自主研究作为待测能力。 |
| **Bridgewater AIA** | 🟡 [Bloomberg 转载，2024-07-08](https://www.wealthmanagement.com/artificial-intelligence/bridgewater-co-cio-sees-ai-adding-incredible-strength-for-investors)：报道新设约 20 亿美元、主要依赖 ML 决策的基金。✅ [AIA Labs 官网，核验 2026-09-18](https://www.bridgewater.com/aia-labs) 自述系统已管理数十亿美元并持续学习。 | 是“AI 不应永远只做文字助手”的正证据；❌ 未找到足以独立复现其自主研究、资金治理及策略净业绩的公开材料，不能拿募资规模当胜率。 |
| **WorldQuant BRAIN** | ✅ [官方发布，2022-08-02](https://www.worldquant.com/ideas/worldquant-launches-brain-platform-and-inaugural-global-alphathon-competition/)：通过统一平台模拟、提交 alpha，并发展研究顾问。 | 可借统一输入、模拟与筛选入口；大量投稿不等于大量独立信号。❌ 未找到公开、可审计的投稿到实盘净收益转化率。 |
| **Alameda／FTX；Three Arrows** | ✅ [美国 DOJ，2024-03-28](https://www.justice.gov/archives/opa/pr/samuel-bankman-fried-sentenced-25-years-his-orchestration-multiple-fraudulent-schemes)：客户资金被挪用等欺诈导致 SBF 获刑。✅ [3AC 法院文件，2022-07-15](https://3acliquidation.com/wp-content/uploads/2022/08/2022.07.15-3AC-SUM-2542-of-2022-Order-of-Court-ORC-3661.pdf) 确认 6/27 获委任清盘人及资产处分限制。 | 两案原因不能混写成“量化失灵”。我们的启示是场所／对手方敞口上限、抵押品压力测试、资产与研究账户隔离、外部记录对账；模型分数不能替代偿付与提款能力。 |
| **Terra／Jump** | ✅ [SEC Tai Mo Shan 案，2024-12-20](https://www.sec.gov/newsroom/press-releases/2024-212)：Jump 子公司在 2021 年 UST 脱锚时买入支持价格，相关安排使“算法自行恢复锚定”的宣传误导投资者；和解约 1.23 亿美元。 | 不把有人救市的历史当作稳定机制被证明；模拟抵押品脱锚、流动性枯竭与支持者退出。不能把 Jump 写成 2022 年已经倒闭的机构。 |
| **Wintermute 私钥事件** | 🟡 [Blockworks，2022-09-20](https://blockworks.com/news/wintermute-whacked-by-160m-hack-exploiting-known-vulnerability) 报道约 1.6 亿美元被盗，并称企业仍有偿付能力。✅ [1inch，2022-09-15](https://1inch.com/blog/post/a-vulnerability-disclosed-in-profanity-an-ethereum-vanity-address-tool) 披露 Profanity 弱随机种子导致密钥可恢复。 | 区分事件归因报道与已复现的工具漏洞。用安全随机生成密钥、有限交易钱包、隔离签名；暴露时更换管理权限及授权，不只转走余额。不是增加一个“风险 Agent”就能解决。 |

## F. 学术创新：迁移方法，不照搬收益榜

| 类别／研究与核验深度 | 真正有用的理念 | 证据边界与我们的选择 |
|---|---|---|
| 自动 alpha：🟡 [AlphaGen，2023-05-25](https://arxiv.org/abs/2306.12964) | RL 搜索公式，以加入组合后的贡献作为奖励。 | 股票模拟正结果；优先迁移“增量贡献”目标，不能视为 BTC 实盘证据。 |
| 研究闭环：✅ [R&D-Agent-Quant v2，2025-09-25，§2–3](https://arxiv.org/html/2505.15155v2) | 因子与模型交替优化，假设、实现、评价、反馈组成可执行研究循环。 | 主实验 CSI300，测试 2017–2020；历史切分不能单独排除后来训练的 LLM 先验。迁移接口与实验记录，不接收论文倍数收益作为交付标准。 |
| 受约束自改进：✅ [AQuA v2，2026-08-17，§3／附录 B](https://arxiv.org/html/2608.12841v2) | 固定评价和因果算子，允许候选表达式／配置变化；研究记忆改善下一轮搜索。 | 其早期系统以当日全日成交量归一化盘中量，审阅 Agent 未发现前视。后改为受限算子。两个研究系统分别运行，不等于统一 LLM 权重自我训练；迁移结构约束。 |
| 新颖性与双循环：🟡 [AlphaAgent，2025-06-09](https://arxiv.org/abs/2502.16789)；🟡 [QuantAgent，2024-02-06](https://arxiv.org/abs/2402.03755) | 前者用 AST 相似度、复杂度和假设一致性约束因子；后者区分知识内循环与外部测试反馈。 | 表达式新颖不保证经济暴露新颖；外部测试若反复回流，也可能优化研究过程到同一历史。此处 QuantAgent 指 Wang 等 2024 论文。 |
| 选择偏差基础：✅ [DSR，2014-07-31](https://www.davidhbailey.com/dhbpapers/deflated-sharpe.pdf)；🟡 [PBO，2015-02-27，摘要](https://www.davidhbailey.com/dhbpapers/backtest-prob.pdf) | 调整多重搜索、非正态带来的 Sharpe 膨胀，衡量样本内选择的样本外退化。 | 不是 2023 年后新发明；不是对时间泄漏、错误成交模型或缺失试验清单的补救魔法。使用前写清试验族和依赖。 |
| 研究 protocol：✅ [Arnott–Harvey–Markowitz，2018-10-29 稿／2019 发表](https://people.duke.edu/~charvey/Research/Published_Papers/SSRN-id3275654.pdf) | 先规定研究与验证过程，重视有限金融样本和反复搜索。 | 若所指是 A Backtesting Protocol in the Era of Machine Learning，其作者不是 López de Prado。我们预先登记假设、成本和晋级规则，而非事后凑指标。 |
| 整个流程的反证：✅ [Spurious Predictability，2026-04-16，§3](https://arxiv.org/html/2604.15531)；🟡 [What survives honest evaluation?，2026-08-27，摘要](https://arxiv.org/abs/2608.27734) | 把完整搜索过程放入零预测环境；后者特别指出泄漏策略仍可能通过 DSR/PBO。 | 两者都是近期研究，不能当普适定理。我们测零预测环境下的实际假发现率，并检验已知信号恢复能力；不是要求每一次随机回测都恰好零收益。 |
| 复杂度之争：🟡 [Kelly–Malamud–Zhou，JF 2024-02，摘要](https://economics.yale.edu/sites/default/files/2024-01/The%20Journal%20of%20Finance%20-%202023%20-%20KELLY%20-%20The%20Virtue%20of%20Complexity%20in%20Return%20Prediction%20%281%29.pdf)；✅ [Nagel，2025-07-30](https://bpb-us-w2.wpmucdn.com/voices.uchicago.edu/dist/f/575/files/2025/07/Complexity_2.pdf)；🟡 [Kelly–Malamud 回应，2025-07-10，概要](https://www.aqr.com/Insights/Research/Working-Paper/Understanding-The-Virtue-of-Complexity?aqrPDF=1) | KMZ 提出过参数化可改善预测；Nagel 指出特定短窗口 RFF 可机械地变成波动率择时动量；作者强调有效复杂度、隐式正则和集成。 | 争论不支持“模型越大越好”或“大模型一定无效”。用相同数据和预算比较简单动量／风险缩放与复杂模型，并做破坏预测结构的对照。 |
| LLM 前视：✅ [Glasserman–Lin，2023-09-29](https://arxiv.org/html/2309.17322)；🟡 [Sarkar–Vafa，2024-04-11 发布／2024-10-18 修订，摘要](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4754678) | 区分参数记忆、新闻可用时间和语义信号；前者还发现公司知识的干扰效应。 | 写“假设今天是过去”不构成隔离证明；匿名化可作诊断，但不是完整去污染证书。优先封存模型版本后的真实前向评价。 |
| 交易 Agent 正面结果：🟡 [FinCon，2024-11-07](https://arxiv.org/abs/2407.06567)；🟡 [TradingAgents，2025-06-03](https://arxiv.org/abs/2412.20138) | 语言信念反馈、分工和信息合成，论文报告相对基线改善。 | 本轮只读摘要，不能独立确认其净收益与泄漏控制；FinCon 的 verbal reinforcement 不能冒充权重 RL。先与单 Agent、简单规则做消融。 |
| 真实参数训练方向：🟡 [Trading-R1，2025-09-14](https://arxiv.org/abs/2509.11420)；🟡 [FLAG-Trader，ACL 2025-07](https://aclanthology.org/2025.findings-acl.716/) | SFT／课程 RL；将部分微调 LLM 作为策略网络，用交易奖励做梯度优化。 | Trading-R1 摘要所述训练覆盖 18 个月、14 股票，评价 6 股票／ETF；金融序列与市场状态仍有限。小资金能借训练方法，但不会因此增加独立样本。 |
| 反面评估：✅ [Profit Mirage，2025-10-09，§2.1](https://arxiv.org/html/2510.07920v1)；✅ [FINSABER v4，2025-11-24，§4–6／附录](https://arxiv.org/html/2505.07078v4) | 前者报告跨训练截止期 Sharpe 明显衰退；后者扩到长时段、多股票及退市股票，显示先前优势不稳。 | 前者有时期混杂，后者也没有重跑所有闭源框架。可据此提高评价要求，不能推出所有 LLM 交易必败。 |
| 预测市场与加密 | B 所列 ✅ Polymarket 历史套利／2026 NBA 快照研究；链上回归及 🟡 2026 订单流研究。 | 一类提供支付约束，一类提供统计预测；前者仍需执行验证，后者仍需成本及独立评价。二者共同构成统一系统的候选，不能互相替代业务目标。 |

## G. 对 Claude 最强的反对

1. **“切换只凭偏好，D1 决定该信哪套”不成立。** 所有者已明确扩大 Agent／SFT／RL 范围，最新基金文档也撤销了旧禁止项（✅ 2026-09-17，A 中本地原文）。D1 若成功，只证明指定学习路径能恢复注入信号；不能证明真实信息上限、策略收益、会计与执行正确。✅ [旧 ENGINEERING §6.1，2026-09-07](../../../../thinking/onchain-rl-trading/ENGINEERING.md) 提出 `RMSE(I)` 与 `T_crit` 数字；❌ 未找到其对本项目成立的完整条件／估计误差界，也未见所读材料附 D1 运行证据。应补诊断，不应让它投票决定所有者目标。

2. **“支付确定＝自带经济阳性对照”把支付与执行合并了。** 数学反例：同一到期条件下若 A 蕴含 B，买 `NO(A)+YES(B)` 的支付可能是 1 或 2，只是下界为 1；还须总成交成本低于该下界。第一腿成交、第二腿报价消失时，下界组合根本没建成。✅ [2026-04-22 NBA 研究](https://arxiv.org/html/2605.00864) 的深度约束与 ✅ [适配器文档，核验 2026-09-18](https://nautilustrader.io/docs/latest/integrations/polymarket/) 的非原子组合边界，直接反驳“因此适合无条件排第一”。它是很好的支付计算对照，不是保证盈利对照。

3. **“BTC 是事件率最低选择”证据过窄。** ✅ [Fed 日历，核验 2026-09-18](https://www.federalreserve.gov/monetarypolicy/fomccalendars.htm) 证明期限内无新例会，不证明 BTC 订单流、资金状态和其他信息无检验事件。反过来，高频报价也不等于独立样本。我支持修改／明确 FOMC 范围，反对据此直接替换所有者的 BTC/ETH 方向。

4. **“Jev 只能提取，不能决策”混淆合理首期位置与永久角色限制。** ✅ [TypeSafe，2026-09-15](https://typesafe.ai/blog/introducing-system-one-models-and-jev) 的任务定义包含类型化选择；🟡 [FLAG-Trader，2025-07](https://aclanthology.org/2025.findings-acl.716/) 提供 LLM 策略网络的正面研究方向。两者均不能证明 Jev 可赚钱，却足以要求我们用受约束对照实验判定，而非先验封死交易动作格。

5. **“同一候选必须完全串行、并行必须跨独立市场”过强。** D 中固定更新算法的逐步评价是逻辑反例。多个候选可以共享未来数据，前提是把选择纳入评价并控制多重试验；扩市场不能自动得到独立性。✅ [Spurious Predictability，2026-04-16，§3](https://arxiv.org/html/2604.15531) 明确讨论相关候选与有效搜索规模。

我同意 Claude 的三点：必须加入阳性／阴性与执行对照；未写出现金流、持仓、平仓或到期结算路径前，HL × Polymarket 不能算已成立的套利支柱；前向时间无法靠多开 Agent 加速。本轮 ❌ 未找到跨场所支柱的完整支付证明。

## H. 哪些证据会改变结论

| 新证据 | 我将如何改判 |
|---|---|
| Polymarket 在我们可取得的数据、延迟和目标资金规模下，连续记录可成交深度、双腿完成率、未完成损失、结算规则、全成本及资金占用；预登记评价显示更快覆盖完整业务链路。 | 可以将其升为首实例。已有他人利润是正线索，仍须证明我们的执行条件。 |
| BTC/ETH 同信息集对照中，链上特征在时间正确、成本压力及未见状态下没有增量，且已知信号对照证明流程有检测能力。 | 降低该特征族优先级；先调整候选信息和机制，不因一个失败模型就宣判整个资产不可预测。 |
| HIP-3 得到精确合约／预言机／会话版本、可复现支付路径及实际成交证据，收益不依赖理想同步成交。 | 可升序；仅有会话规律或重开价相关性不够。 |
| Jev／自训快模型在冻结版本的前向决策中，扣除成本后稳定优于同信息的简单模型，且风险拒绝、超时与异常输入检查通过。 | 扩大其有限动作权限；类型合法率或厂商速度数字不足以触发扩权。 |
| Nautilus 锁定版本无法通过订单生命周期、断线恢复和账户对账；已有更简单组件可以通过同一组检查。 | 更换底座；底座是手段，不是必须捍卫的目标。 |
| 日期推进后仍没有真实权重更新、可用产品，或无法累计约定实盘时长。 | 报告原交付目标未达成，并明确缺项；不能把它重新命名为“研究 MVP 已成功”来降低验收。 |

第二轮应集中反驳这些改变排序的条件与证据，不继续以框架名称争输赢。当前建议保留所有者的完整自主循环目标，以最小单场所实例检验整圈，同时允许证据改变首个业务实例。
