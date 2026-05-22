---
name: fmz-skill
description: FMZ 量化交易平台 API 开发助手。当用户需要编写 FMZ（发明者量化）交易策略、查阅 FMZ API 文档、调试策略代码、了解实盘/回测机制、编写数字货币交易机器人时使用。触发关键词：FMZ、发明者量化、量化交易策略、FMZ API、FMZ 文档、实盘、回测、数字货币交易机器人、GetTicker、GetDepth、Buy、Sell、exchange.GetAccount、_C、_G、_N、_D、_Cross、Dial、HttpQuery、SetContractType。
---

# FMZ 量化交易平台开发助手

## 何时使用

当用户请求涉及以下任一场景时激活本 skill：
- 编写、修改或调试 FMZ 量化交易策略代码（JavaScript/Python/C++）
- 查阅 FMZ 平台 API 函数（Global 函数、exchange 成员函数、TA/Talib 指标等）
- 理解 FMZ 实盘与回测环境的差异
- 编写数字货币交易机器人、网格策略、CTA 趋势跟踪、套利策略等
- 使用 FMZ 的 WebSocket（Dial）、HTTP 请求（HttpQuery）、数据库（DBExec）等功能
- 处理多线程、实盘间通信、错误过滤等高级特性

## 何时不使用

- 用户问的是其他量化平台（如 QuantConnect、Backtrader、VN.PY 等）——本 skill 仅覆盖 FMZ
- 纯 Python/JavaScript/C++ 语言语法问题——应使用通用编程帮助
- 区块链/Web3 合约开发——FMZ 的 Web3 模块仅用于简单的链上交互

## FMZ 平台概述

FMZ（发明者量化）是一个量化交易 SaaS 平台，支持通过 JavaScript、Python、C++ 编写策略并在云端托管运行。核心架构：

- **策略代码** → 运行在 **托管者**（Docker 容器）上 → 通过 **exchange** 对象访问交易所 API
- 每个策略实例称为一个**实盘**（Robot），有唯一实盘 ID
- 支持**回测系统**（`IsVirtual()` 返回 `true`）和**实盘环境**

### 三大语言对比

| 特性 | JavaScript | Python | C++ |
|------|-----------|--------|-----|
| 推荐度 | ⭐⭐⭐ 首选 | ⭐⭐ | ⭐ |
| 时间戳 | 毫秒 | **秒** | 毫秒 |
| _D() 格式 | `yyyy-MM-dd hh:mm:ss` | `%Y-%m-%d %H:%M:%S` | `%Y-%m-%d %H:%M:%S` |
| Dial 支持 | 完整（含 mqtt/nats/amqp/kafka） | 基础 | 基础 |
| HttpQuery | ✅ | ❌（用 urllib） | ✅ |
| HttpQuery_Go | ✅ | ❌ | ❌ |
| JSON.parse/stringify | ✅（扩展 safeStr） | ❌（用 json 库） | ❌（用 json::parse） |
| __Serve | ✅ | ❌ | ❌ |
| exchange.Go | ✅ | ✅ | ✅ |

## 输入要求

开始工作前，确认以下信息：
1. **目标语言**：JavaScript / Python / C++
2. **运行环境**：实盘 / 回测 / 两者兼容
3. **交易所**：如币安、OKX、火币等
4. **策略类型**：CTA、网格、套利、马丁格尔等（如适用）

## 核心工作流

### 1. 查阅 API → 读取 `references/api-reference.md`

当需要查看函数签名、参数、返回值、多语言示例时，读取该文件。包含：
- 30+ 个 Global 函数完整参考
- 12 个分类的所有 exchange 成员函数
- 结构体定义（Ticker, Depth, Order, Account, Position 等）
- 内置变量和常量（PERIOD_*, ORDER_STATE_*, 等）

### 2. 选择策略模板 → 读取 `references/strategy-templates.md`

包含常见策略类型的骨架代码（JavaScript 版）：
- CTA 趋势跟踪（EMA 交叉）
- 网格交易
- 马丁格尔
- 套利策略（跨交易所价差）
- 冰山委托/做市策略
- 实盘间通信（SetChannelData/GetChannelData）

### 3. 遵循最佳实践 → 读取 `references/best-practices.md`

涵盖：
- 容错处理（_C / _CDelay）
- 错误过滤（SetErrorFilter）
- 日志与状态显示（LogStatus 表格）
- 持久化存储（_G / DBExec）
- 回测兼容性处理（IsVirtual）
- Sleep 使用规范（Python 避免 time.sleep）
- 精度处理（_N / exchange.SetPrecision）

### 4. 选择语言 → 读取 `references/language-guide.md`

多语言差异详解、时间戳处理、HTTP 请求替代方案、WebSocket 使用差异、数据序列化方案。

### 5. 遇到问题 → 读取 `references/faq.md`

常见陷阱和解决方案。

## 快速参考：最常用 API

### 生命周期
```
main()          — 策略入口
onexit()        — 策略退出时调用（清理资源）
Sleep(ms)       — 休眠毫秒数
IsVirtual()     — 判断是否回测环境
```

### 行情获取（Market）
```js
exchange.SetContractType("BTC_USDT")  // 设置交易对（期货必备）
exchange.GetTicker()       // 行情 ticker → {Last, High, Low, Buy, Sell, Volume, Time}
exchange.GetDepth()        // 深度 → {Asks:[{Price,Amount}], Bids:[{Price,Amount}], Time}
exchange.GetRecords(k周期) // K线 → [{Time, Open, High, Low, Close, Volume}]
exchange.GetTrades()       // 逐笔成交 → [{Id, Time, Price, Amount, Type}]
```

### 交易执行（Trade）
```js
exchange.Buy(price, amount)               // 买入（price=-1 为市价）
exchange.Sell(price, amount)              // 卖出（price=-1 为市价）
exchange.GetOrder(orderId)                // 查询订单 → {Id, Price, Amount, DealAmount, Status, Type}
exchange.GetOrders()                      // 获取所有未完成订单
exchange.CancelOrder(orderId)             // 取消订单
exchange.SetPrecision(pricePrec, amtPrec) // 设置精度（下单前必须调用）
```

### 账户（Account）
```js
exchange.GetAccount()  // → {Balance, FrozenBalance, Stocks, FrozenStocks}
exchange.GetPosition() // 期货持仓（需先 SetContractType）
```

### 容错与工具
```js
_C(exchange.GetTicker)           // 自动重试直到成功
_CDelay(2000)                     // 设置 _C 重试间隔(ms)
_N(num, precision)               // 格式化浮点数精度
_D(timestamp, format)            // 时间戳转字符串
_G("key", value)                 // 持久化存储（实盘）
_G("key")                        // 读取持久化值
_G(null)                         // 清除所有持久化数据
```

## 文件索引

| 文件 | 用途 | 何时读取 |
|------|------|----------|
| [`references/api-reference.md`](references/api-reference.md) | 完整 API 参考（所有函数、参数、返回值、示例） | 查阅任何 API 函数时 |
| [`references/strategy-templates.md`](references/strategy-templates.md) | 常见策略模板 | 需要策略骨架代码时 |
| [`references/best-practices.md`](references/best-practices.md) | 最佳实践与规范 | 编写/审查策略代码时 |
| [`references/language-guide.md`](references/language-guide.md) | 三语言差异详解 | 跨语言移植或多语言开发时 |
| [`references/faq.md`](references/faq.md) | 常见问题与陷阱 | 遇到报错或疑惑时 |

## 在线文档

FMZ 官方语法手册：https://www.fmz.com/syntax-guide
