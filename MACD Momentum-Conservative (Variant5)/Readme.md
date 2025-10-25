# MACD Momentum Strategy — Conservative Variant 5

A sophisticated Pine Script v5 trading strategy combining MACD momentum analysis with conservative entry filters and PineConnector webhook integration for automated trading.

## 🎯 Strategy Overview

This strategy implements a **conservative momentum-based approach** using MACD indicators with multiple confirmation filters to reduce false signals and improve trade quality.

### Key Features

- ✅ **Multi-Layer Entry Confirmation**: Combines MACD crossovers, histogram momentum, volume, and candle patterns
- ✅ **Trend Filtering**: EMA-based trend detection to trade with the prevailing direction
- ✅ **Volatility Filtering**: Optional ATR-based volatility threshold
- ✅ **Session Filtering**: Trade only during specific market hours
- ✅ **Flexible Exit Logic**: 
  - Traditional stop-loss and take-profit (percentage-based)
  - Trailing stops for profit protection
  - EMA counter-trend exit system
- ✅ **Take-Profit Detection**: Tracks when trades exit at TP vs SL with cooldown periods
- ✅ **PineConnector Integration**: Ready-to-use webhook payloads for automated execution
- ✅ **Visual Debugging**: On-chart condition markers and info dashboard

---

## 📋 Entry Logic (Conservative Variant 5)

A trade signal is generated when **ALL** of the following conditions are met:

### Long Entry Requirements:
1. **MACD Crossover Confirmation**: MACD line crossed above signal line on previous bar
2. **Histogram Momentum**: Histogram is positive AND rising
3. **Volume Confirmation**: Current volume > 20-bar SMA
4. **Bullish Candle Pattern**: 
   - Close > Open (green candle)
   - Close > EMA trend line
   - Close > Previous high
5. **Trend Alignment**: Price above trend EMA
6. **Strength Filter** (optional): MACD line > threshold (default 1.5)
7. **Market Conditions**: Passes volatility and session filters (if enabled)
8. **Post-TP Cooldown**: Minimum bars elapsed since last take-profit exit (default 3)

### Short Entry Requirements:
Same logic in reverse (crossunder, negative histogram falling, bearish candle, etc.)

---

## ⚙️ Configuration Parameters

### MACD & Trend
| Parameter | Default | Description |
|-----------|---------|-------------|
| `fastLength` | 12 | MACD fast EMA period |
| `slowLength` | 26 | MACD slow EMA period |
| `signalLength` | 6 | MACD signal line period |
| `emaLength` | 50 | Trend filter EMA period |
| `strengthLevel` | 1.5 | Minimum MACD strength threshold |
| `useFilter` | true | Enable/disable strength filter |

### Volatility Filter
| Parameter | Default | Description |
|-----------|---------|-------------|
| `useVolatility` | false | Enable ATR volatility filter |
| `atrLength` | 14 | ATR calculation period |
| `atrMult` | 0.5 | ATR threshold multiplier |

### Risk Management
| Parameter | Default | Description |
|-----------|---------|-------------|
| `stopLossValue` | 1.0% | Stop-loss distance from entry |
| `takeProfitValue` | 2.0% | Take-profit distance from entry |
| `useTrailingStop` | true | Enable trailing stop |
| `trailPercent` | 0.8% | Trailing stop distance |
| `bufferTicks` | 2 | Price buffer to avoid premature stops |

### PineConnector Settings
| Parameter | Default | Description |
|-----------|---------|-------------|
| `orderSize` | 0.1 | Position size in lots/contracts |
| `connectorFormat` | PineConnector CSV | Webhook payload format |
| `pine_license_id` | (empty) | Your PineConnector license ID |

---

## 🔗 PineConnector Setup

### Step 1: Configure Strategy
1. Add your PineConnector **License ID** in the strategy settings
2. Enable "Send alert() from script"
3. Choose webhook format (CSV or JSON)

### Step 2: Create TradingView Alert
1. Right-click chart → "Add Alert"
2. **Condition**: Select "Any alert() function call"
3. **Webhook URL**: Enter your PineConnector webhook endpoint
4. **Message**: Leave as `{{strategy.order.alert_message}}`
5. **Trigger**: "Once Per Bar Close"

### Step 3: Webhook Payload Examples

**PineConnector CSV Format (Long):**
```
YOUR_LICENSE_ID,BUY,BTCUSD,0.1,49500.00,51000.00,auto
```

**JSON Format (Long):**
```json
{
  "action":"BUY",
  "symbol":"BTCUSD",
  "size":0.1,
  "entry_price":50000.00,
  "sl_pct":1.0,
  "tp_pct":2.0,
  "sl_price":49500.00,
  "tp_price":51000.00
}
```

---

## 📊 Visual Indicators

### Chart Markers (when "Show condition checks" enabled):
- 🟡 **Yellow Circle**: MACD crossover detected
- 🟢 **Green Square**: Histogram rising
- 🔵 **Blue Cross**: Volume confirmation
- 🔷 **Aqua Triangle Up**: Bullish candle confirmation
- 🟢 **Large Green Triangle**: Final BUY signal
- 🔴 **Large Red Triangle**: Final SELL signal
- 🟣 **Fuchsia X**: Take-profit exit detected

### Info Dashboard (Top-Right):
- Current trend direction
- Buy condition status
- Individual filter states (Histogram/Volume/Candle)
- Last TP exit price and cooldown countdown
- Current position and P/L

---

## 🧪 Backtesting Recommendations

### Initial Settings:
- **Capital**: $10,000
- **Commission**: 0.02%
- **Timeframe**: 1H or 4H (conservative approach works best on higher TFs)
- **Assets**: Works well on trending markets (forex, crypto, indices)

### Optimization Tips:
1. **Start Conservative**: Use default settings first
2. **Adjust EMA Length**: Lower (20-30) for faster markets, higher (100+) for slower
3. **Tune Stop/TP Ratio**: 1:2 risk-reward is balanced; adjust based on asset volatility
4. **Session Filter**: Enable for forex during high-liquidity sessions (London/NY)
5. **Volatility Filter**: Enable in ranging markets to reduce false signals

### Performance Metrics to Monitor:
- **Win Rate**: Target >45% (conservative strategy may have lower win rate but better R:R)
- **Profit Factor**: Target >1.5
- **Max Drawdown**: Should stay under 20% with proper position sizing
- **Average Trade Duration**: Varies by timeframe (1H: hours, 4H: days)

---

## ⚠️ Important Notes

### Known Limitations:
1. **Lagging Indicators**: MACD is trend-following; expect some lag on entries
2. **Choppy Markets**: Strategy may underperform in low-volatility sideways markets
3. **Slippage**: Backtests don't account for real-world slippage; add safety margin
4. **Webhook Delays**: Network latency can affect live trade execution

### Risk Warnings:
- **This is NOT financial advice** — use at your own risk
- **Always test thoroughly** in demo accounts before live trading
- **Use proper position sizing** — never risk more than 1-2% per trade
- **Market conditions change** — past performance doesn't guarantee future results

---

## 🛠️ Installation

### TradingView:
1. Open TradingView Pine Editor
2. Copy the entire script
3. Paste into a new indicator
4. Click "Add to Chart"
5. Configure parameters in strategy settings

### Version Requirements:
- Pine Script: v5
- TradingView: Pro+ plan (for webhooks)

---

## 📝 Changelog

### Version 5 (Current)
- ✅ Fixed TP detection logic with tolerance matching
- ✅ Added post-TP cooldown period to prevent premature re-entries
- ✅ Improved EMA counter-trend exit system
- ✅ Enhanced visual debugging with comprehensive info table
- ✅ Fixed `prevPosSize` handling for accurate exit detection

---

## 🤝 Contributing

Found a bug or have a feature suggestion? Please:
1. Test thoroughly to reproduce the issue
2. Provide detailed description with screenshots
3. Include your TradingView settings and timeframe

---

## 📄 License

This strategy is provided as-is for educational and research purposes. Use in live trading at your own risk.

---

## 📞 Support

For PineConnector setup issues, visit: [PineConnector Documentation](https://pineconnector.com/docs)

---

**Happy Trading! 📈**

*Remember: The best strategy is one you understand completely and have tested extensively.*
