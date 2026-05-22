# FMZ 多语言开发指南

> JavaScript、Python、C++ 三语言的差异详解及互迁指南。

## 目录

- [1. 语言选择建议](#1-语言选择建议)
- [2. 时间戳差异](#2-时间戳差异)
- [3. API 兼容性矩阵](#3-api-兼容性矩阵)
- [4. HTTP 请求](#4-http-请求)
- [5. 数据序列化](#5-数据序列化)
- [6. WebSocket/Dial](#6-websocketdial)
- [7. C++ 特殊语法](#7-c-特殊语法)
- [8. Python 特殊注意](#8-python-特殊注意)
- [9. 代码迁移速查](#9-代码迁移速查)

---

## 1. 语言选择建议

| 语言 | 推荐场景 | 原因 |
|------|---------|------|
| **JavaScript** | 首选，绝大多数策略 | API 最完整（Dial 全部协议、__Serve、HttpQuery_Go、JSON 扩展），文档示例最丰富 |
| **Python** | 需要 Python 生态库时 | API 支持较完整，回测兼容性好，但缺少部分高级功能 |
| **C++** | 高性能计算场景 | 运行时效率最高，但 API 支持最少，语法最复杂 |

---

## 2. 时间戳差异（重要！）

这是三语言之间最容易出错的差异。

| 语言 | 时间戳单位 | _D() 格式 |
|------|-----------|-----------|
| JavaScript | **毫秒** (ms) | `yyyy-MM-dd hh:mm:ss` |
| Python | **秒** (s) | `%Y-%m-%d %H:%M:%S` |
| C++ | **毫秒** (ms) | `%Y-%m-%d %H:%M:%S` |

### 常见错误

```js
// JavaScript: 毫秒级时间戳
var ts = new Date().getTime()          // 1700000000000
var ts = UnixNano() / 1000000          // 毫秒级
Log(_D(ts))                            // ✅ 正确

// ⚠️ 如果从 Python 代码复制了秒级时间戳到 JS
Log(_D(1700000000))                    // ❌ 显示1970年！
```

```python
# Python: 秒级时间戳
ts = time.time()                       # 1700000000.123
Log(_D(int(ts)))                       # ✅ 正确

# ⚠️ 如果从 JS 代码复制了毫秒级时间戳到 Python
Log(_D(1700000000000))                 # ❌ 显示非常远的未来！
```

### 互转

```js
// JS: 毫秒 → 秒
var secTs = Unix()  // 直接获取秒级时间戳
var secTs = Math.floor(Date.now() / 1000)

// Python: 秒 → 毫秒
msTs = int(time.time() * 1000)
```

---

## 3. API 兼容性矩阵

| 功能 | JavaScript | Python | C++ |
|------|-----------|--------|-----|
| Global 函数（Sleep, _C, _G, _N, _D, _Cross） | ✅ | ✅ | ✅ |
| exchange.Buy/Sell/GetOrder/GetOrders | ✅ | ✅ | ✅ |
| exchange.Go (并发) | ✅ | ✅ | ✅ |
| HttpQuery | ✅ | ❌ | ✅ |
| HttpQuery_Go | ✅ | ❌ | ❌ |
| Dial (TCP/WS) | ✅ | ✅ | ✅ |
| Dial (mqtt/nats/amqp/kafka) | ✅ | ❌ | ❌ |
| Dial (数据库连接) | ✅ | ❌ | ❌ |
| __Serve | ✅ | ❌ | ❌ |
| JSON.parse (扩展 safeStr) | ✅ | ❌ | ❌ |
| JSON.stringify | ✅ | ❌ | ❌ |
| EventLoop | ✅ | ✅ | ✅ |
| SetChannelData/GetChannelData | ✅ | ✅ | ⚠️ |
| Mail/Mail_Go | ✅ | ✅ | ✅ |
| Encode | ✅ | ✅ | ✅ |
| _G() 无参返回实盘ID | ✅ | ✅ | ❌ |

---

## 4. HTTP 请求

### JavaScript — HttpQuery

```js
// GET 请求
var res = HttpQuery("https://api.example.com/data")

// POST 请求（JSON body）
var options = {
    method: "POST",
    body: JSON.stringify({key: "value"}),
    headers: {"Content-Type": "application/json"},
    timeout: 5000  // 5秒超时
}
var res = HttpQuery("https://api.example.com/endpoint", options)

// 代理
HttpQuery("socks5://127.0.0.1:1080/http://api.example.com/")
```

### Python — urllib

```python
import urllib.request
import json

# GET
res = urllib.request.urlopen("https://api.example.com/data").read().decode('utf-8')

# POST
data = json.dumps({"key": "value"}).encode('utf-8')
req = urllib.request.Request("https://api.example.com/endpoint", data=data, method="POST")
req.add_header("Content-Type", "application/json")
res = urllib.request.urlopen(req).read().decode('utf-8')
```

### C++ — HttpQuery

```cpp
// GET
auto res = HttpQuery("https://api.example.com/data");

// POST
json options = R"({
    "method": "POST",
    "body": "{\"key\":\"value\"}",
    "headers": {"Content-Type": "application/json"}
})"_json;
auto res = HttpQuery("https://api.example.com/endpoint", options);
```

---

## 5. 数据序列化

### JavaScript — 原生 JSON

```js
var obj = {a: 1, b: "hello"}
var str = JSON.stringify(obj)          // → '{"a":1,"b":"hello"}'
var parsed = JSON.parse(str)           // → {a: 1, b: "hello"}

// 大数值安全解析
var bigNum = JSON.parse('{"num":8754613216564987646512354656874651651358}', true)
// → {"num": "8754613216564987646512354656874651651358"}  // 字符串，无精度丢失
```

### Python — json 库

```python
import json

obj = {"a": 1, "b": "hello"}
s = json.dumps(obj)                    # → '{"a": 1, "b": "hello"}'
parsed = json.loads(s)                 # → {"a": 1, "b": "hello"}
```

### C++ — json 库（nlohmann/json）

```cpp
// 解析字符串
auto obj = json::parse(R"({"a": 1, "b": "hello"})");

// 构造 JSON
json table = R"({
    "type": "table",
    "title": "Status",
    "cols": ["Key", "Value"],
    "rows": []
})"_json;
table["rows"].push_back({"price", 50000});

// 序列化
string s = table.dump();
```

---

## 6. WebSocket/Dial

三语言 Dial 函数签名一致，但细节处理不同：

```js
// JavaScript
var ws = Dial("wss://stream.binance.com:9443/ws/btcusdt@ticker")
if (ws) {
    while (true) {
        var data = ws.read()
        if (data) { /* 处理 */ }
        if (data === "") { /* 连接断开 */ break; }
    }
    ws.close()
}
```

```python
# Python
ws = Dial("wss://stream.binance.com:9443/ws/btcusdt@ticker")
if ws:
    while True:
        data = ws.read()
        if data:
            # 处理
            pass
        if data == "":
            break
    ws.close()
```

```cpp
// C++ — 用 .Valid 检查连接，用 "" 判断断开
auto ws = Dial("wss://stream.binance.com:9443/ws/btcusdt@ticker");
if (ws.Valid) {
    while (true) {
        auto data = ws.read();
        if (data != "") { /* 处理 */ }
        if (data == "") { break; }  // 断开
    }
    ws.close();
}
```

---

## 7. C++ 特殊语法

### 变量声明
```cpp
auto ticker = exchange.GetTicker();  // auto 自动推导
Ticker ticker;                        // 显式类型（用于 wait 解包）
```

### exchange.Go().wait() 解包
```cpp
auto r = exchange.Go("GetTicker");
Ticker ticker;
r.wait(ticker);   // 传入引用解包
Log(ticker.Last);
```

### vector 替代数组
```cpp
vector arr1 = {1,2,3,4,5,6,8,8,9};
vector arr2 = {2,3,4,5,6,7,7,7,7};
auto cross = _Cross(arr1, arr2);
```

### json 操作
```cpp
json obj = json::parse(str);
obj["key"] = "value";
string output = obj.dump();
```

---

## 8. Python 特殊注意

### Sleep vs time.sleep
```python
# ✅ FMZ 平台 Sleep
Sleep(1000)

# ❌ 标准库 time.sleep — 回测会极慢
import time; time.sleep(1)
```

### _D() 使用秒级时间戳
```python
# 当前时间
_D(int(time.time()))

# 从毫秒级转换
_D(int(js_timestamp / 1000))
```

### Encode 必须显式传 keyFormat
```python
Encode("sha256", "raw", "hex", "data", "", "")       # ✅ Python 写法
Encode("sha256", "raw", "hex", "data", "string", "key")  # ✅ 带 key
```

### 空值：None 而非 null
```python
_G("key", None)       # 删除键
_G(None)              # 删除所有键
```

---

## 9. 代码迁移速查

| 操作 | JavaScript | Python | C++ |
|------|-----------|--------|-----|
| 空值判断 | `if (!x)` | `if not x:` | `if (x == "")` |
| 数组长度 | `arr.length` | `len(arr)` | `arr.size()` |
| 数组最后元素 | `arr[arr.length-1]` | `arr[-1]` | `arr.back()` |
| 注释 | `//` 或 `/* */` | `#` | `//` 或 `/* */` |
| for 循环 | `for (var i=0;i<n;i++)` | `for i in range(n):` | `for (int i=0;i<n;i++)` |
| 字符串拼接 | `"a" + "b"` | `"a" + "b"` | `"a" + "b"` |
| 布尔值 | `true`/`false` | `True`/`False` | `true`/`false` |
| 空值 | `null` | `None` | `NULL` 或 `nullptr` |
| 当前时间(ms) | `Date.now()` | `int(time.time()*1000)` | `Unix()*1000` |
| 当前时间(s) | `Unix()` | `int(time.time())` | `Unix()` |
| JSON解析 | `JSON.parse(s)` | `json.loads(s)` | `json::parse(s)` |
| JSON序列化 | `JSON.stringify(o)` | `json.dumps(o)` | `obj.dump()` |
| WebSocket断开 | `data === ""` | `data == ""` | `data == ""` |
| Dial连接检查 | `if (ws)` | `if ws:` | `if (ws.Valid)` |
| exchange.Go结果 | `r.wait()` | `r.wait()` | `Type v; r.wait(v);` |
