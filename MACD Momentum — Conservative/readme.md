Below is a concise, clear wrap-up of how the strategy decides to enter, close, set stop-loss and take-profit — including the EMA-exit add-on, trailing stops, buffers, and webhook behavior. I reference the variable names used in the script so you can map each rule back to the code quickly.

High-level summary
Entry logic: Variant 5 (Conservative) — a one-bar confirmed MACD crossover + histogram momentum + above-average volume + confirming candle above/below 50 EMA + optional MACD strength filter + session/volatility checks.
Stop-loss / Take-profit: percent-based SL and TP (computed from entry price) with a configurable small tick buffer. Applied via strategy.exit() (fixed or with trailing).
Trailing stop: optional; implemented as a trailing stop in price units (computed from trailPercent of entry price).
EMA counter-trend exit: optional; two modes:
Market Close: immediately close the position when price crosses the EMA against the trade.
Set Stop at EMA: attach/update a protective stop at the EMA level (plus/minus buffer).
Webhook behavior: when a trade is entered/exited (and if sendAlertsFromScript = true), the script calls alert(payload, ...) with either PineConnector CSV or JSON payload. TradingView alert must be configured to trigger on "Any alert() function call" and send to your PineConnector webhook.
Entry conditions (long and short) These are the boolean variables used:
crossover_onebar

In code: (macdLine[1] > signalLine[1]) and (macdLine[2] <= signalLine[2])
Meaning: the MACD line crossed above the signal line on the previous bar (one-bar confirmation).
histRising / histFalling

histRising: histLine > 0 and ta.change(histLine) > 0
histFalling: histLine < 0 and ta.change(histLine) < 0
Meaning: MACD histogram is on the expected side and increasing magnitude.
volOk

volOk: volume > ta.sma(volume, 20)
Meaning: volume above 20-bar average on the signal bar.
confirmBull / confirmBear

confirmBull: close > open and close > emaTrend and close > high[1]
confirmBear: close < open and close < emaTrend and close < low[1]
Meaning: the signal bar is a confirming candle oriented with the trade and above/below the EMA and stronger than previous bar.
strengthPassBuy / strengthPassSell

strengthPassBuy: (not useFilter) or (macdLine > strengthLevel)
strengthPassSell: (not useFilter) or (macdLine < -strengthLevel)
Meaning: optional numeric MACD threshold for extra strength.
validMarket

True when session filter and optional volatility filter allow trading.
Final entry booleans:

buyCond = crossover_onebar and histRising and volOk and confirmBull and strengthPassBuy and validMarket
sellCond = crossunder_onebar and histFalling and volOk and confirmBear and strengthPassSell and validMarket
Note: Because calc_on_every_tick=false the conditions evaluate at bar close.

Entry execution & duplicate prevention
When buyCond / sellCond are true and strategy.position_size == 0, script does:
strategy.entry("LONG_ENTRY", strategy.long, qty = orderSize) or strategy.entry("SHORT_ENTRY", strategy.short, qty = orderSize)
Sends a dynamic webhook payload via alert(payload_buy/payload_sell) if sendAlertsFromScript = true.
The script uses var string lastSignalSent to avoid re-sending the same entry signal repeatedly while the position is open. lastSignalSent is reset to "NONE" when strategy.position_size == 0.
Optional retry logic (commented) can re-enable sending after N bars if an external executor fails to open the position.
Stop-loss and take-profit calculation (Percent mode)
Input variables:
stopLossValue (percent), takeProfitValue (percent)
bufferTicks (integer) → buf = bufferTicks * syminfo.mintick
At entry price entryPrice (the script uses strategy.position_avg_price once in position), prices are computed:
For a long:

longStop := entryPrice * (1 - stopLossValue / 100) - buf
Effect: a price below entry by stopLossValue% minus small tick buffer.
longTP := entryPrice * (1 + takeProfitValue / 100)
For a short:

shortStop := entryPrice * (1 + stopLossValue / 100) + buf

Effect: a price above entry by stopLossValue% plus buffer.
shortTP := entryPrice * (1 - takeProfitValue / 100)

These are applied with strategy.exit(..., stop=..., limit=...) for each side while strategy.position_size != 0.

Trailing stop
If useTrailingStop is true:
trailPointsLong := entryPrice * (trailPercent / 100)
trailPointsShort := entryPrice * (trailPercent / 100)
Then strategy.exit uses trail_points=trailPointsLong/Short when not na. trail_points expects price distance (the script supplies absolute price distance computed from the entry price and trailPercent).
Note: trail_points is an absolute price distance (not percent) in this implementation — we convert percent->price using the entry price.
EMA counter-trend exit (new scenario)
Inputs:
emaExitEnabled: boolean
emaExitMode: "Market Close" or "Set Stop at EMA"
emaExitBufferTicks: ticks to offset the protective stop
emaExitSendWebhook: whether to send a webhook on EMA exit events
Detection:
emaCrossDown := ta.crossunder(close, emaTrend) // price crossed below EMA
emaCrossUp := ta.crossover(close, emaTrend) // price crossed above EMA
Behavior when a position exists:
For a long (position_size > 0):
If emaExitMode == "Market Close" and emaCrossDown occurs → strategy.close("LONG_ENTRY") and optional webhook EXIT payload sent.
If emaExitMode == "Set Stop at EMA" → compute ema_stop_long = emaTrend - (emaExitBufferTicks * tick) and call strategy.exit(id="EMA_PROT_LONG", from_entry="LONG_ENTRY", stop=ema_stop_long). The protective stop is updated every bar.
For a short (position_size < 0):
If emaExitMode == "Market Close" and emaCrossUp occurs → strategy.close("SHORT_ENTRY")
If emaExitMode == "Set Stop at EMA" → ema_stop_short = emaTrend + buffer and strategy.exit(... stop=ema_stop_short)
Notes:

The "Set Stop at EMA" mode keeps attaching/updating a protective stop at EMA so that when price reaches EMA the broker triggers the stop order (acting as a counter-trend protective guard).
The "Market Close" mode issues an immediate close — in backtest this is strategy.close; when used with PineConnector you should also send an EXIT webhook so your external executor knows the signal.
Webhook payloads / alerts
Two payload formats selectable by connectorFormat:
PineConnector CSV (default): example line format used in script: pine_license_id + ",BUY," + symbol + "," + orderSize + "," + sl_price + "," + tp_price + ",auto" Example: 8426965401813,BUY,XAUUSD,0.10,1840.50,1860.00,auto
JSON (custom): example: {"action":"BUY","symbol":"XAUUSD","size":0.1,"entry_price":1850.00,"sl_price":1840.5,"tp_price":1860.0}
The script builds payload_buy and payload_sell strings at top-level and calls:
alert(payload_buy, alert.freq_once_per_bar_close)
Important: alert(payload, ...) uses a series string and must be delivered by creating a TradingView alert that listens to "Any alert() function call" and points to your PineConnector webhook URL.
alertcondition() is present in the script but its message is a constant token (e.g., "PINE_BUY" / "PINE_SELL") because alertcondition(message=...) requires a compile-time constant. Use these only if you want a constant, small token or a UI-visible alert.
How exits interact (order of precedence)
EMA-exit Market Close triggers a strategy.close(...) which closes the position immediately and therefore cancels open strategy.exit protective orders.
EMA-exit "Set Stop at EMA" is implemented via strategy.exit(...) with a stop. This is an additional exit in parallel to the percent-based stop/tp/trailing stop. The broker will act on whichever stop/limit is hit first; in the script, both the percent stop and the EMA protective stop can coexist (different strategy.exit ids).
strategy.exit for regular percent-based stop/TP are attached while position exists (id: LONG_EXIT/SHORT_EXIT).
Trailing stop (if provided) is applied via strategy.exit(..., trail_points=...) as part of LONG_EXIT/SHORT_EXIT.
Implementation details / constants and helpers
syminfo.mintick → smallest price unit (tick)
buf = bufferTicks * syminfo.mintick → small buffer added/subtracted to stops to avoid immediate hit at market micro-prices
Prices are formatted with format.mintick when used in payload strings.
All condition-check plotting (plotshape) is declared at top-level and gated with showChecks (plotshape(showChecks and condition, ...)) because Pine disallows plotting calls inside local blocks.
Example numeric calculation (long)
Inputs for example:
entryPrice = 1850.00
stopLossValue = 1.0 (%)
takeProfitValue = 2.0 (%)
bufferTicks = 2; assume syminfo.mintick = 0.01 → buf = 0.02
Computed:
SL price = 1850 * (1 - 0.01) - 0.02 = 1850 * 0.99 - 0.02 = 1831.5 - 0.02 = 1831.48
TP price = 1850 * 1.02 = 1887.00
trailPointsLong (if trailPercent=0.8) = 1850 * 0.008 = 14.8 (so trailing stop distance = 14.8 price units)
The strategy will call strategy.exit(id="LONG_EXIT", from_entry="LONG_ENTRY", stop=1831.48, limit=1887.00, trail_points=14.8) (trail_points included only if enabled)
Practical checklist to verify behavior in your environment
For live/external execution via PineConnector → MT5:
Confirm TradingView alert uses condition "Any alert() function call" and URL is your PineConnector webhook.
Confirm pine_license_id (if required) is correct for PineConnector CSV.
Confirm symbol mapping in PineConnector to your MT5 broker symbol (e.g., XAUUSD vs GOLD vs XAUUSDm).
Confirm SL/TP units expected by your EA (price vs pips vs percent); adjust script payload accordingly.
Test with manual curl payloads to PineConnector first, then with script alert send on a demo account.
For backtesting/visual:
Leave strategy.entry calls enabled so the TradingView strategy simulates fills and the table shows P/L and Position state.
Use showChecks = true to debug which subchecks are passing.
Suggested small improvements you might add
Grace period for EMA exit: require N bars after entry before EMA-exit is allowed to avoid immediate whipsaw exits.
Add an order_id or timestamp in webhook payload so your connector/EA can deduplicate.
Add unit selection for SL/TP payloads (price vs pips vs percent) to match EA expectations.
Optional automatic retry logic (resend webhook after N bars if no external position opened).
If you want, I can:

Produce a one-page quick reference table (CSV) of the exact variable → formula mappings for copy/paste into documentation or README.
Modify payload to match the exact PineConnector CSV schema your connector instance expects (paste a working CSV example from the PineConnector log and I’ll adapt the script).
Add an EMA-exit grace period (require N bars after entry) and return the updated script.
