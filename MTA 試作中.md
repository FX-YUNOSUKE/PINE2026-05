//@version=5
indicator('MTA v2.1', overlay = true, max_lines_count = 500, max_labels_count = 500)
import DevLucem/ZigLib/1 as ZigZag
Depth     = input.int(3, '深さ', minval = 1, step = 1, inline = 'zz1', group = 'ZigZag')
Deviation = input.int(5, '偏差', minval = 1, step = 1, inline = 'zz1', group = 'ZigZag')
Backstep  = input.int(3, 'Backstep', minval = 2, step = 1, inline = 'zz2', group = 'ZigZag')
lineThick = input.int(2, '線太さ', minval = 1, maxval = 4, inline = 'zz2', group = 'ZigZag')
showZZ    = input.bool(true, 'ZZ線', inline = 'zz3', group = 'ZigZag')
upColor   = input.color(color.lime, '↑', inline = 'zz3', group = 'ZigZag')
dnColor   = input.color(color.red, '↓', inline = 'zz3', group = 'ZigZag')
showChart = input.bool(true, 'チャート足', inline = 'm1', group = 'MTA表示')
showSel   = input.bool(true, '選択足', inline = 'm1', group = 'MTA表示')
selTFin   = input.timeframe('60', '選択足TF', inline = 'm2', group = 'MTA表示')
lblOff    = input.int(10, 'ラベルOFS', minval = 0, inline = 'm2', group = 'MTA表示')
cUp   = input.color(color.blue, 'チャート足↑', inline = 'c1', group = 'MTAの色')
cDn   = input.color(color.red, '↓', inline = 'c1', group = 'MTAの色')
sUp   = input.color(color.aqua, '選択足↑', inline = 'c2', group = 'MTAの色')
sDn   = input.color(color.fuchsia, '↓', inline = 'c2', group = 'MTAの色')
trCol = input.color(color.gray, '残置', inline = 'c2', group = 'MTAの色')
almChart = input.bool(true, 'チャート足', inline = 'a1', group = 'アラート')
almSel   = input.bool(true, '選択足', inline = 'a1', group = 'アラート')
almOccur = input.bool(true, '発生', inline = 'a2', group = 'アラート')
almBreak = input.bool(true, 'ブレイク', inline = 'a2', group = 'アラート')
almUp    = input.bool(true, '上昇系', inline = 'a3', group = 'アラート')
almDn    = input.bool(true, '下降系', inline = 'a3', group = 'アラート')
f_tfJP(string tf) =>
    s = timeframe.in_seconds(tf)
    m = s / 60
    out = tf
    if m < 60
        out := str.tostring(m) + '分足'
    else if m < 1440
        out := str.tostring(m / 60) + '時間足'
    else if m < 10080
        out := str.tostring(m / 1440) + '日足'
    else if m < 43200
        out := str.tostring(m / 10080) + '週足'
    else
        out := str.tostring(m / 43200) + 'ヶ月足'
    out
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
            for j = 0 to n - 1 by 1
                if array.get(zH, j) and array.get(zB, j) > aBar and array.get(zP, j) > aPx
                    bIdx  := j
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
            else
                rlo = float(na)
                rix = -1
                for j = 0 to n - 1 by 1
                    if not array.get(zH, j) and array.get(zB, j) > aBar
                        if na(rlo) or array.get(zP, j) < rlo
                            rlo := array.get(zP, j)
                            rix := j
                if rix != -1 and (na(mta) or rlo != mta)
                    mta     := rlo
                    mtaTime := array.get(zT, rix)
    if trend == -1
        if not na(mta) and close > mta
            trend := 0
            aPx   := na
            aBar  := -1
        else if not na(aPx) and close < aPx
            bIdx  = -1
            for j = 0 to n - 1 by 1
                if (not array.get(zH, j)) and array.get(zB, j) > aBar and array.get(zP, j) < aPx
                    bIdx  := j
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
            else
                rhi = float(na)
                rix = -1
                for j = 0 to n - 1 by 1
                    if array.get(zH, j) and array.get(zB, j) > aBar
                        if na(rhi) or array.get(zP, j) > rhi
                            rhi := array.get(zP, j)
                            rix := j
                if rix != -1 and (na(mta) or rhi != mta)
                    mta     := rhi
                    mtaTime := array.get(zT, rix)
    ended = trend == 0 and not na(mta)
    [mta, trend, mtaTime, ended]
selTF = selTFin == '' ? timeframe.period : selTFin
[cM, cTr, cMt, cEnd] = request.security(syminfo.tickerid, timeframe.period, f_mtaCalc(), lookahead = barmerge.lookahead_off, gaps = barmerge.gaps_off)
[sM, sTr, sMt, sEnd] = request.security(syminfo.tickerid, selTF, f_mtaCalc(), lookahead = barmerge.lookahead_off, gaps = barmerge.gaps_off)
[cdir, cz1, cz2] = ZigZag.zigzag(low, high, Depth, Deviation, Backstep)
var line zz = na
if showZZ
    zz := line.new(cz1, cz2, xloc.bar_time, extend.none, color.new(cdir > 0 ? upColor : dnColor, 0), width = lineThick)
    if not na(zz[1])
        if cdir == cdir[1]
            line.delete(zz[1])
        else
            line.set_extend(zz[1], extend.none)
var line  lC = na
var label bC = na
if showChart and not na(cM) and not na(cMt) and cMt > 0
    col = cTr == 1 ? cUp : cTr == -1 ? cDn : trCol
    st  = cTr == 1 ? label.style_label_up : label.style_label_down
    ln  = cEnd ? line.style_dotted : line.style_solid
    if na(lC)
        lC := line.new(cMt, cM, cMt + 1, cM, xloc = xloc.bar_time, extend = extend.right, color = col, width = 2, style = ln)
    else
        line.set_xy1(lC, cMt, cM)
        line.set_xy2(lC, cMt + 1, cM)
        line.set_color(lC, col)
        line.set_style(lC, ln)
    if na(bC)
        bC := label.new(time, cM, 'MTA', xloc = xloc.bar_time, yloc = yloc.price, style = st, color = color.new(color.white, 100), textcolor = col, size = size.small)
    else
        label.set_xy(bC, time + lblOff * (timeframe.in_seconds() * 1000), cM)
        label.set_text(bC, 'MTA')
        label.set_textcolor(bC, col)
        label.set_style(bC, st)
else
    if not na(lC)
        line.delete(lC)
        lC := na
    if not na(bC)
        label.delete(bC)
        bC := na
var line  lS = na
var label bS = na
if showSel and not na(sM) and not na(sMt) and sMt > 0
    col2 = sTr == 1 ? sUp : sTr == -1 ? sDn : trCol
    st2  = sTr == 1 ? label.style_label_up : label.style_label_down
    ln2  = sEnd ? line.style_dotted : line.style_solid
    txt2 = 'MTA ' + f_tfJP(selTF)
    if na(lS)
        lS := line.new(sMt, sM, sMt + 1, sM, xloc = xloc.bar_time, extend = extend.right, color = col2, width = 3, style = ln2)
    else
        line.set_xy1(lS, sMt, sM)
        line.set_xy2(lS, sMt + 1, sM)
        line.set_color(lS, col2)
        line.set_style(lS, ln2)
    if na(bS)
        bS := label.new(time, sM, txt2, xloc = xloc.bar_time, yloc = yloc.price, style = st2, color = color.new(color.white, 100), textcolor = col2, size = size.small)
    else
        label.set_xy(bS, time + lblOff * (timeframe.in_seconds() * 1000), sM)
        label.set_text(bS, txt2)
        label.set_textcolor(bS, col2)
        label.set_style(bS, st2)
else
    if not na(lS)
        line.delete(lS)
        lS := na
    if not na(bS)
        label.delete(bS)
        bS := na
sym = syminfo.ticker
if almChart and not na(cTr) and not na(cTr[1])
    if cTr[1] == 0 and cTr == 1 and almOccur and almUp
        alert(sym + ' チャート足MTA 上昇発生 @' + str.tostring(cM), alert.freq_once_per_bar_close)
    if cTr[1] == 0 and cTr == -1 and almOccur and almDn
        alert(sym + ' チャート足MTA 下降発生 @' + str.tostring(cM), alert.freq_once_per_bar_close)
    if cTr[1] == 1 and cTr == 0 and almBreak and almUp
        alert(sym + ' チャート足MTA 上昇終了(ブレイク) @' + str.tostring(cM), alert.freq_once_per_bar_close)
    if cTr[1] == -1 and cTr == 0 and almBreak and almDn
        alert(sym + ' チャート足MTA 下降終了(ブレイク) @' + str.tostring(cM), alert.freq_once_per_bar_close)
if almSel and not na(sTr) and not na(sTr[1])
    stf = f_tfJP(selTF)
    if sTr[1] == 0 and sTr == 1 and almOccur and almUp
        alert(sym + ' ' + stf + 'MTA 上昇発生 @' + str.tostring(sM), alert.freq_once_per_bar_close)
    if sTr[1] == 0 and sTr == -1 and almOccur and almDn
        alert(sym + ' ' + stf + 'MTA 下降発生 @' + str.tostring(sM), alert.freq_once_per_bar_close)
    if sTr[1] == 1 and sTr == 0 and almBreak and almUp
        alert(sym + ' ' + stf + 'MTA 上昇終了(ブレイク) @' + str.tostring(sM), alert.freq_once_per_bar_close)
    if sTr[1] == -1 and sTr == 0 and almBreak and almDn
        alert(sym + ' ' + stf + 'MTA 下降終了(ブレイク) @' + str.tostring(sM), alert.freq_once_per_bar_close)