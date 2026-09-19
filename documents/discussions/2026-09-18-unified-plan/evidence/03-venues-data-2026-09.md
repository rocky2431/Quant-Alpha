# 证据附录 03：链上场所、数据与工具链现状（截至 2026-09-18）

> 研究代理产出，2026-09-18。标记：✅ 已读一手页面（官方文档/GitHub/论文摘要）；🟡 二手来源；❌ 未找到/未验证。此附录是终稿的证据来源之一，不是结论。

## Q1 Hyperliquid 现状

| 事实 | URL | 日期 | 标记 |
|---|---|---|---|
| 官方 S3 仅两桶：`s3://hyperliquid-archive`（L2 快照 `market_data/[date]/[hour]/l2Book/[coin].lz4`，`asset_ctxs`），`s3://hl-mainnet-node-data`（`node_fills_by_block`、`misc_events_by_block` 等）。requester-pays；"约每月上传一次，不保证及时，可能缺失"；不提供 candles/spot 历史，"请自行用 API 录制" | hyperliquid.gitbook.io/hyperliquid-docs/historical-data | 2026-09 | ✅ |
| fills 格式 2025-07-27 切换；全币种约 0.8–1.0 GiB/天，一年拉取三位数美元账单 | pypi.org/project/hyperliquid-data/ | 2026 | 🟡 |
| 节点规格：16 vCPU / 128 GB / 500 GB SSD，建议东京；约 100 GB 日志/天；flags `--write-fills`、`--write-raw-book-diffs`、`--write-hip3-oracle-updates`、`--write-misc-events`、`--batch-by-block`；L4 快照由 `compute-l4-snapshots` 从每 10,000 块状态离线计算 | github.com/hyperliquid-dex/node | 2026-09 | ✅ |
| 限速：REST 每 IP 1200 权重/分；WS 每 IP 最多 10 连接、1000 订阅、2000 msg/分 | hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits | 2026-09 | ✅ |
| 公共 WS `l2Book` 降频：2026-06 调整，`fast:true` 5 档/0.5s；默认 20 档/2s，拟改 5s | hyperliquid.gitbook.io/.../websocket/subscriptions ; quicknode.com/blog/hyperliquid-foundation-websocket-changes | 2026-06 | ✅/🟡 |
| 全深度逐块 L2/L4 需自建节点或付费 gRPC（QuickNode `StreamL2Book`/`StreamL4Book`） | github.com/quicknode/Hyperliquid-Orderbook-Radar | 2026-07 | 🟡 |
| HIP-3 规模：2026-08-25 快照 144 个在线 builder 市场、OI $3.78B；trade.xyz OI >$4B、8 月占全站约 55% 成交；Q2 trade.xyz 占 HIP-3 成交 95.1%；Felix/Ventuals/Dreamcash 三家部署者已关停 | hyperliquidguide.com/markets/hip-3 ; coinmetrics.substack.com/p/tradexyzs-role-in-hyperliquid ; hyperliquidresearch.xyz/reporting/tradexyz-2026-q2 | 2026-08/09 | 🟡 |
| HIP-3 费率（trade.xyz）：taker 0.090% / maker 0.030%（原生的 2 倍）；Growth mode ≥90% 减免，加密相关及黄金不可用 | docs.trade.xyz/perp-mechanics/fees | 2026-09 | ✅ |
| 会话机制：Relayer 约每 3 秒推送 oracle/mark/external；收盘后 oracle 走 EWMA 内部机制；Discovery Bounds 将 mark 限于参考价 ±(1/最大杠杆)，触发阈值后棘轮重锚，每市场每方向有限次 reset；外部恢复时参考价回到外部价、计数归零；界外爆仓价不会被清算；v2 方法学已替换 v1 | docs.trade.xyz/perp-mechanics/discovery-bounds ; /oracle-price ; /changelog/discovery-bounds-v2 | 2026-09 | ✅ |
| 2026-07-28 xyz:SKHYNIX 闪崩：韩国 NXT 盘前一笔 1 股在跌停价成交，oracle 4 秒内 $1,131→$955，约 960 个多头被清算（$57.4M 仓位），约 100 个空头被 ADL；trade.xyz 称 oracle"按规格工作"，一次性自愿赔付 | galaxy.com/insights/research/hyperliquid-tradexyz-oracle-liquidations ; nexusmutual.io/blog/xyz-skhynix-flash-crash-on-hyperliquid-incident-report | 2026-07 | 🟡 |
| 2025-10-10 崩盘：12 分钟内 162 个资产 ADL $2.10B、清算 $5.51B；首个 ADL 在首笔清算后 61.7s | github.com/ConejoCapital/HyperMultiAssetedADL | 2025-11 | 🟡 |
| 2026-01-31 鲸鱼 $570M ETH 多头被清算；2026-02-05 $200M 50x ETH 清算致 HLP 亏 $4M，HL 随后降杠杆 BTC 40x / ETH 25x | ainvest.com ; cryptoslate.com | 2026-01/02 | 🟡 |
| 隐藏单：Aster 2025-06-23 上线 Hidden Orders；Hyperliquid 截至 2026-03 无隐藏单，簿面/仓位全透明 | globenewswire.com（Aster） ; hyperliquidguide.com/compare/hyperliquid-vs-asterdex | 2025-06 / 2026-03 | ✅/🟡 |
| HIP-4 结果市场：2026-05-02 主网上线，每日 06:00 UTC 按 HL BTC mark 结算的二元合约；6 月扩展到 ETH/HYPE/SOL；8 月中起收费 | hyperliquidguide.com/ecosystem/hip-4-outcome-trading | 2026-05/08 | 🟡 |

## Q2 Polymarket 现状

| 事实 | URL | 日期 | 标记 |
|---|---|---|---|
| 国际站费率（仅 taker，按 `feeRate·C·p(1-p)`）：Crypto 0.07、Sports 0.05、Finance/Politics 0.04、Geopolitics 0；maker 返佣 Crypto 20%、Sports 15%、其余 25%；以 `feesEnabled` 为准 | docs.polymarket.com/trading/fees ; /programs/maker-rebates | 2026-09 | ✅ |
| 变更史：7-24 撮合改异步；8-07 加密 up/down 改为 Chainlink 60s TWAP 结算；8-17 加密市场 taker 延迟 250ms→50ms | docs.polymarket.com/changelog | 2026-07/08 | ✅ |
| Polymarket US（QCX LLC，CFTC DCM）费率自 2026-07-01：taker theta 0.06，maker 返佣 -0.0125 | docs.polymarket.us/fees | 2026-07 | ✅ |
| Combos：国际站 2026-06-10 上线（仅体育，最多 32 腿），RFQ 撮合：做市商 400ms 报价窗、用户 10s 接受窗；US 站 8-21 公开；公开端点仅暴露可用腿，Combo 报价不可被外部观测 | help.polymarket.com/en/articles/15458600-what-are-combos | 2026-06/08 | ✅/🟡 |
| 地域限制三档：OFAC 全封；"前端加 API 仅可平仓"含 US、UK、FR、DE、IT、AU、BR、SG、PL、TW、TH 等 30+；"仅前端平仓、API 开放"：IE、JP、NL、MT | docs.polymarket.com/api-reference/geoblock ; help.polymarket.com/en/articles/13364163-geographic-restrictions | 2026-09 | ✅ |
| 美国路径：2025-07 收购 QCEX；2025-09 CFTC no-action；2026-05-12 取消候补；2026-04-28 Bloomberg 报道正与 CFTC 谈判主站上岸（未决）；2026-05 Kalshi 约 58% vs Polymarket 28% 美国份额 | covers.com ; theblock.co/post/399234 | 2025-12 至 2026-05 | 🟡 |
| CLOB V2 于 2026-04-28 切换：新合约（CTF Exchange V2 `0xE111…996B`，NegRisk V2 `0xe2222…0F59`）、抵押品 USDC.e→pUSD、旧 SDK 完全失效、全部挂单清空 | docs.polymarket.com/v2-migration | 2026-04 | ✅ |
| 官方历史数据：Goldsky 声明 V2 后不再用 subgraph；NautilusTrader 1.224 因"端点已下线"移除 `fetch_orderbook_history`/`fetch_price_history`；Data API `/trades` 在 offset 3,500 处静默返回 400。**结论：官方无历史订单簿**；逐笔需自读 Polygon `OrderFilled` 日志（V2 合约自 2026-04 起）或 Dune | docs.goldsky.com/chains/polymarket ; github.com/nautechsystems/nautilus_trader/releases/tag/v1.224.0 ; pred-markets.com/blog/polymarket-data-api-3500-cap/ | 2026-03/05 | ✅/✅/🟡 |
| 官方 `agent-skills` 仓库仍列出 `getPricesHistory` 与 Goldsky subgraph，与上条矛盾，需在线实测 | github.com/Polymarket/agent-skills/blob/main/market-data.md | — | 🟡（冲突） |
| Dome：2026-02-19 被 Polymarket 收购，所有 API 于 2026-04-28 EOL（已确认）；替代：pmxt、Parsec、polynode、Dune | docs.domeapi.io ; domeapi.io/blog/dome-joins-polymarket ; dune.com/pricing | 2026-02/04 | ✅ |
| UMA 争议：2026-06 "MicroStrategy sells any BTC by May 31" 市场在 8-K 证实卖出后仍经 UMA 投票 98.6% 定为 No；WSJ 分析多数争议中前 10 大钱包 >50% 票权；2026 年至 5 月已 >1,150 个市场进入仲裁 | coindesk.com ; theblock.co/post/403600 ; financemagnates.com | 2026-05/06 | 🟡 |
| 逻辑套利竞争度：AFT 2025 论文证实组合套利已被利用；arXiv 2608.00666 估计累计套利利润 $1.12M（其中 $1.086M 依赖 NegRisk 转换器）；NBA 市场：组合机会 290 次、中位收益 101bp，但 76.9% 受限于平均仅 14.8 股的可执行深度 | doi.org/10.4230/lipics.aft.2025.27 ; arxiv.org/abs/2608.00666v1 ; arxiv.org/html/2605.00864 | 2025-10 / 2026-08 / 2026-05 | ✅ |
| 5m/15m 加密市场机器人：29 个钱包以 Binance 现货为信号在最后 15 秒下单，胜率 >98%；Chainlink vs CF Benchmarks 结算源分歧率 5.7–6.2%；第二周延迟套利已转负 | github.com/OffGrid0xDAO/cross-platform-arbitrage | 2026-03 | 🟡 |

## Q3 预测市场 vs 衍生品的跨场所定价

| 事实 | URL | 日期 | 标记 |
|---|---|---|---|
| arXiv 2606.19517：Polymarket BTC 阈值合约 Yes 价与 Binance 期权隐含数字期权值均值缺口 5.6pp（t=6.46，214 观测） | alphaxiv.org/abs/2606.19517 | 2026-06-17 | ✅ |
| "What price will Bitcoin hit in 2025?"：Polymarket 触及类合约视为 one-touch 障碍期权，用 Deribit 期权定价；delta 对冲显著降低 P&L 方差 | linclund.com/.../What-price-will-Bitcoin-hit-in-2025-1.pdf | 2026-05 | 🟡 |
| Polymarket vs Kalshi 跨场所：2026-08-30 配对 186 个结果对，去除陈旧报价后缺口中位 0.78¢、P90 2.5¢；费改后中价盈亏平衡约 3¢ | predictmarketcap.com/analysis/prediction-market-arbitrage-study ; vultax.com/research/kalshi-polymarket-arbitrage-gaps-after-fees | 2026-08/09 | 🟡 |

## Q4 链上数据与 BTC/ETH 价格发现

| 事实 | URL | 日期 | 标记 |
|---|---|---|---|
| Glassnode：Advanced $49/月仅 24h 分辨率；Professional（报价制）才有 10 分钟分辨率 | studio.glassnode.com/pricing | 2026-09 | ✅ |
| Coin Metrics community API 免费非商用；2025-07 被 Talos 收购 | rfp.wiki | 2026-06 | 🟡 |
| mempool.space REST/WS 免费但限速 | mempool.space/docs/api/rest | 2026-09 | ✅ |
| 链上指标预测力：Grobys 2026：NUPL/MVRV-Z/CVDD 规则跑赢买入持有但全程仅 3 进 3 出，不考虑交易成本 | iris.unito.it/... | 2026-05 | ✅ |
| arXiv 2411.06327：USDT 净流入交易所正向预测 BTC/ETH 收益（仅 1–2h 显著），无成本后交易验证 | arxiv.org/pdf/2411.06327 | 2024-11 | ✅ |
| Shelton 2024：S2F、Metcalfe 样本外几乎无预测力 | ideas.repec.org/a/gam/jjrfmx/v17y2024i10p443 | 2024-10 | ✅ |
| CEX/DEX 份额（CoinGecko）：永续 DEX 份额 2.0%→10.2%；Hyperliquid 占 perp DEX 成交 39.5%、OI 59.1%；Binance 2026 永续份额约 33% | assets.coingecko.com/reports/2026/CoinGecko-2026-State-of-Crypto-Perpetuals-Report.pdf | 2026-05 | ✅ |
| 价格发现：Binance 现货与永续为 BTC 主要价格发现源；2026 预印本：Binance 在所有窗口领先 Hyperliquid，Hasbrouck 信息份额 63–83%；HL 上 658 个"知情钱包"整体跟随 Binance | papers.ssrn.com/sol3/papers.cfm?abstract_id=5070964 ; researchsquare.com/article/rs-10147582/v1.pdf | 2025 / 2026 | ✅ |

## Q5 工具链

| 事实 | URL | 日期 | 标记 |
|---|---|---|---|
| NautilusTrader 最新：2.0.0rc5 预发布（2026-09-15；本附录原写 rc4，Codex 第二轮核验发布页更正）；1.x 最新 1.231.0；1.226 Polymarket 迁 CLOB V2/pUSD、HL fundingHistory/Depth10；1.227 HIP-4 结果合约建模 | github.com/nautechsystems/nautilus_trader/releases | 2026-09 | ✅ |
| 两适配器仍在高频修复：HL 侧修 ADL 成交方向解析、fill 早于 order 缓存、HIP-3 含 `*` 符号 panic；Polymarket 侧修 V2 overfill、IOC 误为 FOK、trade_id 碰撞等。判断：可用但属"beta 成熟度" | 同上 | 2026 | ✅ |
| Hyperliquid 开源录制：`Giri-Aayush/hyperliquid-data-pipeline`（WS spool 加 S3 归档加节点 raw book diffs → L4，Parquet）；`imperator-co/order_book_server`（Rust，从本地节点流出 l2/l4/trades）；`hyperliquid-data` PyPI | github.com/Giri-Aayush/hyperliquid-data-pipeline ; github.com/imperator-co/order_book_server | 2026 | ✅/🟡 |
| Polymarket 开源录制：`pmxt-dev/polymarket-orderbook-collector`（2026-07-31，Rust WS 全市场 firehose → ClickHouse → Parquet）；`weiminglong/poly-book`；`jamtho/polymarket-fetcher` | github.com/pmxt-dev/polymarket-orderbook-collector ; github.com/weiminglong/poly-book ; github.com/jamtho/polymarket-fetcher | 2025–2026 | ✅ |

## Q6 监管与运维

| 事实 | URL | 日期 | 标记 |
|---|---|---|---|
| CFTC：2026-06-12 发布预测市场 NPRM；2026-02 撤回 2024 年拟禁政治合约的提案 | cftc.gov ; financialmarkets.law/prediction-markets/ | 2026-06 | ✅/🟡 |
| 州级冲突：第三巡回 2026-04 判 CEA 先占；第九巡回 2026-08-28 判各州可按赌博监管，巡回分裂 | cnn.com/2026/08/28/business/states-prediction-markets-gambling-federal-appeals-court | 2026-08 | 🟡 |
| 非美方向：巴西 2026-04 封禁 Polymarket；法国 2026-07 ISP 层封网站 | datawallet.com/crypto/polymarket-restricted-countries | 2026-07/08 | 🟡 |
| Hyperliquid 监管：2026-02 成立 Hyperliquid Policy Center；07-14 与 SEC Crypto Task Force 会面；08-24 致函要求把股票永续归类为 security futures；HL 目前不向美国用户开放 | chaintechdaily.com ; beincrypto.com | 2026-07/08 | 🟡 |
| 托管教训：Bybit 2025-02-21 约 $1.5B：Safe{Wallet} 开发者被社工→前端 JS 注入→三名硬件签名人"盲签" delegatecall；密钥未泄露、阈值满足。教训：签名机与交易构造环境是弱点，需独立于 UI 的载荷校验/模拟 | blog.blockstream.com/what-the-bybit-exploit-reveals-about-enterprise-custody/ | 2025-02 | 🟡 |
| HL 自动化最佳实践：API/agent 钱包只能下单不能提币；每主账户最多 3 个命名 agent；agent 180 天过期；一策略一 agent；主账户用 Safe 2/3 硬件多签 | medium.com/@hugo0x18/... | 2026-03/04 | 🟡（官方页未直接读取） |

## 对首个实例选择的直接含义（只看数据可得性与事件频率）

**(c) HIP-3 会话结构 ≥ (a) BTC/ETH 方向 > (d) 跨场所相对价值 > (b) Polymarket 内部逻辑套利。**

- **(c)**：机制公开、参数化、可复现；事件频率高且可预排：约 250 个 HIP-3 市场 × 每周 5 个收盘/重开加周末，约每周上千个"冻结→再收敛"事件。短板：历史短（多数市场 2026 年上线）、S3 archive 是否含 `xyz:` 市场 l2Book **未验证（❌）**、部署者集中、费率是原生 2 倍。
- **(a)**：数据最厚，但价格发现在 Binance，HL 是跟随者；链上指标的成本后样本外证据薄弱；"事件"不离散，验证回路难闭合。
- **(d)**：Polymarket 加密市场专业钱包已占据 >98% 胜率、边际第二周即衰减；期权隐含 vs Polymarket 5.6pp 缺口是研究级信号，但需期权数据与结算源风险建模。更可控的替代：HL 站内 HIP-4 每日二元 vs 同站永续，但流动性尚小。
- **(b)**：官方历史订单簿不存在，只能从 2026-04-28 V2 合约起自读链上成交；可执行组合机会深度中位约 15 股；Combos 走 RFQ 不可观测；UMA 争议 2026 年 >1,150 起。数据可得性和事件质量四者中最差。

## 必须今天开始录制的数据清单

1. **Hyperliquid 公共 WS**（所有 `xyz:` HIP-3 市场加 BTC/ETH/HYPE/SOL 原生加全部 HIP-4 结果市场）：`l2Book` 两路（`fast:true` 与默认）、`trades`、`bbo`、`activeAssetCtx`、`allMids`；每帧记录本地接收时间戳并原样落盘。注意 10 连接/1000 订阅上限，需分片。
2. **每日快照**：`perpDexs`、每个 dex 的 `meta`/`metaAndAssetCtxs`、trade.xyz Specification Index 版本化存档，参数会变且不公告。
3. **尽快自建 HL 非验证节点**：`--write-fills --write-hip3-oracle-updates --write-raw-book-diffs --write-misc-events --batch-by-block`；同时补拉 S3 `l2Book` 与 `node_fills_by_block`。
4. **Binance BTC/ETH USDT 永续加现货**：`bookTicker`、`aggTrade`、`depth@100ms`，作为价格发现基准；如可得加 Deribit 期权报价。
5. **Polymarket**：Gamma 全市场元数据（含 `feesEnabled`/resolution 源）；Market WS 至少覆盖全部 crypto 类市场；RTDS `crypto_prices_twap_sixty`；Polygon `OrderFilled` 日志（V2 合约，从 2026-04-28 起回填）；UMA 提案/争议/投票事件；`GET /v1/rfq/combo-markets` 每日快照。
6. **事件/事故台账**：HL 公告、trade.xyz changelog、Polymarket changelog、UMA 争议清单，按日期落表供回测做"制度断点"标注。
7. **Kalshi**（若可开户）：加密合约簿与成交。

未验证/待办：S3 archive 中 HIP-3 市场覆盖；CryptoQuant 定价；HL 官方 API wallet 页原文。
