//@version=5
// MTA v2.1 + MTF (2026-05-18) 仕様書v2.1完全版に忠実・MTF拡張。[検証:mtaTF=60]
indicator('MTA v2.1', overlay = true, max_lines_count = 500, max_labels_count = 500)

import DevLucem/ZigLib/1 as ZigZag

Depth     = input.int(3, 'Depth', minval = 1, step = 1, group = 'ZigZag Config')
Deviation = input.int(5, 'Deviation', minval = 1, step = 1, group = 'ZigZag Config')
Backstep  = input.int(3, 'Backstep', minval = 2, step = 1, group = 'ZigZag Config')
lineThick = input.int(2, 'Line Thickness', minval = 1, maxval = 4, group = 'ZigZag')
showZZ    = input.bool(true, 'Show ZigZag line (chart TF)', group = 'ZigZag')
upColor   = input.color(color.lime, 'Bull Color', group = 'ZigZag')
dnColor   = input.color(color.red, 'Bear Color', group = 'ZigZag')
lblOff    = input.int(10, 'ラベル右オフセット(bars)', minval = 0, group = 'Display')
mtaTF     = input.timeframe('60', 'MTA時間足 (空=チャート足)', group = 'MTF')

f_mtaCalc() =>
    [direction, z1, z2] = ZigZag.zigzag(low, high, Depth, Deviation, Backstep)
    zpx = z2.price
    zix = z2.index
    ztm = z2.time
    var array<float> zP = array.new_float()
    var array<bool>  zH = array.new_bool()
    var array<int>   zB = array.new_int()
    var array<int>   zT = array.new_int()
    if not na(direction) and not na(direction[1]) and direction != direction[1]
        pPrice = zpx[1]
        pBar   = na(zix[1]) ? -1 : int(zix[1])
        pTime  = na(ztm[1]) ? -1 : int(ztm[1])
        pHigh  = direction[1] > 0
        if not na(pPrice) and pBar >= 0
            if array.size(zP) == 0
                array.push(zP, pPrice)
                array.push(zH, pHigh)
                array.push(zB, pBar)
                array.push(zT, pTime)
            else if pBar > array.last(zB)
                li = array.size(zP) - 1
                if array.last(zH) == pHigh
                    prevP  = array.last(zP)
                    better = pHigh ? pPrice > prevP : pPrice < prevP
                    if better
                        array.set(zP, li, pPrice)
                        array.set(zB, li, pBar)
                        array.set(zT, li, pTime)
                else
                    array.push(zP, pPrice)
                    array.push(zH, pHigh)
                    array.push(zB, pBar)
                    array.push(zT, pTime)
    var int   trend   = 0
    var float aPx     = na
    var int   aBar    = -1
    var float mta     = na
    var int   mtaTime = -1
    n = math.min(array.size(zP), math.min(array.size(zH), array.size(zB)))
    if trend == 0 and n >= 3
        if not array.get(zH, n - 1)
            a1P = array.get(zP, n - 3)
            h1P = array.get(zP, n - 2)
            a2P = array.get(zP, n - 1)
            if a2P > a1P and close > h1P
                trend   := 1
                aPx     := h1P
                aBar    := array.get(zB, n - 2)
                mta     := a2P
                mtaTime := array.get(zT, n - 1)
        if trend == 0 and array.get(zH, n - 1)
            h1P = array.get(zP, n - 3)
            l1P = array.get(zP, n - 2)
            h2P = array.get(zP, n - 1)
            if h2P < h1P and close < l1P
                trend   := -1
                aPx     := l1P
                aBar    := array.get(zB, n - 2)
                mta     := h2P
                mtaTime := array.get(zT, n - 1)
    if trend == 1
        if not na(mta) and close < mta
            trend := 0
            aPx   := na
            aBar  := -1
        else if not na(aPx) and close > aPx
            bIdx  = -1
            found = false
            for j = 0 to n - 1 by 1
                if not found and array.get(zH, j) and array.get(zB, j) > aBar and array.get(zP, j) > aPx
                    bIdx  := j
                    found := true
            if bIdx >= 0
                nlo = float(na)
                nix = -1
                bB  = array.get(zB, bIdx)
                for j = 0 to n - 1 by 1
                    if not array.get(zH, j) and array.get(zB, j) > aBar and array.get(zB, j) < bB
                        if na(nlo) or array.get(zP, j) < nlo
                            nlo := array.get(zP, j)
                            nix := j
                if nix != -1
                    mta     := nlo
                    mtaTime := array.get(zT, nix)
                    aPx     := array.get(zP, bIdx)
                    aBar    := bB
    if trend == -1
        if not na(mta) and close > mta
            trend := 0
            aPx   := na
            aBar  := -1
        else if not na(aPx) and close < aPx
            bIdx  = -1
            found = false
            for j = 0 to n - 1 by 1
                if not found and not array.get(zH, j) and array.get(zB, j) > aBar and array.get(zP, j) < aPx
                    bIdx  := j
                    found := true
            if bIdx >= 0
                nhi = float(na)
                nix = -1
                bB  = array.get(zB, bIdx)
                for j = 0 to n - 1 by 1
                    if array.get(zH, j) and array.get(zB, j) > aBar and array.get(zB, j) < bB
                        if na(nhi) or array.get(zP, j) > nhi
                            nhi := array.get(zP, j)
                            nix := j
                if nix != -1
                    mta     := nhi
                    mtaTime := array.get(zT, nix)
                    aPx     := array.get(zP, bIdx)
                    aBar    := bB
    ended = trend == 0 and not na(mta)
    [mta, trend, mtaTime, ended]

selTF = mtaTF == '' ? timeframe.period : mtaTF
[hM, hTr, hMt, hEnd] = request.security(syminfo.tickerid, selTF, f_mtaCalc(), lookahead = barmerge.lookahead_off, gaps = barmerge.gaps_off)

var line zz = na
if showZZ
    [cdir, cz1, cz2] = ZigZag.zigzag(low, high, Depth, Deviation, Backstep)
    zz := line.new(cz1, cz2, xloc.bar_time, extend.none, color.new(cdir > 0 ? upColor : dnColor, 0), width = lineThick)
    if not na(zz[1])
        if cdir == cdir[1]
            line.delete(zz[1])
        else
            line.set_extend(zz[1], extend.none)

var line  lMta  = na
var label lbMta = na
if not na(hM) and not na(hMt) and hMt > 0
    mtaCol = hTr == 1 ? color.blue : hTr == -1 ? color.red : color.new(color.gray, 0)
    lblSt  = hTr == 1 ? label.style_label_up : label.style_label_down
    lnSt   = hEnd ? line.style_dotted : line.style_solid
    if na(lMta)
        lMta := line.new(hMt, hM, hMt + 1, hM, xloc = xloc.bar_time, extend = extend.right, color = mtaCol, width = 2, style = lnSt)
    else
        line.set_xy1(lMta, hMt, hM)
        line.set_xy2(lMta, hMt + 1, hM)
        line.set_color(lMta, mtaCol)
        line.set_style(lMta, lnSt)
    if na(lbMta)
        lbMta := label.new(time, hM, 'MTA', xloc = xloc.bar_time, yloc = yloc.price, style = lblSt, color = color.new(color.white, 100), textcolor = mtaCol, size = size.small)
    else
        label.set_xy(lbMta, time + lblOff * (timeframe.in_seconds() * 1000), hM)
        label.set_text(lbMta, 'MTA')
        label.set_textcolor(lbMta, mtaCol)
        label.set_style(lbMta, lblSt)
