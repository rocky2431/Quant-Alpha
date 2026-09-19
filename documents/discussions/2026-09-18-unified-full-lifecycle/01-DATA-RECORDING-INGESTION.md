# 01 录制与接入

> **2026-09-18 当前讨论稿｜按 UNIFIED-PLAN v1 细化｜未批准实施。** [索引](README.md) · [总架构](ARCHITECTURE.md) · [MVP](MVP.md) · 上游：研究问题与 `Candidate` 登记簿 · 下游：[02 清洗与 EnvState](02-CLEANING-ALIGNMENT-ENVSTATE.md)、[04 提取](04-EXTRACTION-SEMANTICS.md)

## 1. 目的与边界

把两轨研究问题转换为具体、可取得、可追溯的数据需求，并从第 0 天开始录制不可追补的原始帧。接入完成的标准是覆盖与使用条件成立，不是接口返回一次成功。录制的原则是先落盘再解析：每一帧原样保存并附本地接收时间，解析出错不丢原帧。

输入为 `Candidate` 两张卡片、交易工具候选、时间与延迟需求、数据预算约束、所有者对主体资格核实的裁定。输出为来源清单、覆盖报告与 `RawRecord`，交给 02 处理；原始内容同时供 04 回查证据。

本环节不做去重、对齐或语义判断。它只保证"当时收到了什么、何时收到、从哪里收到"可以被完整回答。

**第 0 周任务**：核实交易主体的司法辖区与 Hyperliquid 账户资格，未核实前不建立任何真实账户依赖；Polymarket 只进入第二档录制，且以资格核实为前提。

## 2. 对象与契约

### 2.1 `RawRecord`

| 字段 | 含义与限制 |
|---|---|
| `source_id` | 来源标识：`hl_ws`、`hl_node`、`hl_s3`、`hl_info_rest`、`binance_ws`、`tradexyz_docs`、`hl_announcements`、第二档另加 |
| `stream` | 频道或文件类型：`l2Book_fast`、`l2Book_slow`、`trades`、`bbo`、`activeAssetCtx`、`allMids`、`node_fills`、`hip3_oracle_updates`、`raw_book_diffs`、`misc_events`、`asset_ctxs`、`meta_snapshot`、`spec_index_snapshot`、`announcement` |
| `record_id` | 来源侧标识，如 WS 序号、区块高度加索引、文件路径加行号；缺失时用内容哈希并标注 |
| `symbol` / `dex` | 市场标识与所属 DEX；HIP-3 市场保留 `xyz:` 前缀与部署者标识 |
| `payload` | 原始字节或原始 JSON，不做字段重命名 |
| `content_hash` | 原始内容哈希，用于版本判定与去重前置 |
| `event_time` | 来源声明的事件发生时刻，可能为空 |
| `published_at` | 来源对外发布时刻，可能为空 |
| `first_observed_at` | **本系统本地接收时刻**，实时录制必填；历史回填为空并在 `provenance` 说明 |
| `processing_ready_at` | 由 02 或 04 回填，本环节留空 |
| `usable_at` | 由 02 确定，本环节留空 |
| `retrieved_at` | 本次取得数据的时刻；对回填等于下载时刻，不能冒充历史接收时刻 |
| `provenance` | `live_recording` 或 `backfill`；回填注明桶、文件、上传时间 |
| `completeness` | 序号是否连续、是否有缺口、缺口区间 |
| `recorder_version` | 录制程序版本与配置哈希 |

### 2.2 覆盖报告

| 字段 | 含义 |
|---|---|
| `track_id` | A、B 或共享 |
| `stream` 与 `symbol` 覆盖区间 | 起止时间、连续时段、缺段清单 |
| 事件簇计数 | 按 `event_cluster_id` 统计的独立事件簇数，不是 K 线数、不是市场数 |
| 历史与实时差异 | 同一 `stream` 在回填与实时录制中的字段集与语义差异 |
| 延迟可信度 | 实时录制的接收延迟分布；回填标"无接收时间" |
| 成本 | 存储、带宽、S3 requester-pays 账单、IP 数 |

## 3. 具体过程

1. 从两张 `Candidate` 卡片列出必要字段、最低粒度、观察期与允许延迟，说明每个字段对假设的预期贡献。A 轨要 BTC/ETH 原生永续的簿、成交、资金费、oracle；B 轨要选定 `xyz:` 市场的簿、成交、`activeAssetCtx` 中 oracle/mark/external、oracle 更新流与规范索引。
2. 检查历史与实时路径能否提供同一字段与语义。HL 官方 S3 有 L2 快照、asset contexts、节点 fills、区块与非交易事件，约每月上传一次且可能缺失，官方建议自行录制 ✅ [E3 Q1]。历史缺的字段明确为缺失。
3. 启动第一档实时录制。公共 WS 每 IP 最多 10 连接、1000 订阅，订阅上限不因多开连接而解除；2000 条/分是发送限制 ✅ [E3 Q1，Codex 核验]。按标的数量规划 IP 数，每个连接的订阅清单写进 `recorder_version` 配置。
4. 部署 HL 非验证节点，flags 为 `--write-fills --write-hip3-oracle-updates --write-raw-book-diffs --write-misc-events --batch-by-block`；保留每 10,000 块的 abci 状态以便离线计算 L4 快照。节点规格 16 vCPU、128 GB、500 GB SSD，默认约 100 GB 日志/天 ✅。
5. 补拉 S3 `asset_ctxs`、`l2Book`、`node_fills_by_block`。第 1 周内核清 S3 对 `xyz:` 市场的覆盖，这是 B 轨前三个工程日的任务之一；结论写进覆盖报告，未核清前 B 轨不扩展。
6. 录制 Binance BTC/ETH USDT 永续与现货的 `bookTicker`、`aggTrade`、`depth@100ms`，作为价格发现基准。它是参考流，不是交易场所。
7. 每日快照：`perpDexs`、各 dex 的 `meta` 与 `metaAndAssetCtxs`、trade.xyz 规范索引与 changelog、HL 费用页。每次快照记 `content_hash`，内容变化生成新版本，供 03 生成 `spec_version`。
8. 事件与事故台账：HL 公告、trade.xyz changelog、监管与场所事故，按日期落表，供 02 标注制度断点、供 10 做 regime 分组。
9. 生成覆盖报告：事件簇数、连续时段、缺段、粒度、延迟可信度、历史与实时差异、成本。交给 02 与 10。
10. 第二档在预算允许且资格核实后启动：全部 HIP-4 结果市场；Polymarket 元数据、crypto 类市场 WS、Polygon `OrderFilled` 日志、UMA 争议事件；Deribit 期权报价。

## 4. 双轨示例

**轨道 A（假设配置）。** 一台采集机两个 IP，IP-1 订阅 BTC、ETH 原生永续的 `l2Book_fast`、`l2Book_slow`、`trades`、`bbo`、`activeAssetCtx`，IP-2 订阅 Binance 四个流。某帧 `trades` 在 `first_observed_at = 12:00:00.412` 落盘，`event_time` 为来源时间戳 `12:00:00.380`；两者差 32 毫秒进入延迟分布。当日 `metaAndAssetCtxs` 快照哈希与前一日不同，差异是 BTC 最大杠杆从 40 变 25，生成新 `spec_version` 候选交 03。

**轨道 B（假设配置）。** 选定 3 个商品类 `xyz:` 市场。除与 A 相同的簿与成交流外，节点 `hip3_oracle_updates` 记录约每 3 秒一条 oracle/mark/external 三价；trade.xyz 规范索引每日快照记录会话时间、discovery bounds、重锚次数。某周五收盘后 `activeAssetCtx` 的 `external` 字段停止变化，这一状态变化本身是 `RawRecord`，由 02 解释为会话转 internal。S3 覆盖核查结果假设为：`asset_ctxs` 自 2025-10 起含该市场，`l2Book` 缺 2026-03 至 2026-05 三个月，写进覆盖报告并标为 B 轨历史复现的边界。

## 5. 失败观测与处置

| 观测 | 候选原因 | 区分方法 | 返回环节 |
|---|---|---|---|
| WS 帧序号跳跃 | 断连、订阅超限、限速 | 对照连接日志与限额；缺口记为 `gap` 行不静默跳过 | 01 修录制；02 标缺失 |
| 同一 `stream` 回填与实时字段集不同 | 来源格式切换，如 fills 格式 2025-07 切换 🟡 | 覆盖报告列字段差异；分版本解析 | 01、02 |
| 存在 API 但缺必要历史 | S3 缺段、市场晚于归档起点 | `aws s3 ls --request-payer requester` 实测覆盖 | 调整假设声明，不先训练弥补 |
| 回填数据被当作实时可得 | `retrieved_at` 冒充 `first_observed_at` | `provenance` 字段检查 | 02 附延迟假设 |
| 规则页变化未被记录 | 快照频率不足或页面结构变化 | 哈希比对失败告警 | 01；03 漂移分诊 |
| 节点追块落后 | 资源不足、网络、高波动期 | 追块延迟告警；与公网 API 抽样比对 | 01；12 运维降级 |
| 事件簇数远低于预期 | 市场多但同冲击、覆盖短 | 覆盖报告按 `event_cluster_id` 计数 | 10 限定统计结论 |

## 6. 两速 AI 在本环节的权限

| 环节动作 | 慢速推理 Agent | 快速类型化模型 | 数值/确定性程序 | 裁决者 |
|---|---|---|---|---|
| 从假设列数据需求 | 提议字段清单与预期贡献 | — | — | 人工确认加覆盖报告 |
| 录制配置 | 提议订阅分片与 IP 规划 | — | 执行录制、序号检查、落盘 | 帧连续性检查 |
| 规则页快照解析 | 提议解析规则；读公告起草变更摘要 | 可对公告做类型化分类初筛 | 哈希比对、版本生成 | 03 对账 CI |
| 覆盖报告 | 解释缺口原因 | — | 生成计数与区间 | 10 按报告限定结论 |
| 任何账户或密钥 | 不可触及 | 不可触及 | 采集区无密钥 | 权限分区 |

## 7. 分阶段能力与证据

| 阶段 | 本环节实质内容 | 验证与推进条件 |
|---|---|---|
| 最小 MVP（对应 10 月 15 日改约交付） | 第一档录制连续运行；节点部署；S3 补拉与 `xyz:` 覆盖结论；Binance 参考流；每日规则快照；覆盖报告 | 帧带本地接收时间；缺口显式记录；两轨覆盖匹配实际实验声明；成本与 IP 数进预算表 |
| 第二步：针对失败补强 | 若错误来自延迟或缺段，补相应数据；若机制需要新增信息，增加一条有明确假设的来源 | 比较补入前后覆盖与下游增量；无增益来源可停用 |
| 第三步：跨状态与运行验证 | 断流恢复、来源改版、高波动期限速劣化、第二档启动 | 真实观测日志与覆盖报告支持目标时效；不能用插值掩盖断流 |
| 完整能力：可复用研究产品 | 可查询的数据目录、来源版本与可用性记录，按任务选择信息集与预算 | 新任务可说明数据来处与限制；新增场所或付费源须有用途与授权 |

## 8. 依赖与待定

02 决定哪些原始记录在某时点有效并生成 `EnvState`；03 从规则快照生成 `spec_version` 并消费 fills 与 asset contexts 做对账；04 判断文本归属；09 决定必须具备哪些执行字段；10 判断样本能支撑多强的结论。待定项为主体资格核实结果、B 轨市场清单、S3 对 `xyz:` 的实测覆盖、IP 数与存储预算、第二档启动条件。其未定不阻止第一档录制，但不得在实际运行前隐式使用默认值。
