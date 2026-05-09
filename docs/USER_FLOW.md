# Quanty — User Flow

## First Launch — Setup

1. Open http://localhost:5173
2. Settings page loads (first-time setup detection)
3. Select tracked assets from the top 20 Hyperliquid list (default: all 20)
4. Set initial paper balance (default: $50,000)
5. Set risk parameters: max position size, daily loss limit, slippage
6. Click "Start Pipeline"
7. Backend connects to Hyperliquid WebSocket
8. Candle engine starts building 1m/5m/15m/1h/4h/1d candles
9. Dashboard appears with live price cards

## Daily Use

### Dashboard
- See 20 price cards with 24h change, color-coded
- Pipeline status block showing each layer's last run time and confidence
- Risk dashboard: daily PnL, drawdown, open positions count

### Rankings
- RIS Matrix: all tracked assets sorted strongest to weakest
- Per-asset trend scores with timeframe alignment details
- Track RIS history to see how asset relationships evolve

### Pipeline Viewer
- Layer 1: current ranking list with RIS values
- Layer 2: top trend scores with MA alignment details
- Layer 3: pending confirmations - what passed/failed each check
- Layer 4: open orders and position sizing decisions
- Log: chronological event feed of all pipeline decisions

### Portfolio
- PnL summary (total, realized, unrealized, today's change)
- Positions table with entry price, current price, PnL
- Order history
- Equity curve
- Per-asset PnL breakdown

### Chat Panel (floating, available from any page)
- "Why is SOL ranked neutral when it was top 3 yesterday?"
- "Add a confirmation rule: only trade if volume is above average"
- "Optimize Layer 1 to weight 24h double the 1h"
- Hermes edits config, hot-reloads, pipeline uses new params

## Backtesting Flow

1. Navigate to Backtest page
2. Select asset universe (default: all tracked)
3. Set date range
4. Click "Load current params" to snapshot pipeline_params.yaml
5. Click "RUN BACKTEST"
6. Results load: metrics grid + equity curve + trade markers
7. Layer Contribution Analysis shows which layers drove PnL
8. Tweak params via Chat or Settings, re-run, compare

## Refining Rules

Type in Chat Panel at any time:

- "Add a confirmation rule: only trade if the last 3 candles are all in the signal direction"
  → Hermes generates Python code, adds to Layer 3, appears in Pipeline Viewer immediately

- "What happens if I increase max position size to 15%?"
  → Hermes simulates: shows risk/exposure impact before committing

## Key Principle

There is no manual trading. No buy/sell buttons. The pipeline generates signals,
the paper trader executes automatically. Your job is to refine pipeline parameters
and confirmation rules. Manual override is v0.2.