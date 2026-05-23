//@version=5
// B LINE v6.6 — ピーク・ペア探索(B候補ループ) ダブルトップ/ボトム検出 + ライフサイクル + MTF (2026-05-21)
// v6.6: 古いパターンも残るよう max_boxes_count 30→100、lifeBars 既定 50→200、maxHistory 既定 5→10。
//   (maxHistory を上げると box が 30 上限を超えてランタイムエラーになっていたため上限を引き上げ)
// A = 「C から左へ伸ばした水平線(cP)が左山/左谷と交わる点」。CDE本数の上下限は可変入力(0.5〜2.0)。
// v6.4: E を D後で最初に割れたバーに遡及特定（検知遅れ/cde水増し解消）。
// v6.5 修正: B(左の山/谷)を「絶対極値」でなく「D側から近い候補を順に試し最初に成立したもの」に。
//   絶対極値だと遠い深安値/高値を掴み B→C>maxBars で A走査範囲外→「Aが無い」と誤判定し W が全却下されていた。
indicator('B LINE ABCDE ダブル v6 MTF', overlay = true, max_boxes_count = 100, max_bars_back = 500)

import DevLucem/ZigLib/1 as ZigZag

/// INPUT
Depth     = input.int(5, '深さ', minval = 1, step = 1, inline = 'z1', group = 'ZigZag')
Deviation = input.int(5, '偏差', minval = 1, step = 1, inline = 'z1', group = 'ZigZag')
Backstep  = input.int(3, 'Backstep', minval = 2, step = 1, inline = 'z2', group = 'ZigZag')

minBars      = input.int(15, 'ABC最小本数', minval = 3, step = 1, inline = 'b1', group = 'B LINE')
maxBars      = input.int(45, 'ABC最大本数', minval = 10, step = 1, inline = 'b1', group = 'B LINE')
peakTol      = input.float(0.40, '2山の高さ許容(×山高)', minval = 0.0, maxval = 1.0, step = 0.05, group = 'B LINE')
heightRatio  = input.float(0.5, 'CDE高さ比(×ABC高)', minval = 0.1, step = 0.1, group = 'B LINE')
cdeBarMin    = input.float(0.5, 'CDE本数 下限(×ABC)', minval = 0.1, step = 0.1, inline = 'cd', group = 'B LINE')
cdeBarMax    = input.float(2.0, 'CDE本数 上限(×ABC)', minval = 1.0, step = 0.1, inline = 'cd', group = 'B LINE')
minHeightATR = input.float(0.0, 'B-C最小高さ(ATR×) 0=無制限', minval = 0.0, step = 0.1, group = 'B LINE')
lifeBars     = input.int(200, '色付き表示期間(本)', minval = 5, step = 5, group = 'B LINE')
maxHistory   = input.int(10, '履歴保持数', minval = 0, step = 1, group = 'B LINE')

showChart = input.bool(true, 'チャート足', inline = 'm1', group = 'B LINE表示')
showSel   = input.bool(true, '選択足', inline = 'm1', group = 'B LINE表示')
selTFin   = input.timeframe('60', '選択足TF', inline = 'm2', group = 'B LINE表示')

cMTopC = input.color(color.new(color.fuchsia, 70), 'チャート足 M TOP', inline = 'c1', group = 'B LINE色')
cWBotC = input.color(color.new(color.aqua, 70),    'チャート足 W BOTTOM', inline = 'c1', group = 'B LINE色')
sMTopC = input.color(color.new(color.red, 70),     '選択足 M TOP', inline = 'c2', group = 'B LINE色')
sWBotC = input.color(color.new(color.blue, 70),    '選択足 W BOTTOM', inline = 'c2', group = 'B LINE色')
cBrkC  = input.color(color.gray, 'チャート足再貫通', inline = 'c3', group = 'B LINE色')
sBrkC  = input.color(color.gray, '選択足再貫通', inline = 'c3', group = 'B LINE色')

almChart = input.bool(true, 'チャート足', inline = 'a1', group = 'アラート')
almSel   = input.bool(true, '選択足', inline = 'a1', group = 'アラート')

/// CALC FUNCTION (MTF用)
f_bZones() =>
    atr14 = ta.atr(14)
    [dir, z1, z2] = ZigZag.zigzag(low, high, Depth, Deviation, Backstep)
    cpx = z2.price
    cix = z2.index
    ctm = z2.time
    var array<float> pP    = array.new_float()
    var array<bool>  pHi   = array.new_bool()
    var array<int>   pT    = array.new_int()
    var array<int>   pB    = array.new_int()
    var array<float> pWick = array.new_float()
    var array<float> pBody = array.new_float()
    pivNew = not na(dir) and not na(dir[1]) and dir != dir[1]
    if pivNew
        pivBar = na(cix[1]) ? -1 : int(cix[1])
        pivTime = na(ctm[1]) ? -1 : int(ctm[1])
        if pivBar >= 0 and pivTime >= 0
            pivOfs = bar_index - pivBar
            if pivOfs >= 0 and pivOfs < 1000
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
                    array.push(pB, pivBar)
                    array.push(pWick, qwk)
                    array.push(pBody, qbd)
                else
                    nn = array.size(pP) - 1
                    array.set(pP, nn, qp)
                    array.set(pHi, nn, qh)
                    array.set(pT, nn, pivTime)
                    array.set(pB, nn, pivBar)
                    array.set(pWick, nn, qwk)
                    array.set(pBody, nn, qbd)
                while array.size(pP) > 30
                    array.shift(pP)
                    array.shift(pHi)
                    array.shift(pT)
                    array.shift(pB)
                    array.shift(pWick)
                    array.shift(pBody)
    var bool  mPrep    = false
    var float mNeck    = na
    var float mB       = na
    var int   mCBar    = na
    var int   mAbc     = na
    var float mZoneT   = na
    var float mZoneB   = na
    var int   mZoneLT  = na
    var bool  wPrep    = false
    var float wNeck    = na
    var float wB       = na
    var int   wCBar    = na
    var int   wAbc     = na
    var float wZoneT   = na
    var float wZoneB   = na
    var int   wZoneLT  = na
    var bool  mActive   = false
    var bool  mReBroken = false
    var int   mFormBar  = na
    var float mAZoneT   = na
    var float mAZoneB   = na
    var int   mAZoneLT  = na
    var bool  wActive   = false
    var bool  wReBroken = false
    var int   wFormBar  = na
    var float wAZoneT   = na
    var float wAZoneB   = na
    var int   wAZoneLT  = na
    // ===== ピーク・ペア探索（新ピボット確定時。B候補をD側から順に試し最初に成立したもの採用）=====
    if pivNew
        n = array.size(pP)
        // --- M TOP: 最新ピボットが高値(D候補) ---
        if n >= 4 and array.get(pHi, n - 1)
            dP = array.get(pP, n - 1)
            dBar = array.get(pB, n - 1)
            biM = n - 3
            mStop = false
            while biM >= 0 and not mStop
                sBarM = array.get(pB, biM)
                if dBar - sBarM > maxBars * 2
                    mStop := true
                else if array.get(pHi, biM)
                    bP = array.get(pP, biM)
                    bBar = array.get(pB, biM)
                    int cIdx = -1
                    float cLowP = na
                    kk = biM + 1
                    while kk <= n - 2
                        if not array.get(pHi, kk)
                            lv = array.get(pP, kk)
                            if na(cLowP) or lv < cLowP
                                cLowP := lv
                                cIdx := kk
                        kk += 1
                    if cIdx >= 0
                        cP = cLowP
                        cBar = array.get(pB, cIdx)
                        cOfsM = bar_index - cBar
                        bOfsM = bar_index - bBar
                        int aOfsM = -1
                        ofsM = bOfsM
                        limM = cOfsM + maxBars
                        while ofsM <= limM and aOfsM < 0
                            if low[ofsM] <= cP
                                aOfsM := ofsM
                            ofsM += 1
                        if aOfsM >= 0
                            abcBars = aOfsM - cOfsM
                            abcH = bP - cP
                            cdeH = dP - cP
                            okAbc  = abcBars >= minBars and abcBars <= maxBars
                            okPeak = abcH > 0 and dP <= bP + peakTol * abcH and dP >= bP - peakTol * abcH
                            okH    = cdeH >= heightRatio * abcH
                            okFlat = minHeightATR <= 0.0 or abcH >= minHeightATR * atr14
                            if okAbc and okPeak and okH and okFlat
                                zTm = array.get(pBody, cIdx)
                                zBm = array.get(pWick, cIdx)
                                zLTm = array.get(pT, cIdx)
                                dOfsE = bar_index - dBar
                                int eOfsE = -1
                                oe = dOfsE - 1
                                while oe >= 0 and eOfsE < 0
                                    if low[oe] < cP
                                        eOfsE := oe
                                    oe -= 1
                                if eOfsE >= 0
                                    eBarE = bar_index - eOfsE
                                    cdeR = eBarE - cBar
                                    if cdeR >= math.round(cdeBarMin * abcBars) and cdeR <= math.round(cdeBarMax * abcBars)
                                        mActive   := true
                                        mReBroken := false
                                        mFormBar  := eBarE
                                        mAZoneT   := zTm
                                        mAZoneB   := zBm
                                        mAZoneLT  := zLTm
                                    mPrep := false
                                else
                                    mPrep   := true
                                    mNeck   := cP
                                    mB      := bP
                                    mCBar   := cBar
                                    mAbc    := abcBars
                                    mZoneT  := zTm
                                    mZoneB  := zBm
                                    mZoneLT := zLTm
                                mStop := true
                if not mStop
                    biM := biM - 1
        // --- W BOT: 最新ピボットが安値(D候補) ---
        if n >= 4 and not array.get(pHi, n - 1)
            dP = array.get(pP, n - 1)
            dBar = array.get(pB, n - 1)
            biW = n - 3
            wStop = false
            while biW >= 0 and not wStop
                sBarW = array.get(pB, biW)
                if dBar - sBarW > maxBars * 2
                    wStop := true
                else if not array.get(pHi, biW)
                    bPW = array.get(pP, biW)
                    bBarW = array.get(pB, biW)
                    int cIdxW = -1
                    float cHiPW = na
                    kkW = biW + 1
                    while kkW <= n - 2
                        if array.get(pHi, kkW)
                            hv = array.get(pP, kkW)
                            if na(cHiPW) or hv > cHiPW
                                cHiPW := hv
                                cIdxW := kkW
                        kkW += 1
                    if cIdxW >= 0
                        cPW = cHiPW
                        cBarW = array.get(pB, cIdxW)
                        cOfsW = bar_index - cBarW
                        bOfsW = bar_index - bBarW
                        int aOfsW = -1
                        ofsW = bOfsW
                        limW = cOfsW + maxBars
                        while ofsW <= limW and aOfsW < 0
                            if high[ofsW] >= cPW
                                aOfsW := ofsW
                            ofsW += 1
                        if aOfsW >= 0
                            abcBarsW = aOfsW - cOfsW
                            abcHW = cPW - bPW
                            cdeHW = cPW - dP
                            okAbcW  = abcBarsW >= minBars and abcBarsW <= maxBars
                            okPeakW = abcHW > 0 and dP >= bPW - peakTol * abcHW and dP <= bPW + peakTol * abcHW
                            okHW    = cdeHW >= heightRatio * abcHW
                            okFlatW = minHeightATR <= 0.0 or abcHW >= minHeightATR * atr14
                            if okAbcW and okPeakW and okHW and okFlatW
                                zTw = array.get(pWick, cIdxW)
                                zBw = array.get(pBody, cIdxW)
                                zLTw = array.get(pT, cIdxW)
                                dOfsEw = bar_index - dBar
                                int eOfsEw = -1
                                oew = dOfsEw - 1
                                while oew >= 0 and eOfsEw < 0
                                    if high[oew] > cPW
                                        eOfsEw := oew
                                    oew -= 1
                                if eOfsEw >= 0
                                    eBarEw = bar_index - eOfsEw
                                    cdeRw = eBarEw - cBarW
                                    if cdeRw >= math.round(cdeBarMin * abcBarsW) and cdeRw <= math.round(cdeBarMax * abcBarsW)
                                        wActive   := true
                                        wReBroken := false
                                        wFormBar  := eBarEw
                                        wAZoneT   := zTw
                                        wAZoneB   := zBw
                                        wAZoneLT  := zLTw
                                    wPrep := false
                                else
                                    wPrep   := true
                                    wNeck   := cPW
                                    wB      := bPW
                                    wCBar   := cBarW
                                    wAbc    := abcBarsW
                                    wZoneT  := zTw
                                    wZoneB  := zBw
                                    wZoneLT := zLTw
                                wStop := true
                if not wStop
                    biW := biW - 1
    // ===== E形成 + ライフサイクル（確定足のみ = repaint防止）=====
    if barstate.isconfirmed
        if mPrep and not na(mCBar) and not na(mAbc) and bar_index - mCBar > math.round(cdeBarMax * mAbc)
            mPrep := false
        if mPrep and not na(mB) and close > mB
            mPrep := false
        if mPrep and not mActive and low < mNeck
            cdeBars = bar_index - mCBar
            if cdeBars >= math.round(cdeBarMin * mAbc) and cdeBars <= math.round(cdeBarMax * mAbc)
                mActive   := true
                mReBroken := false
                mFormBar  := bar_index
                mAZoneT   := mZoneT
                mAZoneB   := mZoneB
                mAZoneLT  := mZoneLT
            mPrep := false
        if mActive
            if not mReBroken and close > mAZoneT
                mReBroken := true
            if bar_index - mFormBar > lifeBars
                mActive   := false
                mReBroken := false
                mAZoneT   := na
                mAZoneB   := na
                mAZoneLT  := na
                mFormBar  := na
        if wPrep and not na(wCBar) and not na(wAbc) and bar_index - wCBar > math.round(cdeBarMax * wAbc)
            wPrep := false
        if wPrep and not na(wB) and close < wB
            wPrep := false
        if wPrep and not wActive and high > wNeck
            cdeBarsW = bar_index - wCBar
            if cdeBarsW >= math.round(cdeBarMin * wAbc) and cdeBarsW <= math.round(cdeBarMax * wAbc)
                wActive   := true
                wReBroken := false
                wFormBar  := bar_index
                wAZoneT   := wZoneT
                wAZoneB   := wZoneB
                wAZoneLT  := wZoneLT
            wPrep := false
        if wActive
            if not wReBroken and close < wAZoneB
                wReBroken := true
            if bar_index - wFormBar > lifeBars
                wActive   := false
                wReBroken := false
                wAZoneT   := na
                wAZoneB   := na
                wAZoneLT  := na
                wFormBar  := na
    float zoneTop = na
    float zoneBtm = na
    int   zoneLT  = na
    bool  isMTop  = false
    bool  bBrk    = false
    bool  bActive = false
    pickM = mActive and (not wActive or (not na(mFormBar) and not na(wFormBar) and mFormBar >= wFormBar))
    pickW = wActive and (not mActive or (not na(mFormBar) and not na(wFormBar) and wFormBar > mFormBar))
    if pickM
        zoneTop := mAZoneT
        zoneBtm := mAZoneB
        zoneLT  := mAZoneLT
        isMTop  := true
        bBrk    := mReBroken
        bActive := true
    else if pickW
        zoneTop := wAZoneT
        zoneBtm := wAZoneB
        zoneLT  := wAZoneLT
        isMTop  := false
        bBrk    := wReBroken
        bActive := true
    [zoneTop, zoneBtm, zoneLT, isMTop, bBrk, bActive]

/// MTF CALL
selTF = selTFin == '' ? timeframe.period : selTFin
[cZT, cZB, cZLT, cIM, cBrk, cAct] = request.security(syminfo.tickerid, timeframe.period, f_bZones(), lookahead = barmerge.lookahead_off, gaps = barmerge.gaps_off)
[sZT, sZB, sZLT, sIM, sBrk, sAct] = request.security(syminfo.tickerid, selTF, f_bZones(), lookahead = barmerge.lookahead_off, gaps = barmerge.gaps_off)

/// PLOT
var box cBBox = na
var box sBBox = na
var int cPrevLT = -1
var int sPrevLT = -1
var array<box> cBHist = array.new<box>()
var array<box> sBHist = array.new<box>()

rightTime = time + 50 * timeframe.in_seconds() * 1000

if showChart and cAct and not na(cZT) and not na(cZLT)
    cBCol  = cBrk ? cBrkC : (cIM ? cMTopC : cWBotC)
    cBBg   = cBrk ? color.new(cBrkC, 90) : (cIM ? color.new(cMTopC, 80) : color.new(cWBotC, 80))
    cBStyl = cBrk ? line.style_dotted : line.style_solid
    isNewC = na(cBBox) or cPrevLT != cZLT
    if isNewC
        if not na(cBBox)
            array.push(cBHist, cBBox)
            while array.size(cBHist) > maxHistory
                oldest = array.shift(cBHist)
                if not na(oldest)
                    box.delete(oldest)
        cBBox := box.new(cZLT, cZT, rightTime, cZB, cBCol, 1, cBStyl, extend.none, xloc.bar_time, cBBg)
        cPrevLT := cZLT
    else
        box.set_left(cBBox, cZLT)
        box.set_top(cBBox, cZT)
        box.set_right(cBBox, rightTime)
        box.set_bottom(cBBox, cZB)
        box.set_border_color(cBBox, cBCol)
        box.set_border_style(cBBox, cBStyl)
        box.set_bgcolor(cBBox, cBBg)
else
    if not na(cBBox)
        box.delete(cBBox)
        cBBox := na
        cPrevLT := -1

if showSel and sAct and not na(sZT) and not na(sZLT)
    sBCol  = sBrk ? sBrkC : (sIM ? sMTopC : sWBotC)
    sBBg   = sBrk ? color.new(sBrkC, 90) : (sIM ? color.new(sMTopC, 80) : color.new(sWBotC, 80))
    sBStyl = sBrk ? line.style_dotted : line.style_solid
    isNewS = na(sBBox) or sPrevLT != sZLT
    if isNewS
        if not na(sBBox)
            array.push(sBHist, sBBox)
            while array.size(sBHist) > maxHistory
                oldest = array.shift(sBHist)
                if not na(oldest)
                    box.delete(oldest)
        sBBox := box.new(sZLT, sZT, rightTime, sZB, sBCol, 2, sBStyl, extend.none, xloc.bar_time, sBBg)
        sPrevLT := sZLT
    else
        box.set_left(sBBox, sZLT)
        box.set_top(sBBox, sZT)
        box.set_right(sBBox, rightTime)
        box.set_bottom(sBBox, sZB)
        box.set_border_color(sBBox, sBCol)
        box.set_border_style(sBBox, sBStyl)
        box.set_bgcolor(sBBox, sBBg)
else
    if not na(sBBox)
        box.delete(sBBox)
        sBBox := na
        sPrevLT := -1

/// ALERT
sym = syminfo.ticker
if almChart and not na(cBrk) and not na(cBrk[1])
    if cBrk and not cBrk[1]
        msg = cIM ? 'チャート足 M TOP ネックライン 再貫通' : 'チャート足 W BOTTOM ネックライン 再貫通'
        alert(sym + ' ' + msg, alert.freq_once_per_bar_close)
if almSel and not na(sBrk) and not na(sBrk[1])
    if sBrk and not sBrk[1]
        msg = sIM ? '選択足 M TOP ネックライン 再貫通' : '選択足 W BOTTOM ネックライン 再貫通'
        alert(sym + ' ' + msg, alert.freq_once_per_bar_close)
