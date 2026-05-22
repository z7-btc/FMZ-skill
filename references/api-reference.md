# FMZ 完整 API 参考

> 基于 FMZ 官方语法手册，覆盖 30+ Global 函数和 12 个 API 分类。

## 目录

- [1. Global 全局函数](#1-global-全局函数)
- [2. Log 日志函数](#2-log-日志函数)
- [3. Market 市场行情](#3-market-市场行情)
- [4. Trade 交易函数](#4-trade-交易函数)
- [5. Account 账户函数](#5-account-账户函数)
- [6. Futures 期货函数](#6-futures-期货函数)
- [7. NetSettings 网络设置](#7-netsettings-网络设置)
- [8. Threads 多线程](#8-threads-多线程)
- [9. Web3 区块链](#9-web3-区块链)
- [10. TA 技术分析](#10-ta-技术分析)
- [11. Talib TA-Lib](#11-talib-ta-lib)
- [12. OS 系统函数](#12-os-系统函数)
- [结构体定义](#结构体定义)
- [内置变量与常量](#内置变量与常量)

---

## 1. Global 全局函数

### Sleep(millisecond)
休眠指定毫秒数。
- **参数**: `millisecond` (number, 必选) — 毫秒
- **支持最小**: `0.000001` (1纳秒)
- **范围**: 全部语言（JS/Python/C++）
- **注意**: Python 中必须使用 `Sleep()` 而非 `time.sleep()`，否则回测会极慢

```js
Sleep(1000 * 10)  // 10秒
```

### IsVirtual()
判断是否在回测环境中运行。
- **返回**: `true`=回测, `false`=实盘
- **范围**: 全部语言

```js
if (IsVirtual()) {
    Log("回测环境")
}
```

### _C(pfn, ...args)
容错重试函数。持续调用 `pfn` 直到返回非空/非假值。
- **参数**: `pfn` (function, 必选) — 函数引用（不是调用！）
- **参数**: `...args` — 传递给 pfn 的参数
- **返回**: pfn 的返回值
- **默认重试间隔**: 3秒，可通过 `_CDelay(ms)` 修改
- **范围**: 全部语言

```js
var ticker = _C(exchange.GetTicker)       // 注意：无括号！
var records = _C(exchange.GetRecords, PERIOD_D1)
_CDelay(2000)  // 重试间隔改为2秒
```

适用函数: GetTicker, GetDepth, GetTrades, GetRecords, GetAccount, GetOrders, GetOrder, GetPositions

### _CDelay(millisecond)
设置 `_C()` 重试间隔。
- **参数**: `millisecond` (number) — 毫秒
- **默认**: 3000

### _G(k, v)
持久化全局字典，数据保存到实盘数据库 `kvdb` 表。
- `_G()` — 返回实盘 ID
- `_G("key", value)` — 设置/更新键值
- `_G("key")` — 读取键值
- `_G("key", null)` — 删除单个键
- `_G(null)` — 删除所有键
- **范围**: 全部语言（C++ 不支持 `_G()` 无参调用）
- **注意**: 回测结束后数据清除；实盘重启后数据保留

数据存储位置: `/logs/storage/{robotId}/{robotId}.db3`

### _N(num, precision)
格式化浮点数精度。
- **参数**: `num` (number, 必选)
- **参数**: `precision` (number, 可选, 默认 4)
- **支持负数精度**: `_N(1300, -3)` → 1000
- **范围**: 全部语言

### _D(timestamp, fmt)
时间戳转换为格式化字符串。
- `_D()` — 当前时间
- `_D(timestamp)` — 指定时间戳
- `_D(timestamp, fmt)` — 自定义格式
- **JS** 格式: `yyyy-MM-dd hh:mm:ss`，时间戳为**毫秒**
- **Python** 格式: `%Y-%m-%d %H:%M:%S`，时间戳为**秒**！
- **C++** 格式: `%Y-%m-%d %H:%M:%S`，时间戳为**毫秒**
- **注意**: 显示基于托管者系统本地时间

### _Cross(arr1, arr2)
计算两数组的交叉周期数。
- **返回**: 正数=上穿距今周期数，负数=下穿距今周期数，0=当前相同
- **范围**: 全部语言

```js
var fastMA = TA.MA(records, 5)
var slowMA = TA.MA(records, 20)
var cross = _Cross(fastMA, slowMA)
if (cross > 0 && cross < 3) { /* 近期金叉 */ }
```

### Dial(address, timeout/options)
原始 Socket/WebSocket/数据库连接。
- **支持协议**: `tcp://`, `udp://`, `tls://`, `unix://`, `wss://`, `ws://`
- **通信协议**: `mqtt://`, `nats://`, `amqp://`, `kafka://` (仅 JS)
- **数据库**: `sqlite3://`, `mysql://`, `postgres://`, `clickhouse://`
- **返回**: 连接对象 `{read, write, close, Valid}` (超时返回 null)
- **范围**: 全部语言（通信协议和数据库仅 JS）

address 参数格式（`|` 分隔选项）:
```
wss://ws.okx.com:8443/ws/v5/public|compress=gzip_raw&mode=recv&reconnect=true
```

关键选项:
| 参数 | 说明 |
|------|------|
| `compress=gzip/gzip_raw` | 压缩方式 |
| `mode=dual/send/recv` | 压缩模式 |
| `reconnect=true` | 自动重连 |
| `interval=10000` | 重连间隔(ms)，默认 1000 |
| `payload=...` | 重连时发送的订阅消息 |
| `proxy=socks5://user:pwd@ip:port` | socks5 代理 |
| `enableCompression=true` | WebSocket compression |

read() 行为:
| 场景 | 无参数 | -1 | -2 | 2000(ms) |
|------|--------|----|----|-----------|
| 缓冲区有数据 | 立即返回最旧 | 返回最旧 | 返回最新 | 返回最旧 |
| 缓冲区无数据 | 阻塞等待 | 立即返回 null | 返回 null | 等待超时 |
| 连接断开 | 返回 "" | 返回 "" | 返回 "" | 返回 "" |

WebSocket 缓冲区上限: 2000 条消息。

### HttpQuery(url, options)
发送 HTTP 请求（同步）。
- **参数**: `url` (string, 必选)
- **参数**: `options` (object, 可选) — `{method, body, charset, cookie, debug, headers, timeout}`
- **返回**: `debug=false` 返回 body 字符串；`debug=true` 返回完整响应对象
- **范围**: JS / C++（Python 使用 urllib）
- **回测**: 仅支持 GET，相同 URL 最多 20 次且缓存数据

支持 socks5 代理:
```js
HttpQuery("socks5://127.0.0.1:8889/http://www.baidu.com/")
HttpQuery("socks5://user:pwd@ip:port/http://target.url/")
```

### HttpQuery_Go(url, options)
HTTP 请求异步版。立即返回并发对象，通过 `.wait()` 获取结果。
- **返回**: 并发对象 `{wait()}`
- **范围**: 仅 JS
- **回测**: 不支持

```js
var r1 = HttpQuery_Go("https://api1.com/tickers")
var r2 = HttpQuery_Go("https://api2.com/tickers")
var result1 = r1.wait()  // 阻塞等待
var result2 = r2.wait()
```

### Encode(algo, inputFormat, outputFormat, data, keyFormat?, key?)
编码/加密/哈希函数。
- **参数**: `algo` (string) — 算法：`"raw"`, `"md5"`, `"sha256"`, `"sha512"`, `"sha1"`, `"keccak256"`, `"sha3.256"`, `"sha3.512"`, `"ripemd160"`, `"blake2b.256"`, `"blake2b.512"`, `"ed25519"`, `"ed25519.seed"`, `"ed25519.md5"`, `"ed25519.sha512"` 等
- **参数**: `inputFormat` — `"raw"`, `"hex"`, `"base64"`, `"string"`
- **参数**: `outputFormat` — `"raw"`, `"hex"`, `"base64"`, `"string"`
- **参数**: `data` (string) — 待处理数据
- **参数**: `keyFormat` — 同 inputFormat
- **参数**: `key` — HMAC 密钥
- **字符串编解码**: `"text.encoder.utf8"`, `"text.decoder.utf8"`, `"text.encoder.gbk"`, `"text.decoder.gbk"`
- **范围**: 全部语言（Python 必须显式传 keyFormat/key 为 `""`）
- **注意**: 仅支持实盘

```js
Log(Encode("sha256", "raw", "hex", "hello", "raw", "key123"))
Log(Encode("text.encoder.utf8", "raw", "hex", "你好"))
```

### Mail(smtpServer, smtpUsername, smtpPassword, mailTo, title, body)
发送邮件（同步）。
- **SSL**: `ssl://smtp.qq.com:465`
- **非SSL**: `smtp.qq.com:587`
- **范围**: 全部语言
- **回测**: 不生效

### Mail_Go(...)
Mail 异步版本。参数同 Mail。
- **返回**: 并发对象 `{wait()}`
- **范围**: 仅 JS
- **回测**: 不生效

### DBExec(sql)
数据库操作（SQLite）。
- **内存数据库**: SQL 以 `:` 开头
- **文件数据库**: SQL 不以 `:` 开头
- **返回**: `{columns:[...], values:[...]}`
- **范围**: 全部语言
- **回测**: 不支持
- **注意**: 系统保留表 `kvdb, cfg, log, profit, chart` 不要操作

参数化查询: `DBExec("UPDATE t SET col=? WHERE id=?", newVal, id)`

### UUID()
生成 32 位 UUID 字符串。
- **范围**: 全部语言
- **回测**: 不支持

### Unix()
秒级时间戳。
- **返回**: number (秒级)
- **范围**: 全部语言

### UnixNano()
纳秒级时间戳。
- **返回**: number
- **毫秒转换**: `UnixNano() / 1000000`
- **范围**: 全部语言

### MD5(data)
MD5 哈希。
- **返回**: 32 位十六进制字符串
- **范围**: 全部语言

### GetOS()
获取操作系统信息。如 `darwin/amd64`, `linux/amd64`。
- **范围**: 全部语言

### GetPid()
获取实盘进程 ID。
- **范围**: 全部语言

### GetLastError()
获取最近一次 API 调用的错误信息。
- **返回**: string
- **范围**: 全部语言
- **回测**: 不起作用

### SetErrorFilter(filters)
过滤错误日志（正则表达式，`|` 分隔）。
- **多次调用累积生效**，`SetErrorFilter("")` 重置
- **范围**: 全部语言

```js
SetErrorFilter("502:|503:|tcp|character|unexpected|network|timeout|WSARecv|Connect|GetAddr|no such|reset|http|received|EOF|reused")
```

### GetCommand()
获取策略交互控件命令。
- **返回格式**: `"ControlName:Data"` 或仅 `"ControlName"`
- **范围**: 全部语言
- **回测**: 不生效

### GetMeta()
获取策略注册码的 Meta 值。
- **返回**: string（最大 190 字符）
- **范围**: 全部语言
- **回测**: 不起作用

### JSON.parse(s, safeStr?)
解析 JSON 字符串（JS 扩展版）。
- **参数**: `safeStr` (bool) — 将大数值解析为字符串避免精度丢失
- **范围**: 仅 JS

```js
JSON.parse('{"num": 8754613216564987646512354656874651651358}', true)
// → {"num": "8754613216564987646512354656874651651358"}
```

### JSON.stringify(obj)
对象序列化为 JSON 字符串。
- **范围**: 仅 JS

### SetChannelData(data)
在实盘频道上发布状态数据（实盘间通信）。
- **参数**: `data` — JSON 可序列化，序列化后 **≤ 1024 字节**
- **范围**: 全部语言（C++ 需确认）
- **回测**: 可能受限
- 每次调用**覆盖**旧数据，不追加
- 支持跨平台：外部系统 POST 到 `https://www.fmz.com/api/v1?method=pub&robot={id}&channel={uuid}`

### GetChannelData(channelId)
订阅频道数据。
- **参数**: `channelId` — 实盘 ID（`_G()` 获取）或 32 位 UUID
- **返回**: 首次调用返回 null，需重试
- 非阻塞调用
- **范围**: 全部语言
- **回测**: 可能受限

### EventLoop(timeout?)
事件监听循环。
- **参数**: `timeout` (ms) — 0=等待事件, >0=超时, <0=立即返回
- **返回**: `{Seq, Event, ThreadId, Index, Nano}`
- **回调队列上限**: 500 个
- **范围**: 全部语言
- **回测**: 不支持
- **注意**: 首次调用才初始化监听；在事件回调之后调用会错过事件

### exchange.Go(methodName, ...args)
并发执行 exchange 成员函数。返回并发对象。
- **返回值用 `.wait()` 获取**
- **范围**: 全部语言

```js
var r1 = exchange.Go("GetTicker")
var r2 = exchange.Go("GetDepth")
// ... 做其他事情 ...
var ticker = r1.wait()
var depth = r2.wait()
```

### __Serve(serveURI, handler, ...args)
创建 HTTP/TCP/WebSocket 服务。
- **限制**: 仅 JS
- **回测**: 不支持
- 服务线程与全局作用域隔离（不支持闭包/外部变量）

```js
var server = __Serve("http://:8088?gzip=true", function(ctx) {
    var path = ctx.path()
    if (path == "/") {
        ctx.write(JSON.stringify({status: "ok"}))
    } else if (path == "/wss") {
        if (ctx.upgrade("websocket")) {
            while (true) {
                var msg = ctx.read(10)
                if (!msg) break
                ctx.write("echo: " + msg)
            }
        }
    }
})
```

ctx 对象方法: `proto()`, `host()`, `path()`, `query(key)`, `rawQuery()`, `headers()`, `header(key)`, `method()`, `body()`, `setHeader(k,v)`, `setStatus(code)`, `remoteAddr()`, `localAddr()`, `upgrade("websocket")`, `read(ms)`, `write(s)`

---

## 2. Log 日志函数

| 函数 | 说明 | 范围 |
|------|------|------|
| `Log(...args)` | 输出日志（支持颜色 `"#FF0000"`） | 全部 |
| `LogProfit(profit)` | 记录收益值 | 全部 |
| `LogProfitReset()` | 重置收益记录 | 全部 |
| `LogStatus(msg)` | 更新状态栏（支持 Markdown 表格） | 全部 |
| `EnableLog(enable)` | 启用/禁用日志 | 全部 |
| `Chart(options)` | 绘制图表 | 全部 |
| `KLineChart(options)` | 绘制 K 线图 | 全部 |
| `LogReset(keep)` | 清除日志（keep=1 保留最近） | 全部 |
| `LogVacuum()` | 清理日志数据库 | 全部 |
| `console.log(...)` | 标准控制台输出 | JS |
| `console.error(...)` | 标准错误输出 | JS |

### LogStatus 表格格式
```js
var table = {
    type: 'table',
    title: '状态',
    cols: ['项目', '值'],
    rows: [['价格', 50000], ['持仓', 0.1]]
}
LogStatus('`' + JSON.stringify(table) + '`')
```

---

## 3. Market 市场行情

**前置步骤**: 期货需先调用 `exchange.SetContractType("BTC_USDT")`。

| 函数 | 说明 | 返回 |
|------|------|------|
| `exchange.GetTicker()` | 行情 ticker | `{Last, High, Low, Buy, Sell, Volume, Time}` |
| `exchange.GetDepth()` | 深度数据 | `{Asks:[{Price,Amount}], Bids:[{Price,Amount}], Time}` |
| `exchange.GetTrades()` | 逐笔成交 | `[{Id, Time, Price, Amount, Type}]` |
| `exchange.GetRecords(period)` | K线数据 | `[{Time, Open, High, Low, Close, Volume}]` |
| `exchange.GetPeriod()` | 当前K线周期（秒） | number |
| `exchange.SetMaxBarLen(len)` | 最大K线长度 | - |
| `exchange.GetRawJSON()` | 原始JSON | string |
| `exchange.GetRate()` | 汇率 | number |
| `exchange.SetData(key, value)` | 设置连接池数据 | - |
| `exchange.GetData(key)` | 获取连接池数据 | - |
| `exchange.GetMarkets()` | 获取市场列表 | - |
| `exchange.GetTickers()` | 获取多个交易对行情 | - |

K线周期常量: `PERIOD_M1`, `PERIOD_M3`, `PERIOD_M5`, `PERIOD_M15`, `PERIOD_M30`, `PERIOD_H1`, `PERIOD_H2`, `PERIOD_H4`, `PERIOD_H6`, `PERIOD_H12`, `PERIOD_D1`, `PERIOD_D3`, `PERIOD_W1`

---

## 4. Trade 交易函数

| 函数 | 说明 |
|------|------|
| `exchange.Buy(price, amount)` | 买入（price=-1 市价） |
| `exchange.Sell(price, amount)` | 卖出（price=-1 市价） |
| `exchange.CreateOrder(type, price, amount, ...)` | 创建高级订单 |
| `exchange.CancelOrder(orderId)` | 取消订单 |
| `exchange.GetOrder(orderId)` | 查询订单 → `{Id, Price, Amount, DealAmount, AvgPrice, Status, Type}` |
| `exchange.GetOrders()` | 所有未完成订单 |
| `exchange.GetHistoryOrders()` | 历史订单 |
| `exchange.CreateConditionOrder(...)` | 创建条件单 |
| `exchange.ModifyOrder(orderId, ...)` | 修改订单 |
| `exchange.ModifyConditionOrder(...)` | 修改条件单 |
| `exchange.CancelConditionOrder(id)` | 取消条件单 |
| `exchange.GetConditionOrder(id)` | 查询条件单 |
| `exchange.GetConditionOrders()` | 所有条件单 |
| `exchange.GetHistoryConditionOrders()` | 历史条件单 |
| `exchange.SetPrecision(pricePrec, amtPrec)` | **下单前必须设置精度** |
| `exchange.SetRate(rate)` | 设置汇率 |
| `exchange.IO(...)` | 扩展IO（Web3等） |
| `exchange.Log(...)` | 交易相关日志 |
| `exchange.Encode(...)` | 交易相关编码 |
| `exchange.Go(method, ...)` | 并发调用 |

### 订单状态
- `ORDER_STATE_PENDING` — 未完成
- `ORDER_STATE_CLOSED` — 已完成
- `ORDER_STATE_CANCELED` — 已取消
- `ORDER_STATE_UNKNOWN` — 未知

### 订单类型
- `ORDER_TYPE_BUY` — 买入
- `ORDER_TYPE_SELL` — 卖出

---

## 5. Account 账户函数

| 函数 | 说明 | 返回 |
|------|------|------|
| `exchange.GetAccount()` | 账户信息 | `{Balance, FrozenBalance, Stocks, FrozenStocks}` |
| `exchange.GetAssets()` | 资产列表 | - |
| `exchange.GetName()` | 交易所名称 | string |
| `exchange.GetLabel()` | 交易所标签 | string |
| `exchange.GetCurrency()` | 计价货币 | string |
| `exchange.SetCurrency(currency)` | 设置货币 | - |
| `exchange.GetQuoteCurrency()` | 报价货币 | string |

---

## 6. Futures 期货函数

| 函数 | 说明 |
|------|------|
| `exchange.GetPositions()` | 获取持仓 → `[{Type, Amount, Price, Margin, Profit}]` |
| `exchange.SetMarginLevel(level)` | 设置保证金杠杆 |
| `exchange.SetDirection(dir)` | 设置方向 `PD_LONG` / `PD_SHORT` |
| `exchange.SetContractType("BTC_USDT")` | **设置合约类型（必须第一步调用）** |
| `exchange.GetContractType()` | 获取当前合约类型 |
| `exchange.GetFundings()` | 获取资金费率 |

### 持仓方向常量
- `PD_LONG` — 做多
- `PD_SHORT` — 做空

### 仓位偏移常量
- `ORDER_OFFSET_OPEN` — 开仓
- `ORDER_OFFSET_CLOSE` — 平仓

---

## 7. NetSettings 网络设置

| 函数 | 说明 |
|------|------|
| `exchange.SetBase(url)` | 设置交易所 API 基础 URL |
| `exchange.GetBase()` | 获取 API 基础 URL |
| `exchange.SetProxy(proxy)` | 设置代理 |
| `exchange.SetTimeout(ms)` | 设置超时 |

---

## 8. Threads 多线程

### threading 模块
| 函数/对象 | 说明 |
|-----------|------|
| `threading.Thread(fn)` | 创建线程 |
| `threading.getThread(id)` | 获取线程 |
| `threading.mainThread` | 主线程 |
| `threading.currentThread` | 当前线程 |
| `threading.Lock()` | 互斥锁 `{acquire, release}` |
| `threading.Condition()` | 条件变量 `{notify, notifyAll, wait, acquire, release}` |
| `threading.Event()` | 事件 `{set, clear, wait, isSet}` |
| `threading.Dict()` | 线程安全字典 `{get, set}` |
| `threading.pending` | 待处理线程列表 |

### Thread 对象
- `peekMessage(ms)` — 检查消息
- `postMessage(msg)` — 发送消息
- `join()` — 等待线程结束
- `terminate()` — 终止线程
- `getData(key)` / `setData(key, value)` — 线程数据
- `id` — 只读 ID
- `name` — 只读名称
- `eventLoop(ms)` — 线程事件循环

---

## 9. Web3 区块链

通过 `exchange.IO(...)` 调用：

| 操作 | 说明 |
|------|------|
| `exchange.IO("abi", abiJson)` | 设置 ABI |
| `exchange.IO("api", blockChain, contractAddr, method, ...)` | 调用合约方法 |
| `exchange.IO("encode", ...)` | ABI 编码 |
| `exchange.IO("encodePacked", ...)` | Packed 编码 |
| `exchange.IO("decode", ...)` | ABI 解码 |
| `exchange.IO("key", privateKey)` | 设置私钥 |
| `exchange.IO("api", ...)` | 区块链 API 请求 |
| `exchange.IO("address")` | 获取地址 |
| `exchange.IO("base", url)` | 设置节点 URL |

---

## 10. TA 技术分析

| 函数 | 说明 | 返回 |
|------|------|------|
| `TA.MACD(records, fast, slow, signal)` | MACD | `[DIF数组, DEA数组, MACD柱数组]` |
| `TA.KDJ(records, n, k, d)` | KDJ | `[K数组, D数组, J数组]` |
| `TA.RSI(records, period)` | RSI | `[RSI值数组]` |
| `TA.ATR(records, period)` | ATR | `[ATR值数组]` |
| `TA.OBV(records)` | OBV | `[OBV值数组]` |
| `TA.MA(records, period)` | 移动平均 | `[MA值数组]` |
| `TA.EMA(records, period)` | 指数移动平均 | `[EMA值数组]` |
| `TA.BOLL(records, period, multiplier)` | 布林带 | `[上轨数组, 中轨数组, 下轨数组]` |
| `TA.Alligator(records)` | 鳄鱼线 | `[唇, 齿, 颚]` |
| `TA.CMF(records, period)` | 蔡金资金流 | `[CMF值数组]` |
| `TA.Highest(records, period)` | 最高值 | `[最高值数组]` |
| `TA.Lowest(records, period)` | 最低值 | `[最低值数组]` |
| `TA.SMA(records, period)` | 简单移动平均 | `[SMA值数组]` |

---

## 11. Talib TA-Lib

包含 100+ 技术分析指标，包括蜡烛形态、重叠研究、动量指标、波动率指标等。

### 蜡烛形态（部分）
`talib.CDL2CROWS`, `talib.CDL3BLACKCROWS`, `talib.CDL3INSIDE`, `talib.CDL3OUTSIDE`, `talib.CDLDOJI`, `talib.CDLDOJISTAR`, `talib.CDLDRAGONFLYDOJI`, `talib.CDLENGULFING`, `talib.CDLEVENINGSTAR`, `talib.CDLHAMMER`, `talib.CDLHANGINGMAN`, `talib.CDLHARAMI`, `talib.CDLMORNINGSTAR`, `talib.CDLSHOOTINGSTAR` 等 60+ 种

### 数学函数
`talib.ACOS`, `talib.ASIN`, `talib.ATAN`, `talib.CEIL`, `talib.COS`, `talib.EXP`, `talib.FLOOR`, `talib.LN`, `talib.LOG10`, `talib.MAX`, `talib.MIN`, `talib.SIN`, `talib.SQRT`, `talib.TAN`, `talib.SUM`, `talib.MINMAX` 等

### 重叠研究
`talib.BBANDS`, `talib.DEMA`, `talib.EMA`, `talib.HT_TRENDLINE`, `talib.KAMA`, `talib.MA`, `talib.MAMA`, `talib.MIDPOINT`, `talib.MIDPRICE`, `talib.SAR`, `talib.SAREXT`, `talib.SMA`, `talib.T3`, `talib.TEMA`, `talib.TRIMA`, `talib.WMA`

### 动量指标
`talib.ADX`, `talib.ADXR`, `talib.APO`, `talib.AROON`, `talib.AROONOSC`, `talib.BOP`, `talib.CCI`, `talib.CMO`, `talib.DX`, `talib.MACD`, `talib.MACDEXT`, `talib.MACDFIX`, `talib.MFI`, `talib.MINUS_DI`, `talib.MINUS_DM`, `talib.MOM`, `talib.PLUS_DI`, `talib.PLUS_DM`, `talib.PPO`, `talib.ROC`, `talib.ROCP`, `talib.ROCR`, `talib.ROCR100`, `talib.RSI`, `talib.STOCH`, `talib.STOCHF`, `talib.STOCHRSI`, `talib.TRIX`, `talib.ULTOSC`, `talib.WILLR`

---

## 12. OS 系统函数

| 函数 | 说明 |
|------|------|
| `os.open(path, mode)` | 打开文件 |
| `os.fgets(file)` | 读取一行 |
| `os.fputs(file, str)` | 写入字符串 |
| `os.mmap(path)` | 内存映射文件 |
| `os.getRootDir()` | 根目录路径 |
| `os.listFiles(dir)` | 列出文件 |
| `os.exists(path)` | 路径是否存在 |
| `os.remove(path)` | 删除文件 |
| `os.mkdir(path)` | 创建目录 |
| `os.rmdir(path)` | 删除目录 |
| `os.rename(old, new)` | 重命名 |
| `os.stat(path)` | 文件状态 |
| `os.exit()` | 退出程序 |

### File 对象
`close()`, `puts(str)`, `printf(fmt, ...)`, `flush()`, `tell()`, `seek(offset, whence)`, `eof()`, `read(len)`, `write(str)`, `getline()`, `toString()`

---

## 结构体定义

### Ticker
```
{Last, High, Low, Buy, Sell, Volume, Time}
```

### Depth
```
{Asks: [{Price, Amount}], Bids: [{Price, Amount}], Time}
```

### Record (K线)
```
{Time, Open, High, Low, Close, Volume}
```

### Order
```
{Id, Price, Amount, DealAmount, AvgPrice, Status, Type, ContractType}
```

### Account
```
{Balance, FrozenBalance, Stocks, FrozenStocks}
```

### Position
```
{Type, Amount, Price, Margin, Profit, ContractType}
```

### Trade
```
{Id, Time, Price, Amount, Type}
```

### OrderBook
```
{Price, Amount}
```

---

## 内置变量与常量

### 交易所引用
- `exchange` — 主交易所（添加的第一个交易所对象）
- `exchanges` — 所有交易所对象数组
- `EXCHANGE` — 交易所枚举名称

### 订单状态
- `ORDER_STATE_PENDING` — 未完成
- `ORDER_STATE_CLOSED` — 已完成
- `ORDER_STATE_CANCELED` — 已取消
- `ORDER_STATE_UNKNOWN` — 未知

### 订单类型
- `ORDER_TYPE_BUY` — 买入
- `ORDER_TYPE_SELL` — 卖出

### 条件单类型
- `ORDER_CONDITION_TYPE_OCO` — OCO
- `ORDER_CONDITION_TYPE_TP` — 止盈
- `ORDER_CONDITION_TYPE_SL` — 止损
- `ORDER_CONDITION_TYPE_GENERIC` — 通用

### K线周期
- `PERIOD_M1`, `PERIOD_M3`, `PERIOD_M5`, `PERIOD_M15`, `PERIOD_M30`
- `PERIOD_H1`, `PERIOD_H2`, `PERIOD_H4`, `PERIOD_H6`, `PERIOD_H12`
- `PERIOD_D1`, `PERIOD_D3`, `PERIOD_W1`

### 日志类型
- `LOG_TYPE_BUY` — 买入
- `LOG_TYPE_SELL` — 卖出
- `LOG_TYPE_CANCEL` — 取消

### 持仓方向
- `PD_LONG` — 做多
- `PD_SHORT` — 做空

### 偏移方向
- `ORDER_OFFSET_OPEN` — 开仓
- `ORDER_OFFSET_CLOSE` — 平仓
