//@version=5
strategy('Double BCD Strategy v0.4 EMA Filter', overlay = true, max_bars_back = 500, default_qty_type = strategy.percent_of_equity, default_qty_value = 10, initial_capital = 10000, commission_type = strategy.commission.percent, commission_value = 0.04, slippage = 2, pyramiding = 0, process_orders_on_close = false, calc_on_every_tick = false)

import DevLucem/ZigLib/1 as ZigZag

/// INPUT — ZigZag
Depth     = input.int(5, '深さ', minval = 1, step = 1, inline = 'z1', group = 'ZigZag')
Deviation = input.int(5, '偏差', minval = 1, step = 1, inline = 'z1', group = 'ZigZag')
Backstep  = input.int(3, 'Backstep', minval = 2, step = 1, inline = 'z2', group = 'ZigZag')

/// INPUT — Pattern
minBars      = input.int(15, '2山間隔 最小(本) B↔D', minval = 3, step = 1, inline = 'b1', group = 'Pattern')
maxBars      = input.int(60, '2山間隔 最大(本) B↔D', minval = 10, step = 1, inline = 'b1', group = 'Pattern')
peakTol      = input.float(0.40, '2山の高さ許容(×山高)', minval = 0.0, maxval = 1.0, step = 0.05, group = 'Pattern')
heightRatio  = input.float(0.5, '右山の深さ比 (D-C)/(B-C)', minval = 0.1, step = 0.1, group = 'Pattern')
minHeightATR = input.float(1.5, '山高 最小(ATR×) 0=無制限', minval = 0.0, step = 0.1, group = 'Pattern')
trendFilter  = input.bool(true, 'トレンド整合', group = 'Pattern')

/// INPUT — Trade
tradeMTop = input.bool(true, 'M TOP で売り', inline = 't1', group = 'Trade')
tradeWBot = input.bool(true, 'W BOTTOM で買い', inline = 't1', group = 'Trade')

slBufATR   = input.float(0.0, '損切りバッファ (ATR×)', minval = 0.0, step = 0.1, group = 'Trade')
useATRexit = input.bool(false, 'ATR固定RR決済', group = 'Trade')
slATRmult  = input.float(1.5, '損切り (ATR×)', minval = 0.1, step = 0.1, group = 'Trade')
rrRatio    = input.float(2.0, 'リスクリワード比', minval = 0.5, step = 0.1, group = 'Trade')

/// INPUT — EMA FILTER
useEMAfilter = input.bool(true, '200EMAフィルター', group = 'Trend Filter')
emaLen       = input.int(200, 'EMA期間', minval = 10, step = 10, group = 'Trend Filter')

/// INPUT — Date Filter
useDateFilter = input.bool(false, '期間フィルタを使う', group = 'Backtest')
fromDate = input.time(timestamp('2020-01-01 00:00'), '開始', group = 'Backtest')
toDate   = input.time(timestamp('2026-01-01 00:00'), '終了', group = 'Backtest')

atrG = ta.atr(14)
ema200 = ta.ema(close, emaLen)

inWindow = not useDateFilter or (time >= fromDate and time <= toDate)

/// CALC FUNCTION
f_bcd() =>
    atr14 = ta.atr(14)

    [dir, z1, z2] = ZigZag.zigzag(low, high, Depth, Deviation, Backstep)

    cpx = z2.price
    cix = z2.index

    var array<float> pP  = array.new_float()
    var array<bool>  pHi = array.new_bool()
    var array<int>   pB  = array.new_int()

    pivNew = not na(dir) and not na(dir[1]) and dir != dir[1]

    if pivNew
        pivBar = na(cix[1]) ? -1 : int(cix[1])

        if pivBar >= 0
            qp = cpx[1]
            qh = dir[1] > 0

            needPush = array.size(pP) == 0

            if not needPush
                needPush := array.get(pHi, array.size(pHi) - 1) != qh

            if needPush
                array.push(pP, qp)
                array.push(pHi, qh)
                array.push(pB, pivBar)
            else
                nn = array.size(pP) - 1
                array.set(pP, nn, qp)
                array.set(pHi, nn, qh)
                array.set(pB, nn, pivBar)

            while array.size(pP) > 30
                array.shift(pP)
                array.shift(pHi)
                array.shift(pB)

    var bool  mWatch = false
    var float mB = na
    var float mC = na
    var int   mBBar = na
    var int   mCBar = na
    var float mRunHi = na
    var int   mRunHiBar = na

    var bool  wWatch = false
    var float wB = na
    var float wC = na
    var int   wBBar = na
    var int   wCBar = na
    var float wRunLo = na
    var int   wRunLoBar = na

    bool  mForm = false
    float mBp = na
    float mCp = na

    bool  wForm = false
    float wBp = na
    float wCp = na

    if pivNew
        n = array.size(pP)

        if n >= 2
            newestHi = array.get(pHi, n - 1)

            if not newestHi
                bCand = array.get(pP, n - 2)

                okTrend = true

                if trendFilter and n >= 4
                    okTrend := bCand > array.get(pP, n - 4)

                if okTrend
                    mB := bCand
                    mC := array.get(pP, n - 1)

                    mBBar := array.get(pB, n - 2)
                    mCBar := array.get(pB, n - 1)

                    mRunHi := array.get(pP, n - 1)
                    mRunHiBar := mCBar

                    mWatch := true

            else
                bCandW = array.get(pP, n - 2)

                okTrendW = true

                if trendFilter and n >= 4
                    okTrendW := bCandW < array.get(pP, n - 4)

                if okTrendW
                    wB := bCandW
                    wC := array.get(pP, n - 1)

                    wBBar := array.get(pB, n - 2)
                    wCBar := array.get(pB, n - 1)

                    wRunLo := array.get(pP, n - 1)
                    wRunLoBar := wCBar

                    wWatch := true

    if barstate.isconfirmed

        // M TOP
        if mWatch

            if high > mRunHi
                mRunHi := high
                mRunHiBar := bar_index

            mElapsed = bar_index - mCBar
            mTol = peakTol * (mB - mC)

            if mRunHi > mB
                mWatch := false

            else if mElapsed > maxBars
                mWatch := false

            else
                if low < mC

                    peakDist = mRunHiBar - mBBar

                    okDist = peakDist >= minBars and peakDist <= maxBars
                    okEq   = mRunHi <= mB and mRunHi >= mB - mTol
                    okDep  = (mB - mC) > 0 and (mRunHi - mC) >= heightRatio * (mB - mC)
                    okATR  = minHeightATR <= 0.0 or (mB - mC) >= minHeightATR * atr14

                    if okDist and okEq and okDep and okATR
                        mForm := true
                        mBp := mB
                        mCp := mC

                    mWatch := false

        // W BOTTOM
        if wWatch

            if low < wRunLo
                wRunLo := low
                wRunLoBar := bar_index

            wElapsed = bar_index - wCBar
            wTol = peakTol * (wC - wB)

            if wRunLo < wB
                wWatch := false

            else if wElapsed > maxBars
                wWatch := false

            else
                if high > wC

                    troughDist = wRunLoBar - wBBar

                    okDistW = troughDist >= minBars and troughDist <= maxBars
                    okEqW   = wRunLo >= wB and wRunLo <= wB + wTol
                    okDepW  = (wC - wB) > 0 and (wC - wRunLo) >= heightRatio * (wC - wB)
                    okATRW  = minHeightATR <= 0.0 or (wC - wB) >= minHeightATR * atr14

                    if okDistW and okEqW and okDepW and okATRW
                        wForm := true
                        wBp := wB
                        wCp := wC

                    wWatch := false

    [mForm, mBp, mCp, wForm, wBp, wCp]

[mForm, mBp, mCp, wForm, wBp, wCp] = f_bcd()

/// ENTRY + EXIT
var float tpPrice = na
var float slPrice = na

entryRef = close

// SHORT
if inWindow and tradeMTop and mForm and strategy.position_size == 0

    if useATRexit
        risk = slATRmult * atrG
        slPrice := entryRef + risk
        tpPrice := entryRef - risk * rrRatio

    else
        h = mBp - mCp
        tpPrice := mCp - h
        slPrice := mBp + slBufATR * atrG

    shortEmaOK = not useEMAfilter or (entryRef < ema200 and slPrice < ema200)

    if shortEmaOK
        strategy.entry('Short', strategy.short)


// LONG
if inWindow and tradeWBot and wForm and strategy.position_size == 0

    if useATRexit
        risk = slATRmult * atrG
        slPrice := entryRef - risk
        tpPrice := entryRef + risk * rrRatio

    else
        h = wCp - wBp
        tpPrice := wCp + h
        slPrice := wBp - slBufATR * atrG

    longEmaOK = not useEMAfilter or (entryRef > ema200 and slPrice > ema200)

    if longEmaOK
        strategy.entry('Long', strategy.long)


// EXIT
if strategy.position_size < 0
    strategy.exit('Exit Short', from_entry = 'Short', stop = slPrice, limit = tpPrice)

if strategy.position_size > 0
    strategy.exit('Exit Long', from_entry = 'Long', stop = slPrice, limit = tpPrice)


// DATE FILTER
if useDateFilter and not inWindow and strategy.position_size != 0
    strategy.close_all()


/// VISUAL
plot(ema200, title = 'EMA200', color = color.orange, linewidth = 2)

plotshape(mForm,
     title = 'M TOP',
     style = shape.triangledown,
     location = location.abovebar,
     color = color.red,
     size = size.small)

plotshape(wForm,
     title = 'W BOTTOM',
     style = shape.triangleup,
     location = location.belowbar,
     color = color.blue,
     size = size.small)