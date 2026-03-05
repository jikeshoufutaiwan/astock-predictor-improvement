# 每日持仓推送服务

> 自动获取最新持仓数据并推送到 QQ

---

## 功能说明

这个定时任务会每天自动：

1. **获取最新持仓数据** - 从 `final_portfolio_*.csv` 读取
2. **计算实时涨跌幅** - 调用腾讯财经 API
3. **生成操作建议** - 基于止损/止盈线
4. **推送到 QQ** - 使用 OpenClaw 的消息推送

---

## 定时任务配置

### 任务 1: 早盘提醒 (09:15)

在开盘前推送今日操作计划：

```json
{
  "action": "add",
  "job": {
    "name": "早盘操作计划",
    "schedule": {
      "kind": "cron",
      "expr": "15 9 * * 1-5",
      "tz": "Asia/Shanghai"
    },
    "sessionTarget": "isolated",
    "wakeMode": "now",
    "deleteAfterRun": false,
    "payload": {
      "kind": "agentTurn",
      "message": "你是一个A股量化交易助手。请执行以下任务并输出结果：\n\n1. 读取最新的持仓文件 (final_portfolio_*.csv)\n2. 获取今日待买入/卖出股票列表\n3. 生成今日操作计划\n\n要求：\n- 输出格式简洁\n- 标注止损止盈价格\n- 提醒集合竞价注意事项",
      "deliver": true,
      "channel": "qqbot",
      "to": "78F3BBFC429738F6B34F8A4942196D51"
    }
  }
}
```

### 任务 2: 午盘总结 (11:35)

中午收盘后推送上午表现：

```json
{
  "action": "add",
  "job": {
    "name": "午盘持仓总结",
    "schedule": {
      "kind": "cron",
      "expr": "35 11 * * 1-5",
      "tz": "Asia/Shanghai"
    },
    "sessionTarget": "isolated",
    "wakeMode": "now",
    "deleteAfterRun": false,
    "payload": {
      "kind": "agentTurn",
      "message": "你是一个A股量化交易助手。请执行午盘总结：\n\n1. 获取持仓股票上午涨跌幅\n2. 计算账户总盈亏\n3. 检查是否有触发止损/止盈\n4. 给出下午操作建议\n\n要求：\n- 用表格展示持仓涨跌\n- 标注需要关注的股票\n- 简洁明了",
      "deliver": true,
      "channel": "qqbot",
      "to": "78F3BBFC429738F6B34F8A4942196D51"
    }
  }
}
```

### 任务 3: 收盘总结 (15:10)

收盘后推送当日总结：

```json
{
  "action": "add",
  "job": {
    "name": "收盘持仓总结",
    "schedule": {
      "kind": "cron",
      "expr": "10 15 * * 1-5",
      "tz": "Asia/Shanghai"
    },
    "sessionTarget": "isolated",
    "wakeMode": "now",
    "deleteAfterRun": false,
    "payload": {
      "kind": "agentTurn",
      "message": "你是一个A股量化交易助手。请执行收盘总结：\n\n1. 获取所有持仓股票今日收盘价和涨跌幅\n2. 计算账户今日盈亏和总盈亏\n3. 更新持仓成本和止损止盈线\n4. 生成明日操作预览\n\n要求：\n- 输出完整的持仓表格\n- 标注今日触发止损/止盈的股票\n- 给出明日关注点",
      "deliver": true,
      "channel": "qqbot",
      "to": "78F3BBFC429738F6B34F8A4942196D51"
    }
  }
}
```

### 任务 4: 晚间分析 (20:00)

晚间生成明日计划：

```json
{
  "action": "add",
  "job": {
    "name": "晚间交易计划",
    "schedule": {
      "kind": "cron",
      "expr": "0 20 * * 1-5",
      "tz": "Asia/Shanghai"
    },
    "sessionTarget": "isolated",
    "wakeMode": "now",
    "deleteAfterRun": false,
    "payload": {
      "kind": "agentTurn",
      "message": "你是一个A股量化交易助手。请生成明日交易计划：\n\n1. 运行 complete_stock_selector_v3.py 选股\n2. 对比当前持仓，决定调仓\n3. 生成明日买入/卖出清单\n4. 更新 final_portfolio 文件\n\n要求：\n- 输出选股结果\n- 说明调仓理由\n- 列出具体操作步骤",
      "deliver": true,
      "channel": "qqbot",
      "to": "78F3BBFC429738F6B34F8A4942196D51"
    }
  }
}
```

---

## 部署步骤

### Step 1: 添加定时任务

使用 OpenClaw 的 cron 工具添加任务：

```bash
# 添加早盘提醒
openclaw cron add --name "早盘操作计划" --expr "15 9 * * 1-5" --tz "Asia/Shanghai"

# 添加午盘总结
openclaw cron add --name "午盘持仓总结" --expr "35 11 * * 1-5" --tz "Asia/Shanghai"

# 添加收盘总结
openclaw cron add --name "收盘持仓总结" --expr "10 15 * * 1-5" --tz "Asia/Shanghai"

# 添加晚间分析
openclaw cron add --name "晚间交易计划" --expr "0 20 * * 1-5" --tz "Asia/Shanghai"
```

### Step 2: 查看任务状态

```bash
openclaw cron list
```

### Step 3: 测试任务

```bash
openclaw cron run "早盘操作计划"
```

---

## 核心数据文件

### 持仓文件位置

```
ml-service/
├── final_portfolio_*.csv          # 最终持仓
├── final_portfolio_complete_*.csv # 完整持仓（含因子）
├── daily_reviews/                 # 每日回顾
│   └── review_YYYYMMDD.csv
└── logs/                          # 日志目录
    └── data_guardian_*.log
```

### 持仓文件格式

```csv
code,name,shares,buy_price,current_price,weight,layer,stop_loss,take_profit,notes
000001,平安银行,100,10.50,10.80,3.25%,核心,-5%,+15%,蓝筹白马
```

---

## 注意事项

1. **交易时间**: 任务只在交易日执行 (周一到周五)
2. **节假日处理**: 需要配合 `china_trading_calendar.py` 判断交易日
3. **数据延迟**: 实时价格可能有 1-2 分钟延迟
4. **止损触发**: 需要人工确认执行

---

## 扩展功能

### 添加企业微信推送

```json
{
  "payload": {
    "kind": "agentTurn",
    "message": "...",
    "deliver": true,
    "channel": "wecom",
    "to": "your_wecom_id"
  }
}
```

### 添加邮件推送

```json
{
  "payload": {
    "kind": "agentTurn",
    "message": "...",
    "deliver": true,
    "channel": "email",
    "to": "your_email@example.com"
  }
}
```

---

*本服务需要 OpenClaw Gateway 保持运行。*
