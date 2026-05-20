//@version=5
// C LINE v2.1 — クラスタリングCラインゾーン + 上位足MTF + ブレイク色カスタム (2026-05-20)
// PDF「The Power of Horizontal Lines」定義準拠。MTA v2.2 と同じ MTF パターン。
// 計算関数 f_cZones() を request.security で 2系統呼び、xloc.bar_time で描画。
indicator('C LINE クラスタライン MTF', overlay = true, max_boxes_count = 20)

import DevLucem/ZigLib/1 as ZigZag

/// INPUT
Depth     = input.int(3, '深さ', minval = 1, step = 1, inline = 'z1', group = 'ZigZag')
Deviation = input.int(5, '偏差', minval = 1, step = 1, inline = 'z1', group = 'ZigZag')
Backstep  = input.int(3, 'Backstep', minval = 2, step = 1, inline = 'z2', group = 'ZigZag')
tolC      = input.float(0.5, '許容幅(ATR×)', minval = 0.1, step = 0.1, group = 'C LINE')
maxPivots = input.int(6, '検索ピボット数', minval = 3, step = 1, group = 'C LINE')

showChart = input.bool(true, 'チャート足', inline = 'm1', group = 'C LINE表示')
showSel   = input.bool(true, '選択足', inline = 'm1', group = 'C LINE表示')
selTFin   = input.timeframe('60', '選択足TF', inline = 'm2', group = 'C LINE表示')

cHiC  = input.color(color.new(color.orange, 70), 'チャート足抵抗', inline = 'c1', group = 'C LINE色')
cLoC  = input.color(color.new(color.green, 70), 'チャート足支持', inline = 'c1', group = 'C LINE色')
sHiC  = input.color(color.new(color.red, 70), '選択足抵抗', inline = 'c2', group = 'C LINE色')
sLoC  = input.color(color.new(color.blue, 70), '選択足支持', inline = 'c2', group = 'C LINE色')
cBrkC = input.color(color.gray, 'チャート足ブレイク', inline = 'c3', group = 'C LINE色')
sBrkC = input.color(color.gray, '選択足ブレイク', inline = 'c3', group = 'C LINE色')

almChart = input.bool(true, 'チャート足', inline = 'a1', group = 'アラート')
almSel   = input.bool(true, '選択足', inline = 'a1', group = 'アラート')

/// CALC FUNCTION (MTF用)
f_cZones() =>
    atr14 = ta.atr(14)
    [dir, z1, z2] = ZigZag.zigzag(low, high, Depth, Deviation, Backstep)
    cpx = z2.price
    cix = z2.index
    ctm = z2.time
    var array<float> pP    = array.new_float()
    var array<bool>  pHi   = array.new_bool()
    var array<int>   pT    = array.new_int()
    var array<float> pWick = array.new_float()
    var array<float> pBody = array.new_float()
    pivNew = not na(dir) and not na(dir[1]) and dir != dir[1]
    if pivNew
        pivBar  = na(cix[1]) ? -1 : int(cix[1])
        pivTime = na(ctm[1]) ? -1 : int(ctm[1])
        if pivBar >= 0 and pivTime >= 0
            pivOfs = bar_index - pivBar
            if pivOfs >= 0 and pivOfs < 500
                qp  = cpx[1]
                qh  = dir[1] > 0
                qwk = qh ? high[pivOfs] : low[pivOfs]
                qbd = qh ? math.max(open[pivOfs], close[pivOfs]) : math.min(open[pivOfs], close[pivOfs])
                needPush = array.size(pP) == 0
                if not needPush
                    needPush := array.get(pHi, array.size(pHi) - 1) != qh
                if needPush
                    array.push(pP, qp)
                    array.push(pHi, qh)
                    array.push(pT, pivTime)
                    array.push(pWick, qwk)
                    array.push(pBody, qbd)
                else
                    n = array.size(pP) - 1
                    array.set(pP, n, qp)
                    array.set(pHi, n, qh)
                    array.set(pT, n, pivTime)
                    array.set(pWick, n, qwk)
                    array.set(pBody, n, qbd)
                while array.size(pP) > maxPivots
                    array.shift(pP)
                    array.shift(pHi)
                    array.shift(pT)
                    array.shift(pWick)
                    array.shift(pBody)
    var float hiTop = na
    var float hiBtm = na
    var int   hiLT  = na
    var bool  hiBrk = false
    var float loTop = na
    var float loBtm = na
    var int   loLT  = na
    var bool  loBrk = false
    if pivNew
        qhNew = dir[1] > 0
        tol   = tolC * atr14
        sz    = array.size(pP)
        if qhNew
            hiI1 = -1
            hiI2 = -1
            i = sz - 1
            while i >= 0 and hiI2 < 0
                if array.get(pHi, i)
                    if hiI1 < 0
                        hiI1 := i
                    else if math.abs(array.get(pP, i) - array.get(pP, hiI1)) <= tol
                        hiI2 := i
                i := i - 1
            if hiI1 >= 0 and hiI2 >= 0
                hiTop := math.max(array.get(pWick, hiI1), array.get(pWick, hiI2))
                hiBtm := math.max(array.get(pBody, hiI1), array.get(pBody, hiI2))
                hiLT  := math.min(array.get(pT, hiI1), array.get(pT, hiI2))
                hiBrk := false
            else
                if not na(hiTop) and hiBrk
                    hiTop := na
                    hiBtm := na
                    hiLT  := na
                    hiBrk := false
        else
            loI1 = -1
            loI2 = -1
            j = sz - 1
            while j >= 0 and loI2 < 0
                if not array.get(pHi, j)
                    if loI1 < 0
                        loI1 := j
                    else if math.abs(array.get(pP, j) - array.get(pP, loI1)) <= tol
                        loI2 := j
                j := j - 1
            if loI1 >= 0 and loI2 >= 0
                loTop := math.min(array.get(pBody, loI1), array.get(pBody, loI2))
                loBtm := math.min(array.get(pWick, loI1), array.get(pWick, loI2))
                loLT  := math.min(array.get(pT, loI1), array.get(pT, loI2))
                loBrk := false
            else
                if not na(loTop) and loBrk
                    loTop := na
                    loBtm := na
                    loLT  := na
                    loBrk := false
    if barstate.isconfirmed
        if not hiBrk and not na(hiTop)
            if close > hiTop
                hiBrk := true
        if not loBrk and not na(loBtm)
            if close < loBtm
                loBrk := true
    [hiTop, hiBtm, hiLT, hiBrk, loTop, loBtm, loLT, loBrk]

/// MTF CALL
selTF = selTFin == '' ? timeframe.period : selTFin
[cHiTop, cHiBtm, cHiLT, cHiBrk, cLoTop, cLoBtm, cLoLT, cLoBrk] = request.security(syminfo.tickerid, timeframe.period, f_cZones(), lookahead = barmerge.lookahead_off, gaps = barmerge.gaps_off)
[sHiTop, sHiBtm, sHiLT, sHiBrk, sLoTop, sLoBtm, sLoLT, sLoBrk] = request.security(syminfo.tickerid, selTF, f_cZones(), lookahead = barmerge.lookahead_off, gaps = barmerge.gaps_off)

/// PLOT
var box cHiBox = na
var box cLoBox = na
var box sHiBox = na
var box sLoBox = na

rightTime = time + 50 * timeframe.in_seconds() * 1000

// チャート足 HIGH
if showChart and not na(cHiTop) and not na(cHiLT)
    cHiCol  = cHiBrk ? cBrkC : cHiC
    cHiBg   = cHiBrk ? color.new(cBrkC, 90) : color.new(cHiC, 80)
    cHiStyl = cHiBrk ? line.style_dotted : line.style_solid
    if na(cHiBox)
        cHiBox := box.new(cHiLT, cHiTop, rightTime, cHiBtm, cHiCol, 1, cHiStyl, extend.none, xloc.bar_time, cHiBg)
    else
        box.set_left(cHiBox, cHiLT)
        box.set_top(cHiBox, cHiTop)
        box.set_right(cHiBox, rightTime)
        box.set_bottom(cHiBox, cHiBtm)
        box.set_border_color(cHiBox, cHiCol)
        box.set_border_style(cHiBox, cHiStyl)
        box.set_bgcolor(cHiBox, cHiBg)
else
    if not na(cHiBox)
        box.delete(cHiBox)
        cHiBox := na

// チャート足 LOW
if showChart and not na(cLoTop) and not na(cLoLT)
    cLoCol  = cLoBrk ? cBrkC : cLoC
    cLoBg   = cLoBrk ? color.new(cBrkC, 90) : color.new(cLoC, 80)
    cLoStyl = cLoBrk ? line.style_dotted : line.style_solid
    if na(cLoBox)
        cLoBox := box.new(cLoLT, cLoTop, rightTime, cLoBtm, cLoCol, 1, cLoStyl, extend.none, xloc.bar_time, cLoBg)
    else
        box.set_left(cLoBox, cLoLT)
        box.set_top(cLoBox, cLoTop)
        box.set_right(cLoBox, rightTime)
        box.set_bottom(cLoBox, cLoBtm)
        box.set_border_color(cLoBox, cLoCol)
        box.set_border_style(cLoBox, cLoStyl)
        box.set_bgcolor(cLoBox, cLoBg)
else
    if not na(cLoBox)
        box.delete(cLoBox)
        cLoBox := na

// 選択足 HIGH
if showSel and not na(sHiTop) and not na(sHiLT)
    sHiCol  = sHiBrk ? sBrkC : sHiC
    sHiBg   = sHiBrk ? color.new(sBrkC, 90) : color.new(sHiC, 80)
    sHiStyl = sHiBrk ? line.style_dotted : line.style_solid
    if na(sHiBox)
        sHiBox := box.new(sHiLT, sHiTop, rightTime, sHiBtm, sHiCol, 2, sHiStyl, extend.none, xloc.bar_time, sHiBg)
    else
        box.set_left(sHiBox, sHiLT)
        box.set_top(sHiBox, sHiTop)
        box.set_right(sHiBox, rightTime)
        box.set_bottom(sHiBox, sHiBtm)
        box.set_border_color(sHiBox, sHiCol)
        box.set_border_style(sHiBox, sHiStyl)
        box.set_bgcolor(sHiBox, sHiBg)
else
    if not na(sHiBox)
        box.delete(sHiBox)
        sHiBox := na

// 選択足 LOW
if showSel and not na(sLoTop) and not na(sLoLT)
    sLoCol  = sLoBrk ? sBrkC : sLoC
    sLoBg   = sLoBrk ? color.new(sBrkC, 90) : color.new(sLoC, 80)
    sLoStyl = sLoBrk ? line.style_dotted : line.style_solid
    if na(sLoBox)
        sLoBox := box.new(sLoLT, sLoTop, rightTime, sLoBtm, sLoCol, 2, sLoStyl, extend.none, xloc.bar_time, sLoBg)
    else
        box.set_left(sLoBox, sLoLT)
        box.set_top(sLoBox, sLoTop)
        box.set_right(sLoBox, rightTime)
        box.set_bottom(sLoBox, sLoBtm)
        box.set_border_color(sLoBox, sLoCol)
        box.set_border_style(sLoBox, sLoStyl)
        box.set_bgcolor(sLoBox, sLoBg)
else
    if not na(sLoBox)
        box.delete(sLoBox)
        sLoBox := na

/// ALERT
sym = syminfo.ticker
if almChart and not na(cHiBrk) and not na(cHiBrk[1])
    if cHiBrk and not cHiBrk[1]
        alert(sym + ' チャート足 C LINE 抵抗ブレイク', alert.freq_once_per_bar_close)
    if cLoBrk and not cLoBrk[1]
        alert(sym + ' チャート足 C LINE 支持ブレイク', alert.freq_once_per_bar_close)
if almSel and not na(sHiBrk) and not na(sHiBrk[1])
    if sHiBrk and not sHiBrk[1]
        alert(sym + ' 選択足 C LINE 抵抗ブレイク', alert.freq_once_per_bar_close)
    if sLoBrk and not sLoBrk[1]
        alert(sym + ' 選択足 C LINE 支持ブレイク', alert.freq_once_per_bar_close)
