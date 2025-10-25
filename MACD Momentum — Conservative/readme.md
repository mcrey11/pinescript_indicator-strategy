Options (choose one)

A — Webhook-only (no strategy trades): you only want alerts sent to PineConnector and do NOT want the Pine script to place strategy.entry orders. Useful when PineConnector / your broker executes the real orders and you only want Pine to generate signals.
B — Strategy + webhook: you want the strategy to still place backtest / paper trades in TradingView AND also send the webhook. (This is what the current script does, but I can refine timing/cooldown.)
C — Use strategy orders differently: place limit/stop orders from Pine (strategy.order with limit/stop) rather than immediate market entry. Use this if you want entries at specific price levels.
Which one do you want? Below are ready-to-paste code snippets and a short explanation for each.

A — Webhook-only (remove strategy.entry)

Behavior: script will never place strategy.entry orders. It only sends alert(payload, ...) when the signal triggers. Useful when PineConnector is the only executor.
Replace the entry blocks with this:

pine
// WEBHOOK-ONLY: no strategy.entry calls, only send alert payloads
if (buyCond and lastSignalSent != "BUY")
    // only alert — PineConnector will execute the trade
    if sendAlertsFromScript
        alert(payload_buy, alert.freq_once_per_bar_close)
    lastSignalSent := "BUY"

if (sellCond and lastSignalSent != "SELL")
    if sendAlertsFromScript
        alert(payload_sell, alert.freq_once_per_bar_close)
    lastSignalSent := "SELL"

// optional: reset lastSignalSent after N bars if you want to re-send same side later
var int barsSinceSignal = 0
if lastSignalSent == "NONE"
    barsSinceSignal := 0
else
    barsSinceSignal += 1
// e.g. allow resend after 3 bars:
if barsSinceSignal >= 3
    lastSignalSent := "NONE"
Notes for A:

Create a TradingView alert with Condition = this indicator and choose "Any alert() function call", set the PineConnector webhook URL.
Advantage: no double-execution risk between strategy and PineConnector. Backtest results inside TradingView will be meaningless (no entries), but you can still use the script for signals only.
B — Strategy + webhook but better cooldown and send webhook only once per actual order

Current script places strategy.entry and sends alert at same time. If PineConnector takes time to execute and you want to avoid re-sends while you have an open position, use the position check to stop re-sending and reset only when position closes (or after X bars). Use this version if you want both TradingView backtests and real trade execution via PineConnector.
Replace the entry blocks with this:

pine
// STRATEGY + WEBHOOK: place tradingview entry and send webhook once per open-position lifecycle
if (buyCond and strategy.position_size == 0 and lastSignalSent != "BUY")
    strategy.entry("LONG_ENTRY", strategy.long, qty = orderSize)
    if sendAlertsFromScript
        alert(payload_buy, alert.freq_once_per_bar_close)
    lastSignalSent := "BUY"

// send for short
if (sellCond and strategy.position_size == 0 and lastSignalSent != "SELL")
    strategy.entry("SHORT_ENTRY", strategy.short, qty = orderSize)
    if sendAlertsFromScript
        alert(payload_sell, alert.freq_once_per_bar_close)
    lastSignalSent := "SELL"

// Reset logic: only reset after the position actually closes
if (strategy.position_size == 0)
    lastSignalSent := "NONE"
Refinements you might want:

If PineConnector can fail, you might want a retry flow: reset lastSignalSent after N bars so the script will re-send the webhook if position didn't open externally. (Add barsSinceSignal counter similar to A.)
If PineConnector requires a unique order ID per signal, include time or an incrementing id in the payload.
C — Place limit/stop entry orders from Pine (strategy.order) and send webhook for info only

Use strategy.order with limit or stop price. Example placing a buy-limit at current close - delta (or at next bar open is typical in backtest).
Example placing a limit buy slightly below close:

pine
limitDelta = 0.002 * close  // 0.2% below current close, adjust as needed
buyLimitPrice = close - limitDelta

if (buyCond and strategy.position_size == 0 and lastSignalSent != "BUY")
    strategy.order(id="LONG_LIMIT", long=true, qty=orderSize, limit=buyLimitPrice)
    if sendAlertsFromScript
        alert(payload_buy, alert.freq_once_per_bar_close)
    lastSignalSent := "BUY"
Notes for C:

strategy.order with limit places a limit order in the backtest; behavior in real-time depends on calc_on_every_tick and other factors.
You can also use stop orders (strategy.order with stop parameter).
If you prefer, I can:

Produce a full script implementing the Webhook-only version (A) so nothing is entered by TradingView and PineConnector handles all execution.
Or produce a full script implementing Version B with an automatic retry after N bars if a position didn't open externally (useful if PineConnector fails occasionally).
Or implement version C with configurable limit/stop offsets and a debug flag to show payloads on the chart.


Variant 5 (the conservative entry logic) into a complete Pine v5 strategy file and kept the PineConnector webhook support and exit/stop logic intact.

(short):

Replaced the previous buy/sell conditions with the conservative Variant 5 logic: one-bar MACD confirmation + histogram momentum rising/falling + above-average volume + confirming candle relative to EMA + optional MACD strength filter. This reduces false entries and delays entry by one bar after the crossover (more robust).
Kept webhook payloads (CSV/JSON) and dynamic alert(payload, ...) calls. alertcondition() remains top-level with constant messages (required by Pine).
Added a showChecks debug toggle to plot small markers for the sub-checks and a dashboard table row that shows which subchecks passed on the current bar so you can visually verify why the strategy did or didn't enter.
Preserved stop/TP/trailing-stop logic and single-entry-per-position lifecycle behavior (lastSignalSent prevents duplicate sends until position closes).
How to test it quickly:

Load the script in TradingView on the symbol/timeframe you trade.
Use a timeframe where MACD/EMA behavior is meaningful (e.g., 15m, 1h).
Keep sendAlertsFromScript = true and create one TradingView Alert with Condition = this indicator and "Any alert() function call" and set your PineConnector webhook URL. The script will call alert(payload, ...) when a trade is placed and push the dynamic CSV/JSON to the webhook.
Use the showChecks toggle to see markers and the dashboard to confirm which part blocked/allowed the entry.
If you want the script to only send webhooks and NOT place paper/backtest trades in TradingView, I can remove strategy.entry calls and keep only alert(payload, ...). Tell me and I will provide that version.
