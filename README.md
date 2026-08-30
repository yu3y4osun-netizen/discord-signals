# discord-signals

Discord 付费群抓取的结构化信号仓库（私有，仅限双方个人量化研究使用，不得对外分发）。

## 数据说明

- 接口契约见 [CONTRACT.md](CONTRACT.md)（schema、字段枚举、幂等规则）
- 路径规则：`signals/YYYY/MM/DD/{channel_key}_{message_id}.json`
- 一个 Discord 消息 = 一个文件，文件名即幂等键
- `_heartbeat.json` 记录各频道最后成功抓取时间，用于判断数据新鲜度

## 消费方拉取方式

```bash
git fetch origin
git diff --name-only HEAD origin/main   # 看有哪些新文件
git pull                                # 或直接拉
```

建议每 5~30 分钟拉一次，按 message_id 去重后入库。
