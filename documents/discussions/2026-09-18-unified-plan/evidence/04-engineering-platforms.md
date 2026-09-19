# 证据附录 04：研究到生产平台工程证据（2026-09-18）

> 研究代理产出。标记：✅ 已读一手页面 | 🟡 二手/摘要 | ❌ 未找到。此附录是终稿的证据来源之一，不是结论。

## Q1 顶级机构公开工程写作与可提炼的设计模式

| 结论 | 来源 | 日期 | 标记 |
|---|---|---|---|
| Jane Street 研究负责人 In Young Cho：研究工具追求灵活 vs 生产要求健壮之间存在张力；"quick and dirty" 研究工具直接进生产反复"bite us"；notebook 不是生产框架 | signalsandthreads.com/finding-signal-in-the-noise/ | 2025-03 | ✅ |
| Jane Street ML-infra：训练运行的可复现性是首要 | signalsandthreads.com/the-uncertain-art-of-accelerating-ml-models/ | 2024-10 | ✅ |
| Jane Street 并非单语言：OCaml 为主、Python 做数据分析/ML；重要/共享函数宁可写在 OCaml | signalsandthreads.com/python-ocaml-and-machine-learning/ | 2020-10 | ✅ |
| 「Why OCaml」：类型系统作为设计工具；"交易系统最珍贵的能力是不交易" | blog.janestreet.com/why-ocaml/ | 2016 | ✅ |
| HRT：Trading Tech（70% C++）与 R&D（70% Python）分工；挂单真相以多播事件加日志重建，进程宕机也能恢复 | hudsonrivertrading.com/hrtbeat/engineering-and-interviewing-at-hrt/ | 2025-10 | ✅ |
| HRT Blobby：小团队自建分布式 FS，"只在必要时引入复杂度"，append-only | hudsonrivertrading.com/hrtbeat/distributed-filesystem-for-scalable-research/ | 2025-08 | ✅ |
| HRT Python 代码库百万行"tangle"→分层架构加 CI 依赖规则检查 | hudsonrivertrading.com/hrtbeat/dependency-graph-python-codebase/ | 2023-06 | ✅ |
| Man Group ArcticDB：无服务器、不可变版本、time travel；供应商修数据是常态，必须能回看"我的系统当时看到什么"；BSL 1.1 两年后转 Apache 2.0 | infoq.com/presentations/arcticdb/ ; github.com/man-group/arcticDB | 2024-08 / 2026-01 | ✅ |
| Two Sigma：Memento 框架强调数据增量转换的可复现；2025 "Treating data as code"（数据契约、pipeline 版本化以支持 replay）；Ben Wellington 强调用 point-in-time 训练的开源 LLM 控制时间泄漏 | github.com/twosigma/memento ; twosigma.com/articles/treating-data-as-code-at-two-sigma/ ; twosigma.com/articles/platform-thinking-three-views-from-two-sigma-leaders/ | 2023/2025-11/2025-10 | ✅ |
| Optiver："保持仿真对生产诚实，研究不能以生产无法复现的方式 work"；可复现与可追溯"不可妥协" | optiver.com/insights/technology-blog/designing-for-latency-and-iteration/ ; /research-at-scale/ | 2026-07 / 2025-12 | ✅ |
| Citadel Securities：研究平台上云，>100 万核并发，度量"每研究小时成本" | cloud.google.com/transform/citadel-securities-reimagine-... | 2024-04 | ✅ |
| XTX TernFS：500PB、文件不可变、快照防误删 | xtxmarkets.com/tech/2025-ternfs/ | 2025-09 | ✅ |
| Quantopian 失败：888 策略回测 Sharpe 对 OOS 几乎无预测力（R²≈0.01–0.02）；回测越多 IS-OOS 差距越大；黑箱 IP 保护⇒无法审因果⇒过拟合 | papers.ssrn.com/abstract=2745220 ; quantrocket.com/blog/quantopian-shutting-down/ | 2016 / 2020 | ✅/🟡 |
| Numerai：NMR 质押等于 skin in the game 作为准入门槛；MMC 奖励与元模型正交的信号 | docs.numer.ai | 2019–2023 | ✅ |
| IMC、Jump 研究平台写作 | — | — | ❌ |

**提炼模式**：(a) 研究与生产共享同一数据平面与同一执行语义；(b) 版本化/不可变数据是默认；(c) "真相"以事件流加日志重建，而非查询在线进程；(d) 语言选择是"一强类型核心加 Python 控制面"的多语言。

## Q2 Point-in-time 工具与已记录的泄漏 bug

| 结论 | 来源 | 日期 | 标记 |
|---|---|---|---|
| Qlib #770：NORM processor 在 train/test 切分前拟合导致泄漏；维护者以文档回应，未改默认 | github.com/microsoft/qlib/issues/770 | — | ✅ |
| Qlib #2080：同一天预测值随 test 段结束日期变化 | github.com/microsoft/qlib/issues/2080 | 2026-01 | ✅ |
| backtesting.py #277（预计算指标可能前视）；PR #1230 修 ATR bfill 前视 | github.com/kernc/backtesting.py/issues/277 | 2021 / 2025-02 | ✅ |
| HKUDS/AI-Trader #76：回测中 agent 通过搜索工具读到 4 天后新闻 | github.com/HKUDS/AI-Trader/issues/76 | 2025-11 | ✅ |
| DuckDB ASOF JOIN：语义、比 window 加 inequality 快数个量级 | duckdb.org/2023/09/15/asof-joins-fuzzy-temporal-lookups | 2023-09 | ✅ |
| Polars join_asof：默认 backward；#21693 带 by 分组时不检查组内排序 | github.com/pola-rs/polars/issues/21693 | 2025-03 | ✅ |
| Tecton：用 `_effective_timestamp`（特征在在线库可用的最早时间）做 AS OF join，而非事件时间 | docs.tecton.ai | — | ✅ |
| Feast：entity_df 缺 event_timestamp 则退化为"现在"，静默返回未来值 | theneuralbase.com | — | 🟡 |
| Iceberg：快照加 tag 防 GC，snapshot_id 记入 MLflow 复现训练集 | iceberg.apache.org/docs/1.6.0/spark-queries/ | 2026-05 | ✅ |
| Bridgewater 数据库"bitemporally-modeled" | bridgewater.com/aia-labs | — | ✅ |
| 静态泄漏 lint（shift(-n)、center=True、bfill、fit-before-split）：lookahead-lint / backtest-guard / nullius | github.com/bmouler/lookahead-lint ; blu3c0ral/nullius | — | 🟡 |

**小团队 2025–26 实际用法**：Parquet/DuckDB/Polars 本地 as-of join 加 Iceberg/ArcticDB 版本化是主流；重型 feature store 被视为 overkill。关键坑：**事件时间 ≠ 可用时间**。

## Q3 实验/管线管理与预注册

| 结论 | 来源 | 日期 | 标记 |
|---|---|---|---|
| Metaflow：每 step 持久化数据加代码加依赖的不可变快照，内容寻址存储 | docs.metaflow.org/internals/technical-overview | — | ✅ |
| Kedro #3578 讨论 ETL 痛点；Codex 第二轮核验正文不含"不保证可复现"的引语，本条降级为"Kedro 是编排器，不冻结外部数据" | github.com/kedro-org/kedro/issues/3578 | 2024-01-30 | 🟡 |
| 科学预注册：结构化模板比开放式更能约束"研究者自由度"但都不能消除 | journals.plos.org/plosbiology/...pbio.3000937 | 2020-12 | ✅ |
| Kaggle 公开榜可被无数据"爬榜"；Ladder 机制只在显著改进时才释放分数 | blog.mrtz.org/2015/03/09/competition.html | 2015 | ✅ |
| 量化领域"哈希锁阈值"实践：SHA-256 预注册 AcceptanceCriteria 并 CI 校验；要求 n_configurations_tried 作为必填输入 | github.com/tony140222/validation-protocol ; foolproof-labs/falsification-ledger | 2026 | 🟡 |
| arXiv 框架：IS→purged WFA→OOS，阈值在打开 OOS 前预承诺 | arxiv.org/pdf/2603.09219 | 2026-03 | 🟡 |

**"冻结管线为工件"语义**：Metaflow 最接近；DVC 次之；MLflow/W&B 本身是记录器；Kedro/Dagster/Prefect 是编排器，不冻结外部数据。

## Q4 交易引擎骨架（Python/Rust 小团队，2026）

| 结论 | 来源 | 日期 | 标记 |
|---|---|---|---|
| NautilusTrader 1.231.0（2026-08-02）为最后一个 Cython v1 版；2.0.0rc3/rc4 Rust 加 PyO3 运行时进入 RC | github.com/nautechsystems/nautilus_trader/releases | 2026-08/09 | ✅ |
| 架构：单线程内核确定性事件序、backtest/sandbox/live 共享 NautilusKernel；同进程不支持多个 LiveNode，隔离用多进程 | nautilustrader.io/docs/latest/concepts/architecture/ | — | ✅ |
| Event Sourcing 页：事件库是状态历史的"durable authority"，cache 是投影；录制命令、原始 venue 报告、对账输出 | nautilustrader.io/docs/latest/concepts/event_sourcing/ | — | ✅ |
| DST 页：madsim 替换 tokio，同 seed 复现调度/定时/随机 | nautilustrader.io/docs/latest/concepts/dst/ | — | ✅ |
| 许可 LGPL-3.0-or-later；维护者立场：内部使用/不分发无需开源 | github.com/nautechsystems/nautilus_trader/issues/1197 | — | ✅ |
| LEAN：Apache 2.0，C# 核心加 Python.Net 桥（慢） | github.com/QuantConnect/Lean | — | ✅ |
| Hummingbot：Hyperliquid 连接器 ✅ 含 HIP-3；Polymarket 连接器 ❌ 未找到 | hummingbot.org/exchanges/hyperliquid/ | 2026 | ✅/❌ |
| 自研 asyncio 风险：ccxt bybit 子串分派 34% orderbook 帧误路由致 stale/crossed book | github.com/ccxt/ccxt/issues/28857 | — | ✅ |

**评价**：多 venue crypto 加预测市场且要 backtest-live parity，Nautilus 是唯一同时具备 Hyperliquid 加 Polymarket 适配器、事件溯源、DST 的开源引擎；代价是 v1→v2 迁移期 API 波动、LGPL 合规心智负担、适配器仍在修竟态。

## Q5 事件溯源、对账与状态失同步事故

| 结论 | 来源 | 日期 | 标记 |
|---|---|---|---|
| Knight Capital：8 台服务器中 1 台未部署新代码，复用旧 flag 激活死代码；97 封告警邮件无人看；无 kill switch，回滚方向错误加剧；45 分钟亏 $460M | sec.gov/files/litigation/admin/2013/34-70694.pdf | 2013-10 | ✅ |
| Hummingbot #4991：成交后立刻重启产生重复成交记录且数量翻倍 | github.com/hummingbot/hummingbot/issues/4991 | 2022-01 | ✅ |
| Hummingbot #7294：Hyperliquid 返回 MarketOrderFailureEvent 后订单实际成交→重试导致重复订单 | .../issues/7294 | 2024-11 | ✅ |
| Nautilus #1205/#3081/#3176：重启对账是高发 bug 面（dYdX 同一 fill 挂两次；IBKR 每次重启生成重复合成订单） | github.com/nautechsystems/nautilus_trader/issues/1205, 3081, 3176 | 2023–2025 | ✅ |
| freqtrade #11461：调仓时撤单与成交竟态→两笔都成交 | github.com/freqtrade/freqtrade/issues/11461 | 2025-03 | ✅ |
| passivbot #980：成交摄入滞后→同一入场重复下单，仓位到 142% 限额 | github.com/enarjord/passivbot/issues/980 | — | ✅ |
| ccxt #18028：Binance 期货 WS 重复 FILL 推送 | github.com/ccxt/ccxt/issues/18028 | 2023-05 | ✅ |
| marketmind：同一帧 3 个 WS 事件并行处理→仓位双计；修法"信任交易所而非加法" | github.com/nathanssantos/marketmind/commit/03e5255 | 2026-05 | ✅ |
| 个人复盘：OKX API 返回 code 0 但止损单在交易所侧 failed | supa.is/article/two-losses-two-bugs-okx-hyperliquid-post-mortem | 2026-03 | ✅ |
| 2025-04-15 AWS 东京故障后多 venue 重连未重拍快照→本地"flat"实际多头约 $180K | electronictradinghub.com/crypto-market-making-position-drift-... | 2026-06 | 🟡 |
| Polymarket：2026-02-19 链下已"executed"而链上 matchOrders 因 nonce 过期回滚；2026-09-01 DB 副本滞后进入 cancel-only | status.polymarket.com/cmtj44v7b0m6b13mv6828j2gj | 2026 | 🟡/✅ |

## Q6 AI 代理入环

| 结论 | 来源 | 日期 | 标记 |
|---|---|---|---|
| METR RCT：16 名资深开源开发者、246 任务，允许 AI 时慢 19%，而自评快 20% | metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/ | 2025-07 | ✅ |
| Man Numeric AlphaGPT：Idea/Implementer/Evaluator 加 orchestrator；"AI 或人产生的信号必须通过相同阈值"；全过程日志加投委会加代码审查双轨 | man.com/insights/what-ai-can-do-for-alpha | 2025-11 | ✅ |
| Balyasny：12+ 维内部评测管线；agent 只能用被批准的工具/数据，"模型不是控制本身" | openai.com/index/balyasny-asset-management/ ; claude.com/blog/working-at-the-frontier-how-balyasny-... | 2026-03 / 2026-09-17 | ✅ |
| Bridgewater PAT：把校验写进普通 Python 架构"agent 无法忘记验证"；"Teach"按钮→先写会失败的 benchmark 再修→PR 人审 | insights.ml4trading.io/p/how-bridgewater-engineers-a-research | 2026-07 | 🟡 |
| Two Sigma：LLM 特征工程需用 point-in-time 训练的模型 | twosigma.com/articles/platform-thinking-... | 2025-10 | ✅ |
| RD-Agent(Q)：Research/Development 两段，Validation 用 Qlib 真实回测；注意其模板沿用 Qlib 的 handler-level 归一化 | microsoft.com/en-us/research/publication/rd-agent-quant | 2025-10 | ✅ |
| agent-lightning：Algorithm 与 Runner 之间零直接通信，仅经 LightningStore | microsoft.github.io/agent-lightning/latest/deep-dive/birds-eye-view/ | — | ✅ |
| LLM 回测泄漏研究群：Profit Mirage、Look-Ahead-Bench、Temporal Leakage、Leakage-safe LLM strategy discovery（agent 只能通过 registry 校验工具行动） | arxiv 2510.07920, 2601.13770, 2608.02985, 2608.27734 | 2025-10 至 2026-09 | 🟡 |

## Q7 胶水单体 vs 微服务；三区部署

| 结论 | 来源 | 日期 | 标记 |
|---|---|---|---|
| 交易系统咨询方经验：微服务需 3 名全职 DevOps vs 单体 0.5；<20 人团队协调成本大于收益；推荐"模块化单体，日后可抽取" | nordvarg.com/blog/microservices-vs-monolith | 2024-10 | 🟡 |
| Nautilus 官方：一进程一节点多策略；隔离用多进程 | nautilustrader.io/docs/latest/concepts/architecture/ | — | ✅ |
| Kraken 官方：一进程一 API key（nonce 隔离、最小权限、故障归因）；子账户隔离资金/风险面；子账户禁止直接提币 | blog.kraken.com/product/api/unlocked-6-multi-strategy-operations-subaccounts-api-keys | 2026-05 | ✅ |
| Binance/Coinbase 权限模型：TRADE 与 Withdraw 分离；IP 白名单是提币类操作前置 | trilicity.com/articles/api-key-security-best-practices/ | 2026-07 | 🟡 |
| 顶级机构公开的"三区（采集/研究/执行）"部署论证 | — | — | ❌ 未找到；为基于 key/nonce/延迟证据的推论 |

## Q8 多候选并发生命周期与资本闸门

| 结论 | 来源 | 日期 | 标记 |
|---|---|---|---|
| Man AlphaGPT：AI/人类信号进入实盘前通过**相同**阈值 | man.com/insights/what-ai-can-do-for-alpha | 2025-11 | ✅ |
| Numerai：staking 作为资本/信任闸门；24 个重叠 round 同时在评 | docs.numer.ai/numerai-tournament/submissions | — | ✅ |
| 四闸门框架：预注册搜索空间→DSR/PBO→≥6 个月 paper→25% 名义起步、5% 回撤 kill、live Sharpe<50% paper 即停 | equationstocapital.com/research/papers/paper-08-... | — | 🟡 |
| 个人 12 策略平台复盘：两条已退役策略仍占 62.7% 资本份额，分配状态与风控层"两个真相无人比对" | github.com/Ssebv/systematic-trading-showcase | — | 🟡 |
| 顶级机构公开的策略生命周期状态机框架 | — | — | ❌ 未找到 |

## 给 1–3 人团队的推荐工程栈（带证据）

1. **拓扑：模块化单体加进程级隔离，不做微服务。** 三个进程（采集 / 研究批处理 / 执行），共享一个 Python 包与一个 Parquet/Iceberg 数据平面。
2. **数据层：Parquet 加 DuckDB/Polars as-of join，版本化用 Iceberg tag 或 ArcticDB。** 每张特征表三列时间：event_ts、available_ts（含供应商延迟）、ingest_ts；训练/评估只允许按 available_ts 做 backward as-of。
3. **研究管线为一等版本对象：Metaflow 或 DVC 冻结代码加配置加数据指针；MLflow 只做记录，阈值/验收标准作为哈希锁定的元数据。**
4. **验收：CI 判、agent 不判。** 预注册 JSON（阈值、trial 计数、数据快照 id、代码 SHA）先提交再跑 OOS，CI 校验哈希。加静态泄漏 lint 与 prefix-invariance 测试。凡 agent 生成的代码，只允许通过白名单特征工具访问数据。
5. **执行：NautilusTrader v2（LGPL）为骨架，自己的策略/适配器保持独立包。** 开启 event store，把 venue 原始报告全录；另跑独立 REST 对账循环。上线前用 DST 种子回放加强制重启对账演练。
6. **密钥：三类 key 三台机器。** 采集用只读 key；执行用 trade-only 加 IP 白名单，永不开提币；研究区无 key。

## 五个最常见的工程失败模式与预防

1. **"事件时间"当"可用时间"用（look-ahead）。** 预防：available_ts 强制列加 backward as-of 加泄漏 lint 加 prefix-invariance 回归测试。
2. **重启/重连后的状态失同步与重复订单。** 预防：客户端幂等 order id、"订单离簿但无成交摄入"即冻结该标的新开仓、序列号缺口必重拍快照、独立 REST 对账 K 次确认。
3. **"API 说成功"≠"交易所已生效"，且 WS 事件并发处理双计。** 预防：下单后回查状态、以交易所返回的绝对仓位覆盖本地加法、单线程内核序列化事件。
4. **多重检验/搜索强度不入账。** 预防：trial registry 为必填、DSR/PBO 门、阈值先哈希后开 OOS、AI 与人同阈值。
5. **部署与死代码路径失控加无 kill switch。** 预防：不可变构建工件加版本一致性校验、feature flag 不复用、告警必须可动作、kill switch 独立于策略进程并定期演练。

**主要未找到项**：IMC/Jump 研究平台写作；WorldQuant BRAIN 官方设计文档；顶级机构公开的策略生命周期状态机；Hummingbot Polymarket 连接器；"三区部署"的机构级论证。
