# FMZ 常见问题与陷阱

> FMZ 开发中常见错误及解决方案。

## 目录

- [1. 下单相关](#1-下单相关)
- [2. HTTP/网络相关](#2-http网络相关)
- [3. 回测相关](#3-回测相关)
- [4. 时间相关](#4-时间相关)
- [5. WebSocket/Dial 相关](#5-websocketdial-相关)
- [6. 多线程相关](#6-多线程相关)
- [7. 存储相关](#7-存储相关)
- [8. Python 特定问题](#8-python-特定问题)
- [9. C++ 特定问题](#9-c-特定问题)

---

## 1. 下单相关

### Q: 下单失败，报错 "invalid amount precision"
**原因**: 未调用 `exchange.SetPrecision()` 或精度设置不正确。

**解决**:
```js
exchange.SetPrecision(2, 3)  // 币安 BTC/USDT: 价格2位, 数量3位
```

### Q: Buy/Sell 返回的订单 ID 为 null
**原因**: `_C()` 包裹下单函数后，_C 期望返回非空值，但下单本身可能被 _C 干扰；或交易所 API 返回异常。

**解决**: 不要用 `_C()` 包裹下单函数。
```js
var orderId = exchange.Buy(-1, 0.01)  // 直接调用
if (!orderId) {
    Log("下单失败:", GetLastError())
}
```

### Q: 期货下单方向不对
**原因**: 未调用 `SetDirection()`。

**解决**:
```js
exchange.SetContractType("swap")
exchange.SetDirection("buy")      // 或 "sell"
exchange.Buy(-1, amount)
```

### Q: CancelOrder 后订单仍然存在
**原因**: 取消是异步的，需要等待。

**解决**:
```js
exchange.CancelOrder(orderId)
Sleep(1000)  // 等待取消生效
var order = exchange.GetOrder(orderId)
if (order.Status == ORDER_STATE_CANCELED) {
    Log("取消成功")
}
```

---

## 2. HTTP/网络相关

### Q: HttpQuery 返回空或超时
**原因**: 回测中 HttpQuery 仅支持 GET 且同 URL 最多 20 次；实盘中可能网络不通。

**解决**:
- 回测：减少 URL 变体，合并请求
- 实盘：检查托管者网络，使用 `options.timeout` 设置超时
```js
var options = { timeout: 10000 }  // 10秒超时
var res = HttpQuery(url, options)
```

### Q: HttpQuery 代理不生效
**格式**:
```js
// socks5 代理
HttpQuery("socks5://user:pwd@ip:port/http://target/")
// 仅对当前调用生效，非全局
```

### Q: Python 中如何发 HTTP 请求？
Python 不支持 `HttpQuery`，使用标准库：
```python
import urllib.request, json
res = urllib.request.urlopen(url).read().decode('utf-8')
data = json.loads(res)
```

---

## 3. 回测相关

### Q: 回测结果和实盘不一致
**常见原因**:
1. `IsVirtual()` 区分逻辑有误
2. 回测中 `DBExec`/`_G`/`Dial` 等不支持
3. 回测中 `HttpQuery` 缓存导致数据过期
4. Python 用了 `time.sleep()` 而非 `Sleep()`

**解决**: 在回测中充分使用 `Log()` 输出中间值对比。

### Q: 回测速度极慢
**原因**: 如果是 Python 策略，可能使用了 `time.sleep()`。

**解决**: 替换所有 `time.sleep(n)` 为 `Sleep(n*1000)`。

### Q: 回测中 exchange.GetAccount() 余额不对
回测系统使用模拟余额，初始值在策略参数中设置。检查回测参数中的初始资产配置。

---

## 4. 时间相关

### Q: _D() 显示 1970 年
**原因**: 时间戳单位错误。JS/C++ 需要毫秒，Python 需要秒。

**解决**:
```js
// JS: 毫秒
Log(_D(Date.now()))
// Python: 秒
Log(_D(int(time.time())))
```

### Q: K 线数据时间戳解析错误
Record.Time 是毫秒级（所有语言统一），但 `_D()` 在不同语言中期望不同单位：
```js
// JS: 直接用
Log(_D(records[0].Time))

// Python: 需要除以 1000
Log(_D(int(records[0].Time / 1000)))
```

### Q: 回测中时间推进不正确
确保使用 `Sleep()` 而非 `time.sleep()`，使用 `exchange.GetRecords()` 获取回测时间序列。

---

## 5. WebSocket/Dial 相关

### Q: Dial 连接后 read() 一直阻塞
**原因**: 无参数的 `read()` 在无数据时阻塞。

**解决**: 根据场景选择合适的 read 参数：
```js
ws.read(1000)   // 1秒超时
ws.read(-1)     // 立即返回（有数据返数据，无数据返null）
ws.read(-2)     // 立即返回最新数据，丢弃缓冲区旧数据
```

### Q: WebSocket 自动重连不生效
**原因**: 未在地址参数中设置 `reconnect=true`。

**解决**:
```js
var ws = Dial("wss://api.example.com/ws|reconnect=true&interval=5000")
```

### Q: read() 返回空字符串但连接未断开
`read()` 返回 `""` 表示连接断开或出错。检查：
1. 是否设置了超时参数导致超时返回
2. 服务器是否主动关闭了连接
3. 网络是否稳定

### Q: WebSocket 数据积压，处理滞后
实时推送频率高，策略主循环处理慢。

**解决**:
- 使用 `ws.read(-2)` 只取最新数据
- 优化主循环中数据处理速度
- 将数据处理放到单独线程

---

## 6. 多线程相关

### Q: 线程中无法访问 exchange 对象
**原因**: 线程间变量不共享，exchange 对象在主线程中定义。

**解决**: 通过参数传递或使用线程安全数据结构（ThreadDict）。

### Q: EventLoop 错过事件
**原因**: 在并发调用 exchange.Go() 后才首次调用 EventLoop()。

**解决**: 在主循环开始前调用一次 EventLoop(-1) 初始化监听：
```js
EventLoop(-1)  // 初始化事件监听
var r1 = exchange.Go("GetTicker")
var r2 = exchange.Go("GetDepth")
var event = EventLoop(0)  // 等待事件
```

---

## 7. 存储相关

### Q: _G() 保存的数据丢失
**原因**: 
- 回测结束后会清除
- 调用了 `_G(null)` 清除所有数据
- 数据库文件损坏或被删除

**数据位置**: 托管者目录 `/logs/storage/{robotId}/{robotId}.db3`

### Q: DBExec 报错 "no such table"
需要先创建表：
```js
DBExec(":CREATE TABLE IF NOT EXISTS my_table (id TEXT PRIMARY KEY, val REAL)")
```

### Q: DBExec 数据没有持久化
检查 SQL 是否以 `:` 开头（内存数据库，重启丢失）。

**文件数据库**（持久化）:
```js
DBExec("CREATE TABLE IF NOT EXISTS my_table (...)")
```

---

## 8. Python 特定问题

### Q: Encode 报错 "missing argument"
Python 中 Encode 必须传入所有 6 个参数，用空字符串占位：
```python
Encode("sha256", "raw", "hex", "data", "", "")
```

### Q: _G() 无参调用不返回实盘ID
Python 中应使用 `_G()` 返回实盘 ID，如果返回 None，检查实盘是否已创建。

### Q: JSON 数字精度丢失
Python 标准 json 库对大数字可能有精度问题。考虑使用 `decimal` 模块处理。

### Q: 策略参数变量报 NameError
**原因**: FMZ 策略参数还没在网页编辑器「策略参数」面板中添加，直接运行代码导致变量未定义。

**错误示例**:
```
NameError: name 'FastPeriod' is not defined
```

**解决**: 在 `main()` 开头对每个策略参数做 try/except 兜底：
```python
def main():
    try:
        FastPeriod
    except NameError:
        FastPeriod = 5         # 默认值

    try:
        SlowPeriod
    except NameError:
        SlowPeriod = 20
    # ... 其他参数同理
```

**原理**: FMZ 先注入面板参数再执行策略。面板有参数时 try 不触发；没有时 except 赋默认值。

### Q: _D() 显示 1970 年（Python 回测）
**原因**: `time.time()` 在 FMZ Python 回测环境中可能返回模拟时间（非真实秒级时间戳），导致 `_D()` 解析为 1970 年。

**解决**（按推荐度排序）:
```python
# 方案 1（最推荐）：无参调用，FMZ 自动取当前时间
_D()

# 方案 2：使用 FMZ 内置 Unix() 获取秒级时间戳
_D(Unix())

# 方案 3（不推荐）：time.time() 在回测中不可靠
# _D(int(time.time()))     ← 可能导致 1970 年！
```

### Q: Log 中使用 emoji 报 pyodide.ffi.ConversionError
**原因**: FMZ Python 环境基于 Pyodide（Python → JavaScript 桥接），emoji 字符无法在两种语言间正确转换。

**错误示例**:
```
pyodide.ffi.JsException: RangeError: Invalid code point NaN
pyodide.ffi.ConversionError: Conversion from python to javascript failed
```

**解决**: Python 策略中**禁止使用任何 emoji**，全部用纯文本方括号标签代替：
```python
# ❌ 错误
Log("📈 金叉信号", "#FF0000")

# ✅ 正确
Log("[金叉] 金叉信号", "#FF0000")
Log("[+] 开多成功", "#00FF00")
Log("[-] 开多失败", "#FF0000")
```

### Q: 持仓查询后 pos[0] 报 IndexError
**原因**: 多次调用 `exchange.GetPosition()` 时，只更新了 `pos` 变量但没有同步更新 `hasPosition` 等关联变量。交易操作后 pos 可能变成空列表，但 `hasPosition` 仍是旧的 `True`，导致访问 `pos[0]` 越界。

**解决**:
- 交易操作前后必须重新查询持仓并同步更新所有关联变量
- 不同用途的持仓查询用不同变量名（如 `pos` vs `currentPos`），避免混淆

```python
# ✅ 状态栏展示时独立查询
currentPos = exchange.GetPosition()
currentHasPos = currentPos is not None and len(currentPos) > 0
if currentHasPos:
    posType = "多头" if currentPos[0].Type == PD_LONG else "空头"
```

---

## 9. C++ 特定问题

### Q: exchange.Go().wait() 不返回值
C++ 的 `wait()` 需要通过引用参数接收：
```cpp
Ticker ticker;
auto r = exchange.Go("GetTicker");
r.wait(ticker);  // 传入 ticker 引用
Log(ticker.Last);
```

### Q: 数组不能用 [] 初始化
C++ 中数组用 vector:
```cpp
vector arr = {1, 2, 3};
```

### Q: JSON 操作复杂
C++ 使用内置的 nlohmann/json 库：
```cpp
json obj = json::parse(str);
string val = obj["key"];
obj["newKey"] = 123;
```

### Q: Dial 连接检查
C++ 用 `.Valid` 而非布尔判断：
```cpp
auto ws = Dial("wss://...");
if (ws.Valid) { /* 已连接 */ }
```
