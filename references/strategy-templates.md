# FMZ 策略模板

> 常见量化交易策略骨架代码。所有模板为 JavaScript 语言，运行前需添加交易所并设置交易对。

## 目录

- [1. CTA 趋势跟踪（EMA 交叉）](#1-cta-趋势跟踪ema-交叉)
- [2. 网格交易策略](#2-网格交易策略)
- [3. 马丁格尔策略](#3-马丁格尔策略)
- [4. 跨交易所价差套利](#4-跨交易所价差套利)
- [5. 冰山委托/做市策略](#5-冰山委托做市策略)
- [6. 实盘间通信（频道）](#6-实盘间通信频道)

---

## 1. CTA 趋势跟踪（EMA 交叉）

最经典的趋势策略：快线上穿慢线做多，下穿做空。

```js
// 参数
var fastPeriod = 5
var slowPeriod = 20

function main() {
    // 期货需设置合约类型
    exchange.SetContractType("swap")
    exchange.SetMarginLevel(10)
    exchange.SetPrecision(2, 3)

    while (true) {
        var records = _C(exchange.GetRecords, PERIOD_H1)
        if (!records || records.length < slowPeriod + 5) {
            Sleep(1000)
            continue
        }

        var fastMA = TA.EMA(records, fastPeriod)
        var slowMA = TA.EMA(records, slowPeriod)

        var fastNow = fastMA[fastMA.length - 1]
        var slowNow = slowMA[slowMA.length - 1]
        var cross = _Cross(fastMA, slowMA)

        var pos = exchange.GetPositions()
        // pos 为空表示无持仓；需要 SetContractType 后才能正常返回

        if (cross > 0 && cross <= 2) {  // 近期金叉
            if (pos && pos.length > 0) {
                // 有空头则先平仓
                exchange.SetDirection("closebuy")
                exchange.Buy(-1, Math.abs(pos[0].Amount))
            }
            exchange.SetDirection("buy")
            var acc = _C(exchange.GetAccount)
            var amount = _N(acc.Balance * 0.95 / fastNow, 3)
            exchange.Buy(-1, amount)
            Log("开多", fastNow, amount, "#FF0000")
        } else if (cross < 0 && cross >= -2) {  // 近期死叉
            if (pos && pos.length > 0) {
                exchange.SetDirection("closesell")
                exchange.Sell(-1, Math.abs(pos[0].Amount))
            }
            exchange.SetDirection("sell")
            var acc = _C(exchange.GetAccount)
            var amount = _N(acc.Balance * 0.95 / fastNow, 3)
            exchange.Sell(-1, amount)
            Log("开空", fastNow, amount, "#00FF00")
        }

        LogStatus(_D(), "\n快线:", fastNow, "\n慢线:", slowNow)
        Sleep(60 * 1000)
    }
}
```

---

## 2. 网格交易策略

在价格区间内布设等差数列挂单，低买高卖赚取波动收益。

```js
// 网格参数
var gridCount = 10          // 网格层数
var gridLower = 40000       // 网格下限
var gridUpper = 50000       // 网格上限
var totalInvest = 100       // 总投资额(USDT)

function main() {
    exchange.SetPrecision(2, 3)
    var perAmount = _N(totalInvest / gridCount, 3)

    // 计算网格价格
    var gridPrices = []
    var step = (gridUpper - gridLower) / (gridCount - 1)
    for (var i = 0; i < gridCount; i++) {
        gridPrices.push(_N(gridLower + step * i, 2))
    }

    while (true) {
        var ticker = _C(exchange.GetTicker)
        if (!ticker) { Sleep(5000); continue }

        var currentPrice = ticker.Last

        // 取消所有旧网格订单
        var orders = _C(exchange.GetOrders)
        for (var i = 0; i < orders.length; i++) {
            exchange.CancelOrder(orders[i].Id)
        }

        // 重新布设网格
        for (var j = 0; j < gridPrices.length; j++) {
            if (gridPrices[j] < currentPrice) {
                // 价格下方：挂买单
                exchange.Buy(gridPrices[j], perAmount)
            } else if (gridPrices[j] > currentPrice) {
                // 价格上方：挂卖单
                exchange.Sell(gridPrices[j], perAmount)
            }
        }

        var table = {
            type: 'table', title: '网格状态',
            cols: ['当前价', '下限', '上限', '层数', '每层数量'],
            rows: [[currentPrice, gridLower, gridUpper, gridCount, perAmount]]
        }
        LogStatus('`' + JSON.stringify(table) + '`')
        Sleep(30 * 1000)  // 30秒刷新
    }
}
```

---

## 3. 马丁格尔策略

亏损后加仓的逆势策略（高风险！需要充足资金）。

```js
var baseAmount = 0.01       // 基础开仓量
var multiplier = 2          // 加仓倍数
var maxLayers = 5           // 最大加仓层数
var takeProfit = 0.02       // 止盈比例 2%
var priceStep = 0.01        // 加仓间隔 1%

function main() {
    exchange.SetContractType("swap")
    exchange.SetMarginLevel(20)
    exchange.SetPrecision(2, 3)

    var currentLayer = 0
    var entryPrice = 0
    var currentAmount = baseAmount

    while (true) {
        var ticker = _C(exchange.GetTicker)
        if (!ticker) { Sleep(5000); continue }
        var price = ticker.Last

        var pos = exchange.GetPositions()
        var hasPosition = pos && pos.length > 0

        if (!hasPosition && currentLayer == 0) {
            // 初始开仓（做多）
            exchange.SetDirection("buy")
            exchange.Buy(-1, currentAmount)
            entryPrice = price
            currentLayer = 1
            Log("初始开仓", price, currentAmount, "#FF0000")
        }

        if (hasPosition) {
            var avgPrice = pos[0].Price
            var profitRate = (price - avgPrice) / avgPrice

            if (profitRate >= takeProfit) {
                // 止盈，全部平仓
                exchange.SetDirection("closebuy")
                exchange.Sell(-1, Math.abs(pos[0].Amount))
                currentLayer = 0
                currentAmount = baseAmount
                Log("止盈平仓", price, "#00FF00")
            } else if (profitRate <= -(currentLayer * priceStep)) {
                // 亏损加仓
                if (currentLayer < maxLayers) {
                    var addAmount = _N(currentAmount * multiplier, 3)
                    exchange.SetDirection("buy")
                    exchange.Buy(-1, addAmount)
                    currentAmount = addAmount
                    currentLayer++
                    Log("加仓第" + currentLayer + "层", price, addAmount, "#FF0000")
                }
            }
        }

        var table = {
            type: 'table', title: '马丁格尔状态',
            cols: ['价格', '层数', '均价', '当前数量'],
            rows: [[price, currentLayer, hasPosition ? pos[0].Price : 0, currentAmount]]
        }
        LogStatus('`' + JSON.stringify(table) + '`')
        Sleep(10000)
    }
}
```

---

## 4. 跨交易所价差套利

监控两个交易所同一交易对价差，价差大于阈值时套利。

```js
var spreadThreshold = 0.005  // 价差阈值 0.5%

function main() {
    if (exchanges.length < 2) {
        throw "需要添加至少2个交易所"
    }
    var exA = exchanges[0]
    var exB = exchanges[1]

    exA.SetPrecision(2, 3)
    exB.SetPrecision(2, 3)

    while (true) {
        var tickerA = _C(exA.GetTicker)
        var tickerB = _C(exB.GetTicker)

        if (!tickerA || !tickerB) { Sleep(5000); continue }

        var spread = (tickerA.Buy - tickerB.Sell) / tickerB.Sell

        var table = {
            type: 'table', title: '价差监控',
            cols: ['交易所A买一', '交易所B卖一', '价差%', '阈值%'],
            rows: [[tickerA.Buy, tickerB.Sell, _N(spread * 100, 3), spreadThreshold * 100]]
        }
        LogStatus('`' + JSON.stringify(table) + '`')

        if (spread > spreadThreshold) {
            // A 买入，B 卖出
            var accA = _C(exA.GetAccount)
            var amount = _N(accA.Balance * 0.5 / tickerA.Buy, 3)
            exA.Buy(tickerA.Buy, amount)
            exB.Sell(tickerB.Sell, amount)
            Log("套利执行!", tickerA.Buy, tickerB.Sell, amount, "#FF0000")
        }

        Sleep(5000)
    }
}
```

---

## 5. 冰山委托/做市策略

将大单拆分成多个小单隐藏真实意图。

```js
var totalAmount = 1.0        // 总数量
var sliceCount = 10          // 拆分份数
var priceOffset = 0.001      // 价格偏移

function main() {
    exchange.SetPrecision(2, 3)

    while (true) {
        var ticker = _C(exchange.GetTicker)
        if (!ticker) { Sleep(5000); continue }

        // 取消所有未完成订单
        var orders = _C(exchange.GetOrders)
        for (var i = 0; i < orders.length; i++) {
            exchange.CancelOrder(orders[i].Id)
        }
        Sleep(1000)

        var sliceAmount = _N(totalAmount / sliceCount, 3)
        var basePrice = ticker.Last

        // 布设冰山买单和卖单
        for (var j = 0; j < sliceCount; j++) {
            var buyPrice = _N(basePrice * (1 - priceOffset * (j + 1)), 2)
            var sellPrice = _N(basePrice * (1 + priceOffset * (j + 1)), 2)
            exchange.Buy(buyPrice, sliceAmount)
            exchange.Sell(sellPrice, sliceAmount)
        }

        LogStatus(_D(), "冰山委托已布设", basePrice, sliceCount * 2, "单")
        Sleep(60 * 1000)
    }
}
```

---

## 6. 实盘间通信（频道）

### 广播端 — 发布行情数据

```js
function main() {
    var robotId = _G()
    while (true) {
        var ticker = _C(exchange.GetTicker)
        if (!ticker) { Sleep(5000); continue }

        var data = {
            symbol: "BTC_USDT",
            price: ticker.Last,
            time: Date.now()
        }
        SetChannelData(data)
        LogStatus("广播端 [Bot " + robotId + "]", _D(), data.price)
        Sleep(60000)
    }
}
```

### 订阅端 — 接收并跟单

```js
function main() {
    var sourceRobotId = "632799"  // 替换为广播端实盘ID

    while (true) {
        var data = GetChannelData(sourceRobotId)
        if (data !== null) {
            Log("收到频道数据:", data.price, _D(data.time))
            // 在此执行跟单逻辑
        } else {
            Log("等待首次数据同步...")
        }
        Sleep(5000)
    }
}
```

### 跨平台接入（TradingView Webhook）

外部系统通过 HTTP POST 发送到：
```
https://www.fmz.com/api/v1?method=pub&robot={实盘ID}&channel={UUID}
```

策略端订阅：
```js
function main() {
    var uuid = "6BC42A119B5DBFA2188A8279DA3B5C30"
    while (true) {
        var data = GetChannelData(uuid)
        if (data !== null) {
            Log("外部信号:", data)
            // 解析 data.action, data.symbol, data.price 执行交易
        }
        Sleep(1000)
    }
}
```
