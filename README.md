# Session Volume Structure Levels (HVN & LVN)

## Overview

This indicator analyzes volume distribution within a user-defined session and highlights price levels associated with higher and lower traded volume. It focuses on identifying High Volume Nodes (HVNs) and Low Volume Nodes (LVNs) as reference levels for intraday market structure analysis.

## What the script does

Within the selected session window, the script evaluates volume across price buckets and identifies:

High Volume Nodes (HVNs): price levels with relatively high traded volume

Low Volume Nodes (LVNs): price levels with relatively low traded volume

Instead of displaying a full volume profile, the script plots these derived levels directly on the chart for clarity.

## How the levels are calculated

During the session window, the script:

Tracks volume across discrete price buckets

Aggregates volume per bucket

Classifies buckets using configurable thresholds:

HVN → volume above a defined tier

LVN → volume below a defined tier

Each detected level is plotted as a horizontal reference line.

## Optional signal logic (experimental)

The script can optionally generate basic entry markers based on price interaction with HVN/LVN levels.

ATR-based exit logic is included for testing and visualization purposes only.

## Visual output

Session-based HVN and LVN lines

Minimal, uncluttered chart presentation

Optional enable/disable controls for plotted elements

## Inputs & customization

Session window

Volume bin size

HVN / LVN multipliers

Visibility toggles for structure levels

## Use cases

This tool is intended for studying intraday volume structure across markets such as:

Futures

Indices

FX sessions

Crypto (custom session timing)

## Important note

This script is designed for analysis and experimentation. It does not constitute a complete trading system and is best used as a structural reference component within a broader trading approach.

## Automate It with PineGen AI

Take this strategy to the next level with PineGen AI automation:

Website: https://www.pinegen.ai/

Twitter: https://x.com/PineGenAI

Telegram Channel: https://t.me/PineGenAI

YouTube Channel: https://www.youtube.com/@pinegenai

## Pine Script Code

```pine
//@version=6
strategy("Session Volume Profile Sniffer – HVN & Rejection Zones (Fixed & Improved)",
     overlay=true,
     initial_capital=10000,
     commission_type=strategy.commission.percent,
     commission_value=0.04,
     pyramiding=0)

//──────────────────────────────────────────
// Inputs
//──────────────────────────────────────────
sessionTime   = input.session("0900-1600", "Session Time")
hvnMultiplier = input.float(1.6, "HVN Multiplier", minval=1)
lvnMultiplier = input.float(0.75, "LVN Multiplier", minval=0.1)
minBins       = input.int(6, "Minimum Volume Bins", minval=1)
showZones     = input.bool(true, "Show HVN / LVN Lines")

// Risk settings
atrMultSL     = input.float(1.0, "ATR Stop-Loss Multiplier", step=0.1)
atrMultTP     = input.float(2.5, "ATR Take-Profit Multiplier", step=0.1)
avoidChopTime = input.bool(true, "Avoid Mid-Session Chop (11:30–13:30)")

//──────────────────────────────────────────
// Session boolean
//──────────────────────────────────────────
inSession = not na(time(timeframe.period, sessionTime))
sessionChange = inSession and not inSession[1]

//──────────────────────────────────────────
// Arrays per session
//──────────────────────────────────────────
var float[] volArr = array.new_float()
var float[] priceArr = array.new_float()

if sessionChange
    array.clear(volArr)
    array.clear(priceArr)

//──────────────────────────────────────────
// Build volume profile per session
//──────────────────────────────────────────
if inSession
    bucket = math.round(close / syminfo.mintick) * 1.0
    idx = array.indexof(priceArr, bucket)
    if idx == -1
        array.push(priceArr, bucket)
        array.push(volArr, volume)
    else
        array.set(volArr, idx, array.get(volArr, idx) + volume)

//──────────────────────────────────────────
// HVN / LVN Detection (safe loops)
//──────────────────────────────────────────
float hvnPrice = na
float lvnPrice = na

if array.size(volArr) >= minBins
    // compute average volume across bins
    sumV = 0.0
    for i = 0 to array.size(volArr) - 1
        sumV := sumV + array.get(volArr, i)
    avgV = sumV / array.size(volArr)

    // HVN: bins significantly above average
    for i = 0 to array.size(volArr) - 1
        if array.get(volArr, i) > avgV * hvnMultiplier
            // choose the most recent HVN found in session (overwrites older)
            hvnPrice := array.get(priceArr, i) * syminfo.mintick

    // LVN: bins significantly below average
    for i = 0 to array.size(volArr) - 1
        if array.get(volArr, i) < avgV * lvnMultiplier
            lvnPrice := array.get(priceArr, i) * syminfo.mintick

//──────────────────────────────────────────
// Plot HVN / LVN
//──────────────────────────────────────────
if showZones and not na(hvnPrice)
    line.new(
        x1 = bar_index - 1, y1 = hvnPrice,
        x2 = bar_index,     y2 = hvnPrice,
        xloc = xloc.bar_index,
        color = color.new(color.orange, 0),
        width = 2,
        extend = extend.none
    )

if showZones and not na(lvnPrice)
    line.new(
        x1 = bar_index - 1, y1 = lvnPrice,
        x2 = bar_index,     y2 = lvnPrice,
        xloc = xloc.bar_index,
        color = color.new(color.green, 0),
        width = 2,
        extend = extend.none
    )

//──────────────────────────────────────────
// Trend filter (200 EMA)
//──────────────────────────────────────────
ema200 = ta.ema(close, 200)
bullTrend = close > ema200
bearTrend = close < ema200

//──────────────────────────────────────────
// Reject Confirmation (stronger conditions)
//──────────────────────────────────────────
bullReject = not na(lvnPrice) and low <= lvnPrice and close > open and close > lvnPrice
bearReject = not na(hvnPrice) and high >= hvnPrice and close < open and close < hvnPrice

//──────────────────────────────────────────
// Avoid mid-session chop (local time via hour/minute)
//──────────────────────────────────────────
hh = hour(time)
mm = minute(time)
inChopWindow = (hh == 11 and mm >= 30) or (hh == 12) or (hh == 13 and mm < 30)
chopZone = avoidChopTime and inChopWindow

//──────────────────────────────────────────
// Entry Conditions (filtered)
//──────────────────────────────────────────
longSignal  = bullTrend and bullReject and not chopZone
shortSignal = bearTrend and bearReject and not chopZone

//──────────────────────────────────────────
// Execute trades with ATR-based exits
//──────────────────────────────────────────
atr = ta.atr(14)

if longSignal and strategy.position_size == 0
    entryPrice = close
    stopPrice = entryPrice - atr * atrMultSL
    limitPrice = entryPrice + atr * atrMultTP
    strategy.entry("Long", strategy.long)
    strategy.exit("Exit Long", "Long", stop = stopPrice, limit = limitPrice)

if shortSignal and strategy.position_size == 0
    entryPrice = close
    stopPrice = entryPrice + atr * atrMultSL
    limitPrice = entryPrice - atr * atrMultTP
    strategy.entry("Short", strategy.short)
    strategy.exit("Exit Short", "Short", stop = stopPrice, limit = limitPrice)

//──────────────────────────────────────────
// Optional visuals: show current HVN/LVN labels
//──────────────────────────────────────────
if not na(hvnPrice)
    label.new(bar_index, hvnPrice, text = "HVN", color = color.orange, textcolor = color.white, style=label.style_label_left, yloc=yloc.price)
if not na(lvnPrice)
    label.new(bar_index, lvnPrice, text = "LVN", color = color.green, textcolor = color.white, style=label.style_label_left, yloc=yloc.price)
```
