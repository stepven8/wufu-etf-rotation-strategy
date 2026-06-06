# 五福 ETF 轮动策略说明

本仓库保存从 `五福.txt` 中独立提取出来的“策略3：ETF轮动策略”。原始多策略文件中包含小市值、ETF反弹、ETF轮动、白马攻防等多个模块，本仓库版本只保留 ETF 轮动策略及其运行所需的基础配置、交易函数、记录函数和调度入口。

策略文件：

- `五福_ETF轮动策略.txt`：聚宽 JoinQuant 策略代码，可直接放入聚宽策略编辑器中运行。

## 一句话概括

这是一个“多资产 ETF 动量轮动策略”：每天从预设的大 ETF 池中筛选可交易标的，计算最近 25 个交易日的加权趋势得分，买入得分最高的 1 只 ETF；如果没有合格 ETF，则尝试切换到货币基金 ETF `511880.XSHG` 防御。

## 策略运行环境

该策略使用聚宽 API，核心依赖包括：

- `jqdata`
- `jqfactor`
- `finance`
- `get_price`
- `attribute_history`
- `get_current_data`
- `get_extras`
- `order`
- `run_daily`
- `record`

本地 Python 环境无法直接完成完整回测，因为这些接口由聚宽平台提供。仓库中的语法检查只能验证 Python 代码结构，不代表可在本地独立取得行情数据。

## 交易标的池

策略的动量排名候选池由三部分组成：

1. 固定 ETF 池 `g.etf_pool_3`；
2. 全市场 ETF 中昨日成交金额最大的 5 只；
3. 全市场 ETF 中先按过去 5 个交易日平均成交金额筛选高流动性候选，再从这些候选里选出过去 5 个交易日涨幅最大的 5 只。

三部分会合并去重后一起进入原有动量排名。固定池标的不会被修改、覆盖或删除；每天变化的是昨日成交金额前 5，以及“5 日高成交候选中的 5 日涨幅前 5”。如果动态 ETF 已经存在于固定池中，只保留固定池中的那一份，不重复计算，也不会额外向后补足“新增 ETF 数量”。

如果 `g.enable_dynamic_etf_pool = False`，策略不会获取这两组动态 ETF，最终只使用固定池 `g.etf_pool_3` 进行动量排名。

固定 ETF 池覆盖范围很广，主要分为以下几类：

### 商品与另类资产

- 黄金 ETF：`518880.XSHG`
- 有色 ETF：`159980.XSHE`
- 豆粕 ETF：`159985.XSHE`
- 原油 LOF：`501018.XSHG`
- 白银 LOF：`161226.XSHE`
- 能源化工 ETF：`159981.XSHE`

这类资产用于捕捉商品趋势，与股票宽基通常相关性较低。

### 美国及全球科技宽基

- 纳指、纳指 100、纳指科技、生物科技
- 标普 500、标普消费
- 道琼、美国 50

这类标的用于捕捉海外权益资产和科技成长风格趋势。

### 日本、欧洲、亚太、中东

- 日经 ETF、日经 225 ETF
- 德国 ETF、法国 ETF
- 亚太精选 ETF
- 沙特 ETF
- 中韩半导体 ETF
- 东南亚科技 ETF

这类标的提升全球区域分散度。

### 港股、中概、沪港深

- 港股通互联网
- 恒生科技
- 中概互联网
- 恒生指数
- 港股红利、港股通红利、港股高股息、港股红利低波
- 沪港深 300、沪港深 500

这类标的用于捕捉港股和中概资产阶段性趋势，也兼顾红利低波风格。

### A 股宽基与风格

- 沪深 300、中证 500、上证 50、上证指数
- 创业板、创业板成长
- 科创 50
- 中证 1000、中证 2000、中证 A500
- 红利低波、价值 100、自由现金流

这类标的是 A 股市场内部的主要轮动方向。

### 固收与现金防御

- 可转债 ETF：`511380.XSHG`
- 国债 ETF：`511010.XSHG`
- 城投债 ETF：`511220.XSHG`

这些固定池标的会和两组动态 ETF 一起参与动量排名，不等同于最终防御资产。策略最终防御资产单独设置为 `511880.XSHG`。

## 核心参数总览

| 参数 | 当前值 | 含义 |
| --- | --- | --- |
| `g.portfolio_value_proportion` | `[0, 0, 1, 0]` | 只启用第 3 个策略，即 ETF 轮动，占组合资金 100%。 |
| `g.lookback_days` | `25` | 计算中期动量得分的回看周期。 |
| `g.holdings_num` | `1` | 最终持仓数量，只持有排名最高的 1 只 ETF。 |
| `g.defensive_etf` | `511880.XSHG` | 无合格进攻 ETF 时使用的防御资产。 |
| `g.min_money` | `5000` | 单次调仓差额低于 5000 元时不交易，避免碎片化交易。 |
| `g.loss` | `0.97` | 最近 3 个单日价格比值中，任一天跌破 0.97 即过滤该 ETF。 |
| `g.min_score_threshold` | `0` | 动量得分必须大于 0。 |
| `g.max_score_threshold` | `100.0` | 动量得分必须小于 100。 |
| `g.enable_volume_check` | `True` | 开启放量过热过滤。 |
| `g.volume_lookback` | `5` | 成交量均值回看天数。 |
| `g.volume_threshold` | `2` | 当日分钟成交量累计超过过去 5 日平均成交量 2 倍时视为异常放量。 |
| `g.volume_return_limit` | `1` | 如果异常放量且年化收益率超过 100%，过滤该 ETF。 |
| `g.use_short_momentum_filter` | `True` | 开启短期动量过滤。 |
| `g.short_lookback_days` | `10` | 短期动量回看周期。 |
| `g.short_momentum_threshold` | `0.0` | 10 日年化动量必须不低于 0。 |
| `g.enable_profit_protection` | `True` | 开启盈利保护。 |
| `g.profit_protection_lookback` | `1` | 盈利保护参考最近 1 日最高价。 |
| `g.profit_protection_threshold` | `0.05` | 当前价较最近高点回落 5% 时触发保护。 |
| `g.profit_protection_check_times` | `['11:00']` | 每天 11:00 执行盈利保护检查。 |
| `g.enable_premium_filter` | `True` | 开启 ETF 溢价率过滤。 |
| `g.premium_threshold` | `0.20` | 溢价率超过 20% 时过滤。 |
| `g.rankings_cache` | `{'date': None, 'data': None}` | 当日排名缓存，保证同一天卖出和买入使用同一批排名结果。 |
| `g.enable_dynamic_etf_pool` | `True` | 动态 ETF 总开关。为 `True` 时追加两组动态 ETF；为 `False` 时只使用固定池。 |
| `g.dynamic_yesterday_money_etf_count` | `5` | 取昨日成交金额最大的 5 只全市场 ETF。 |
| `g.dynamic_avg_money_candidate_count` | `50` | 先按过去 5 日平均成交金额选出前 50 只高流动性 ETF。 |
| `g.dynamic_avg_money_days` | `5` | 平均成交金额的计算窗口。 |
| `g.dynamic_return_etf_count` | `5` | 在高流动性 ETF 候选中取过去 5 日涨幅最大的 5 只。 |
| `g.dynamic_return_days` | `5` | 涨幅计算窗口。 |
| `g.dynamic_money_etf_cache` | `{'date': None, 'yesterday': [], 'money_return_5d': []}` | 当日动态 ETF 池缓存。 |

## 每日调度流程

策略通过 `schedule_tasks(context)` 挂载定时任务：

| 时间 | 函数 | 作用 |
| --- | --- | --- |
| 09:00 | `morning_greet` | 打印策略存活确认日志。 |
| 11:00 | `profit_protection_check_172` | 盈利保护检查，符合条件时卖出持仓。 |
| 14:00 | `strategy_3_sell` | 根据当日 ETF 排名执行卖出。 |
| 14:01 | `strategy_3_buy` | 根据当日 ETF 排名执行买入。 |
| 15:01 | `make_record` | 记录 ETF 轮动策略收益。 |
| 15:02 | `print_summary` | 打印持仓表和账户汇总。 |

卖出在 14:00，买入在 14:01，中间间隔 1 分钟。这样做可以先处理不再属于目标池的旧持仓，再用释放出的资金买入新目标。

## 排名缓存机制

函数：`get_cached_rankings(context)`

策略在同一个交易日内只计算一次 ETF 排名：

1. 获取当前日期 `context.current_dt.date()`。
2. 如果 `g.rankings_cache['date']` 不是今天，则重新调用 `get_ranked_etfs_172(context)`。
3. 将排名结果保存到 `g.rankings_cache`。
4. 当天后续卖出、买入复用同一份排名。

这个设计非常关键，因为卖出函数和买入函数只相差 1 分钟。如果两次分别重新计算，可能因为分钟价格、成交量等数据变化导致目标 ETF 不一致，从而出现卖出和买入逻辑互相打架。缓存让当天调仓目标稳定。

动态 ETF 池也按交易日缓存。策略用 `context.previous_date` 作为统计截止日，所以昨日成交金额、过去 5 日平均成交金额和过去 5 日涨幅都不包含当日盘中数据。

## 动态 ETF 池

新增的动态池只改变“哪些 ETF 有资格进入动量排名”，不改变动量评分公式、过滤顺序、买卖时间、盈利保护、防御资产或最终持仓数量。

动态池由 `g.enable_dynamic_etf_pool` 统一控制：

- `True`：固定 ETF 池 + 昨日成交金额前 5 + 过去 5 日均额前 50 中的涨幅前 5；
- `False`：只使用固定 ETF 池，不获取全市场 ETF 列表，也不拉取动态池行情。

### 昨日成交金额前 5

函数通过 `get_all_securities(['etf'], date=context.previous_date)` 获取全市场 ETF，再用日线 `money` 字段统计昨日成交金额：

```python
get_price(
    all_etfs,
    end_date=context.previous_date,
    count=1,
    frequency='daily',
    fields=['money'],
    panel=False,
)
```

成交金额为空、非数字或小于等于 0 的 ETF 不参与排序。剩余 ETF 按昨日成交金额从高到低取前 5。

### 过去 5 日高成交候选中的涨幅前 5

这组动态 ETF 不再拆成“过去 5 日平均成交金额前 5”和“过去 5 日涨幅前 5”两个独立池，而是一个连续筛选流程：

1. 从全市场 ETF 中获取截至 `context.previous_date` 的最近 5 个交易日行情；
2. 对每只 ETF 计算过去 5 日平均成交金额；
3. 按平均成交金额从高到低选出前 50 只高流动性 ETF；
4. 只在这 50 只 ETF 内计算过去 5 日涨幅；
5. 按 5 日涨幅从高到低取前 5。

为了节省运算时间，策略一次性获取 `money` 和 `close` 两个字段，同一份 5 日行情数据同时用于平均成交金额和区间涨幅计算：

```python
get_price(
    all_etfs,
    end_date=context.previous_date,
    count=5,
    frequency='daily',
    fields=['money', 'close'],
    panel=False,
)
```

过去 5 日涨幅公式：

```text
过去5日涨幅 = 最近第5个交易日收盘价 / 最近第1个交易日收盘价 - 1
```

只有最近 5 个交易日都有有效成交金额、有效收盘价，且成交金额和收盘价都大于 0 的 ETF 才参与这个连续筛选。

### 合并去重规则

最终候选池构造顺序为：

```text
固定 ETF 池
+ 昨日成交金额前5
+ 过去5日均额前50中的涨幅前5
```

去重时保留首次出现的位置。因此：

- 固定池优先；
- 昨日成交金额前 5 只补充固定池中没有的 ETF；
- 过去 5 日均额前 50 中的涨幅前 5 只补充前两部分中都没有的 ETF；
- 如果两组动态 ETF 互相重复，只保留一份；
- 最多新增 10 只动态 ETF，但实际新增数量可能因为重复而少于 10 只。

## 候选 ETF 过滤流程

核心函数：`calculate_momentum_metrics_172(context, etf)`

固定池和两组动态 ETF 合并去重后，每只 ETF 必须依次通过以下过滤条件，才会进入最终排名。

### 1. 停牌过滤

在 `get_ranked_etfs_172` 中先检查：

```python
if current_data[etf].paused:
    continue
```

停牌 ETF 直接跳过，不参与计算。

### 2. 历史数据长度过滤

策略获取：

```python
lookback = max(g.lookback_days, g.short_lookback_days) + 20
prices = attribute_history(etf, lookback, '1d', ['close', 'high'])
```

当前参数下，`max(25, 10) + 20 = 45`，也就是取 45 个交易日数据。若数据长度小于 `g.lookback_days = 25`，该 ETF 被过滤。

### 3. 盈利保护过滤

计算动量前会调用：

```python
if check_profit_protection_172(etf, context):
    return None
```

这个逻辑不仅用于持仓卖出，也用于候选 ETF 过滤。若某 ETF 当前价格相对最近高点回撤超过阈值，则不作为新买入候选。

当前参数为：

- 回看高点：最近 `1` 日最高价。
- 回撤阈值：`5%`。
- 触发条件：`当前价 <= 最近高点 * (1 - 0.05)`。

### 4. 溢价率过滤

开启参数：

```python
g.enable_premium_filter = True
g.premium_threshold = 0.20
```

策略取前一个交易日作为净值比较日：

```python
prev_date = get_trade_days(end_date=context.current_dt.date(), count=2)[0]
```

溢价率公式：

```text
溢价率 = (场内收盘价 - 基金单位净值) / 基金单位净值
```

净值数据获取顺序：

1. 优先使用 `get_extras('unit_net_value', ...)`。
2. 如果取不到，再查询 `finance.FUND_NET_VALUE`。
3. 最多向前回看若干交易日寻找有效净值。

过滤规则：

- 如果成功取得溢价率，且 `premium > 0.20`，过滤该 ETF。
- 如果净值或溢价率获取失败，返回 `None`，不会因为数据缺失直接过滤。

也就是说，这个过滤器只在“明确知道溢价过高”时排除标的。

### 5. 放量过热过滤

开启参数：

```python
g.enable_volume_check = True
g.volume_lookback = 5
g.volume_threshold = 2
g.volume_return_limit = 1
```

计算方式：

1. 用 `attribute_history` 获取过去 5 日日成交量，计算平均成交量。
2. 用分钟线统计当日从开盘到当前时刻的累计成交量。
3. 若 `当日累计成交量 / 过去5日平均成交量 > 2`，认为出现异常放量。
4. 如果异常放量同时 25 日年化收益率超过 `1`，也就是 100%，则过滤。

这个设计主要防止追入短期极度亢奋、成交拥挤、可能处于冲顶阶段的 ETF。

### 6. 短期动量过滤

开启参数：

```python
g.use_short_momentum_filter = True
g.short_lookback_days = 10
g.short_momentum_threshold = 0.0
```

短期收益计算：

```text
10日收益 = 当前价 / 10日前价格 - 1
10日年化收益 = (1 + 10日收益) ^ (250 / 10) - 1
```

过滤规则：

- 若 `short_ann < 0`，过滤该 ETF。

含义是：只做短期趋势不为负的标的。即使 25 日中期趋势较好，如果最近 10 日已经转弱，也不买。

### 7. 最近三日急跌过滤

参数：

```python
g.loss = 0.97
```

策略计算最近三个连续单日价格比：

```python
day1 = price_series[-1] / price_series[-2]
day2 = price_series[-2] / price_series[-3]
day3 = price_series[-3] / price_series[-4]
```

如果三天中任意一天跌幅超过 3%，即：

```text
min(day1, day2, day3) < 0.97
```

则过滤该 ETF。

这个规则是一个短期风险过滤器，避免买入刚发生过明显单日下跌的标的。

### 8. 动量得分阈值过滤

最终排名前，还要求：

```python
g.min_score_threshold < score < g.max_score_threshold
```

当前值为：

```text
0 < score < 100
```

得分小于等于 0 的标的不买，得分极端异常高于 100 的标的也排除。

## 动量得分计算

核心思想是：不仅看涨幅，还看趋势是否平滑稳定。

策略使用最近 `g.lookback_days + 1` 个价格点，当前为 26 个价格点：

```python
recent_y = np.log(price_series[-(g.lookback_days + 1):])
x = np.arange(len(recent_y))
weights = np.linspace(1, 2, len(recent_y))
slope, intercept = np.polyfit(x, recent_y, 1, w=weights)
```

### 为什么使用对数价格

对价格取自然对数后，线性斜率近似代表连续复利收益率。这样不同价格区间的 ETF 可以放在同一套收益尺度下比较。

### 为什么使用加权回归

权重从 1 线性增加到 2：

```python
weights = np.linspace(1, 2, len(recent_y))
```

越接近当前的价格点权重越高，说明策略更重视最近走势。

### 年化收益率

回归斜率转为年化收益：

```python
ann_ret = math.exp(slope * 250) - 1
```

含义是：如果最近 25 日趋势延续一年，大致对应多少年化收益。

### 趋势拟合优度 R²

策略计算加权残差平方和和总平方和：

```python
ss_res = np.sum(weights * (recent_y - (slope * x + intercept)) ** 2)
ss_tot = np.sum(weights * (recent_y - np.mean(recent_y)) ** 2)
r2 = 1 - ss_res / ss_tot if ss_tot != 0 else 0
```

R² 越高，说明价格越接近一条平滑上升趋势；R² 越低，说明走势噪音大、上下震荡多。

### 最终得分

```python
score = ann_ret * r2
```

所以策略偏好：

- 年化收益率高；
- 趋势连续、平滑、可解释度高；
- 最近走势没有明显转弱；
- 没有高溢价、异常放量过热、急跌等风险。

## 买入规则

买入函数：`strategy_3_buy(context)`

完整流程：

1. 调用 `get_cached_rankings(context)` 获取当日 ETF 排名。
2. 从排名结果中按顺序取前 `g.holdings_num` 只 ETF。
3. 当前参数下只取 1 只，即排名第一的 ETF。
4. 如果没有合格 ETF，则检查防御 ETF `511880.XSHG` 是否可买。
5. 如果防御 ETF 可用，则目标 ETF 变为 `511880.XSHG`。
6. 如果防御 ETF 也不可用，则当天不买。
7. 如果当前还持有不在目标列表里的 ETF，买入函数直接返回，等待卖出函数先处理。
8. 目标仓位为总资产的 100%，平均分配给目标 ETF。
9. 当前只有 1 只目标 ETF，所以理论上满仓买入该 ETF。
10. 若当前持仓市值和目标市值差异超过 5%，或者当前没有持仓，则下单调整。

仓位计算：

```python
total_val = context.portfolio.total_value * g.portfolio_value_proportion[2]
val_per = total_val / len(target_etfs)
```

当前 `g.portfolio_value_proportion[2] = 1`，所以 ETF 轮动策略使用账户总资产。

调仓触发条件：

```python
if abs(current_val - val_per) > val_per * 0.05 or current_val == 0:
    smart_order_target_value_172(etf, val_per, context)
```

也就是说，如果现有仓位和目标仓位偏差不超过 5%，不会频繁再平衡。

## 卖出规则

卖出函数：`strategy_3_sell(context)`

完整流程：

1. 获取当日缓存排名。
2. 从排名前 `g.holdings_num` 个结果中筛选得分不低于 `g.min_score_threshold` 的 ETF。
3. 若没有目标 ETF，则尝试使用防御 ETF `511880.XSHG`。
4. 构建目标集合 `target_set`。
5. 遍历当前策略3持仓。
6. 如果某只持仓不在目标集合中，则清仓卖出。

卖出逻辑核心不是固定止损，而是“排名驱动换仓”：只要当前持仓不再属于当日目标 ETF，就卖出。

## 盈利保护与止损机制

本策略没有传统意义上的“买入后跌破成本价 X% 止损”模块，主要风险控制来自以下几层。

### 1. 持仓盈利保护

函数：

- `profit_protection_check_172`
- `check_profit_protection_172`

每天 11:00 检查当前策略3持仓：

```text
如果 当前价 <= 最近1日最高价 * 0.95，则卖出
```

当前参数：

- `g.enable_profit_protection = True`
- `g.profit_protection_lookback = 1`
- `g.profit_protection_threshold = 0.05`

这更像是“高点回撤保护”，并不要求持仓一定盈利。只要价格相对近期高点回撤达到 5%，就会触发卖出。

### 2. 候选标的盈利保护过滤

同一个 `check_profit_protection_172` 也被用于候选 ETF 筛选。如果某只 ETF 当前已经从近期高点回落 5%，它不会被新买入。

### 3. 最近三日急跌过滤

如果最近三天里任意一天跌幅超过 3%，该 ETF 不进入排名。

这避免策略在明显下跌冲击后立即接入。

### 4. 短期动量转弱过滤

10 日年化动量低于 0 的 ETF 不买。

### 5. 排名换仓退出

每天 14:00 如果当前持仓已经不是目标 ETF，就卖出。这是一种趋势跟随策略常见的动态退出机制。

### 6. 防御资产切换

如果没有合格进攻 ETF，策略不强行空仓，而是尝试买入：

```text
511880.XSHG
```

通常这类标的被用作现金管理或货币基金 ETF 防御。

## 防御 ETF 可用性检查

函数：`check_defensive_etf_available_172(context)`

防御 ETF 需要满足：

1. 未停牌；
2. 如果涨停价有效，当前价不能达到涨停；
3. 如果跌停价有效，当前价不能达到跌停。

代码还专门处理实盘中 `high_limit` 或 `low_limit` 为 `NaN` 或 `0` 的情况：

```python
if pd.notna(high_limit) and high_limit > 0:
    ...
```

这能减少实盘数据异常导致的误判。

## 下单规则

核心下单函数：`smart_order_target_value_172(security, target_value, context)`

### 目标金额转股数

```python
target_amount = int(target_value / price)
target_amount = (target_amount // 100) * 100
```

策略按 100 份取整，符合 A 股 ETF 交易单位习惯。

如果目标金额大于 0，但计算后不足 100 份，则强制设置为 100：

```python
if target_amount <= 0 and target_value > 0:
    target_amount = 100
```

### 涨跌停保护

买入时：

- 如果当前价达到有效涨停价，不买。

卖出时：

- 如果当前价达到有效跌停价，不卖。

### 最小交易金额

```python
if 0 < abs(diff) * price < g.min_money:
    return False
```

交易差额小于 5000 元时不下单，减少无意义的小额调仓。

### 可卖数量限制

卖出时使用：

```python
closeable = cur_pos.closeable_amount
diff = -min(abs(diff), closeable)
```

只卖可卖数量，避免当天买入无法卖出的 T+1 限制问题。

### 持仓归属记录

买入成功后：

```python
g.strategy_holdings[3].append(security)
g.stock_strategy[security] = 3
```

卖出到目标金额 0 后：

```python
g.strategy_holdings[3].remove(security)
del g.stock_strategy[security]
```

这些记录用于后续卖出、收益拆分和持仓表展示。

## 回测与交易成本设置

函数：`set_backtest()`

### 避免未来函数

```python
set_option("avoid_future_data", True)
```

要求聚宽尽量避免未来数据。

### 基准

```python
set_benchmark("159919.XSHE")
```

使用沪深 300 ETF `159919.XSHE` 作为基准。

### 使用真实价格

```python
set_option("use_real_price", True)
```

使用真实价格模式。

### 滑点

```python
set_slippage(FixedSlippage(0.002), type="stock")
set_slippage(PriceRelatedSlippage(0.0001), type="fund")
```

股票使用固定滑点，基金使用按价格比例滑点。

### 手续费

策略分别设置股票、基金、货币基金的交易成本：

| 类型 | 印花税 | 佣金 | 最低佣金 |
| --- | --- | --- | --- |
| stock | 卖出 0.0005 | 0.85 / 10000 | 5 |
| fund | 0 | 0.0002 | 5 |
| mmf | 0 | 0 | 0 |

ETF 主要走 `fund` 成本配置。

## 收益记录与持仓展示

### 收益记录

函数：`make_record(context)`

每天 15:01 运行，统计策略3持仓市值和浮动盈亏，并调用：

```python
record(ETF轮动=...)
```

记录 ETF 轮动策略相对初始资金的收益百分比。

### 收盘排名预览

`make_record` 还会在收盘后重新检测最新 ETF 动量排名，并打印前 5 名：

```text
排名1: xxx 得分: x.xxxx
排名2: xxx 得分: x.xxxx
...
```

这方便观察第二天可能的候选方向。

### 持仓表

函数：`print_summary(context)`

每天 15:02 打印：

- 所属策略；
- 股票代码；
- 股票名称；
- 持仓数量；
- 持仓价格；
- 当前价格；
- 盈亏数额；
- 盈亏比例；
- 股票市值；
- 仓位占比；
- 总市值；
- 总资产。

## 核心交易思路

策略本质是趋势跟随，不预测市场涨跌，而是每天回答三个问题：

1. 固定池和高成交 ETF 中，哪类资产趋势最强？
2. 这个趋势是否足够平滑、近期没有明显转弱或过热？
3. 如果没有合格趋势资产，是否应该转入现金防御？

它的核心优势是：

- 多资产轮动，不局限于 A 股；
- 固定池之外，每天自动纳入昨日成交金额最高的 ETF，以及近 5 日高成交候选中涨幅最强的 ETF；
- 使用年化收益率乘 R²，兼顾涨幅和趋势质量；
- 只持有 1 只 ETF，信号非常集中；
- 有溢价率、放量、短期动量、急跌、高点回撤等多重过滤；
- 有防御资产兜底；
- 卖出和买入共用当日缓存排名，避免同日信号不一致。

它的潜在风险是：

- 只持有 1 只 ETF，集中度高；
- 趋势策略在震荡市可能反复换仓；
- QDII ETF 可能存在净值滞后和溢价率数据缺失；
- 20% 溢价阈值较宽，若用于实盘可以按个人风险偏好调低；
- 11:00 盈利保护可能在盘中波动中提前卖出，随后尾盘又重新评估买入；
- 策略依赖聚宽数据质量，特别是分钟成交量、ETF净值、涨跌停字段。

## 参数调整建议

以下不是源策略规则的一部分，只是阅读代码后的调参方向。

### 更稳健

- 将 `g.holdings_num` 从 `1` 提高到 `2` 或 `3`，降低单一 ETF 集中风险。
- 将 `g.premium_threshold` 从 `0.20` 下调到 `0.02` 或 `0.03`，更严格控制 QDII 高溢价风险。
- 提高 `g.min_money`，减少小账户频繁微调。
- 将 `g.profit_protection_threshold` 从 `0.05` 放宽到 `0.07` 或 `0.10`，降低盘中洗出概率。

### 更激进

- 保持 `g.holdings_num = 1`。
- 缩短 `g.lookback_days`，让策略对趋势切换更敏感。
- 关闭或放宽放量过热过滤。

任何参数修改都应重新回测，并特别关注换手率、最大回撤、单笔极端亏损和 QDII 溢价风险。

## 文件来源说明

本仓库策略文件来自本地 `Jq策略/五福.txt` 的精简版，只保留“策略3：ETF轮动策略”。提取时做过以下校验：

- 源文件未修改；
- 新文件语法检查通过；
- 策略3核心函数与源文件对应代码块保持一致；
- 删除了原多策略中的策略1、策略2、策略4代码和调度引用。

## 免责声明

本仓库仅用于量化策略研究和代码整理，不构成投资建议。ETF、LOF、QDII、债券基金等品种均存在价格波动、流动性、溢价率、汇率、跟踪误差和数据质量风险。实盘使用前请在聚宽环境中充分回测、模拟盘验证，并结合个人风险承受能力调整参数。
