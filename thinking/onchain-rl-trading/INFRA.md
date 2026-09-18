# 链上量化研究基建设计 v2

**日期**：2026-09-07 ｜ **目标**：机会无关的体系——发现、建模、证伪、跑实质资金
**关系**：`RESULT.md` 是当期市场分析（会过期），本文是产生和淘汰机会的机器（不过期）。
**v2 修订**：v1 的「spec 作为唯一真相源、自动生成完整仿真环境」被前人证据推翻，已拆分降级。见第零章。

---

## 零、v1 的三处错误与修正

### 错误 1：「spec 自动生成完整仿真环境」会崩

**不是维护成本问题，是抽象层次不可调和。**

能被精化成代码的规格，比"不同步的"规格**难写得多**，因为你失去了一整个抽象层次（Hillel Wayne, 2024-03-19）。而要生成能跑 RL 的仿真器，spec 必须包含队列位置、部分成交、taker/maker 判定、tick/lot 舍入、拒单条件、auction 切换的每一个分支——**写到这个粒度，spec 就是实现，只是换了个语法。**

证据链：
- **Sour Protocol** 的 Kani harness 因 CBMC 45 秒预算把 u64 收窄到 u8 甚至 u4；`u128 / 1_000_000` 的除法在预算内判不了，退回随机 proptest。**这还只是会计算术，不含撮合。**
- **Vega `0053-PERP`** 是目前公开最详尽的 perp 机制规格（clamp 函数、TWAP 采样、auction 期间按比例折算 funding、scaling 与 bounds 的先后顺序、40+ 条带唯一 ID 的验收准则、配 `approbation` 产出 spec↔测试覆盖矩阵）。**而它只是 perp 产品的一个模块，全套 89+ 文件，由专职团队维护——然后项目于 2024-09 被治理投票关停。**
- **Clockwork Finance Framework**（IEEE S&P 2023）在 K framework 里写了 4 个 DeFi 协议的可执行模型，声称 attack-exhaustive，从真实交易活动挖出平均 **$56M/月** 的可提取价值。**代码最后一次 push 是 2021-09-08。**

**修正**：spec 只写**会计核心**，不写撮合。

### 错误 2：「一套求解器跑所有 spec」的收益被高估

求解器泛化的瓶颈从来不是环境接口（Gymnasium 已经解决了），而是**每个 venue 的状态空间、动作约束、奖励尺度都不一样**——你还是要为每个 venue 调特征、调奖励、调风险约束。Bayforest/Bertsekas 那篇 MPC 的动作空间是 d=11；换个 venue 换个动作语义，QP 的结构就变了。

**修正**：这是**架构整洁性收益，不是研究效率收益**。不要拿它当立项理由。

### 错误 3：接缝扫描器的定位过高

必须区分两件事：
- 接缝**存在**——Hyperliquid funding 有 ±0.05% clamp、封顶 4%/h、用 spot oracle 而非 mark price 计价。**这是官方文档第一段就写着的公开信息。**
- 接缝**可利用**——在当前市场状态下 clamp 正在 binding、方向可预测、幅度超过费用+滑点+资金成本。**这需要具体状态、具体流动性、具体成本。**

DeFiPoser 与 FlashSyn 都在做后者，且**都必须接触具体链上状态**才能判断盈利性。静态扫描在语法层，可利用性在状态层。

**修正**：降级为"结构化假设生成器"——它输出清单，不输出 alpha。每一条都必须走完整仿真+回测才算数。

### 但核心洞察成立，而且比 v1 描述的更有价值

**所有反对形式化的证据——Hillel Wayne 的同步成本、Rockwell Collins 的 308 小时/指令、IronSpec 的 vacuous spec 问题——都假设 spec 与实现的一致性靠人来维持。**

**链上不是这样：有真值，校验可以自动、连续、无人值守地跑。这把"规格漂移"从人工负担变成了 CI 信号。**

而这件事——把仿真器状态与链上真值做逐字段 conformance——**在 DeFi/perp 领域没有任何人做过**。通用范式存在（Quint trace validation），Ethereum 共识层存在（SpecTrum / Fluffy / Forky），**中间这一段是空的**。

**并且 perp 的会计核心比想象中小**：Hyperliquid 完整的 funding + margin + liquidation 大约是 5 个公式加十几个参数，**300–800 行可执行代码**，不是 89 个 spec 文件。而它覆盖的恰恰是**你的 PnL 真正依赖的部分**。撮合细节反正校验不了（HL 不公开 L3），那就别 spec 它。

---

## 一、不变量（不变）

```
s = (p, m)
p  外生价格   转移不可知        → 预测器给分布，承认有信息天花板
m  机械状态   转移是已发布的公式 → 写死，不学；且【依赖你的动作】
a  动作       受 m 的硬约束      → 保证金、bounds、订单上限
r  奖励       (m, p) 的闭式函数  → 费用/funding/滑点/清算罚金全可算
```

**这个分解在所有场景成立。** 也解释了过去为什么卡住：Qlib、随机森林、FinRL 组合**全是纯 p 的问题**，而 alpha 在 m 上。

本次探究找到的每一个机会都来自读公式，没有一个来自数据挖掘：funding 死区（`clamp(0.0001−P, ±0.0005)`）、impact price 杠杆失配（$6k 控制量 vs 按全部 OI 的计价量）、HIP-3 会话结构、部分清算 20%/30s、ADL 排序 `(mark/entry)×(notional/account_value)`、跨链 oracle 领先 19.4s。

---

## 二、真正要造的东西：会计核心 + 重放校验断言集

### 2.1 范围（严格）

**写进 spec 的**（Hyperliquid，字段均已核验）：
```
funding:      F = avg_premium + clamp(0.0001 − premium, −0.0005, +0.0005)
              premium 每 5s 采样、按小时平均、按 1/8h 支付、封顶 4%/h
              impact_price_difference = max(impact_bid − oracle, 0) − max(oracle − impact_ask, 0)
              impact notional = 20,000 USDC (BTC/ETH) / 6,000 USDC (其他)
              ★ 结算用 spot oracle price 换算名义值，不是 mark price
              ★ HIP-3 用【不同的】公式：premium = 0.5*(impact_bid+impact_ask)/oracle − 1（无 clamp）
margin:       cross / isolated；maintenance = 最大杠杆下初始保证金的一半（1.25%–16.7%）
              分档：mm_rate(tier n) = (tier n 初始保证金率)/2
              mm_deduction(n) = mm_deduction(n−1) + notional_lower_bound(n)*(mmr(n)−mmr(n−1))
liquidation:  liq_price = price − side*margin_available/position_size/(1 − l*side)，l = 1/MAINT_LEV
              >$100k 仓位每次只发 20% 到盘口；30 秒冷却期内改为整仓
              权益 < (2/3)*mm → backstop 转 HLP，维持保证金不退还
adl:          排序索引 = (mark/entry) × (notional/account_value)，按上一个 mark price 成交
constraints:  max_market_order_ntl = {≥25x: 30e6, 20–25x: 5e6, 10–20x: 2e6, else: 500e3}
```

**不写进 spec 的**：撮合、队列位置、部分成交顺序。理由：HL 不公开 L3，**校验不了的东西不要 spec**。用 `ordersim` 那种诚实做法——标注为"在明确的队列假设下重建的 virtual MBO"，并把假设写在 manifest 里。

**注意一个静默错算的陷阱**：HIP-3 perps 用**和普通 perps 完全不同**的 premium 公式。spec 里如果没有"这个 market 属于哪一类"的判别逻辑，整套东西会**静默地算错**。

### 2.2 重放校验：这是地基

```
拿历史重放 → 逐字段比对「spec 预测的状态」vs「链上真实状态」
可交叉校验的真值：每小时 funding rate、每笔 funding payment、
                  每个清算事件的触发时刻与成交价、liquidationPx、markPx、premium、openInterest
```

**每日跑，进 CI。** spec 错了或 venue 悄悄改了参数，**CI 当天就红**，而不是等你亏钱才发现。这是这套设计相对"手写回测器"的唯一、也是全部的优势。

现成骨架：**Quint** 的 trace validation（1662★，2026-08 活跃）——官方原话："使用生产环境的数据，但期望来自模型……问 Quint：这个 transition/state 序列在我的模型里可能吗？"配标准 ITF trace 格式和 `quint-connect`（Rust，81★）。Informal 自己在一个项目上"不到 1 人周找到一个 bug"。

方法论参照：**SpecTrum**（arXiv:2608.17738）把 Ethereum consensus-spec 的**隐式有效性条件显式化为 if-premises**，定义 premise coverage，发现 27 个跨客户端分歧，**其中 22 个不加显式 premise 就找不到**。

### 2.3 ⚠️ 真正的前置瓶颈：可观测性，不是维护

**`impact_bid_px`（20,000 USDC 名义值的平均执行价）你必须自己重建 L2 book 才能算，而 premium 是每 5 秒采样、按小时平均的（720 个样本）。**

→ **没有 5 秒粒度的 L2 快照，你根本无法逐字段验证 funding。**
→ **你的校验精度上限由你的行情记录精度决定。**
→ **这个前置工程比 spec 本身大。**

而 Hyperliquid 官方 S3 归档的 `l2Book` 是**按小时目录、约每月上传一次、不保证及时、数据可能缺失**（官方原文）。所以：**从第一天起就自己录 5 秒 L2 快照**。这件事没有捷径，且**每晚一天就永久少一天可校验的历史**。

---

## 三、不要从零造：站在 NautilusTrader 上

| 项目 | 状态（2026-09-07 核实）| 它已经解决了什么 |
|---|---|---|
| **NautilusTrader** | **28,552★，当天仍在 push，v2.0.0rc4（2026-09-02），LGPL-3.0** | Rust 核心 + Python 控制面；**backtest / sandbox / live 共享同一个 `NautilusKernel`**；**Event Sourcing**（消息总线边界捕获、per-run seq、cache 是投影不是真相源）；**Deterministic Simulation Testing**（seed 可重放，pre-commit hook 强制）；**Execution reconciliation**；**已有 dYdX / Hyperliquid / Lighter / Deribit 适配器** |

**它的 `concepts/` 文档（architecture、event_sourcing、deterministic_simulation_testing、execution_reconciliation、backtesting）是目前能找到的最详尽的现代交易研究平台架构公开文档**，而且它自曝短板（单进程不能并发多 node、分配器占热路径近一半时间）。

**已知代价**：LGPL-3.0（改引擎本身需开源改动）；venue 语义写在 adapter/matching engine 的 Rust 代码里，不是从 spec 生成的。

**结论**：把会计核心 spec 做成 Nautilus 之上的**校验层与 venue 模型**，而不是另起炉灶。你要新增的那一层——"venue 语义由 spec 声明并被链上真值校验"——在 LEAN 里是 `BrokerageModel` 的 C# 代码，在 Nautilus 里是 Rust 代码，在 Jane Street 是 OCaml 代码。**所有公开架构里都不存在这一层。这是你的增量。**

其他参考实现：**Percolator**（`aeyakovenko/percolator`，560★，2026-09-04 活跃）——Solana 创始人写的 perp 风险引擎，Kani 机械检验，含 funding 累积、warmup、ADL waterfall。可读的对照物。

**明确不用**：ABIDES（已 ARCHIVED，且只有连续双向拍卖，**无保证金/funding/清算概念**）、JAX-LOB（2023-10 停更，**硬编码 LOBSTER 股票消息格式**，无 funding/保证金/清算，接自定义 venue 等于重写）、FinRL（维护方自陈失修）、Qlib 默认管道（切分前归一化）。

---

## 四、接缝清单（降级后的定位）

**它是结构化假设生成器，输出清单不输出 alpha。每一条必须走完整仿真+回测才算数。**

| # | 接缝类型 | 已验证实例 | **强制零假设** |
|---|---|---|---|
| 1 | Clamp / floor / cap | funding 死区 −4bp~+6bp，46–73% 小时数落入 | 死区内本来就没人交易 |
| 2 | 控制量 vs 计价量失配 | impact notional $6k 决定按 $79M OI 收的 funding → AAVE 34,830× | 维持压制的对抗成本 > 收益（premium 720 个样本/小时 → 要压住大半小时） |
| 3 | 冻结 / 陈旧状态 | HIP-3 休市冻结 external；Chainlink heartbeat 跨链差 1–2 数量级 | **价差只是在给期货曲线/持有成本定价** ← 石油 funding |
| 4 | 强制流 | 清算 20%/30s、ADL、重开强制结算 | 强制流不经过你能触及的场所（10/10 有 **62.6% 走 backstop 不打盘口**）|
| 5 | 阈值离散跳变 | >$100k 整仓→20%；margin tier 边界；重锚定次数用尽 | 跳变点已有人守 |
| 6 | 更新频率失配 | oracle 3s vs CEX 连续；Optimism 领先 Arbitrum 19.4s | 延迟窗口 < 你的执行延迟 |

**两条纪律**：
1. **候选在零假设被证伪前不进策略层。** 这条规则的价值已被本次探究验证——石油永续 −24.97%/年 极其诱人，但零假设「只是在给 backwardation 定价」**尚未被证伪**。
2. **按证伪成本排序，不按预期收益排序。** 吞吐量来自快速淘汰，不是来自押中。

**如果将来真要做自动发现**：抄 **FlashSyn**（ICSE 2024）的路子，不要抄形式化的路子。它**明确放弃了符号执行**（理由：DeFi 动作逻辑"对标准求解器太复杂，即使已知动作序列，朴素符号执行也可能因约束过于复杂导致求解器超时"），改用**黑盒采样 + 多项式回归/最近邻逼近状态转移 + 反例驱动细化**（在 fork 链上跑，实际利润与估计偏差 >5% 就当反例回收重拟合）。结果：18 个 benchmark，**人手写精确 action summary 只合成出 7 个攻击，数值逼近合成出 16 个**。**同一领域内，用真值校准的近似模型打败了精确形式模型。**

---

## 五、price 模块（不变）

两种 price：**p̂**（外生价格在未来 h 的**分布**，给 MPC 做风险约束）和 **v̂**（无市可参考时的公允价，HIP-3 休市期间需要）。

契约：`predict(context, horizon) -> Distribution`。**必须是分布，不是点估计。**

三条铁律：
1. **先量天花板再建模型。** `RMSE(I) = σ·e^{−I}`，把现有模型的 R² 放上去比。差距小 = 信息天花板 = 换模型是浪费。配套算 `T_crit = C_z⁻¹σ²logP/B²`（R²=2–3% 时，P=12,000 需约 31 年数据；P=15 也要约 9 年）。**这把"要不要换模型"从争论变成算术。**
2. **预测器质量与策略表现分开记账。** 必须随时能回答"这次变好是预测变好了还是规划变好了"。
3. **换信息集，不换模型容量。** Alpha158 是纯价量、60 天窗口、无基本面——你在它上面把模型推到 Transformer 停在 IC 0.02–0.05（官方 benchmark 里 Linear 0.0397 打赢 Transformer 0.0264）。链上你有它买不到的：全局持仓与清算图、premium/impact price 机制状态、跨场所 basis、**会话状态**（距重开多久、bounds 剩多少、重锚定用了几次）。

**泄漏探针常驻 CI**：把 forward return 直接当特征喂进管道。学不会 → 管道坏了（FinRL #1248 就是这个结果），任何新算法都是浪费。

---

## 六、策略层（不变）

```
policy = f_θ(structural_skeleton, state)
```
- 骨架是**解析解**（Almgren-Chriss、Avellaneda-Stoikov、会话收敛的规则式解）
- RL / 优化**只调少量参数**（风险厌恶、偏斜、执行比例、报价深度）
- **最坏退化回骨架**，不是退化到发散的网络

证据：Wang/Gao/Li（*Finance and Stochastics* 2026）用 AC 闭式解参数化 → 参数极少 → 有线性收敛证明 → **同时打败 plug-in 经典控制和深度 model-free SAC**；Alpha-AS（PLOS ONE 2022）RL 只调 AS 的风险厌恶与偏斜；Hendricks & Wilcox RL 只调执行比例 → IS 改善 4.8–10.3%。
反面：Spooner（AAMAS 2018）full-state 自由出价 agent **10/10 只股票为负**且训练发散。

**MPC 的实测锚点**（arXiv:2603.28898，Bayforest + Bertsekas，2026-08-24）：求解器 **Clarabel**，**约 1 毫秒/决策**，动作空间 **d=11**，硬件 2× AMD EPYC 7R13；6 个月 NASDAQ level-3 数据、1200 instrument/天；相对 spread-crossing 基准降低 **40–50%** schedule shortfall。作者的设计需求原话："在实盘里我们同时管理数百到数千个订单。如果每次日内决策要好几毫秒，等真正行动时状态信息已经很陈旧了。"

**求解器选型**：先用 `qpsolvers` 在真实问题维度上跑基准；热路径在 Rust 且含方差/锥约束 → **Clarabel**（唯一有同场景实测）；纯 QP 且参数逐 tick 微变 → **OSQP**（warm start + 因子分解缓存 + codegen）；含离散决策 → **HiGHS**。

**硬禁止**：端到端方向预测；LLM 直接决策（Profit Mirage：越过知识截止后 Sharpe 衰减 51–62%）；学习式 sizing；Dreamer/MuZero 式学习型世界模型。

**基线是一等公民**：必须自适应、公平实现、同环境评估。Solovyev(2019) 的唯一严格对比中**自适应 AC 打败了每一个 RL 配置**；Gašperov 综述里 22 篇 RL 做市论文 **22/22 全报正面结果、只有 18% 用了 AS/GLFT 基线**。

---

## 七、验证闸门（不变）

G0 spec 重放一致性 → G1 泄漏探针 → G2 信息天花板 → G3 自适应基线 → G4 跨 ≥10 种子 → **G5 证伪审计** → G6 T2 反应环境 → G7 T3 影子 → G8 小额真钱

**"快"来自不用每次重新设计验证，不是来自跳过验证。**

**G5 是守门员**：把整条流程跑在零可预测性的合成数据上，**应当赚不到钱**。还能赚 → 工作流有泄漏，前面全作废。在缺少长期实盘记录时，这是唯一能替代实盘的证伪机制。

三档环境：**T1 重放**（真实簿+真实 funding+真实清算规则，对手冻结）｜**T2 反应**（注入校准过的响应 agent，校准目标须链上可观测）｜**T3 影子**（实时 listen-only，不需资金，给机械层地面真值）。

---

## 八、状态层

**Hyperliquid 必须自建非验证节点**（`hyperliquid-dex/node`，497★，Apache-2.0）。理由是硬的：公网 API 限速 1200 权重/分钟、`clearinghouseState` 权重 2 → 600 账户/分钟，扫完 4.5 万地址要 **75 分钟/IP**；实测按权益 top 2,498 账户只还原全局 OI 的 **34.9%**。

节点给你：`periodic_abci_states`（每 10,000 块全状态快照）、`compute-l4-snapshots --includeUsers`（**L4 全簿含挂单归属**）、`--serve-info`（本地 info server，官方称可缓解限速）。配置 16 vCPU / 128GB RAM / 500GB SSD，约 100GB 日志/天。

增量：WS `trades`（带 `users:[buyer,seller]`）+ `userEvents.WsLiquidation` + `WsFill.liquidation`（**能区分 market 与 backstop**）。

**状态是事件溯源的**：不维护可变的"当前状态"对象，存事件流，状态是事件流的折叠。任意时点可重建 → T1 重放天然成立；spec 变了可重放验证；实盘出问题可精确重现。（NautilusTrader 已经这么做了，直接用。）

EVM 侧（若扩到 AMM）：**Reth ExEx**（post-execution hook，reorg-aware；Paradigm 称 reorg tracker <20 LoC、indexer <250 LoC，**直插 reth 数据库的 indexer 比走 JSON-RPC 快 1–2 个数量级**）。一次性历史抽取用 **Cryo**（1,581★，但 2025-01 后停滞）。

---

## 九、落地：4 周硬门槛

**单 venue、窄 spec、有明确停止条件。**

| 周 | 内容 |
|---|---|
| **第 0 天** | **立刻开始录 5 秒粒度 L2 快照。** 每晚一天就永久少一天可校验历史 |
| 1 | HL 非验证节点跑起来；接 NautilusTrader 的 Hyperliquid adapter；拉 3 个月历史（每小时 funding rate + 每笔 funding payment + 每个清算事件） |
| 2 | 写会计核心 spec：funding 累积、cross/isolated 保证金、清算触发与价格、ADL。**不写撮合。** Rust + proptest（快）或 Quint（有模型检查 + ITF trace） |
| 3 | 逐字段对账：funding rate 对到多少位小数、每个账户的 funding payment、每个清算的触发时刻与成交价 |
| 4 | 把对账做成每日 CI；同时并行跑 **G1 泄漏探针 + G2 天花板度量**（在你现有 Qlib pipeline 上，只要几天） |

**停止条件（硬）**：4 周内做不到「重放 3 个月历史，funding payment 逐笔对上，清算事件触发区块不差超过 1 个」→ **这条路停**。因为这已经是最简单的 venue、最小的机制子集、最公开的规格。

**成功判据不是"spec 写完了"，是「某天 CI 红了，然后你发现 Hyperliquid 悄悄改了 impact notional」。** 那一刻这套东西才证明了价值——它是唯一能自动检测机制漂移的东西。

**如果三个月里校验从没红过**，说明要么覆盖太浅，要么这个 venue 根本没在变——那这套基建的边际价值就很低，应当重新评估。

之后：P2 接缝清单 + 证伪石油 backwardation → P3 T1 环境 + 基线 + G5 → P4 MPC/结构化 RL + T2 → P5 T3 影子（**必须跨越至少一次高波动事件**）→ P6 小额真钱分层放量。

---

## 十、这套东西会怎么死

| 失败模式 | 缓解 |
|---|---|
| **永远在建基建、从不交易** | **最大的风险。** 4 周硬门槛 + 单 venue + 只写会计核心。DSL 留到后期，先用 Rust/Python 直写 |
| **行情记录精度不够，spec 根本没法校验** | **第 0 天就开始录 5 秒 L2。** 这是不可追补的 |
| **HIP-3 与普通 perp 公式不同导致静默错算** | spec 必须带 market 类型判别；对账覆盖两类市场 |
| **接缝清单全是假阳性** | 强制零假设 + 按证伪成本排序 |
| **可观测性前提消失** | Aster 主打 Hidden Orders + ZK 隐藏仓位，官方定位是防 liquidation hunting。spec 层要能表达"部分可观测"，策略层要能在信息降级时优雅退化 |
| **实测都在清淡日做的** | 所有速率/深度/还原率实测均在低活跃时段。**P5 影子必须跨越至少一次高波动事件**——10/10 那种时刻正是限速和追块最可能垮掉的时候 |

---

## 十一、一句话

**别造通用框架，也别指望 spec 生成仿真器。造一个 300–800 行的会计核心，让它每天被链上真值自动校验一次——这件事没人做过，而且只有链上能做。RL 只在这个被校验过的机械动力学上做有限期规划，价格永远是可替换、独立记账的模块。第 0 天就开始录 5 秒 L2 快照，第 4 周如果对不上账就停。**
