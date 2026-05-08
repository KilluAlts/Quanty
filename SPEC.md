# Quanty — v0.1 Specification

> A paper trading & strategy development platform for Hyperliquid, with real-time WebSocket data, a strategy studio, backtester, conversational AI integration, and a web dashboard.

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
                            │   subscription    │
                            │   management)     │
                            └────────┬─────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
     ┌────────▼────────┐   ┌────────▼────────┐   ┌─────────▼─────────┐
     │  Strategy Engine │   │  Paper Trader   │   │  Backtester       │
     │  (live feed      │   │  (sim fills,    │   │  (historical      │
     │   → signal)      │   │   PnL, pos,     │   │   replay →        │
     │                  │   │   orders)       │   │   metrics)        │
     └────────┬────────┘   └────────┬────────┘   └─────────┬─────────┘
              │                      │                      │
              └──────────────────────┼──────────────────────┘
                                     │
                            ┌────────▼─────────┐
                            │  FastAPI Backend  │
                            │  (REST endpoints  │
                            │   + WebSocket for │
                            │   live UI updates)│
                            └────────┬─────────┘
                                     │
                            ┌────────▼─────────┐
                            │  React Web UI    │
                            │  (Dashboard,      │
                            │   Strategy Studio,│
                            │   Portfolio,      │
                            │   Backtest,       │
                            │   Settings,       │
                            │   Chat Panel)     │
                            └──────────────────┘

Hermes Agent Gateway ←→ Chat Panel (embedded in Web UI)
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
│   │   └── models.py        # Tick, Candle, OrderBook dataclasses
│   ├── strategy/            # Strategy engine
│   │   ├── base.py          # Abstract Strategy class
│   │   └── studio.py        # Strategy loader, hot-reload
│   ├── trader/              # Paper trading engine
│   │   ├── paper.py
│   │   └── models.py        # Position, Order, PnL
│   ├── backtest/            # Backtester
│   │   ├── engine.py
│   │   └── metrics.py       # Sharpe, drawdown, win rate
│   ├── api/                 # FastAPI routes
│   │   ├── feed.py
│   │   ├── strategies.py
│   │   ├── portfolio.py
│   │   └── backtest.py
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── App.tsx
│   │   ├── pages/
│   │   │   ├── LiveFeed.tsx
│   │   │   ├── StrategyStudio.tsx
│   │   │   ├── Portfolio.tsx
│   │   │   ├── Backtest.tsx
│   │   │   └── Settings.tsx
│   │   ├── components/
│   │   │   ├── ChatPanel.tsx
│   │   │   ├── TradingChart.tsx
│   │   │   └── StrategyEditor.tsx
│   │   └── hooks/
│   └── package.json
├── strategies/              # User-created strategy files
│   └── example_sma.py
├── SPEC.md                  # This file
├── CLAUDE.md                # Context for AI agents
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

## 3. Strategy Studio

This is the core value of Quanty. Two complementary interfaces:

### 3.1 Code-Based Strategies

Strategies are Python files dropped into `strategies/`. Each extends a base class:

```python
class SmaCrossover(Strategy):
    name = "SMA Crossover"
    params = {"fast_period": 10, "slow_period": 30}
    
    def on_tick(self, tick: Tick) -> Signal | None:
        # Called on every tick from live feed
        # Calculate, decide, return Signal(BUY/SELL/CLOSE) or None
        ...
    
    def on_candle(self, candle: Candle) -> Signal | None:
        # Called on each new candle close
        ...
```

### 3.2 The Studio UI

The Strategy Studio page in the web UI has:

1. **Strategy List** — all strategies in your `strategies/` folder, with enable/disable toggles
2. **Code Editor** — Monaco editor (VS Code engine) to view and edit strategy Python code
3. **Parameter Panel** — live sliders for tuning strategy params with instant feedback
4. **Live Signal Log** — each signal the strategy generates appears in real time
5. **Performance Mini-View** — running PnL for this strategy, updated live

### 3.3 Hermes Integration

The Chat Panel embedded in the web UI connects to the Hermes Agent gateway. You can type:

> "Build a mean reversion strategy that buys when the price drops 2% below the 20-period EMA and sells when it touches the EMA again. Add a 1% stop loss."

Hermes receives this through the gateway, generates the strategy Python code, stores it in `strategies/`, and the Studio UI picks it up instantly. No restart required.

### 3.4 Strategy Lifecycle

```
Create (UI or Hermes) → Edit (code or params) → Paper Trade (live) → Backtest (historical) → Refine → Deploy (v0.2)
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

### 6.1 Live Feed

- Real-time price ticker for tracked pairs (cards with bid/ask/last/change %)
- Order book depth chart (L2 visualization)
- Recent trades feed (scrolling list)
- Connection status indicator (green/red dot)

### 6.2 Strategy Studio

- Strategy list sidebar (with enable/disable toggles)
- Selected strategy: code editor + parameter sliders + live signal log
- "New Strategy" button → opens a prompt to describe in natural language → sends to Hermes chat → code appears

### 6.3 Portfolio

- PnL summary bar (total, realized, unrealized, today's change)
- Positions table (symbol, side, size, entry price, current price, PnL)
- Order history table (time, symbol, side, size, price, fill status)
- Equity curve (mini chart)
- Per-strategy breakdown (each strategy's contribution)

### 6.4 Backtest

- Strategy selector
- Date range selector
- "Run Backtest" button
- Results: metrics grid + equity curve chart + trade markers on price chart
- Compare mode (v0.2): overlay multiple backtests

### 6.5 Settings

- Tracked symbols (add/remove pairs)
- Default slippage
- Initial paper balance
- Hyperliquid wallet address (for future live trading)
- Notification preferences

### 6.6 Chat Panel (Global)

Floating chat panel (like Intercom/Crisp) available from any page. Connects to Hermes Agent gateway. Use cases:

- "Generate a new momentum strategy"
- "Analyze this backtest result"
- "Explain what caused the drawdown on May 5th"
- "Optimize this strategy's parameters"
- "What's my current PnL?"

## 7. Phasing — v0.1 vs v0.2

| Feature | v0.1 | v0.2 |
|---|---|---|
| WebSocket data feed | Yes (Hyperliquid only) | Multi-exchange |
| Paper trading | Yes | Enhanced (more order types) |
| Strategy studio (code) | Yes | + Visual drag-drop builder |
| Strategy studio (Hermes gen) | Yes | + Strategy market / sharing |
| Backtester | Yes (basic metrics) | + Walk-forward, Monte Carlo |
| CLI dashboard | Yes (Rich) | Terminal UI (TUI) |
| Web UI | Yes | + Mobile responsive |
| Live trading | No | Yes (Hyperliquid) |
| Multi-strategy portfolios | Yes | + Correlation analysis |
| Chat panel | Yes | + Voice mode |

## 8. Development Phases

### Phase 1 — Foundation (Cards T1-T4)
- Research Hyperliquid SDK
- Project scaffold + deps
- WebSocket feed manager
- Data models + unified feed interface

### Phase 2 — Core Logic (Cards T5-T7) 
- Strategy interface + example strategies
- Paper trading engine
- Backtester

### Phase 3 — Interface (Cards T8-T10)
- CLI dashboard
- Web UI scaffolding
- Chat panel connecting to Hermes gateway

### Phase 4 — Integration (Cards T11-T12)
- Strategy Studio UI (Monaco editor, params, live signal log)
- Hermes strategy generation pipeline
- End-to-end testing
- Documentation + CLAUDE.md

---

*This spec is a living document. As we build, decisions that diverge from this doc should be noted here with a version bump.*
