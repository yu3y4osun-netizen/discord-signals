# Discord 信号数据接口契约 v0.2

> 数据提供方：leo（Discord 抓取管道，WorkBuddy automation）
> 数据消费方：量化平台（本地部署）
> 状态：v0.2 已按四类信号重构，待消费方确认后冻结

## 1. 总体架构

- 提供方定时抓取 Discord 付费群 4 个频道，解析为结构化信号（+原文），以 JSON 文件形式写入本仓库 `signals/` 目录
- 消费方定时 `git pull`（当前仓库为 public，克隆无需凭证），按文件路径/`message_id` 去重后入库
- 双方互不依赖对方在线；数据延迟 ≤ 抓取周期 + 拉取周期（约 10~60 分钟）
- ⚠️ 仓库当前为 public（试运行期），跑通后将改回 private 并邀请消费方为协作者——**拉取端请同时支持匿名 clone 与凭证 clone**

## 2. 文件组织

```
signals/
  {YYYY}/{MM}/{DD}/            # 日期 = 消息 UTC 日期
    {channel_key}_{message_id}.json
_heartbeat.json                # 各频道最后成功抓取状态
```

- 文件名即幂等键，消费方按路径去重
- UTF-8，单文件单个 JSON 对象

## 3. 统一信封

```json
{
  "schema_version": "0.2",
  "message_id": "1543123805310816359",
  "channel_key": "zhao_short",
  "channel_name": "zhao赵哥-股票-短线-p0",
  "author": "zhao",
  "ts": "2026-08-28T16:47:35.187000+00:00",
  "captured_at": "2026-08-30T12:00:00+08:00",
  "signal_type": "trade",
  "payload": { },
  "raw_text": "**[美股]** 111.7加了6分之一常规仓的soxl~"
}
```

| 字段 | 说明 |
|---|---|
| schema_version | "0.2"；字段变更前升版本 |
| message_id | Discord 消息 ID，全局唯一幂等键 |
| channel_key | 见 §4 频道表 |
| ts / captured_at | 消息原始时间（UTC）/ 抓取时间 |
| signal_type | `trade` / `news` / `market_view` / `community` 四选一 |
| payload | 按 signal_type 变化，见 §5 |
| raw_text | 原文，消费方可二次加工 |

## 4. 频道与信号类型

| channel_key | 频道 | signal_type | 推送策略 |
|---|---|---|---|
| `zhao_short` | zhao赵哥-股票-短线-p0 | `trade`（操作）/ `market_view`（解说） | 全量推 |
| `usstock_news` | 美股投资网-p0 | `trade`（波段机会提示）/ `news`（个股新闻） | 全量推 |
| `lyc_trend` | 洛阳铲a港美投资-核心趋势-p0 | `market_view` | 全量推 |
| `zhao_discussion` | #🔴赵哥讨论区 | `community` | **过滤后推**：仅含个股/操作/明显情绪的消息，寒暄跳过 |

## 5. payload 定义

### 5.1 trade（交易信号）

赵哥实盘操作（`plan_type: "incremental"`）：

```json
{
  "plan_type": "incremental",
  "ticker": "SOXL", "market": "US",
  "action": "buy_add",
  "price": 111.7,
  "size_hint": "1/6", "size_hint_raw": "6分之一常规仓"
}
```

「波段机会提示」完整计划（`plan_type: "full_plan"`）：

```json
{
  "plan_type": "full_plan",
  "ticker": "BJ", "market": "US",
  "action": "buy_open",
  "price": 93.4, "price_note": "市价直接买入，别设限价，不一定买得到",
  "size_hint": "25%",
  "target_price": 110,
  "horizon": "weeks",
  "exit_rule": "到了目标价也必须等通知再卖"
}
```

`action` 枚举：`buy_open`（首次买入）/ `buy_add`（加仓）/ `buy_back`（卖后买回）/ `sell_reduce`（减仓/出一半）/ `sell_close`（清仓止盈）/ `stop_loss`（止损）/ `watch`（关注观察）/ `other`。

**回测语义提示**：`incremental` 是持仓过程流（配对需按 ticker 累积），`full_plan` 是开仓信号 + 退出规则（出场依赖后续通知类消息配对）。

### 5.2 news（个股新闻）

```json
{
  "ticker": "AMZN",
  "headline": "亚马逊签北欧风电长期购电合约保AI数据中心能源",
  "sentiment": 1,
  "summary": "持续加码清洁能源采购支撑云计算扩张，七巨头中走出领先；算力扩建资本开支压力需留意。"
}
```

`sentiment`：`1`（利多）/ `0`（中性）/ `-1`（利空）。

### 5.3 market_view（趋势观点）

```json
{
  "subtype": "daily_review",
  "market": "A股",
  "direction": "cautious_bearish",
  "key_levels": [
    {"index": "上证", "value": 4000, "kind": "resistance"},
    {"index": "创业板", "value": "5日线", "kind": "support_lost"}
  ],
  "sectors_favored": ["贵金属", "农业", "化工"],
  "sectors_avoided": [],
  "summary": "创业板跌破5日线，下周初收不回则本轮科技股反弹有结束风险。"
}
```

- `subtype`：`daily_review`（每日复盘）/ `qa`（[问][答]栏目）/ `thesis`（赛道/方法论长文）
- `direction`：`bullish` / `bearish` / `cautious_bullish` / `cautious_bearish` / `range`
- `qa` 子类型额外含 `question` / `answer` / `topic_tags` 字段
- 盘面解说类（来自 zhao_short 频道的非操作消息）`subtype` 用 `intraday_comment`

### 5.4 community（群友情绪，仅讨论区）

```json
{
  "mood": "fomo",
  "tickers_mentioned": ["SOXL", "MU"],
  "is_following_zhao": true,
  "summary": "群友在赵哥买SOXL时跟单（917成本），并反馈MU尾盘拉高出货模式。"
}
```

`mood` 枚举：`fomo` / `fear` / `confusion` / `confidence` / `greed` / `neutral`。`is_following_zhao` 标记是否为跟单行为反馈。

## 6. 可靠性与约定

- **幂等**：文件名含 message_id；重复拉取由消费方去重
- **顺序**：不保证推送顺序，消费方按 `ts` 排序
- **补发**：解析规则升级导致漏发时按 `schema_version` 重发，文件名加 `.r1` 后缀
- **心跳**：`_heartbeat.json` 含每个频道最后成功抓取时间与 last_message_id，用于判断数据新鲜度
- **合规**：内容来自付费 Discord 群，仅限双方个人量化研究使用，不得对外分发或商业化（仓库 public 为试运行临时状态）

## 7. 历史回填（待消费方确认）

提供方可一次性导出并解析历史原始数据（8 月中旬至今）：
- 赵哥讨论区：8/19~8/25 全量 + 8/25~8/29 缺口段
- zhao短线 / 洛阳铲 / 美股投资网：可按频道深度回填（Discord API 按频道可翻页至建频道起）

## 8. 待消费方确认的问题

1. `trade` 的 action 枚举是否满足回测需要？是否补期权字段（到期日/行权价）？
2. 拉取频率建议 5~30 分钟一次（`git pull` 即可）
3. 历史回填的范围（回填到多早？）
