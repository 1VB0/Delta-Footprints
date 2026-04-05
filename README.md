//# Delta-Footprints
//PineScript Delta Volume Order Flow @1VB0 (GitHub)

//@version=6
indicator("Footprint Delta [S1 + Hollow Candles]", overlay=true, max_boxes_count=500, max_lines_count=500, max_labels_count=500)

// INPUTS
timeframeInput = input.timeframe("1", "Timeframe")
sizeInput      = input.float(20.0, "Row Size")
autoInput      = input.bool(true, "Auto")
show_tooltip   = input.bool(true, "Enable Tooltips")
deltaGap       = input.float(3.0, "Delta Label Gap", minval=1.0)

// COLORS
transparent = color.new(color.white, 100)
bullEdge    = color.lime
bearEdge    = color.red

// LOWER TF DATA
[oArr, hArr, lArr, cArr, vArr] = request.security_lower_tf(syminfo.tickerid, timeframeInput, [open, high, low, close, volume])

// ROW SIZE
float volatility = math.round_to_mintick(0.1 * ta.atr(200))
float parsedSize = autoInput ? volatility : sizeInput
int idx = math.max(int(math.round((high - low) / parsedSize)), 1)

// GAP
float gridGap = ta.atr(14) * deltaGap * 0.15

// ANCHOR
int xLeft  = bar_index
int xRight = bar_index + 1

// FOOTPRINT
float minLevel   = low
float maxLevel   = low
float totalDelta = 0.0

for i = 0 to idx - 1
    maxLevel += (high - low) / idx

    float buyVol  = 0.0
    float sellVol = 0.0

    if array.size(vArr) > 0
        for j = 0 to array.size(vArr) - 1
            float bh = array.get(hArr, j)
            float bl = array.get(lArr, j)
            float bo = array.get(oArr, j)
            float bc = array.get(cArr, j)
            float bv = array.get(vArr, j)

            if bh > minLevel and bl < maxLevel
                if bc >= bo
                    buyVol += bv
                else
                    sellVol += bv

    float delta = buyVol - sellVol
    totalDelta += delta

    color bgCol = delta >= 0 ? color.new(color.lime, 88) : color.new(color.red, 88)
    color css   = delta >= 0 ? color.new(color.lime, 20) : color.new(color.red, 20)

    string bStr = math.abs(buyVol)  >= 1000 ? str.tostring(buyVol / 1000, "#.0") + "K" : str.tostring(buyVol, "#")
    string sStr = math.abs(sellVol) >= 1000 ? str.tostring(sellVol / 1000, "#.0") + "K" : str.tostring(sellVol, "#")
    string dStr = math.abs(delta)   >= 1000 ? str.tostring(delta / 1000, "#.0") + "K" : str.tostring(delta, "#")

    // BOX
    box.new(xLeft, maxLevel, xRight, minLevel,
        text=bStr + "x" + sStr,
        text_color=css,
        text_size=size.tiny,
        bgcolor=bgCol,
        border_color=color.black)

    // TOOLTIP
    if show_tooltip
        tip = "Zone: " + str.tostring(minLevel) + " - " + str.tostring(maxLevel) +
              "\nBuy: " + bStr +
              "\nSell: " + sStr +
              "\nΔ: " + dStr

        label.new(xLeft, (minLevel + maxLevel) / 2,
            text="",
            style=label.style_none,
            color=transparent,
            textcolor=transparent,
            tooltip=tip)

    line.new(xLeft, minLevel, xRight, minLevel, color=color.black)

    minLevel := maxLevel

// TOP BORDER
line.new(xLeft, maxLevel, xRight, maxLevel, color=color.black)

// TOTAL DELTA LABEL
color totalCol = totalDelta >= 0 ? color.new(color.lime, 20) : color.new(color.red, 20)
color totalBg  = totalDelta >= 0 ? color.new(color.lime, 80) : color.new(color.red, 80)

string totalStr =
     math.abs(totalDelta) >= 1000000 ? str.tostring(totalDelta / 1000000, "#.0") + "M" :
     math.abs(totalDelta) >= 1000     ? str.tostring(totalDelta / 1000, "#.0") + "K" :
                                       str.tostring(totalDelta, "#")

label.new(xLeft, low - gridGap,
    text="Σ " + totalStr,
    style=label.style_label_up,
    color=totalBg,
    textcolor=totalCol,
    size=size.tiny)

// =============================
// ✅ HOLLOW CANDLES (ADDED CLEANLY)
// =============================
plotcandle(
    open,
    high,
    low,
    close,
    color = transparent,                        // hollow body
    wickcolor = color.gray,                    // wick
    bordercolor = close > open ? bullEdge : bearEdge // outline
)
