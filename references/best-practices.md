# FMZ 最佳实践

> 策略编写规范与常见模式，避免坑。

## 目录

- [1. 容错处理](#1-容错处理)
- [2. 错误日志过滤](#2-错误日志过滤)
- [3. 精度处理](#3-精度处理)
- [4. 休眠函数使用](#4-休眠函数使用)
- [5. 日志与状态展示](#5-日志与状态展示)
- [6. 持久化存储](#6-持久化存储)
- [7. 回测兼容性](#7-回测兼容性)
- [8. WebSocket 连接管理](#8-websocket-连接管理)
- [9. 策略退出清理](#9-策略退出清理)
- [10. 策略参数管理](#10-策略参数管理)
- [11. Python 策略参数兜底](#11-python-策略参数兜底)
- [12. Python 时间获取规范](#12-python-时间获取规范)
- [13. Python 日志字符规范](#13-python-日志字符规范)
- [14. 持仓查询一致性](#14-持仓查询一致性)

---

## 1. 容错处理

### 使用 _C() 包裹所有网络 IO

交易所 API 调用可能因网络波动失败，必须使用容错函数：

```js
// ✅ 正确：使用 _C 容错
var ticker = _C(exchange.GetTicker)

// ❌ 错误：直接调用可能返回 null 导致后续代码崩溃
var ticker = exchange.GetTicker()
```

### _C() 使用注意事项

- `_C()` 的参数是**函数引用**，不是函数调用：
  ```js
  _C(exchange.GetTicker)      // ✅ 正确
  _C(exchange.GetTicker())    // ❌ 错误！
  ```
- 带参数的函数容错：
  ```js
  _C(exchange.GetRecords, PERIOD_D1)
  ```
- 默认重试间隔 3 秒，可通过 `_CDelay(ms)` 修改
- 适用于: GetTicker, GetDepth, GetTrades, GetRecords, GetAccount, GetOrders, GetOrder, GetPositions
- 也可用于自定义函数，当函数返回 false/null/空值时重试

### 检查 null 值

```js
var ticker = _C(exchange.GetTicker)
if (!ticker) {  // 即使 _C 也可能因为策略被停止而返回 null
    Sleep(5000)
    continue
}
// 安全使用 ticker.Last, ticker.Buy 等
```

---

## 2. 错误日志过滤

大量网络错误日志会导致数据库膨胀、磁盘占满。**建议在 main() 开头设置**：

```js
function main() {
    SetErrorFilter("502:|503:|tcp|character|unexpected|network|timeout|WSARecv|Connect|GetAddr|no such|reset|http|received|EOF|reused")
    // ...
}
```

- 正则表达式，`|` 分隔
- 多次调用**累积**生效，`SetErrorFilter("")` 重置
- WebSocket 的 `read()` 超时错误需单独过滤

---

## 3. 精度处理

### 下单前必须设置精度

```js
exchange.SetPrecision(2, 3)  // 价格2位小数，数量3位小数
```

不设置可能导致下单失败。

### 使用 _N() 格式化数量

```js
var feeRate = 0.001
var amount = _N(balance * (1 - feeRate) / price, 3)
```

### 交易所精度参考

| 交易所 | 常见价格精度 | 常见数量精度 |
|--------|------------|------------|
| 币安 BTC/USDT | 2 | 3 |
| OKX BTC/USDT | 2 | 3 |
| 火币 BTC/USDT | 2 | 4 |

---

## 4. 休眠函数使用

### Python 必须使用 Sleep() 而非 time.sleep()

```python
# ✅ 正确
Sleep(1000)

# ❌ 错误——会导致回测极慢！
import time
time.sleep(1)
```

`Sleep()` 在回测系统中会跳到下一个时间序列而非实际等待。
`time.sleep()` 会真实阻塞，回测时等待实际物理时间。

### 最小休眠和轮询频率

- 实盘轮询间隔建议 ≥ 3 秒（交易所 API 频率限制）
- 回测中 `Sleep(0)` 等价于跳到下一条 K 线
- 高频策略注意避免过度请求触发限频

---

## 5. 日志与状态展示

### 使用 LogStatus 表格

```js
var table = {
    type: 'table',
    title: '策略状态',
    cols: ['指标', '数值'],
    rows: [
        ['当前价格', price],
        ['持仓量', amount],
        ['收益率', profit + '%']
    ]
}
LogStatus('`' + JSON.stringify(table) + '`')
```

### Log 颜色

```js
Log("普通消息")
Log("买入信号", "#FF0000")    // 红色
Log("卖出信号", "#00FF00")    // 绿色
Log("警告信息", "#FFA500")    // 橙色
```

### LogProfit 记录收益

```js
LogProfit(profit)           // 添加收益点
LogProfitReset()            // 重置收益曲线
```

### 定期清理日志

长时间运行的实盘应定期清理：
```js
LogReset(1)   // 保留最近1条日志，清除其余
LogVacuum()   // 回收数据库空间
```

---

## 6. 持久化存储

### _G() 使用规范

```js
// 存储状态（实盘重启后保留）
_G("lastBuyPrice", 50000)
_G("totalProfit", 1234.56)

// 读取
var price = _G("lastBuyPrice")

// 删除单个键
_G("lastBuyPrice", null)

// 清除所有（慎用！）
_G(null)
```

- 数据存储到 SQLite 数据库，实盘停止不会丢失
- 回测结束后清除
- 不要存储过于频繁变化的数据（IO 开销）
- 每个键值对都会写磁盘，高频写入可能导致性能问题

### DBExec() 使用规范

```js
// 内存数据库（推荐，速度最快）
DBExec(":CREATE TABLE IF NOT EXISTS orders (id TEXT, price REAL)")

// 文件数据库（持久化，速度较慢）
DBExec("CREATE TABLE IF NOT EXISTS orders (id TEXT, price REAL)")

// 参数化查询防止 SQL 注入
DBExec("INSERT INTO orders VALUES (?, ?)", orderId, price)
```

- 系统保留表不要操作: `kvdb`, `cfg`, `log`, `profit`, `chart`
- DBExec 仅实盘支持
- 不支持事务

---

## 7. 回测兼容性

### IsVirtual() 环境分支

```js
function main() {
    if (IsVirtual()) {
        Log("回测环境")
        // 回测逻辑：简化处理、跳过实盘专用功能
    } else {
        Log("实盘环境")
        // 实盘逻辑：完整功能
    }
}
```

### 回测不支持的功能

| 功能 | 回测行为 |
|------|---------|
| `DBExec()` | 不支持 |
| `_G()` 持久化 | 回测结束后清除 |
| `GetCommand()` | 不生效 |
| `GetLastError()` | 不起作用 |
| `Mail()` | 不生效 |
| `HttpQuery()` | 仅 GET，同 URL 最多 20 次且缓存 |
| `HttpQuery_Go()` | 不支持 |
| `EventLoop()` | 不支持 |
| `Dial()` | 不支持 |
| `SetChannelData()`/`GetChannelData()` | 可能受限 |
| `UUID()` | 不支持 |

### 回测中的时间处理

```js
// 获取回测中的当前 K 线时间
var records = exchange.GetRecords(PERIOD_D1)
var currentTime = records[records.length - 1].Time
```

---

## 8. WebSocket 连接管理

### 连接检测与重连

```js
var ws = Dial("wss://stream.binance.com:9443/ws/btcusdt@ticker")

while (true) {
    var buf = ws.read()
    if (buf === "") {
        // 连接断开（read 返回空字符串）
        Log("WebSocket 断开，尝试重连", "#FFA500")
        ws.close()
        ws = Dial("wss://stream.binance.com:9443/ws/btcusdt@ticker")
        if (!ws) {
            Log("重连失败，休眠后重试")
            Sleep(10000)
            continue
        }
    }
    if (buf) {
        // 处理数据
        var data = JSON.parse(buf)
        // ...
    }
}
```

### 启用自动重连

```js
var ws = Dial("wss://stream.binance.com:9443/ws/btcusdt@ticker|reconnect=true&interval=5000")
```

### 在 onexit() 中关闭连接

```js
function onexit() {
    if (ws) ws.close()
    Log("连接已关闭")
}
```

### WebSocket read() 缓冲区管理

缓冲区上限 2000 条，如果策略主循环过长可能导致数据积压。高频推送场景使用 `ws.read(-1)` 获取最新数据：

```js
// 丢弃旧数据，只获取最新
var latest = ws.read(-2)
```

---

## 9. 策略退出清理

### onexit() 函数

```js
function onexit() {
    // 关闭所有 WebSocket 连接
    if (ws) ws.close()
    // 取消所有未完成订单（可选）
    var orders = exchange.GetOrders()
    for (var i = 0; i < orders.length; i++) {
        exchange.CancelOrder(orders[i].Id)
    }
    Log("策略已退出，资源已清理")
}
```

- `onexit()` 在策略停止时自动调用，但不是所有场景都能执行到
- 关键资源（如订单清理）不应完全依赖 onexit

---

## 10. 策略参数管理

### 使用交互控件

策略可以添加交互控件，通过 `GetCommand()` 获取用户输入：

```js
while (true) {
    var cmd = GetCommand()
    if (cmd) {
        var parts = cmd.split(":")
        var action = parts[0]
        var value = parts[1]
        Log("命令:", action, value)
        // 处理不同的交互命令
    }
    Sleep(1000)
}
```

### 使用 GetMeta() 区分租用级别

```js
var level = GetMeta()
if (level == "vip") {
    maxPosition = 10
} else if (level == "basic") {
    maxPosition = 1
} else {
    maxPosition = 0.5
}
```

---

## 11. Python 策略参数兜底

### 问题

FMZ 策略参数（在网页编辑器「策略参数」面板中添加的变量）如果还未添加就运行策略，会报 `NameError`：

```
NameError: name 'FastPeriod' is not defined
```

### 解决方案

在 `main()` 开头对每个策略参数做 try/except 兜底，提供默认值：

```python
def main():
    # 策略参数兜底：如果 FMZ 面板已添加参数，用面板值；否则用默认值
    try:
        FastPeriod
    except NameError:
        FastPeriod = 5         # 默认值

    try:
        SlowPeriod
    except NameError:
        SlowPeriod = 20

    try:
        Leverage
    except NameError:
        Leverage = 10

    # ... 后续代码正常使用 FastPeriod, SlowPeriod, Leverage
```

**原理**：FMZ 会先把你面板上定义的参数注入为全局变量，然后才执行策略代码。如果面板里有参数，`try` 块不会触发；如果没有，`except` 赋默认值，策略不至于崩溃。

---

## 12. Python 时间获取规范

### 问题

在 FMZ Python 环境（基于 Pyodide）中，`time.time()` 在回测中可能返回不可靠的模拟时间值，导致 `_D()` 显示 1970 年：

```
当前时间  1970-01-21 14:09:35   ← 错误！
```

这是因为 `time.time()` 返回了一个远小于真实秒级时间戳的值。

### 解决方案

**推荐方案（按优先级）**：

```python
# 方案 1（最推荐）：_D() 无参调用，FMZ 自动取当前时间
_D()

# 方案 2：使用 FMZ 内置 Unix() 获取秒级时间戳（三语言统一，稳定可靠）
_D(Unix())

# 方案 3（不推荐）：time.time() 在回测中不可靠
_D(int(time.time()))     # ← 可能导致 1970 年！
```

**注意**：K 线的 `Record.Time` 在所有语言中都是毫秒级，Python 中传给 `_D()` 需要除以 1000：

```python
records = exchange.GetRecords(PERIOD_H1)
_D(int(records[-1].Time / 1000))   # Record.Time 毫秒 → 秒
```

---

## 13. Python 日志字符规范

### 问题

FMZ Python 环境基于 **Pyodide**（Python → JavaScript 桥接），emoji 表情符号在 Python 字符串转换为 JavaScript 时会触发编码崩溃：

```
pyodide.ffi.JsException: RangeError: Invalid code point NaN
pyodide.ffi.ConversionError: Conversion from python to javascript failed
```

### 解决方案

**Python 策略中禁止使用任何 emoji 表情符号！** 全部用纯文本方括号标签代替：

| ❌ 错误（emoji） | ✅ 正确（纯文本标签） |
|---|---|
| `"📊 回测模式"` | `"[回测] 回测模式"` |
| `"📈 金叉信号"` | `"[金叉] 金叉信号"` |
| `"📉 死叉信号"` | `"[死叉] 死叉信号"` |
| `"🔄 平仓"` | `"[平仓] 平仓"` |
| `"✅ 成功"` | `"[成功] 成功"` |
| `"❌ 失败"` | `"[失败] 失败"` |
| `"⏳ 等待"` | `"[等待] 等待"` |
| `"🛑 停止"` | `"[停止] 停止"` |
| `"🚀 实盘"` | `"[实盘] 实盘"` |

也可以用纯 ASCII 符号替代：

```python
Log("[+] 开多仓成功", "#00FF00")
Log("[-] 开多仓失败", "#FF0000")
Log("[*] 收到信号", "#FFA500")
Log("[.] 等待数据...")
```

**注意**：此问题仅存在于 FMZ Python 环境，JavaScript 策略不受影响。

---

## 14. 持仓查询一致性

### 问题

在主循环中多次调用 `exchange.GetPosition()` 时，如果只在循环开头查一次，然后执行交易操作（平仓/开仓），后面再使用之前的持仓快照就会数据过期，导致 `IndexError`：

```python
# ❌ 错误示例
pos = exchange.GetPosition()          # 第 1 次查询
hasPosition = pos is not None and len(pos) > 0

# ... 平仓操作 ...
exchange.SetDirection("closebuy")
exchange.Buy(-1, abs(pos[0].Amount))  # 平仓后 pos 还是旧数据

pos = exchange.GetPosition()          # 第 2 次查询，pos 被更新
# 但 hasPosition 没更新！

# ... 状态栏展示 ...
if hasPosition:                       # 使用的是过期的 hasPosition！
    posType = "多头" if pos[0].Type == PD_LONG else "空头"  # 可能 IndexError!
```

### 解决方案

**每次需要使用持仓信息时重新查询，且同步更新 all 相关的状态变量**：

```python
# ✅ 正确做法

# 1. 交易前查询
pos = exchange.GetPosition()
hasPosition = pos is not None and len(pos) > 0

# ... 执行交易操作 ...

# 2. 交易后重新查询（pos 和 hasPosition 成对更新）
pos = exchange.GetPosition()
hasPosition = pos is not None and len(pos) > 0  # ← 同步更新！

# 3. 状态栏展示时再次独立查询（用单独的变量名，避免混淆）
currentPos = exchange.GetPosition()
currentHasPos = currentPos is not None and len(currentPos) > 0
if currentHasPos:
    posType = "多头" if currentPos[0].Type == PD_LONG else "空头"
```

**核心原则**：
- 交易操作前后必须重新查询持仓
- `pos` 和 `hasPosition` 必须同时更新，不能只更新一个
- 不同用途的持仓查询用不同变量名（如 `pos` vs `currentPos`），防止误用
