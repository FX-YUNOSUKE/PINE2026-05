# Double BCD Strategy v0.4 — 200EMA位置フィルター改良版

## 改善内容

これまでの v0.3 では、200EMAフィルターを以下のように判定していました。

- ショート：`close < ema200`
- ロング：`close > ema200`

しかしこの判定だと、エントリー足の価格だけで判断してしまうため、  
実際の損切り位置が200EMAをまたいでいるケースでもエントリーしてしまう可能性があります。

今回の改良では、以下の条件に変更しています。

## 新しい200EMAフィルター条件

### ショート条件

ショートは、

- エントリー想定価格が200EMAより下
- 損切り価格も200EMAより下

この2つを満たす時だけエントリーします。

```pine
shortEmaOK = not useEMAfilter or (entryRef < ema200 and slPrice < ema200)
```

### ロング条件

ロングは、

- エントリー想定価格が200EMAより上
- 損切り価格も200EMAより上

この2つを満たす時だけエントリーします。

```pine
longEmaOK = not useEMAfilter or (entryRef > ema200 and slPrice > ema200)
```

---

## 差し替えコード

以下を、元コードの `ENTRY + EXIT` 以降と差し替えてください。

```pine
/// ENTRY + EXIT
var float tpPrice = na
var float slPrice = na

// エントリー判定に使う価格
// ※ strategy.entry は次足約定なので、ここではシグナル確定足の close を想定エントリー価格として使う
entryRef = close

// M TOP 確定 → SHORT
if inWindow and tradeMTop and mForm and strategy.position_size == 0
    if useATRexit
        risk = slATRmult * atrG
        slPrice := entryRef + risk
        tpPrice := entryRef - risk * rrRatio
    else
        h = mBp - mCp
        tpPrice := mCp - h
        slPrice := mBp + slBufATR * atrG

    // ショートは「エントリー想定価格」と「損切り価格」の両方がEMA200より下の時だけ
    shortEmaOK = not useEMAfilter or (entryRef < ema200 and slPrice < ema200)

    if shortEmaOK
        strategy.entry('Short', strategy.short)


// W BOTTOM 確定 → LONG
if inWindow and tradeWBot and wForm and strategy.position_size == 0
    if useATRexit
        risk = slATRmult * atrG
        slPrice := entryRef - risk
        tpPrice := entryRef + risk * rrRatio
    else
        h = wCp - wBp
        tpPrice := wCp + h
        slPrice := wBp - slBufATR * atrG

    // ロングは「エントリー想定価格」と「損切り価格」の両方がEMA200より上の時だけ
    longEmaOK = not useEMAfilter or (entryRef > ema200 and slPrice > ema200)

    if longEmaOK
        strategy.entry('Long', strategy.long)


// TP(limit) + SL(stop)
if strategy.position_size < 0
    strategy.exit('Exit Short', from_entry = 'Short', stop = slPrice, limit = tpPrice)

if strategy.position_size > 0
    strategy.exit('Exit Long', from_entry = 'Long', stop = slPrice, limit = tpPrice)


// 期間フィルタの外に出たら強制クローズ
if useDateFilter and not inWindow and strategy.position_size != 0
    strategy.close_all()


/// VISUAL
plot(ema200, title = 'EMA200', color = color.new(color.orange, 0), linewidth = 2)
plotshape(mForm, title = 'M TOP detected', style = shape.triangledown, location = location.abovebar, color = color.new(color.red, 0), size = size.small)
plotshape(wForm, title = 'W BOTTOM detected', style = shape.triangleup, location = location.belowbar, color = color.new(color.blue, 0), size = size.small)
```

---

## 注意点

`strategy.entry()` は通常、シグナルが出た次の足で約定します。  
そのため、今回のコードでは `entryRef = close` として、  
シグナル確定足の終値をエントリー想定価格として判定しています。

より厳密に「実際の約定価格」で判定したい場合は、  
エントリー後にポジション平均価格 `strategy.position_avg_price` を使った管理ロジックにする必要があります。

ただし、今回の目的である

- 損切り位置が200EMAをまたぐエントリーを避ける
- パターン全体がEMAの同じ側にある時だけ仕掛ける

という目的には、この形がシンプルで扱いやすいです。
