# Discord 信号数据接口契约

**版本：v0.3**（2026-08-30 定稿）
**仓库**：github.com/yu3y4osun-netizen/discord-signals
**消费方**：量化平台（回测 / 模拟盘）
**生产方**：Discord 抓取管道（增量抓取 → 解析 → 推送）

---

## 1. 总览

```
[Discord 4 个频道]
      ↓ 定时抓取（增量，按 message_id 翻页）
[解析为结构化信号 JSON]
      ↓ 逐条推送（一个消息 = 一个文件）
[本仓库 signals/YYYY/MM/DD/ 目录]
      ↓ 消费方 git pull（建议 5~30 分钟一次）
[量化平台：按 message_id 去重入库]
```

### 目录结构

```
signals/
  2026/
    08/
      28/
        zhao_short_1542900000000000001.json      # 赵哥短线·trade
        mgstock_p0_1542900000000000002.json      # 美股投资网·news/trade
        lyc_trend_1542900000000000003.json       # 洛阳铲·market_view
        zhao_discussion_1542900000000000004.json # 讨论区·community
_heartbeat.json                                   # 心跳：各频道最后抓取进度
CONTRACT.md                                       # 本文档
```

- 文件名 = `<channel_key>_<message_id>.json`，**message_id 全局唯一，重复拉取直接跳过（幂等）**
- 日期目录用消息的 **UTC 日期**（Discord 原始 timestamp 即 UTC）

### 频道与 signal_type 对应表

| channel_key | 频道 | signal_type | source_profile | 说明 |
|---|---|---|---|---|
| `zhao_short` | zhao赵哥-股票-短线-p0 | `trade` / `market_view` | `daily` | 60% 实盘操作 + 40% 盘面解说（解说归 market_view） |
| `mgstock_p0` | 美股投资网-p0 | `news` / `trade` | `swing` | 98% 个股新闻 + 2%「波段机会提示」（归 trade） |
| `lyc_trend` | 洛阳铲a港美投资-核心趋势-p0 | `market_view` | — | 每日复盘 / 问答 / 赛道长文 |
| `zhao_discussion` | 赵哥讨论区 | `community` | — | 群友情绪（**过滤后**才推送） |

---

## 2. 统一信封（所有信号文件共用）

| 字段 | 类型 | 必填 | 含义 |
|---|---|---|---|
| `schema_version` | string | ✓ | 契约版本，当前 `"0.3"`。消费方应校验版本，遇不识别版本先看本文档变更记录 |
| `source` | string | ✓ | 固定 `"discord"` |
| `channel_key` | string | ✓ | 频道机器码，见上表 |
| `channel_name` | string | ✓ | 频道人读名 |
| `source_profile` | string | — | **渠道频率档位**（仅 trade/news 渠道携带）：`"daily"`（日内~天频，赵哥）/ `"swing"`（波段/周月频，美股投资网）。market_view/community 类为 null |
| `message_id` | string | ✓ | Discord 消息 ID。**幂等去重键**，同时是文件名的一部分 |
| `author` | string | ✓ | 发言者用户名。注意：`xiaowanglucky` 和 `halandhaha` 是同一个人（Discord ID 1529064430128857139） |
| `ts` | string | ✓ | 消息发出时间，UTC，ISO 8601（Discord 原始值） |
| `captured_at` | string | ✓ | 管道抓取时间，ISO 8601 带时区偏移（+08:00）。`ts` 与 `captured_at` 的差值 = 数据延迟 |
| `signal_type` | string | ✓ | `trade` / `news` / `market_view` / `community`（`analysis` 见第 9 节，规划中） |
| `payload` | object | ✓ | 按类型定义，见下文各节 |
| `raw_text` | string | ✓ | 消息原文（embed 消息取 description），截断至 2000 字符 |
| `parse_confidence` | string | — | 解析置信度：`"high"`（规则化格式）/ `"medium"`（LLM/规则推断）/ `"low"`（仅粗粒度结构化）。低置信度时以 raw_text 为准 |

**时区约定**：`ts` 一律 UTC；`captured_at` 带 +08:00；交易日归组消费方自定。

---

## 3. trade 类（交易信号）

### 触发来源
1. `zhao_short` 频道的实盘操作（格式规整：「111.7加了6分之一常规仓的soxl」）
2. `mgstock_p0` 频道的「波段机会提示」（含完整开仓计划）

### payload 字段

| 字段 | 类型 | 含义 | zhao_short | 波段提示 |
|---|---|---|---|---|
| `plan_type` | string | `"incremental"` 持仓过程流 / `"full_plan"` 完整开仓计划 | incremental | full_plan |
| `ticker` | string | 标的代码（统一大写，如 `SOXL`） | ✓ | ✓ |
| `action` | string | 动作枚举，见下表 | ✓ | 通常 `buy_open` |
| `price` | number\|null | 参考成交价 | ✓ | ✓ |
| `price_note` | string\|null | 价格附注（如「市价买入，别设限价单」） | 少量 | ✓ |
| `size_hint` | string\|null | 仓位提示（如 `"1/6"`、`"25%"`），未提及为 null | ✓ | ✓ |
| `target_price` | number\|null | 目标价 | **null** | ✓ |
| `horizon` | string\|null | 持有周期（`"weeks"` / `"months"`） | **null** | ✓ |
| `exit_rule` | string\|null | 退出纪律原文 | **null** | ✓ |
| `rationale` | string\|null | 操作理由（消息中明确陈述的逻辑） | 有时携带 | 无 |

### action 枚举（8 值）

| 值 | 含义 | 判定标准 | 真实示例 |
|---|---|---|---|
| `buy_open` | 首次建仓买入 | 该标的近期无持仓语境，明确「买入/建仓」 | 波段提示「BJ 市价买入」 |
| `buy_add` | 已有持仓基础上加仓 | 明确「加/补」+ 此前有持仓 | 「111.7 加了 6 分之一常规仓的 soxl」 |
| `buy_back` | 卖出后买回同一标的 | 明确「买回之前卖出的」 | 「WDC 460 买回之前 471 卖出的」 |
| `sell_partial` | 部分减仓/部分止盈 | 「出一半」「出一部分」「止盈」 | 「MU 等尾盘拉升的位置再出」 |
| `sell_close` | 全部清仓该标的 | 「清仓」「全出」 | — |
| `stop_loss` | 止损离场 | 因亏损/破位明确卖出 | — |
| `hold` | 明确表态继续持有不动 | 「拿着不动」「不卖」 | — |
| `mention` | 仅提及标的，无动作指令 | 谈论标的但无买卖 | 「soxl 今天真强」 |

### ⚠️ null 语义与回测配对规则（重要）

- **null = 本条消息不携带该信息**（≠ 0 ≠ 不存在）
- `plan_type: full_plan`（波段提示）：**开仓信号自带完整生命周期**。回测逻辑：开仓 → 等 `exit_rule` 触发或后续同标的 trade 信号。单文件即可回测
- `plan_type: incremental`（赵哥操作）：**退出规则是隐式的**——由后续消息流构成（他后面发的 `sell_partial` / `sell_close` 就是退出动作本身）。回测必须按 ticker 串联时间序列配对（`buy_add` → … → `sell_partial`/`sell_close`），盈亏 = 各笔价差 × 仓位节奏。**单条信号不可独立回测**
- 两类信号的配对引擎必须是两套，`plan_type` 就是分流开关

---

## 4. news 类（个股新闻）

**来源**：`mgstock_p0` 日常新闻流（占该频道 98%，约每半小时一条）。

| 字段 | 类型 | 含义 |
|---|---|---|
| `ticker` | string | 个股代码（消息开头的大写代码） |
| `headline` | string | 新闻标题/首句摘要（≤100 字） |
| `sentiment` | number | `-1`（利空，如「BNTX -7% 终止试验」）/ `+1`（利好，如「GAP +13% 上调指引」）/ `0`（中性）。由消息内涨跌幅符号和语义共同判定 |
| `summary` | string | 完整摘要（≤300 字） |

**用途建议**：回测引擎直接消费价值有限，但适合做**事件驱动因子、个股情绪聚合、新闻流密度指标**。

---

## 5. market_view 类（趋势观点）

**来源**：`lyc_trend` 全部内容 + `zhao_short` 的盘面解说（无明确操作的那部分）。

| 字段 | 类型 | 含义 |
|---|---|---|
| `subtype` | string | `daily_review` 每日复盘 / `qa` 问答栏目 / `thesis` 赛道方法论长文 / `commentary` 盘中短评 |
| `markets` | string[] | 覆盖市场：`["A股","港股","美股"]`，可多值（洛阳铲单条复盘常同时覆盖 A股+港股+美股+黄金+币） |
| `direction` | string | 观点方向（见下枚举） |
| `key_levels` | array | 关键点位，每项 `{"index":"上证","level":4000,"level_type":"resistance|support|support_lost|level","note":"..."}` |
| `sectors_favored` | string[] | 看好板块（如贵金属/农业/化工/创新药） |
| `sectors_avoided` | string[] | 回避/提示风险的板块（如高位半导体设备/存储/光模块/PCB） |
| `risk_notes` | string[] | 风险提示条目（原文摘录） |
| `summary` | string | 100~200 字中文总结（管道捕获时生成，非原文截断） |

### direction 枚举（5 值）

| 值 | 含义 | 典型信号词 |
|---|---|---|
| `bullish` | 看涨 | 明确看多、逢低做多 |
| `bearish` | 看跌 | 明确看空、清仓避险 |
| `cautious_bullish` | 谨慎看多 | 看好但提示波动/仓位控制 |
| `cautious_bearish` | 谨慎看空 | 高位回吐提示、以防守为主（洛阳铲 8/28 复盘即此） |
| `neutral` | 震荡观望 | 观望、等待落地 |

**回测价值**：观点本身不可直接回测，但可做①择时对照基准（观点 vs 后续实际走势的命中率）②与 trade 信号的交叉分析（见第 9 节）。

---

## 6. community 类（群友情绪）

**来源**：`zhao_discussion`（赵哥讨论区），**过滤后推送**。

**过滤规则（进管道需满足其一）**：①提及具体股票/ETF/币种 ②讨论赵哥的某个操作（跟单/反馈/成本）③有明显情绪表达。寒暄/表情包/灌水跳过（预计通过率 30~40%）。

### payload 字段

| 字段 | 类型 | 含义 |
|---|---|---|
| `mood` | string | 发言时的主导情绪，6 值枚举（见下表） |
| `mood_strength` | number | 情绪强度 1~3：`1` 顺带一提 / `2` 明确表达 / `3` 强烈（连续感叹号、大段宣泄） |
| `tickers_mentioned` | string[] | 消息提及的所有代码（大写），无则 `[]` |
| `is_following_zhao` | boolean | `true` = 描述**自己跟单赵哥的实际行为或结果**（「我在 111.5 也跟了 1/6」）；`false` = 仅讨论/情绪，无跟单行为 |
| `summary` | string | 一句话概括（≤50 字） |

### mood 枚举（6 值）

| 值 | 含义 | 典型发言 |
|---|---|---|
| `fomo` | 踏空/追高焦虑 | 「赵哥 soxl 还能上车吗」「刚才没跟到拍大腿」 |
| `fear` | 恐慌 | 「这跌法要不要全跑」「破位了完了」 |
| `confusion` | 困惑求助 | 「1/6 仓是什么意思」「为什么买了又卖」 |
| `confidence` | 笃定/得意 | 「拿稳了坐等起飞」「上次跟这单赚了 8%」 |
| `greed` | 激进加仓欲望 | 「想满仓干了」「借钱梭哈」 |
| `neutral` | 无情绪的客观讨论 | 问技术问题、报价格、贴数据 |

**聚合建议**：单条价值低，按日聚合成「跟风盘情绪指标」价值高（当日 mood 分布 + mood_strength 均值 + is_following_zhao 计数）。可验证「群友最 fomo 时是否是赵哥的出货点」。

---

## 7. 心跳文件 `_heartbeat.json`（仓库根目录）

```json
{
  "updated_at": "2026-08-30T12:30:00+08:00",
  "channels": {
    "zhao_short": {"channel_id": "1460592890525913201", "last_message_id": "...", "last_fetch_at": "..."},
    "mgstock_p0": {"channel_id": "1461005584064184503", "last_message_id": "...", "last_fetch_at": "..."},
    "lyc_trend": {"channel_id": "1516647144935784602", "last_message_id": "...", "last_fetch_at": "..."},
    "zhao_discussion": {"channel_id": "1529062583364223008", "last_message_id": "...", "last_fetch_at": "..."}
  }
}
```

消费方据此判断**数据新鲜度**（updated_at 超过 24h 未更新 = 生产管道异常，勿当作实时数据使用）。

---

## 8. 消费方接入要点

1. **拉取**：`git pull`（每 5~30 分钟一次）；仓库当前为公开，后续可能转私有（转私有后需配只读凭据）
2. **去重**：入库主键用 `message_id`，天然幂等
3. **增量**：只处理 `_heartbeat.json` 的 `last_message_id` 之后的新文件，或直接全量 diff 文件列表
4. **变更通知**：schema 升级会改 `schema_version` + 本文档版本记录，消费前先校验

---

## 9. analysis 类（衍生分析）——**规划中，当前未启用**

> 与原始信号严格分离：所有机器加工的分析独立成类，payload 带 `derived_by: "pipeline"` 与 `derived_from`（引用源消息 id），消费方可整体忽略，不污染原始数据源。
> **纪律：analysis 只使用生成时点之前的数据（防前视偏差）**。计划主链路（四类原始信号）稳定运行两周后启用。

规划中的子类型：

| analysis_type | 内容 | 价值 |
|---|---|---|
| `trade_vs_view` | 每条 trade 信号回看 24h 内最新 market_view，输出 `alignment: aligned/contrarian/neutral/unverifiable` + rationale | 回答「赵哥逆势/顺势加仓哪种胜率高」 |
| `view_conflict` | 两个渠道观点直接冲突时生成 | KOL 分歧度因子 |
| `community_sentiment_daily` | 讨论区按日聚合情绪 | 跟风盘温度计 |
| `trade_echo` | 赵哥操作后 24h 内群友跟单行为聚合（按 ticker 关联） | 赵哥市场影响力半径 |

---

## 10. 版本记录

| 版本 | 日期 | 变更 |
|---|---|---|
| v0.1 | 2026-08-30 | 初版：统一信封 + 三频道 |
| v0.2 | 2026-08-30 | 四类 signal_type（trade/news/market_view/community）；trade 增 plan_type |
| v0.3 | 2026-08-30 | ①trade 增 `source_profile`（swing/daily 渠道频率档位）②market_view 增 `sectors_avoided`/`risk_notes`/summary 生成式总结、`market`→`markets` 数组 ③community 增 `mood_strength`（1~3）④analysis 类立项（规划中）⑤null 语义与回测配对规则成文 ⑥action 8 值枚举释义定稿 |

---

## 附：合规边界

数据来源于付费社群，**仅限数据提供方与其协作者的个人研究使用**，不得对外分发、转售或用于对外服务。仓库当前临时公开（数据初期不敏感），正式运行后转私有。
