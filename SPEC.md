# Quanty — v0.1 Specification

> A systematic multi-asset, multi-timeframe ranking & execution platform for Hyperliquid. Instead of individual strategies, Quanty runs a pipeline: rank all tracked assets → identify strongest/weakest → confirm with price action → execute. Real-time WebSocket data, full backtesting, web dashboard, and Hermes AI integration.

## 1. Architecture Overview

```
                            ┌─────────────────┐
                            │   Hyperliquid    │
                            │   WebSocket API   │
                            │   + REST API     │
                            └────────┬─────────┘
                                     │
                            ┌────────▼─────────┐
                            │  Data Feed Mgr   │
                            │  (WebSocket conn, │
                            │   auto-reconnect, │
                            │   multi-symbol    │
                            │   + multi-tf      │
                            │   subscriptions)  │
                            └────────┬─────────┘
                                     │
                            ┌────────▼─────────┐
                            │    Candle Engine  │
                            │  (builds candles  │
                            │   from ticks,     │
                            │   manages 1m/5m/  │
                            │   15m/1h/4h/1d    │
                            │   per symbol)     │
                            └────────┬─────────┘
                                     │
                            ┌────────▼──────────────────────────┐
                            │         Pipeline Engine            │
                            │                                    │
                            │  Layer 1 — RS Ranking Matrix       │
                            │    Rank all tracked assets by      │
                            │    relative strength across        │
                            │    multiple windows (1h/4h/24h)    │
                            │    → output: ordered asset list    │
                            │                                    │
                            │  Layer 2 — Trend Alignment         │
                            │    Per asset: multi-MA alignment   │
                            │    (5/10/20/50/100 on multiple     │
                            │    timeframes). Score 0-100.       │
                            │    → output: trend score per asset  │
                            │                                    │
                            │  Layer 3 — Confirmation Gate       │
                            │    On ranked/trend-aligned targets:│
                            │    price action patterns, volume   │
                            │    confirmation, MA cross check    │
                            │    → output: validated signal set  │
                            │                                    │
                            │  Layer 4 — Execution Rules         │
                            │    Position sizing, risk mgmt,     │
                            │    max exposure per asset/corr     │
                            │    → output: order list            │
                            │                                    │
                            └────────────────┬──────────────────┘
                                     │
                            ┌────────▼──────────────────────────┐
                            │         Paper Trading Engine      │
                            │  (sim fills, PnL tracking,        │
                            │   position management)            │
                            └────────┬──────────────────────────┘
                                     │
                            ┌────────▼──────────────────────────┐
                            │         FastAPI Backend            │
                            │  (REST + WebSocket for live UI)    │
                            └────────┬──────────────────────────┘
                                     │
                            ┌────────▼──────────────────────────┐
                            │         React Web UI              │
                            │  Dashboard | Rankings | Pipeline   │
                            │  Portfolio | Backtest | Settings   │
                            │  Chat Panel (Hermes gateway)       │
                            └────────────────────────────────────┘
```

### 1.1 Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.11+, FastAPI, Uvicorn |
| Data feed | Hyperliquid Python SDK (WebSocket) |
| Frontend | React + Vite + TypeScript |
| Charts | Lightweight Charts (trading charts) or Recharts |
| Realtime UI | Server-Sent Events (SSE) or WebSocket from FastAPI |
| Terminal | Rich (Python CLI, optional) |
| Hermes Gateway | Built-in HTTP API for chat panel |

### 1.2 Repo Structure

```
Quanty/
├── backend/
│   ├── main.py              # FastAPI entry point
│   ├── feed/                # WebSocket connection manager
│   │   ├── manager.py
│   │   ├── models.py        # Tick, Candle, OrderBook dataclasses
│   │   └── candle_engine.py # Builds multi-tf candles from ticks
│   ├── pipeline/            # Core trading pipeline
│   │   ├── __init__.py
│   │   ├── runner.py        # Orchestrates all layers on each tick/candle
│   │   ├── layer1_ranking.py   # RS ranking matrix
│   │   ├── layer2_trend.py     # Multi-MA trend alignment scores
│   │   ├── layer3_confirmation.py  # Price action + volume gate
│   │   └── layer4_execution.py    # Position sizing + risk mgmt
│   ├── trader/              # Paper trading engine
│   │   ├── paper.py
│   │   └── models.py        # Position, Order, PnL
│   ├── backtest/            # Backtester
│   │   ├── engine.py
│   │   └── metrics.py       # Sharpe, drawdown, win rate
│   ├── api/                 # FastAPI routes
│   │   ├── feed.py
│   │   ├── pipeline.py
│   │   ├── portfolio.py
│   │   └── backtest.py
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── App.tsx
│   │   ├── pages/
│   │   │   ├── Dashboard.tsx   # Live feed + connection status
│   │   │   ├── Rankings.tsx    # RS matrix + trend scores table
│   │   │   ├── Pipeline.tsx    # 4-layer pipeline visualization
│   │   │   ├── Portfolio.tsx
│   │   │   ├── Backtest.tsx
│   │   │   └── Settings.tsx
│   │   ├── components/
│   │   │   ├── ChatPanel.tsx
│   │   │   ├── TradingChart.tsx
│   │   │   ├── RankMatrix.tsx  # RS ranking heatmap
│   │   │   ├── TrendRadar.tsx  # Trend scores visualization
│   │   │   └── PipelineFlow.tsx # Layer status indicators
│   │   └── hooks/
│   └── package.json
├── config/
│   ├── tracked_assets.yaml    # List of symbols to track
│   ├── pipeline_params.yaml   # All pipeline layer parameters
│   └── risk.yaml              # Risk guardrails
├── SPEC.md                    # This file
├── CLAUDE.md                  # Context for AI agents
└── README.md
```

## 2. Data Layer

### 2.1 Hyperliquid WebSocket Feed

The SDK's WebSocket client connects to Hyperliquid's L2 WebSocket endpoint. Quanty subscribes to:

- **Ticker updates** (`allMids` channel) — real-time mid prices for tracked pairs
- **Trades** (`trades` channel) — recent fill data
- **Order book snapshots + updates** (`l2Book` channel) — full depth

The connection manager handles:

- **Automatic reconnection** with exponential backoff (1s, 2s, 4s, 8s, 16s, cap at 60s)
- **Heartbeat / ping** to keep connection alive
- **Subscription management** — add/remove pairs at runtime
- **Normalized output** — all data converted to Quanty's internal model format before reaching the rest of the system

### 2.2 Internal Data Models

```python
@dataclass
class Tick:
    symbol: str
    mid_price: float
    timestamp: datetime
    bid: float
    ask: float
    volume_24h: float

@dataclass
class Candle:
    symbol: str
    timestamp: datetime
    open: float
    high: float
    low: float
    close: float
    volume: float
    interval: str  # "1m", "5m", "15m", "1h", etc.

@dataclass
class OrderBookSnapshot:
    symbol: str
    timestamp: datetime
    bids: list[tuple[float, float]]  # (price, size)
    asks: list[tuple[float, float]]
```

### 2.3 Historical Data

For backtesting, pull historical candles from Hyperliquid's REST API (`/info` endpoint). No API key required for public data. Only recent candles are available (typically 7-30 days depending on interval). For v0.1, this is sufficient. Deep history can be added later as a CSV import or paid data source.

## 3. Pipeline System (Core Engine)

This replaces the "strategy" concept entirely. Quanty runs a 4-layer pipeline on every tick/candle against all tracked assets simultaneously. There are no individual strategies — only pipeline stage configuration.

### 3.1 Candle Engine

Before the pipeline runs, the **Candle Engine** maintains a candle buffer for each symbol across multiple timeframes:

- **1m candles** — built from the raw tick stream (open on the minute, resolve each tick)
- **5m, 15m, 1h, 4h, 1d** — aggregated from the 1m candles
- Pushed to the pipeline on close of each candle interval

This gives every layer access to any timeframe for any asset at any time without redundant calculations.

### 3.2 Layer 1 — RS Ranking Matrix

Runs on a configurable timer (every 15 min / 1h / 4h by default). Processes the **top 20 most liquid Hyperliquid assets** — the full set is defined in `config/tracked_assets.yaml`.

```python
class RankingLayer:
    def compute(self, state: PipelineState) -> RankingOutput:
        for symbol in state.tracked_symbols:
            rank_score = 0
            for window in [1, 4, 24]:  # hours
                change = price_change(symbol, window)
                # Normalize change across all assets → z-score
                rank_score += z_score(change, all_changes)
            # Weight: 1h (immediate) < 4h (medium) < 24h (trend)
        return sorted(symbols, key=rank_score, reverse=True)
```

**Output:** Ordered list of all tracked assets ranked by relative strength. Displayed in the web UI as a **heatmap matrix** (symbols × time windows, color-coded green→red).

**RIS (Relative Intensity Score):** A single composite score (0-100) per asset derived from the z-score sum. Visually: RIS cards in the Rankings page, sorted strongest→weakest, with color intensity showing conviction.

### 3.3 Layer 2 — Trend Alignment

Runs on every candle close for every asset. For each symbol, compute multiple moving averages across multiple timeframes:

```python
class TrendLayer:
    def compute(self, state: PipelineState, rank_ordered: list[str]) -> dict[str, TrendScore]:
        scores = {}
        for symbol in rank_ordered[:10]:  # Only top/bottom 10
            ma_data = {}
            for tf in ["5m", "15m", "1h"]:
                for period in [5, 10, 20, 50, 100]:
                    ma_data[f"{tf}_MA{period}"] = sma(state.candles[symbol][tf], period)
            
            # Count aligned MAs (price above MA = bullish, below = bearish)
            # Weight by timeframe (higher TF = more weight)
            # Score 0-100: 0 = all bearish, 100 = all bullish
            scores[symbol] = TrendScore(
                value=alignment_score(ma_data),
                mas=ma_data,
                dominant_tf=detect_dominant_timeframe(ma_data)
            )
        return scores
```

**Output:** Per-asset trend score (0-100) + which timeframes are aligned vs conflicting. Displayed as a **Trend Radar** — horizontal bars per asset showing score and MA breakdown.

### 3.4 Layer 3 — Confirmation Gate

Only triggers on assets that pass both Layer 1 (top/bottom quartile) AND Layer 2 (score above/below threshold). Checks for:

**Price action patterns:**
- Recent swing high/low breakouts
- Candle body vs wick ratios (momentum confirmation)
- Consecutive bullish/bearish closes

**Volume confirmation:**
- Above-average volume on the signal candle
- Volume increasing in the signal direction

**MA cross verification:**
- Fast MA crossed slow MA recently (if applicable to the asset's timeframe)
- MA slope direction (accelerating or decelerating)

```python
class ConfirmationLayer:
    def compute(self, signals: dict[str, Signal]) -> dict[str, Signal]:
        validated = {}
        for symbol, signal in signals.items():
            checks = [
                recent_breakout(symbol, signal.direction),
                volume_above_avg(symbol, threshold=1.5),
                ma_slope_in_direction(symbol, signal.direction),
            ]
            if sum(checks) >= 2:  # At least 2/3 confirmations
                validated[symbol] = signal.with_confidence(sum(checks) / 3)
        return validated
```

### 3.5 Layer 4 — Execution Rules

Determines what actually happens. For each validated signal:

- **Position sizing** — Kelly fraction or fixed % of portfolio, capped per asset
- **Max exposure** — no more than N% in correlated assets (same sector)
- **Opposite side check** — if already long BTC, don't take a new long on ETH if they're highly correlated
- **Drawdown circuit breaker** — if portfolio is down X% in a day, stop all new entries

```python
class ExecutionLayer:
    def compute(self, validated: dict[str, Signal], portfolio: Portfolio) -> list[Order]:
        orders = []
        for symbol, signal in validated:
            if not self.daily_loss_limit_breached(portfolio):
                size = self.kelly_position_size(signal, portfolio)
                orders.append(Order(symbol=symbol, side=signal.direction, size=size))
        return orders  # → Paper Trading Engine
```

### 3.6 Pipeline Orchestration

The `PipelineRunner` ties it all together:

```python
class PipelineRunner:
    def __init__(self):
        self.layer1 = RankingLayer()
        self.layer2 = TrendLayer()
        self.layer3 = ConfirmationLayer()
        self.layer4 = ExecutionLayer()
    
    async def on_tick(self, ticks: dict[str, Tick]):
        self.candle_engine.update(ticks)
    
    async def on_candle_close(self, candles: dict[str, dict[str, Candle]]):
        # Layer 1 — runs on schedule (e.g., every 1h)
        if self.ranking_due():
            rankings = self.layer1.compute(self.state)
        
        # Layer 2 — every candle close on top/bottom ranked
        trends = self.layer2.compute(self.state, rankings.top_bottom(10))
        
        # Layer 3 — confirm only strong-ranked + strong-trend
        signals = self.layer3.compute(trends)
        
        # Layer 4 — size and execute
        orders = self.layer4.compute(signals, self.portfolio)
        await self.paper_trader.execute(orders)
```

### 3.7 Hermes Integration

Three specific roles:

1. **Pipeline tuning** — "Optimize the Layer 1 ranking to weight 24h momentum higher than 1h" → Hermes edits `pipeline_params.yaml`
2. **Layer 3 rule design** — "Add a confirmation check: only trade if the last 3 candles are all in the signal direction" → Hermes generates the confirmation function
3. **Backtest analysis** — "Why did we get a drawdown on May 3rd?" → Hermes reads the backtest log and explains

The Chat Panel sends these requests to the Hermes gateway. Responses appear inline, and if code/config changes are needed, Hermes writes them directly to the filesystem.

### 3.8 Pipeline Configuration

All config lives in YAML files under `config/`:

```yaml
# pipeline_params.yaml
ranking:
  windows: [1, 4, 24]           # hours
  weights: [0.2, 0.3, 0.5]      # 1h < 4h < 24h
  recalc_interval: 3600          # seconds

trend:
  timeframes: ["5m", "15m", "1h"]
  mas: [5, 10, 20, 50, 100]
  top_n: 10                      # only score top/bottom N ranked

confirmation:
  min_checks_passed: 2           # out of 3
  volume_threshold: 1.5          # vs 24h average
  max_breakout_lookback: 20      # candles

execution:
  position_sizing: "kelly"       # or "fixed_pct"
  max_position_pct: 10           # % of portfolio per asset
  max_correlated_exposure: 25    # % across correlated assets
  daily_loss_limit_pct: 5        # stop new trades if down this much
  slippage: 0.0001               # 0.01%
```

## 4. Paper Trading Engine

### 4.1 Execution Model

- Receives `Signal(BUY/SELL/CLOSE, size, price)` from the strategy engine
- Simulates fill immediately at the next tick's price + configured slippage (default: 0.01%)
- Tracks per-strategy: **positions**, **order history**, **realized/unrealized PnL**, **open orders**

### 4.2 Portfolio State

```python
@dataclass
class PaperPortfolio:
    cash: float
    positions: dict[str, Position]          # symbol → Position
    order_history: list[FilledOrder]
    pnl_total: float
    pnl_unrealized: float
    pnl_realized: float
    start_balance: float                     # for % return calc
    drawdown_peak: float                     # for drawdown tracking
```

### 4.3 Risk Guardrails

Configurable per-strategy:
- **Max position size** (as % of portfolio)
- **Max open positions** (count)
- **Daily loss limit** (stop trading if exceeded)
- **Slippage model** (fixed %, or market impact estimate)

## 5. Backtester

### 5.1 How It Works

1. Fetches historical candles from Hyperliquid REST API or loads from CSV
2. Feeds candles into the strategy's `on_candle()` method in chronological order at simulated speed (or as fast as possible)
3. Routes resulting signals through the paper trading engine
4. Outputs performance metrics at the end

### 5.2 Metrics

| Metric | Description |
|---|---|
| Total Return % | (final_equity - start_balance) / start_balance |
| Sharpe Ratio | Annualized risk-adjusted return |
| Max Drawdown % | Largest peak-to-trough decline |
| Win Rate | Winning trades / total trades |
| Profit Factor | Gross profit / gross loss |
| Total Trades | Number of round-trip trades |
| Avg Hold Time | Average time per position |

### 5.3 Visualization

- **Equity curve** (portfolio value over time, plotted with drawdown shaded)
- **Trade markers** (buy/sell points on price chart)
- **Monthly returns heatmap**
- **Metrics summary card**

## 6. Web UI — Pages

### 6.1 Dashboard

- Real-time price ticker for tracked pairs (cards with bid/ask/last/change %)
- Connection status indicator (green/red dot)
- Pipeline status block — shows each layer's last run time + current output
- **Risk dashboard** — daily PnL, drawdown, positions count, circuit breaker status

### 6.2 Rankings (RS Matrix + Trend Radar)

- **RIS Matrix** — heatmap table (symbols × time windows), color-coded green to red. Sortable by strongest/weakest
- **RIS History** — line chart showing each asset's RIS score over time (track how relationships evolve)
- **Trend Radar** — per-asset horizontal bars showing trend score (0-100) and MA alignment breakdown per timeframe
- **Dominant Timeframe** — mini-badge per asset showing which timeframe has the strongest trend signal

### 6.3 Pipeline Viewer

- **Live layer feed** — each layer's last output shown as a card
- Layer 1: current ranking list with RIS values
- Layer 2: top trend scores with MA alignment details
- Layer 3: pending confirmations panel — shows what passed/failed each check
- Layer 4: current open orders, position sizing decisions
- **Pipeline log** — chronological event feed of all pipeline decisions

### 6.4 Portfolio

- PnL summary bar (total, realized, unrealized, today's change)
- Positions table (symbol, side, size, entry price, current price, PnL)
- Order history table (time, symbol, side, size, price, fill status)
- Equity curve over time
- Per-asset PnL breakdown bar chart
- Daily performance log

### 6.5 Backtest

- Asset universe selector
- Date range selector
- Pipeline config snapshot (which params to test)
- "Run Backtest" button
- Results: metrics grid + equity curve chart + trade markers on price chart
- **Layer analysis** — which layers contributed most to PnL? (ranking decisions vs confirmation gates vs execution rules)
- Compare mode (v0.2): overlay multiple backtests with different params

### 6.6 Settings

- Tracked symbols (select from top 20, add custom)
- All pipeline parameters (YAML editor or structured UI form)
- Risk guardrails (max position, daily loss limit, slippage)
- Initial paper balance
- Hyperliquid wallet address (for future live trading)
- Notification preferences (Discord alerts on large PnL moves)

### 6.7 Chat Panel (Global)

Floating chat panel (like Intercom/Crisp) available from any page. Connects to Hermes Agent gateway. Use cases:

- "Add a new confirmation rule: don't trade if volume is declining over the last 3 candles"
- "Show me why BTC was ranked #3 in the last ranking cycle"
- "Optimize the Layer 2 trend calculation to weight 1h MA alignment higher"
- "Run a backtest with these new settings and show me the Sharpe ratio"
- "What caused the drawdown spike on May 5th?"
- "What's my best performing asset right now?"

## 7. Phasing — v0.1 vs v0.2

| Feature | v0.1 | v0.2 |
|---|---|---|
| WebSocket data feed | Yes (Hyperliquid, top 20 assets) | Multi-exchange |
| Candle engine (multi-tf) | Yes (1m/5m/15m/1h/4h/1d) | + Custom intervals |
| RS Ranking Matrix (Layer 1) | Yes | + Custom ranking formulas |
| Trend Alignment (Layer 2) | Yes (SMA, configurable) | + EMA, VWAP, custom indicators |
| Confirmation Gate (Layer 3) | Yes (3 checks, configurable) | + ML-based confirmation |
| Execution Rules (Layer 4) | Yes (Kelly, risk limits) | + Dynamic sizing algorithms |
| Paper trading | Yes | + Advanced order types |
| Backtester | Yes (basic metrics) | + Walk-forward, Monte Carlo |
| Web UI | Yes (Dashboard, Rankings, Pipeline, Portfolio, Backtest, Settings, Chat) | + Mobile responsive |
| Hermes chat panel | Yes | + Voice mode |
| CLI dashboard | Yes (Rich) | Terminal UI (TUI) |
| Live trading | No | Yes (Hyperliquid) |
| Discord alerts | Yes (via Hermes gateway) | Custom notification rules |
| Multi-asset portfolio tracking | Yes | + Correlation analysis |

## 8. Development Phases

### Phase 1 — Foundation (Cards T1-T4)
- Research Hyperliquid Python SDK + WebSocket patterns
- Project scaffold: FastAPI backend, React frontend, venv, deps
- WebSocket feed manager — connect, subscribe to top 20 tickers, auto-reconnect
- Candle engine — build multi-tf candles (1m/5m/15m/1h/4h/1d) from tick stream
- Data models + unified feed interface

### Phase 2 — Pipeline Core Logic (Cards T5-T8)
- Layer 1 — RS Ranking Matrix (RIS scoring, multi-window, z-score normalization)
- Layer 2 — Trend Alignment (multi-MA, multi-tf, scoring)
- Layer 3 — Confirmation Gate (price action + volume + MA verification)
- Layer 4 — Execution Rules (position sizing, risk guardrails, circuit breaker)
- Pipeline orchestrator (PipelineRunner tying all layers together)

### Phase 3 — Trading + Backtesting (Cards T9-T10)
- Paper trading engine (sim fills, PnL tracking, position management)
- Backtester (historical replay, metrics: Sharpe, drawdown, win rate)

### Phase 4 — Interface (Cards T11-T14)
- CLI dashboard (Rich-based live PnL + pipeline status)
- Web UI scaffold (FastAPI + React, SSE for live updates)
- Dashboard + Rankings pages (RIS matrix, trend radar)
- Pipeline Viewer + Portfolio pages

### Phase 5 — Integration (Cards T15-T17)
- Backtest page with metrics + charts
- Settings page with pipeline config editor
- Chat Panel connected to Hermes gateway
- End-to-end integration test
- Documentation + CLAUDE.md

---

*This spec is a living document. As we build, decisions that diverge from this doc should be noted here with a version bump.*
