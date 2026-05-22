# FMZ 量化交易平台开发助手

> Agent Skill for FMZ Quant Trading Platform — API 参考 · 策略模板 · 最佳实践 · 多语言指南

为 AI 编程助手（如 Roo Code / Claude Code）提供 FMZ（发明者量化）平台的完整开发知识库，覆盖 30+ Global 函数、12 个 API 分类、6 种策略模板、三语言差异详解及常见陷阱。

---

## 目录

- [特性](#特性)
- [文件结构](#文件结构)
- [触发关键词](#触发关键词)
- [快速开始](#快速开始)
- [覆盖范围](#覆盖范围)
  - [API 参考](#api-参考)
  - [策略模板](#策略模板)
  - [最佳实践](#最佳实践)
  - [多语言指南](#多语言指南)
  - [常见问题](#常见问题)
- [语言支持](#语言支持)
- [在线文档](#在线文档)
- [许可](#许可)

---

## 特性

| 特性 | 说明 |
|------|------|
| 📖 **完整 API 参考** | 30+ Global 函数 + 12 大分类的所有 exchange 成员函数，含签名、参数、返回值、代码示例 |
| 📊 **策略模板库** | CTA 趋势跟踪、网格交易、马丁格尔、跨交易所套利、冰山委托、实盘间通信 6 种骨架代码 |
| ✅ **最佳实践** | 容错处理 `_C`、精度处理 `_N`/`SetPrecision`、日志表格 `LogStatus`、持久化 `_G`/`DBExec`、回测兼容等 14 个专题 |
| 🌐 **多语言开发指南** | JavaScript / Python / C++ 三语言差异矩阵、API 兼容性表、代码迁移速查表 |
| 🐛 **FAQ 陷阱库** | 9 大类常见错误及解决方案：下单精度、时间戳单位、回测不一致、WebSocket 积压、Python emoji 崩溃等 |
| ⚡ **触发式工作流** | 按需读取 reference 文件，轻量高效 |

---

## 文件结构

```
fmz-skill/
├── SKILL.md                          # Skill 主文件（入口）
├── README.md                         # 本文件
└── references/
    ├── api-reference.md              # 完整 API 参考
    ├── strategy-templates.md         # 策略模板骨架代码
    ├── best-practices.md             # 最佳实践与编码规范
    ├── language-guide.md             # JavaScript/Python/C++ 三语言差异
    └── faq.md                        # 常见问题与陷阱
```

### 各文件职责

| 文件 | 用途 | 何时读取 |
|------|------|----------|
| [`SKILL.md`](SKILL.md) | Skill 定义、平台概述、工作流指南 | Skill 激活时自动加载 |
| [`references/api-reference.md`](references/api-reference.md) | 30+ Global 函数、12 类 exchange 成员函数、结构体、常量 | 查阅 API 签名/参数/示例 |
| [`references/strategy-templates.md`](references/strategy-templates.md) | CTA/网格/马丁/套利/冰山/频道通信等 JS 骨架代码 | 需要策略代码起点 |
| [`references/best-practices.md`](references/best-practices.md) | 容错、精度、日志、持久化、回测兼容等 14 个规范 | 编写/审查策略代码 |
| [`references/language-guide.md`](references/language-guide.md) | 三语言差异矩阵、API 兼容性表、代码迁移速查 | 跨语言移植或多语言开发 |
| [`references/faq.md`](references/faq.md) | 9 大类常见错误及解决方案 | 遇到报错或疑惑 |

---

## 触发关键词

当用户提及以下任一关键词时，AI 编程助手将自动激活本 Skill：

- **平台名称**: FMZ、发明者量化
- **功能**: 量化交易策略、实盘、回测、数字货币交易机器人
- **API 函数**: `GetTicker`、`GetDepth`、`Buy`、`Sell`、`exchange.GetAccount`
- **工具函数**: `_C`、`_G`、`_N`、`_D`、`_Cross`
- **高级功能**: `Dial`、`HttpQuery`、`SetContractType`

### 适用场景

- ✅ 编写/修改/调试 FMZ 策略代码（JavaScript / Python / C++）
- ✅ 查阅 FMZ API 函数签名、参数、返回值
- ✅ 理解实盘与回测环境差异
- ✅ 数字货币交易机器人开发
- ✅ WebSocket（Dial）、HTTP 请求、数据库操作等高级特性
- ✅ 多线程、实盘间通信、错误过滤

### 不适用场景

- ❌ 其他量化平台（QuantConnect、Backtrader、VN.PY 等）
- ❌ 纯编程语言语法问题
- ❌ 区块链/Web3 合约开发（FMZ 的 Web3 模块仅简单链上交互）

---

## 快速开始

### 安装 Skill

将 `fmz-skill` 目录放置到 Agent Skills 目录：

```
<workspace>/.roo/skills/fmz-skill/     # 项目级
# 或
<home>/.roo/skills/fmz-skill/          # 全局
```

### 使用示例

```
用户: 帮我写一个币安 BTC/USDT 的 EMA 金叉死叉策略，JavaScript 语言，实盘运行
```

AI 助手将自动：
1. 加载 [`SKILL.md`](SKILL.md) 获取平台上下文
2. 读取 [`references/strategy-templates.md`](references/strategy-templates.md) 获取 CTA 趋势跟踪模板
3. 读取 [`references/api-reference.md`](references/api-reference.md) 确认 `TA.EMA`、`_Cross` 等函数签名
4. 读取 [`references/best-practices.md`](references/best-practices.md) 确保容错、精度、日志等符合规范

---

## 覆盖范围

### API 参考

[`references/api-reference.md`](references/api-reference.md) 覆盖 FMZ 平台全部 API：

| 分类 | 函数数量 | 核心函数 |
|------|---------|---------|
| Global 全局函数 | 30+ | `Sleep`, `_C`, `_G`, `_N`, `_D`, `_Cross`, `Dial`, `HttpQuery`, `Encode` |
| Log 日志 | 11 | `Log`, `LogProfit`, `LogStatus`, `Chart`, `KLineChart` |
| Market 行情 | 12 | `GetTicker`, `GetDepth`, `GetRecords`, `GetTrades` |
| Trade 交易 | 20 | `Buy`, `Sell`, `GetOrder`, `GetOrders`, `CancelOrder`, `SetPrecision` |
| Account 账户 | 7 | `GetAccount`, `GetAssets`, `GetName` |
| Futures 期货 | 6 | `GetPositions`, `SetDirection`, `SetMarginLevel`, `SetContractType` |
| NetSettings 网络 | 4 | `SetBase`, `SetProxy`, `SetTimeout` |
| Threads 多线程 | 10+ | `Thread`, `Lock`, `Condition`, `Event`, `ThreadDict` |
| Web3 区块链 | 9 | ABI 设置、合约调用、编解码 |
| TA 技术分析 | 13 | `TA.MA`, `TA.EMA`, `TA.MACD`, `TA.KDJ`, `TA.RSI`, `TA.BOLL` |
| Talib TA-Lib | 100+ | 蜡烛形态、数学函数、重叠研究、动量指标 |
| OS 系统 | 13 | `os.open`, `os.mmap`, `os.listFiles`, `os.stat` |

另含 7 个结构体定义（Ticker、Depth、Order、Account、Position 等）及全部内置常量。

### 策略模板

[`references/strategy-templates.md`](references/strategy-templates.md) 提供 6 种常见策略的 JavaScript 骨架代码：

1. **CTA 趋势跟踪** — EMA 快慢线交叉，含金叉/死叉检测、期货多空切换
2. **网格交易** — 等差数列布单，低买高卖
3. **马丁格尔** — 亏损加仓逆势策略，含止盈与最大层数控制
4. **跨交易所价差套利** — 双交易所价差监控与自动执行
5. **冰山委托/做市** — 大单拆分隐藏真实意图
6. **实盘间通信** — `SetChannelData`/`GetChannelData` 广播与订阅，含 TradingView Webhook 接入

### 最佳实践

[`references/best-practices.md`](references/best-practices.md) 涵盖 14 个关键规范：

- `_C()` 容错处理：函数引用非调用、重试间隔设置
- `SetErrorFilter()` 错误日志过滤：防止数据库膨胀
- `_N()` / `SetPrecision()` 精度处理：下单前必须调用
- `Sleep()` 休眠规范：Python 禁止 `time.sleep()`
- `LogStatus` 表格 + `Log` 颜色 + `LogProfit` 收益记录
- `_G()` / `DBExec()` 持久化存储规范
- `IsVirtual()` 回测兼容分支
- WebSocket 连接管理：重连、缓冲区、`onexit()` 清理
- Python 策略参数兜底、时间获取规范、emoji 禁用
- 持仓查询一致性：交易后必须同步更新

### 多语言指南

[`references/language-guide.md`](references/language-guide.md) 提供三语言全面对比：

- **时间戳差异**: JS/C++ 毫秒 vs Python 秒（最易出错的差异）
- **API 兼容性矩阵**: 27 项功能 × 3 语言逐项对比
- **HTTP 请求方案**: `HttpQuery`(JS/C++) vs `urllib`(Python)
- **数据序列化**: `JSON.parse/stringify`(JS) vs `json`(Python) vs `json::parse`(C++)
- **代码迁移速查表**: 17 个常用操作的三语言对照
- **C++ 特殊语法**: `wait()` 解包、vector、`ws.Valid` 检查
- **Python 特殊注意**: `Encode` 必须显式传参、`None` vs `null`

### 常见问题

[`references/faq.md`](references/faq.md) 收录 9 大类 20+ 个常见陷阱：

| 分类 | 典型问题 |
|------|---------|
| 下单 | 精度错误、订单 ID 为 null、方向不对、取消异步 |
| HTTP/网络 | 回测 GET 限制、代理配置、Python urllib 替代 |
| 回测 | 结果不一致、速度极慢、余额不对 |
| 时间 | `_D()` 显示 1970 年、K线时间戳单位错误 |
| WebSocket/Dial | read() 阻塞、重连不生效、数据积压 |
| 多线程 | exchange 对象不可访问、EventLoop 错过事件 |
| 存储 | `_G()` 数据丢失、DBExec 表不存在、内存库 vs 文件库 |
| Python 特定 | `Encode` 缺参数、NameError 兜底、emoji 崩溃、IndexError |
| C++ 特定 | `wait()` 解包语法、`ws.Valid` 检查、JSON 操作 |

---

## 语言支持

| 语言 | 推荐度 | 特点 |
|------|--------|------|
| **JavaScript** | ⭐⭐⭐ 首选 | API 最完整：Dial 全协议、`__Serve`、`HttpQuery_Go`、JSON 扩展 |
| **Python** | ⭐⭐ | 回测兼容性好，但缺少 `HttpQuery`、`__Serve` 等 |
| **C++** | ⭐ | 运行效率最高，但 API 最少、语法最复杂 |

详细对比见 [`references/language-guide.md`](references/language-guide.md)。

---

## 在线文档

- FMZ 官方语法手册: [https://www.fmz.com/syntax-guide](https://www.fmz.com/syntax-guide)
- FMZ 策略广场: [https://www.fmz.com/square](https://www.fmz.com/square)

---

## 许可

本 Skill 为 FMZ 量化交易平台的辅助开发工具，知识内容源自 FMZ 官方文档。仅供学习和策略开发使用。
