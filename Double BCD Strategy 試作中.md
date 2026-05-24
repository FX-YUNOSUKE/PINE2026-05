//@version=5
// Double BCD Strategy v0.2 — ダブルトップ/ボトムのネックライン確定ブレイクで売買
// 派生元: indicators/b/b.pine (B LINE v7.3)。検出ロジックは同一。MTF撤去でchart-TFのみ＝リペイント最小化。
// セミナーSTEP2 バックテスト教材。手数料0.04%+slippage2込み。動的オフセット履歴参照ゼロ（series[定数]のみ）。
// v0.2 PDCA(2026-05-24): v0.1の「値幅測定TP/SL」はブレイク約定でRR<1:1になり負け(PF0.57)。
//   出口に「ATR基準の固定リスクリワード」(useATRexit)を追加。だが検証でRR1:2は悪化(PF0.41)→棄却。
//   既定は useATRexit=false(v0.1値幅測定) に戻した。useATRexit=true で ATR-RR 実験を再現できる。
// v0.3 PDCA(2026-05-24): トレンドフィルター追加(useEMAfilter,既定ON)。200EMAで大きな流れを見て、
//   流れに沿う側だけ仕掛ける（M TOP売りは close<EMA / W BOTTOM買いは close>EMA）＝純粋逆張り→順張り目線の反転狙い。
//   強トレンドへの逆張りを排除する仮説。useEMAfilter=false で v0.1/v0.2 の挙動に戻る。
strategy('Double BCD Strategy v0.3', overlay = true, max_bars_back = 500, default_qty_type = strategy.percent_of_equity, default_qty_value = 10, initial_capital = 10000, commission_type = strategy.commission.percent, commission_value = 0.04, slippage = 2, pyramiding = 0, process_orders_on_close = false, calc_on_every_tick = false)

import DevLucem/ZigLib/1 as ZigZag

/// INPUT — ZigZag
Depth     = input.int(5, '深さ', minval = 1, step = 1, inline = 'z1', group = 'ZigZag')
Deviation = input.int(5, '偏差', minval = 1, step = 1, inline = 'z1', group = 'ZigZag')
Backstep  = input.int(3, 'Backstep', minval = 2, step = 1, inline = 'z2', group = 'ZigZag')

/// INPUT — Pattern（インジ B LINE と同一）
minBars      = input.int(15, '2山間隔 最小(本) B↔D', minval = 3, step = 1, inline = 'b1', group = 'Pattern')
maxBars      = input.int(60, '2山間隔 最大(本) B↔D', minval = 10, step = 1, inline = 'b1', group = 'Pattern')
peakTol      = input.float(0.40, '2山の高さ許容(×山高)', minval = 0.0, maxval = 1.0, step = 0.05, group = 'Pattern')
heightRatio  = input.float(0.5, '右山の深さ比 (D-C)/(B-C)', minval = 0.1, step = 0.1, group = 'Pattern')
minHeightATR = input.float(1.5, '山高 最小(ATR×) 0=無制限', minval = 0.0, step = 0.1, group = 'Pattern')
trendFilter  = input.bool(true, 'トレンド整合 (M:Bが高値更新 / W:Bが安値更新)', group = 'Pattern')

/// INPUT — Trade
tradeMTop = input.bool(true, 'M TOP で売り', inline = 't1', group = 'Trade')
tradeWBot = input.bool(true, 'W BOTTOM で買い', inline = 't1', group = 'Trade')
slBufATR  = input.float(0.0, '損切りバッファ (ATR×) ※値幅測定版のみ', minval = 0.0, step = 0.1, group = 'Trade')
useATRexit = input.bool(false, 'ATR固定RR決済 ON=v0.2実験/OFF=v0.1値幅測定(既定)', group = 'Trade')
slATRmult  = input.float(1.5, '損切り (ATR×)', minval = 0.1, step = 0.1, group = 'Trade')
rrRatio    = input.float(2.0, 'リスクリワード比 (利確=損切り×)', minval = 0.5, step = 0.1, group = 'Trade')

/// INPUT — Trend Filter (v0.3)
useEMAfilter = input.bool(true, '200EMAトレンドフィルター (売りはEMA下/買いはEMA上)', group = 'Trend Filter')
emaLen       = input.int(200, 'EMA期間', minval = 10, step = 10, group = 'Trend Filter')

/// INPUT — Backtest 期間フィルタ（アウトオブサンプル検証用）
useDateFilter = input.bool(false, '期間フィルタを使う', group = 'Backtest')
fromDate = input.time(timestamp('2020-01-01 00:00'), '開始', group = 'Backtest')
toDate   = input.time(timestamp('2026-01-01 00:00'), '終了', group = 'Backtest')

atrG = ta.atr(14)
ema200 = ta.ema(close, emaLen)
inWindow = not useDateFilter or (time >= fromDate and time <= toDate)

/// CALC FUNCTION — chart-TFでinline呼び。formation event + B/C価格 を返す
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
    // ===== watch 状態 =====
    var bool  mWatch    = false
    var float mB        = na
    var float mC        = na
    var int   mBBar     = na
    var int   mCBar     = na
    var float mRunHi    = na
    var int   mRunHiBar = na
    var bool  wWatch    = false
    var float wB        = na
    var float wC        = na
    var int   wBBar     = na
    var int   wCBar     = na
    var float wRunLo    = na
    var int   wRunLoBar = na
    bool  mForm = false
    float mBp   = na
    float mCp   = na
    bool  wForm = false
    float wBp   = na
    float wCp   = na
    // ===== 新ピボットで watch 開始 =====
    if pivNew
        n = array.size(pP)
        if n >= 2
            newestHi = array.get(pHi, n - 1)
            if not newestHi
                // 新安値ピボット = M TOP ネックライン C、直前高値 = 左山 B
                bCand = array.get(pP, n - 2)
                okTrend = true
                if trendFilter and n >= 4
                    okTrend := bCand > array.get(pP, n - 4)
                if okTrend
                    mB        := bCand
                    mC        := array.get(pP, n - 1)
                    mBBar     := array.get(pB, n - 2)
                    mCBar     := array.get(pB, n - 1)
                    mRunHi    := array.get(pP, n - 1)
                    mRunHiBar := mCBar
                    mWatch    := true
                else
                    mWatch := false
            else
                // 新高値ピボット = W BOTTOM ネックライン C、直前安値 = 左谷 B
                bCandW = array.get(pP, n - 2)
                okTrendW = true
                if trendFilter and n >= 4
                    okTrendW := bCandW < array.get(pP, n - 4)
                if okTrendW
                    wB        := bCandW
                    wC        := array.get(pP, n - 1)
                    wBBar     := array.get(pB, n - 2)
                    wCBar     := array.get(pB, n - 1)
                    wRunLo    := array.get(pP, n - 1)
                    wRunLoBar := wCBar
                    wWatch    := true
                else
                    wWatch := false
    // ===== 確定足ごと: D追跡 → ネック割れ確定 =====
    if barstate.isconfirmed
        // --- M TOP ---
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
                        mBp   := mB
                        mCp   := mC
                    mWatch := false
        // --- W BOTTOM ---
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
                        wBp   := wB
                        wCp   := wC
                    wWatch := false
    [mForm, mBp, mCp, wForm, wBp, wCp]

[mForm, mBp, mCp, wForm, wBp, wCp] = f_bcd()

/// ENTRY + EXIT（値幅測定）
var float tpPrice = na
var float slPrice = na

// M TOP 確定 → SHORT（200EMA下のみ）
if inWindow and tradeMTop and mForm and strategy.position_size == 0 and (not useEMAfilter or close < ema200)
    if useATRexit
        risk = slATRmult * atrG            // v0.2: ATR固定リスクリワード
        slPrice := close + risk
        tpPrice := close - risk * rrRatio
    else
        h = mBp - mCp                       // v0.1: 値幅測定
        tpPrice := mCp - h
        slPrice := mBp + slBufATR * atrG
    strategy.entry('Short', strategy.short)

// W BOTTOM 確定 → LONG（200EMA上のみ）
if inWindow and tradeWBot and wForm and strategy.position_size == 0 and (not useEMAfilter or close > ema200)
    if useATRexit
        risk = slATRmult * atrG
        slPrice := close - risk
        tpPrice := close + risk * rrRatio
    else
        h = wCp - wBp
        tpPrice := wCp + h
        slPrice := wBp - slBufATR * atrG
    strategy.entry('Long', strategy.long)

// 値幅測定 TP(limit) + SL(stop)
if strategy.position_size < 0
    strategy.exit('Exit Short', from_entry = 'Short', stop = slPrice, limit = tpPrice)
if strategy.position_size > 0
    strategy.exit('Exit Long', from_entry = 'Long', stop = slPrice, limit = tpPrice)

// 期間フィルタの外に出たら強制クローズ
if useDateFilter and not inWindow and strategy.position_size != 0
    strategy.close_all()

/// VISUAL: 200EMA + formation足マーカー
plot(useEMAfilter ? ema200 : na, title = 'EMA200', color = color.new(color.orange, 0), linewidth = 2)
plotshape(mForm, title = 'M TOP entry', style = shape.triangledown, location = location.abovebar, color = color.new(color.red, 0), size = size.small)
plotshape(wForm, title = 'W BOTTOM entry', style = shape.triangleup, location = location.belowbar, color = color.new(color.blue, 0), size = size.small)
