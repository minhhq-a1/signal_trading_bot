# signal_trading_bot

**Self-Aware Trend System (SATS)** — Adaptive SuperTrend trading indicator + signal processing backend.

## Project Overview

```
signal_trading_bot/
├── source/
│   ├── self_aware_trend_system_indicator_script.pine    # TradingView Pine Script v1.9.0
│   ├── self_aware_trend_system_indicator_description.md  # Indicator documentation
│   └── trading_btc_signal_system.md                     # BTC signal system design
│
├── docs/superpowers/specs/
│   └── 2026-05-08-sats-signal-backend-design.md        # Backend technical specification
│
└── signal_backend/                                      # Python FastAPI backend (planned)
    ├── README.md
    ├── Dockerfile
    └── docker-compose.yml
```

## Core Components

### 1. SATS Indicator (Pine Script)
- **Location:** `source/self_aware_trend_system_indicator_script.pine`
- **Version:** 1.9.0
- **Purpose:** Adaptive SuperTrend with 4-factor Trend Quality Index (TQI), asymmetric bands, character-flip detection, dynamic TP engine, and on-chart performance tracking
- **Docs:** `source/self_aware_trend_system_indicator_description.md`

### 2. SATS Signal Backend (planned)
- **Location:** `signal_backend/`
- **Stack:** Python 3.11+ / FastAPI / SQLite / Telegram Bot
- **Purpose:** Stateful multi-timeframe decision engine — receives TradingView webhooks, verifies signals, classifies verdict (A+/A/WATCH/BLOCKED), sends formatted alerts to Telegram
- **Spec:** `docs/superpowers/specs/2026-05-08-sats-signal-backend-design.md`

### 3. BTC Signal System Design
- **Location:** `source/trading_btc_signal_system.md`
- **Purpose:** Architecture vision — 3-layer system (Indicator → Backend → Telegram), HTF verification, no-chase engine, replay engine

## Quick Start

### SATS Indicator (TradingView)
1. Copy `source/self_aware_trend_system_indicator_script.pine` into TradingView Pine Editor
2. Add to chart — preset "Auto" auto-detects timeframe
3. Configure webhook alerts pointing to backend endpoint

### SATS Backend (MVP — Local)

```bash
cd signal_backend

# 1. Create virtual environment
python3 -m venv .venv
source .venv/bin/activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Configure
cp .env.example .env
# Edit .env — set WEBHOOK_SECRET, TELEGRAM_BOT_TOKEN, TELEGRAM_CHAT_ID

# 4. Run
uvicorn main:app --reload --port 8000

# 5. Expose for TradingView (ngrok)
ngrok http 8000
# Copy ngrok HTTPS URL → TradingView alert webhook config

# Hoặc dùng cloudflare tunnel
cloudflared tunnel --url http://localhost:8000
```

### SATS Backend (Production — Docker)

```bash
cd signal_backend

# Build + run
docker-compose up -d

# Logs
docker-compose logs -f signal_backend
```

## Architecture

```
TradingView SATS Indicator
        │
        │ JSON webhook (bar_update / signal)
        ▼
   SATS Backend (FastAPI)
        │
        ├── Payload Validator + Normalizer
        ├── State Store (SQLite)
        ├── Entry Quality Engine (Binance WebSocket live price)
        ├── Risk Engine
        ├── Verdict Engine (A+/A/WATCH/BLOCKED)
        └── Telegram Formatter + Sender
        │
        ▼
   Telegram Chat
```

## Key Specs

| Topic | Link |
|---|---|
| Backend technical spec | `docs/superpowers/specs/2026-05-08-sats-signal-backend-design.md` |
| Indicator docs | `source/self_aware_trend_system_indicator_description.md` |
| System architecture | `source/trading_btc_signal_system.md` |

## Backend Phases

| Phase | Content | Status |
|---|---|---|
| 0 | Pine MVP patch — add `event`, `secret`, `bar_time`, `symbol`, `timeframe` to alerts | Pending |
| 1 | Core infrastructure — scaffold, config, SQLite schema, Pydantic models | Pending |
| 2 | Web UI — payload input page, verdict result, manual test flow | Pending |
| 3 | Verdict pipeline — validator, normalizer, entry quality, risk engine, verdict engine, deduper, WebSocket live price | Pending |
| 4 | Telegram integration — formatter, sender, end-to-end flow | Pending |
| 5 | Market context — funding rate, session filter, weekend filter, position sizing | Pending |
| 6 | Dashboard — stats, verdict distribution, recent signals | Pending |

## Notes

- **No repainting:** All indicator signals use `barstate.isconfirmed`
- **Performance stats are walk-forward,** not backtested — honest statistics on your specific instrument
- **Backend verdict is decision-support only** — trade decisions remain yours
- **HTF verification** deferred until Pine emits `bar_update` events (Phase v1)
