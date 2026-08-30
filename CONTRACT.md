# Discord 信号数据接口契约 v0.1

> 数据提供方：leo（Discord 抓取管道）
> 数据消费方：量化平台（本地部署）
> 状态：草案，待消费方确认后冻结字段

## 1. 总体架构

- 提供方定时抓取 Discord 付费群频道，解析为结构化信号（+原文），以 JSON 文件形式写入中转存储
- 消费方定时拉取中转存储的增量文件，按 `message_id` 去重后入库
- 双方互不依赖对方在线，无实时性要求（数据延迟 ≤ 抓取周期 + 拉取周期，约 10~60 分钟）

**中转存储二选一（待定）**：
- 方案 A：GitHub 私有仓库（推荐，免费，天然版本化）
- 方案 B：腾讯云 COS bucket（私有读写，SDK 列举增量）

## 2. 文件组织

```
signals/
  {YYYY}/{MM}/{DD}/
    {channel_key}_{message_id}.json     # 一个 Discord 消息 = 一个文件
    {channel_key}_{message_id}.json
    ...
```

- 文件名即幂等键，消费方按路径去重
- 文件 UTF-8 编码，JSON Lines 之外的任意多余内容都没有

## 3. 记录 Schema

统一信封，`signals` 数组按频道类型变化：

```json
{
  "schema_version": "0.1",
  "message_id": "1543123805310816359",
  "channel_key": "zhao_short",
  "channel_name": "zhao赵哥-股票-短线-p0",
  "author": "zhao",
  "ts": "2026-08-28T16:47:35.187000+00:00",
  "captured_at": "2026-08-30T02:00:00+08:00",
  "signals": [
    {
      "type": "trade",
      "ticker": "SOXL",
      "market": "US",
      "action": "buy_add",
      "price": 111.7,
      "size_hint": "1/6",
      "size_hint_raw": "6分之一常规仓"
    }
  ],
  "sentiments": [],
  "raw_text": "**[美股]** 111.7加了6分之一常规仓的soxl~"
}
```

### 3.1 字段说明

| 字段 | 类型 | 说明 |
|---|---|---|
| schema_version | string | 当前 "0.1"，字段变更前会升版本号 |
| message_id | string | Discord 消息 ID，全局唯一，幂等键 |
| channel_key | string | 频道标识，见 §3.2 |
| ts | string | 消息原始时间（UTC，ISO 8601） |
| captured_at | string | 本方抓取时间（含时区），用于审计 |
| signals | array | 结构化信号，可为空数组（纯闲聊消息不出文件） |
| sentiments | array | 情绪因子，可为空 |
| raw_text | string | 原文（embed description / message content），消费方可二次加工 |

### 3.2 频道与信号类型

| channel_key | 频道 | 主信号类型 | 说明 |
|---|---|---|---|
| `zhao_short` | zhao赵哥-股票-短线-p0 | `trade` | 实盘操作信号：买入/加仓/减仓/止盈，带价格与仓位提示 |
| `lyc_trend` | 洛阳铲a港美投资-核心趋势-p0 | `viewpoint` | 大盘/板块趋势观点，含关键点位（如上证 4000 压力位） |
| `usstock_news` | 美股投资网-p0 | `event` | 个股新闻事件流（机器人定时推送） |

`trade` 信号的 `action` 枚举：`buy_open`（首次买入）/ `buy_add`（加仓）/ `sell_reduce`（减仓）/ `sell_close`（清仓止盈）/ `stop_loss`（止损）/ `watch`（关注观察）。无法归类时为 `other`，以 `raw_text` 为准。

`viewpoint` / `event` 信号字段：

```json
{
  "type": "viewpoint",
  "scope": "index",
  "target": "SHCOMP",
  "stance": "bearish_short_term",
  "levels": [{"kind": "support", "value": 3850}, {"kind": "resistance", "value": 4000}],
  "horizon": "days"
}
```

字段均可为空，解析置信度低时 `signals` 为空、只留 `raw_text`。

## 4. 拉取方式

### 方案 A：GitHub 私有仓库

- 提供方将文件提交到 `signals/` 目录，消费方被邀请为协作者
- 消费方定时 `git fetch` + `git diff --name-only` 得到增量文件列表后读取
- 建议：每日一个分支/目录，消费方记录上次处理到的 commit

### 方案 B：腾讯云 COS

- bucket 私有读写，消费方持有只读子账号密钥
- 消费方定时 `ListObjects`（按 `signals/YYYY/MM/DD/` 前缀）拉增量，下载新文件

## 5. 可靠性与约定

- **幂等**：文件名含 message_id，重复拉取由消费方去重
- **顺序**：不保证推送顺序，消费方按 `ts` 排序
- **补发**：解析规则升级导致漏发时，提供方按 `schema_version` 重发并在文件名加 `.r1` 后缀
- **数据来源合规**：内容来自付费 Discord 群，仅限双方个人量化研究使用，不得对外分发或商业化
- **故障语义**：某频道当日无文件 = 无新消息或抓取失败（提供方会在 README 维护 `_heartbeat.json`，含每个频道最后成功抓取时间，消费方可据此判断数据新鲜度）

## 6. 待消费方确认的问题

1. 中转存储选 A 还是 B？
2. `trade` 信号的 action 枚举是否满足回测需要？是否要补字段（如期权到期日/行权价——群里会讨论期权）？
3. 拉取频率（建议 5~30 分钟一次）？
4. 是否需要历史回填（提供方可一次性导出 8 月中旬至今的全部原始数据 + 解析结果）？
