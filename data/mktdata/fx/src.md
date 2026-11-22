# FX Spot Intraday 数据源调研

> 调研日期：2026-11-22
> 范围：**FX spot** 的 intraday（分钟级及以下）数据源，比较 **历史深度** 与 **频率/粒度**。
> 本文档仅为选型调研，尚未落地任何下载代码。

---

## 0. 结论先行

做 FX spot intraday 数据，主流是两条路线，**不要混在一起比较**：

| 路线 | 代表 | 频率 | 深度 | 覆盖 |
|---|---|---|---|---|
| **A. 免费 tick / M1 深历史** | Dukascopy / HistData / TrueFX | tick、S1、M1+ | **2000 / 2003 → 今天** | 主要货币对 + 交叉盘 |
| **B. 有 API key 的分钟级** | OANDA practice / Alpaca / Twelve Data | M1 起（部分 S5） | 实际多为 **1–3 年** | 1000+ 对，宽但浅 |

**核心陷阱**：免费"API"几乎都把 intraday 历史截断了；而免费"下载站"（Dukascopy / HistData）反而有 20+ 年深度，只是要自己下载 + 聚合。
→ **主数据走 A 路线，API 只用来补当天增量或做 sanity check。**

---

## 1. A 路线：免费深度档（推荐作为主数据源）

| 源 | 最早数据 | 频率 | bid/ask | 品种数 | 获取方式 | 注意 |
|---|---|---|---|---|---|---|
| **Dukascopy** `.bi5` | tick 多数对 **2003-05 起**（EURUSD 2003-05、GBPUSD 2003-05-04、AUDJPY 2003-11-29、AUDCAD 2005-12-26、AUDNZD 2006-12-08）；日线回溯到 **1986**（部分工具报 1973） | tick、S1、M1–H1、日/月 | ✅ 真 bid/ask + tick volume | **~1500+**（FX ~300、金属 50+，另有 CFD 股票/指数/加密） | 免费无 key，HTTP 直下 `datafeed.dukascopy.com/datafeed/{symbol}/{YYYY}/{MM}/{DD}/{HH}h_ticks.bi5` | 逐小时文件，需并发下载；各品种起始日不同 |
| **HistData.com** | EURUSD tick **2000** 起 | M1 + tick | ✅ | ~66（含 XAU/XAG、WTI、指数） | 按月 ZIP，无需注册 | **时间戳是 EST 不是 UTC**；完整下载要拼几千个 zip |
| **TrueFX** (Integral) | **2009-05** 起 | tick（bid/ask） | ✅ 银行间实盘 | ~15 主流对 | 免费注册，按月 zip，URL 规律固定 | 覆盖窄，只有 majors |
| **FXAbsolute** | **2021 → 今** | 已聚合 M1 | ❌ 只有 mid | 15 个品种（28.4M bars） | 免费无 key，有 MCP server | 自称继承 Dukascopy 的缺口；**无 spread**，不适合 <10 pip 策略 |
| **QuantConnect Forex** | **2007-01** 起（72 对） | tick → 任意粒度 | ✅（源自 OANDA / FXCM） | 72 对 | 平台免费回测 / LEAN 本地下载 | 取数要过 LEAN，批量拉取较绕 |
| **Scalar Field** | 声称 **2003-01-02** | 日线 + 分钟/小时 | 声称源自 bid/ask | **1750+** | 免费，`C:EURUSD`，Python `scalarlib.getOHLCV()` | ⚠️ 营销页数据，**intraday 实际回溯到哪天需实测** |

**配套社区工具**：`dukascopy-node`（Node CLI，支持 tick/s1/m1 → csv）、`bhd-dukascopy`（查每品种各周期起始日）、`dukascopy-downloader`（Python）、`digdigdig3`（Rust）。

---

## 2. B 路线：API 档（分钟级）

| API | 频率 | 声称深度 | 免费层真相 | 关键门槛 |
|---|---|---|---|---|
| **OANDA v20** | S5 / M1 / M2 / M4 / M5 / M10 / M15 / M30 / H1…（共 21 档） | **2005 起** | 免费 = 免费 practice account，无 API 订阅费 | `/pricing/range` 每请求 **~5000 条上限**；⚠️ 细粒度（S5/M1）实际保留窗口需实测，官方文档未明确 free 账户截断规则 |
| **Alpaca Forex** | 1Min bars + 5Sec / 1Day rates | 官方"到 2016"（股票口径） | Basic：200 req/min、**15 分钟延迟** | `v1beta1/forex/bars`，符号 `EUR/USD`；**forex 是否在 Basic 覆盖范围不明确**，起点也需实测；数据源是 FXCM / OTC |
| **Twelve Data** | 1min 起 | 声称 **20+ 年**，1300+ 对 | 800 credits/day、8 req/min、**禁商用** | 最省事的"拿 key 就用" |
| **Polygon / Massive** | 1min aggs | Starter $29 起才有 intraday | Basic 5 calls/min、**EOD only、~2 年** | 免费层基本拿不到分钟级 |
| **Alpha Vantage** `FX_INTRADAY` | 1 / 5 / 15 / 30 / 60min | 20 年（指日线） | **25 req/day**，intraday 只给近期 compact 数据 | 完整 intraday 历史要 premium |
| **iTick** | 分钟级 | 2009 起 | Basic 5 calls/min、**仅 2 年历史** | 要 10 年分钟级得买 Starter |
| **FMP** forex intraday | 1min / 5min | intraday 深度有限、偏实时 | 需 key | 不适合做 10 年分钟级回测 |
| **Trading Economics** | 日内 OHLC（单交易日内） | — | 试用 100k points / 100 req | **5 symbols/call、10k rows/call**，~$199/mo 起；**完全没有 tick** |

### 只有日线 / 参考价（做基准和长历史校验，别拿来回测）

- **Frankfurter** — 日线回溯到 1948，免费无 key
- **FRED** — 日线，宏观/汇率参考
- **FxFeed** — 免费层含 1999 起历史 + 任意 base + 小时级
- **Open Exchange Rates** — 1999 起历史**在付费层**（~$12/mo），免费层仅当前汇率
- **exchangerate.dev** — 1999 起历史在 Plus/Pro（$39/mo）
- **Fixer** — 1999 起，Basic $14.99/mo

### 机构 / 付费 tick

LSEG / Bloomberg B-PIPE、ICE、Refinitiv DataScope；或 Massive $29–199/mo。

---

## 3. 会毁掉回测的坑

1. **mid-price vs bid/ask**：FXAbsolute、Scalar、Twelve Data、OANDA 大多只给 mid OHLC，**没有 spread**。Dukascopy / TrueFX / HistData 有真 bid/ask。做 <5 pip 策略必须用后者。
2. **MT4/MT5 的 "tick data" 大多是按 M1 插值合成的**，不可信。FXAbsolute 的 M1 同样是继承 Dukascopy 的缺口。
3. **时区与 bar 边界**：HistData 用 **EST**（不是 EDT），Dukascopy 用 UTC；各家 1 分钟 bar 归属哪一分钟也不同。混源前必须统一到 UTC 并对齐边界。
4. **"spot" 的口径**：Alpaca / QuantConnect 源自 FXCM、OANDA 的 OTC feed，与 ECN 真实 interbank 在小币对、节假日 gap 上有差异。Dukascopy / TrueFX 更接近 interbank。
5. **FX 没有真实 volume**：Dukascopy 的 volume 是 tick count（可用），其他源的 OHLCV 里 V 基本无意义。
6. **免费层的 history 截断是隐形的**（Alpha Vantage / iTick / Polygon 最典型），不要相信标注的 "20 年历史"。

---

## 4. 建议的落地路线（针对 `data/mktdata/fx/`）

- **主仓库**：Dukascopy tick → 落 Parquet / Polars，本地聚合成 M1/M5/…，2003 至今，带 bid/ask。建议目录结构 `data/mktdata/fx/dukascopy/{symbol}/{timeframe}/`。
- **加速 + 交叉校验**：HistData 的 M1（EURUSD 从 2000）直接下，别全量重算；FXAbsolute 的 2021+ M1 当 benchmark 比对。
- **日线长历史**：Dukascopy d1，或 FRED / Frankfurter。
- **当天增量 & 实时**：OANDA practice（或 Alpaca）拉当天 + live quote 增量落盘，形成 daily incremental pipeline。
- **原型期**：Scalar Field / Twelve Data 拿 key 就跑，别先做下载器。

---

## 5. 待拍板 / 待实测

### 需要拍板
1. **要不要 bid/ask**？
   - 要 → 只能走 Dukascopy / TrueFX / HistData 自己建库
   - 不要 → 直接用 Twelve Data / Scalar Field / OANDA，省 ~90% 工程量
2. **需要多少品种**？7 个 majors 还是 300 个交叉盘？
   - 直接决定下载量（Dukascopy 全品种全历史 tick = TB 级；major-only M1 = GB 级）

### 需要实测（本机 WebFetch 被网络策略拦截，以下厂商口径未能直连验证）

建议先花半天做三个探针：
1. 拉 Dukascopy EURUSD 2003 年某个交易日的 tick，确认 `.bi5` 可达性与起始日
2. 拉 Scalar Field 最早可得的分钟数据，确认 intraday 真实深度
3. 用 OANDA free account 拉最早 M1，确认细粒度保留窗口

预期需要校准的点：
- Scalar Field 的 intraday 起始日期
- OANDA free/practice 账户的 S5/M1 历史保留窗口
- Alpaca forex 分钟线的实际起始日期
- QuantConnect forex 各品种的起始日

---

## 参考链接

**免费深度档**
- [dukascopy-node — Historical market tick and OHLC Data Download Tool](https://www.dukascopy-node.app/)
- [Downloading tick data — dukascopy-node](http://www.dukascopy-node.app/downloading-tick-data)
- [Dukascopy DataFeed Guide — TheoryCraft](https://theorycraft-trading.ai/en/docs/guides/dukascopy)
- [HistData.com — Free Forex Historical Data](https://www.histdata.com/?area=downloads)
- [HistData F.A.Q.](https://histdata.com/f-a-q/)
- [philipperemy/FX-1-Minute-Data](https://github.com/philipperemy/FX-1-Minute-Data)
- [TrueFX tick data（镜像页）](https://wwww.spreadsheet1.com/truefx.html)
- [FXAbsolute — Where to Get Free M1 Forex Data](https://fxabsolute.com/where-to-get-free-m1-forex-data)
- [FXAbsolute — Free Forex Candle Data (M1 OHLC)](https://fxabsolute.com/free-forex-candle-data-download)
- [FXAbsolute — MCP Server for Forex Data](https://fxabsolute.com/mcp-server-forex-data)
- [Scalar Field — Forex OHLCV](https://scalarfield.io/docs/market-data/forex-ohlcv)
- [QuantConnect — Forex dataset](https://www.quantconnect.com/docs/v2/cloud-platform/datasets/quantconnect/forex)
- [Where can someone get free high frequency tick forex data? — Quant SE](https://quant.stackexchange.com/questions/34919/where-can-someone-get-free-or-very-cheap-high-frequency-tick-forex-data)

**API 档**
- [OANDA — REST API（历史定价回溯至 2005）](https://www.oanda.com/au-en/trading/platforms/rest-api/)
- [OANDA Pricing API reference](https://apis.io/apis/oanda/oanda-pricing-api)
- [Alpaca — Historical rates for currency pairs](https://docs.alpaca.markets/reference/rates-1)
- [Alpaca Data Timeline support](https://alpaca.markets/support/alpaca-data-timeline)
- [Twelve Data — Free Forex API](https://twelvedata.com/forex)
- [Polygon.io Free Tier Limits](https://aifinhub.io/articles/polygon-io-free-tier-limits)
- [Alpha Vantage API Documentation](https://www.alphavantage.co/documentation)
- [Trading Economics — Markets Intraday Data](https://docs.tradingeconomics.com/markets/intraday)
- [Trading Economics — API Rate Limits](https://docs.tradingeconomics.com/get_started/rate-limits/)
- [FMP — Forex Intraday 1 Min](https://financialmodelingprep.com/developer/docs/stable/forex-intraday-1-min)
- [Open Exchange Rates vs XE](https://exchangerate.dev/compare/openexchangerates-vs-xe)
