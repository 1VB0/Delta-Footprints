//# Delta-Footprints update Confirmed Works Live 19:13 est
//PineScript Delta Volume Order Flow @1VB0 (GitHub)

//@version=6
indicator("Footprint Delta [S1 - Zone Clean]", overlay=true, max_boxes_count=500, max_lines_count=500, max_labels_count=500)

// =============================
// INPUTS
// =============================
timeframeInput = input.timeframe("1", "Timeframe")
sizeInput      = input.float(1.5, "Zone Size (pips)", minval=0.25, maxval=20.0, step=0.25)
autoInput      = input.bool(true, "Auto Size (ATR)")
show_tooltip   = input.bool(true, "Enable Tooltips")
deltaGap       = input.float(3.0, "Delta Gap Below Low", minval=1.0, maxval=10.0, step=0.5)

// =============================
// COLORS
// =============================
transparent = color.new(color.white, 100)

// =============================
// ZOOM — only hides text when zoomed far out
// =============================
int  visibleBars = math.max(int((last_bar_time - chart.left_visible_bar_time) / timeframe.in_seconds(timeframe.period) / 1000), 1)
bool showText    = visibleBars < 400

// =============================
// LOWER TF DATA
// =============================
[oArr, hArr, lArr, cArr, vArr] = request.security_lower_tf(syminfo.tickerid, timeframeInput, [open, high, low, close, volume])

// =============================
// ZONE SIZE
// =============================
float volatility = math.round_to_mintick(0.1 * ta.atr(200))
float parsedSize = autoInput ? volatility : math.round_to_mintick(sizeInput * syminfo.mintick * 10)
int   idx        = math.max(int(math.round((high - low) / parsedSize)), 1)
float gridGap    = ta.atr(14) * deltaGap * 0.15

// =============================
// ANCHOR
// =============================
int xLeft  = bar_index
int xRight = bar_index + 1

// =============================
// FOOTPRINT LOOP
// =============================
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
                float barRange   = math.max(bh - bl, syminfo.mintick)
                float overlap    = math.min(bh, maxLevel) - math.max(bl, minLevel)
                float proportion = math.max(overlap / barRange, 0.0)
                float allocVol   = bv * proportion

                if bc >= bo
                    buyVol  += allocVol
                else
                    sellVol += allocVol

    float delta = buyVol - sellVol
    totalDelta += delta

    string zonePrice = str.tostring(math.round_to_mintick(minLevel), "#.##")
    string bStr = math.abs(buyVol)  >= 1000.0 ? str.tostring(buyVol  / 1000.0, "#.0") + "K" : str.tostring(math.round(buyVol,  0), "#")
    string sStr = math.abs(sellVol) >= 1000.0 ? str.tostring(sellVol / 1000.0, "#.0") + "K" : str.tostring(math.round(sellVol, 0), "#")
    string dStr = math.abs(delta)   >= 1000.0 ? str.tostring(delta   / 1000.0, "#.0") + "K" : str.tostring(math.round(delta,   0), "#")

    color bgCol = delta >= 0.0 ? color.new(color.lime, 88) : color.new(color.red, 88)
    color css   = delta >= 0.0 ? color.new(color.lime, 15) : color.new(color.red, 15)

    box.new(xLeft, maxLevel, xRight, minLevel,
        text         = showText ? bStr + "x" + sStr : "",
        text_color   = css,
        text_size    = size.tiny,
        bgcolor      = bgCol,
        border_color = color.black,
        border_width  = 1)

    if show_tooltip
        string tip = zonePrice + "\nB: " + bStr + "  S: " + sStr + "\nΔ: " + dStr
        label.new(xLeft, (minLevel + maxLevel) / 2.0,
            text      = "",
            style     = label.style_none,
            color     = transparent,
            textcolor = transparent,
            tooltip   = tip)

    line.new(xLeft, minLevel, xRight, minLevel, color=color.black, width=1)
    minLevel := maxLevel

// Top border
line.new(xLeft, maxLevel, xRight, maxLevel, color=color.black, width=1)

// =============================
// TOTAL DELTA — box below wick, no pointer, size.tiny
// Box height matches one zone slice
// =============================
float zoneH     = math.max((high - low) / idx, ta.atr(14) * 0.12)
float deltaBoxT = low - gridGap
float deltaBoxB = deltaBoxT - zoneH

color  totalCol = totalDelta >= 0.0 ? color.new(color.lime, 15) : color.new(color.red, 15)
color  totalBg  = totalDelta >= 0.0 ? color.new(color.lime, 78) : color.new(color.red, 78)
string totalStr = math.abs(totalDelta) >= 1000000.0 ? str.tostring(totalDelta / 1000000.0, "#.0") + "M" : math.abs(totalDelta) >= 1000.0 ? str.tostring(totalDelta / 1000.0, "#.0") + "K" : str.tostring(math.round(totalDelta, 0), "#")

box.new(xLeft, deltaBoxT, xRight, deltaBoxB,
    text         = showText ? "Σ " + totalStr : "",
    text_color   = totalCol,
    text_size    = size.tiny,
    bgcolor      = totalBg,
    border_color = color.black,
    border_width  = 1)

// =============================
// CANDLES
// =============================
plotcandle(open, high, low, close,
    color       = transparent,
    wickcolor   = close > open ? color.new(color.lime, 20) : color.new(color.red, 20),
    bordercolor = transparent)

