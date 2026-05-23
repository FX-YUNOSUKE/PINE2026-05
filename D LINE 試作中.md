//@version=5
// D LINE v2.6 — リアルタイム検出 (3ピボット + closeブレイク) + 上位足MTF + ブレイク色カスタム (2026-05-21)
// 根拠: PDF「The Power of Horizontal Lines」+ ゆうのすけ氏 講義
//   引用1: 「B はトップとボトムのネックライン、D はトレンド中の N 頂点」
//   引用2: 「左の高値を上抜けた時点でトレンド発生が確定して D ラインが発生」← リアルタイム判定
//   引用3: 「D ができるのは MTA を突破し、その後に出来た N の先端」
//   引用4: 「D はトレンドが転換したあとにしか発生しないライン」
//   ユーザー図: 「下降トレンドなら MTA を作った左側の安値が D」
// 実装: 3ピボット (LHL or HLH) でセットアップ判定 → 確定足 close で 直近高安をブレイク → D 即発生
//   上昇D セットアップ: P2=L1, P1=H1=A, P0=L2(B=MTA), P0>P2（安値切上げ）
//     ブレイク: close > P1 (A=H1) → D = P1 (A) の高値ヒゲ〜実体（元レジスタンス → 今は支持）
//   下降D セットアップ: P2=H1, P1=L1=A, P0=H2(B=MTA), P0<P2（高値切下げ）
//     ブレイク: close < P1 (A=L1) → D = P1 (A) の安値ヒゲ〜実体（元サポート → 今は抵抗）
// MTF: MTA v2.2 / C LINE v2.x と同方式 (request.security + 計算関数 + xloc.bar_time)
// 履歴: v1.0 誤定義 (D=P1) → v2.1 訂正 (D=P2) → v2.2 厳密化 → v2.3 4ピボット → v2.4 6ピボット(厳しすぎ)
//       → v2.5 4ピボット (P0=C 確定待ち=遅延) → v2.6 3ピボット+リアルタイムブレイク (遅延ゼロ)
indicator('D LINE Nパターン MTF', overlay = true, max_boxes_count = 10)

import DevLucem/ZigLib/1 as ZigZag

/// INPUT
Depth     = input.int(3, '深さ', minval = 1, step = 1, inline = 'z1', group = 'ZigZag')
Deviation = input.int(5, '偏差', minval = 1, step = 1, inline = 'z1', group = 'ZigZag')
Backstep  = input.int(3, 'Backstep', minval = 2, step = 1, inline = 'z2', group = 'ZigZag')

showChart = input.bool(true, 'チャート足', inline = 'm1', group = 'D LINE表示')
showSel   = input.bool(true, '選択足', inline = 'm1', group = 'D LINE表示')
selTFin   = input.timeframe('60', '選択足TF', inline = 'm2', group = 'D LINE表示')

cDnC  = input.color(color.new(color.yellow, 70), 'チャート足下降D', inline = 'c1', group = 'D LINE色')
cUpC  = input.color(color.new(color.yellow, 70), 'チャート足上昇D', inline = 'c1', group = 'D LINE色')
sDnC  = input.color(color.new(color.orange, 70), '選択足下降D', inline = 'c2', group = 'D LINE色')
sUpC  = input.color(color.new(color.orange, 70), '選択足上昇D', inline = 'c2', group = 'D LINE色')
cBrkC = input.color(color.gray, 'チャート足ブレイク', inline = 'c3', group = 'D LINE色')
sBrkC = input.color(color.gray, '選択足ブレイク', inline = 'c3', group = 'D LINE色')

almChart = input.bool(true, 'チャート足', inline = 'a1', group = 'アラート')
almSel   = input.bool(true, '選択足', inline = 'a1', group = 'アラート')

/// CALC FUNCTION (MTF用)
f_dZones() =>
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
                while array.size(pP) > 10
                    array.shift(pP)
                    array.shift(pHi)
                    array.shift(pT)
                    array.shift(pWick)
                    array.shift(pBody)
    var bool  upPrep    = false
    var float upBreakLv = na
    var float upDw      = na
    var float upDb      = na
    var int   upDt      = na
    var bool  dnPrep    = false
    var float dnBreakLv = na
    var float dnDw      = na
    var float dnDb      = na
    var int   dnDt      = na
    var float zoneTop = na
    var float zoneBtm = na
    var int   zoneLT  = na
    var bool  isHigh  = false
    var bool  dBrk    = false
    if pivNew
        upPrep := false
        dnPrep := false
        if array.size(pP) >= 3
            n = array.size(pP)
            p0p = array.get(pP,  n - 1)
            p0h = array.get(pHi, n - 1)
            p1p = array.get(pP,  n - 2)
            p1h = array.get(pHi, n - 2)
            p1t = array.get(pT,  n - 2)
            p1w = array.get(pWick, n - 2)
            p1b = array.get(pBody, n - 2)
            p2p = array.get(pP,  n - 3)
            p2h = array.get(pHi, n - 3)
            if not p0h and p1h and not p2h and p0p > p2p
                upPrep    := true
                upBreakLv := p1p
                upDw      := p1w
                upDb      := p1b
                upDt      := p1t
            if p0h and not p1h and p2h and p0p < p2p
                dnPrep    := true
                dnBreakLv := p1p
                dnDw      := p1w
                dnDb      := p1b
                dnDt      := p1t
    if barstate.isconfirmed
        if upPrep and not na(upBreakLv) and close > upBreakLv
            zoneTop := upDw
            zoneBtm := upDb
            zoneLT  := upDt
            isHigh  := false
            dBrk    := false
            upPrep  := false
        if dnPrep and not na(dnBreakLv) and close < dnBreakLv
            zoneTop := dnDb
            zoneBtm := dnDw
            zoneLT  := dnDt
            isHigh  := true
            dBrk    := false
            dnPrep  := false
        if not dBrk and not na(zoneTop)
            if isHigh and close > zoneTop
                dBrk := true
            else if not isHigh and close < zoneBtm
                dBrk := true
    [zoneTop, zoneBtm, zoneLT, isHigh, dBrk]

/// MTF CALL
selTF = selTFin == '' ? timeframe.period : selTFin
[cZT, cZB, cZLT, cIH, cBrk] = request.security(syminfo.tickerid, timeframe.period, f_dZones(), lookahead = barmerge.lookahead_off, gaps = barmerge.gaps_off)
[sZT, sZB, sZLT, sIH, sBrk] = request.security(syminfo.tickerid, selTF, f_dZones(), lookahead = barmerge.lookahead_off, gaps = barmerge.gaps_off)

/// PLOT
var box cDBox = na
var box sDBox = na

rightTime = time + 50 * timeframe.in_seconds() * 1000

if showChart and not na(cZT) and not na(cZLT)
    cDCol  = cBrk ? cBrkC : (cIH ? cDnC : cUpC)
    cDBg   = cBrk ? color.new(cBrkC, 90) : (cIH ? color.new(cDnC, 80) : color.new(cUpC, 80))
    cDStyl = cBrk ? line.style_dotted : line.style_solid
    if na(cDBox)
        cDBox := box.new(cZLT, cZT, rightTime, cZB, cDCol, 1, cDStyl, extend.none, xloc.bar_time, cDBg)
    else
        box.set_left(cDBox, cZLT)
        box.set_top(cDBox, cZT)
        box.set_right(cDBox, rightTime)
        box.set_bottom(cDBox, cZB)
        box.set_border_color(cDBox, cDCol)
        box.set_border_style(cDBox, cDStyl)
        box.set_bgcolor(cDBox, cDBg)
else
    if not na(cDBox)
        box.delete(cDBox)
        cDBox := na

if showSel and not na(sZT) and not na(sZLT)
    sDCol  = sBrk ? sBrkC : (sIH ? sDnC : sUpC)
    sDBg   = sBrk ? color.new(sBrkC, 90) : (sIH ? color.new(sDnC, 80) : color.new(sUpC, 80))
    sDStyl = sBrk ? line.style_dotted : line.style_solid
    if na(sDBox)
        sDBox := box.new(sZLT, sZT, rightTime, sZB, sDCol, 2, sDStyl, extend.none, xloc.bar_time, sDBg)
    else
        box.set_left(sDBox, sZLT)
        box.set_top(sDBox, sZT)
        box.set_right(sDBox, rightTime)
        box.set_bottom(sDBox, sZB)
        box.set_border_color(sDBox, sDCol)
        box.set_border_style(sDBox, sDStyl)
        box.set_bgcolor(sDBox, sDBg)
else
    if not na(sDBox)
        box.delete(sDBox)
        sDBox := na

/// ALERT
sym = syminfo.ticker
if almChart and not na(cBrk) and not na(cBrk[1])
    if cBrk and not cBrk[1]
        msg = cIH ? 'チャート足 D LINE 上方ブレイク' : 'チャート足 D LINE 下方ブレイク'
        alert(sym + ' ' + msg, alert.freq_once_per_bar_close)
if almSel and not na(sBrk) and not na(sBrk[1])
    if sBrk and not sBrk[1]
        msg = sIH ? '選択足 D LINE 上方ブレイク' : '選択足 D LINE 下方ブレイク'
        alert(sym + ' ' + msg, alert.freq_once_per_bar_close)
