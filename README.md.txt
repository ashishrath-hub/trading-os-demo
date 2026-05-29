# trading_os — AI Hedge Fund for Indian Markets

> A production-grade algorithmic trading system built for NSE (India), combining real-time market data, machine learning signal filtering, generative AI analysis, and a live monitoring dashboard — running fully autonomously on a daily schedule.

---

## About This Repository

This is a **public portfolio demo** of `trading_os`, a full-stack algorithmic trading system I built independently over several months. The private repository contains live API integrations (Upstox, Telegram, TimescaleDB). This repo documents the architecture, methodology, and results for portfolio purposes.

---

## System Overview

```
                    ┌─────────────────────────────────┐
                    │         Daily Schedule           │
                    │  08:55 Pre-market AI scoring     │
                    │  09:15 Market open (WebSocket)   │
                    │  Every 5 min: Signal scan        │
                    │  12:30 Lunch review (Ollama AI)  │
                    │  15:15 Square off all positions  │
                    │  15:32 Post-market ML pipeline   │
                    └─────────────────────────────────┘
                                   │
          ┌────────────────────────┼──────────────────────────┐
          ▼                        ▼                           ▼
  ┌───────────────┐      ┌──────────────────┐      ┌──────────────────┐
  │  Data Layer   │      │  Signal Engine   │      │   ML Pipeline    │
  │               │      │                  │      │                  │
  │ Upstox WS     │      │ 2-Stage Scanner  │      │ Ensemble Model   │
  │ REST APIs     │      │ 221 NSE Stocks   │      │ RF + GBM + XGB   │
  │ RSS Feeds     │      │ 25+ Indicators   │      │ 15 Features      │
  │ TimescaleDB   │      │ Sentiment Scores │      │ Walk-forward CV  │
  └───────────────┘      └──────────────────┘      └──────────────────┘
          │                        │                           │
          └────────────────────────┼──────────────────────────┘
                                   ▼
                    ┌─────────────────────────────────┐
                    │       Execution & Monitoring     │
                    │                                  │
                    │  Paper Engine (slippage + STT)   │
                    │  1-second Position Monitor       │
                    │  Grafana: 34 panels, 10 rows     │
                    │  FastAPI: 36 endpoints           │
                    │  Telegram Bot: live alerts       │
                    └─────────────────────────────────┘
```

---

## Key Features

### 1. Two-Stage Market Scanner
- **Stage 1 (wide):** Batch-fetches quotes for all 221 candidate stocks in a single API call. Filters by price > ₹50, % change > 0.3%, volume > 0.
- **Stage 2 (deep):** Runs 25+ technical indicators on the ~50–80 stocks that pass Stage 1.
- Previously only 12–24 stocks were analyzed per cycle. The v4 pipeline covers the full NSE universe.

### 2. Five-Level Top-Down Hierarchy
```
L1 Macro      →  VIX regime, NIFTY A/D ratio, USD/INR, Gold, Crude
L2 Breadth    →  Large / Mid / Small cap performance comparison
L3 Sectors    →  All 16 NSE sectoral indices scored (price × 70% + sentiment × 30%)
L4 Strategy   →  Momentum, Quality, Low-Volatility index signals
L5 Stocks     →  Top LONG + SHORT candidates ranked by composite score
```

### 3. Ensemble ML Signal Filter
| Item | Detail |
|---|---|
| Model | RandomForest + GradientBoosting + XGBoost (VotingClassifier) |
| Validation | Walk-forward (train on past, test on future — no data leakage) |
| Features | 15 engineered features (RSI, VWAP%, ATR, sentiment, regime, MACD, BB%, vol ratio, EMA spread, gap%, momentum, ATR%, RSI divergence, time of day, day of week) |
| Safety gate | Model deploys **only** if trained on ≥ 100 real trades and new Sharpe > current Sharpe |
| Current status | 88% accuracy on 41 paper trades (59 trades away from production deployment) |

**Applied three ways:**
- Entry filter: score < 0.50 → signal suppressed
- Size adjustment: 0.40–0.50 → 70% position size; > 0.50 → full size
- Position re-scoring: every 30 seconds on open positions → exits if score < 0.25

### 4. Generative AI Integration (Ollama)
Six integration points per trading day:
1. **08:55** — Batch scores all RSS headlines using `qwen2.5:0.5b` (cached in Redis)
2. **09:02** — Classifies top 20 stocks as `momentum_continuation / mean_reversion / breakout / avoid`
3. **During scan** — Reads cached setup to adjust scoring (zero CPU during market hours)
4. **12:30** — Lunch review: HOLD / TIGHTEN / EXIT on open positions
5. **15:32** — Post-market AI analysis of each trade's outcome
6. **18:00** — Enriches losing trade labels for ML training (`chased_late_momentum` vs generic `false_breakout`)

### 5. Real-Time Infrastructure
- **1-second position monitor** with 3 isolated layers (price/SL/target → smart exit → ML rescore)
- **Background DB writer** (non-blocking `enqueue_sql()` — writes batch in microseconds)
- **Graceful shutdown** via `atexit`: saves capital state, flushes DB, stops all components
- **API circuit breaker**: 5 consecutive failures → 2-min pause + Telegram CRITICAL alert
- **Live reconciliation** every 5 min: detects ghost/orphan positions, alerts via Telegram

---

## Technical Indicators (25+)

| Category | Indicators |
|---|---|
| Trend | EMA 21/50, MACD, ADX, Supertrend |
| Momentum | RSI (Wilder), Momentum 5-bar, Gap % |
| Volatility | ATR, Bollinger Bands, Keltner Channel, India VIX |
| Volume | RVOL (vs 20-day avg), Volume spike detection |
| Price levels | VWAP (daily-reset), Support/Resistance |
| Market regime | Bull / Sideways / Bear / Volatile (from VIX + NIFTY breadth) |

---

## Paper Trading Results

| Day | P&L | Notes |
|---|---|---|
| Day 1 | −₹4,931 | System calibration, thresholds too aggressive |
| Day 2 | −₹2,659 | Improved after RSI gate widening (22–72) |
| Day 3 | +₹300 | First profitable day — ML suppression working |

Virtual capital: ₹4,92,090 (started ₹5,00,000). Execution costs modelled realistically: slippage 0.05%, brokerage ₹20/trade, STT 0.1%.

---

## Dashboard (Grafana)

34 panels across 10 rows:

| Row | Panels |
|---|---|
| Overview | Equity curve, Today P&L, Win rate, Profit factor, Capital |
| Paper Trading | Daily P&L bars, Strategy breakdown, Avg win/loss |
| Risk & Drawdown | Drawdown chart, Loss streak rolling window |
| Signals & ML | Signals per hour, ML score scatter, Model history |
| Market Context | Regime distribution, P&L by regime and hour, Exit reasons |
| System Health | Scan duration, WebSocket disconnects, DB queue depth |
| Performance | Cumulative P&L, Win rate by day, Best/worst stocks table |

---

## API Server (FastAPI — 36 endpoints)

```
GET /market          →  NIFTY, VIX, regime, sector rotation
GET /signals/live    →  Real-time signal bus (current session)
GET /paper/overview  →  Paper trade summary (all strategies)
GET /model           →  ML model history + feature importance
GET /system          →  Operational health: scan times, WS status, DB queue
GET /scanner         →  Trigger a fresh 5-level scan (~30 sec)
```

Full Swagger docs at `localhost:8000/docs`.

---

## Project Structure

```
trading_os/
├── config/
│   ├── settings.py          # All constants from .env
│   ├── instruments.py       # 150+ NSE stocks with Upstox keys
│   ├── trading_rules.py     # 19 hardcoded professional rules
│   └── holidays.py          # NSE 2026 calendar + F&O expiry
├── src/
│   ├── data/                # Market feeds, Upstox client, DB setup
│   ├── signals/             # Technical indicators, scanner, sentiment
│   ├── strategies/          # Intraday scalp/short, swing, BankNIFTY ORB
│   ├── execution/           # Paper engine, order manager, exit engine
│   ├── risk/                # Position sizer, portfolio limits, trailing stop
│   ├── learning/            # Trade journal, ML retrainer, mistake classifier
│   ├── infra/               # Scheduler, Telegram bot, Grafana metrics
│   └── api/                 # FastAPI dashboard server
├── docs/
│   ├── PHASE_1.md           # Foundation (complete)
│   ├── PHASE_2.md           # Upstox live data (complete)
│   ├── PHASE_3.md           # ML learning loop (complete)
│   └── PHASE_4.md           # Dashboard & full sync (complete)
├── notebooks/               # EDA and backtesting notebooks
├── deploy/
│   ├── start_trading_os.bat # Windows auto-start
│   └── watchdog.py          # Component health monitor
├── .env.example             # All required environment variables
└── CLAUDE.md                # Full system documentation
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Python 3.14 |
| Database | TimescaleDB (PostgreSQL + time-series extension) |
| Cache | Redis |
| Dashboard | Grafana |
| API server | FastAPI |
| ML | Scikit-learn, XGBoost |
| AI | Ollama (qwen2.5:0.5b, local inference) |
| Broker API | Upstox V3 (WebSocket + REST) |
| Alerting | Telegram Bot API |
| Containerisation | Docker + Docker Compose |
| IDE | Windsurf / VS Code |
| OS | Windows 11 (i5-12450H, 16 GB RAM — CPU-only inference) |

---

## Setup (Demo / Paper Mode)

```bash
# 1. Clone and install
git clone https://github.com/ashishrath-hub/trading-os-demo
cd trading-os-demo
pip install -r requirements.txt

# 2. Configure environment
cp .env.example .env
# Fill in your Upstox API keys (free account works for paper trading)

# 3. Start infrastructure
docker-compose up -d   # TimescaleDB + Redis + Grafana

# 4. Run
python main.py         # Interactive menu
python main.py test    # System health check (6/6 tests)
```

---

## Author

**Ashish Kumar Rath**
[LinkedIn](https://www.linkedin.com/in/ashish-kumar-rath-3842541a9) · [GitHub](https://github.com/ashishrath-hub) · ashishrath51@gmail.com