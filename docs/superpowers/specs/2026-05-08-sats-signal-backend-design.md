# SATS Signal Backend — Technical Specification

## 1. Overview

**Purpose:** Backend service nhận signal payload từ SATS Pine Script indicator, verify đa timeframe (v1), tính verdict (A+/A/WATCH/BLOCKED), và gửi alert qua Telegram. Giao diện web cho phép nhập payload thủ công và xem dashboard.

**MVP scope:** Không có HTF verification (Pine MVP không phát `bar_update`). Verdict dựa trên payload fields + live price + risk engine. HTF verification bổ sung ở target v1.

**Symbol focus:** BTCUSDT (Binance)

**Stack:**
- Python 3.11+
- FastAPI (web + API)
- SQLite (MVP persistence)
- python-telegram-bot v20
- Uvicorn (ASGI server)
- Jinja2 templates (no frontend framework)
- httpx (async HTTP, Binance API calls)
- websockets (Binance WebSocket live price)

## 2. Architecture

```
┌────────────────────────────────────────────────────────────┐
│                    FastAPI Application                     │
├─────────────┬──────────────┬───────────────┬───────────────┤
│  POST       │   Web UI     │   Dashboard   │   REST API    │
│ /webhook/   │   Form       │   Page        │   /api/       │
│ sats        │   (Jinja2)   │   (Jinja2)    │   signals     │
├─────────────┴──────┬───────┴───────┬───────┴───────────────┤
│           │  Services Layer     │          │               │
│  ┌────────▼──────────▼──────────▼──────────▼──────────┐    │
│  │              Verdict Pipeline                      │    │
│  │  Validator → Normalizer → StateStore               │    │
│  │  → [HTFVerifier DEFERRED]                         │    │
│  │  → EntryQuality → RiskEngine → VerdictEngine      │    │
│  │  → Deduper → TelegramFormatter                     │    │
│  └────────────────────────────────────────────────────┘    │
├────────────────────────────────────────────────────────────┤
│                        External                            │
│  Binance REST API (funding rate) [klines DEFERRED]       │
│  Binance WebSocket (live trade price)                      │
│  Telegram Bot API (alert delivery)                         │
└────────────────────────────────────────────────────────────┘
         │
         ▼
    SQLite Database
    - raw_payloads
    - latest_states
    - signal_decisions
    - signal_outcomes
```

---

## 3. Payload Format

### 3.1 Pine Script Alert

**Decision: MVP không dùng Pine theo kiểu "as-is". MVP dùng một minimal Pine patch: giữ các field SATS đang có và bắt buộc bổ sung `secret`, `event = "signal"`, `bar_time = time`, `symbol`, `timeframe`.**

#### Payload Field Coverage

| Field | MVP webhook contract (TradingView → backend) | MVP normalized payload (internal) | Target v1 |
|-------|----------------------------------------------|-----------------------------------|-----------|
| `event` | Required (`signal`) | Required | Required |
| `secret` | Required | Consumed at validation layer, not required downstream | Required |
| `source` | Optional | Default to `"SATS"` if missing | Required |
| `version` | Optional | Default to `"data-emitter-1.0"` if missing | Required |
| `symbol` | Required (from Pine) | Required | Required |
| `exchange` | Optional | Default to `"BINANCE"` if missing | Required |
| `timeframe` | Required (from Pine) | Required | Required |
| `bar_time` | Required (from Pine `time`) | Required | Required |
| `action` | ✅ Pine alert payload | ✅ | ✅ Pine |
| `price` | ✅ Pine | ✅ | ✅ Pine |
| `entry` | Optional | Default to `price` if missing | ✅ Pine |
| `sl` | ✅ Pine | ✅ | ✅ Pine |
| `tp1` / `tp2` / `tp3` | ✅ Pine | ✅ | ✅ Pine |
| `trend` | Optional | Derive from `action` if missing | Required |
| `score` | ✅ Pine | ✅ | ✅ Pine |
| `tqi` | ✅ Pine | ✅ | ✅ Pine |
| `er` | Optional / usually missing | Null if absent | Required |
| `rsi` | Optional / usually missing | Null if absent | Required |
| `vol_z` | Optional / usually missing | Null if absent | Required |
| `structure` | Optional / usually missing | Null if absent | Required |
| `momentum` | Optional / usually missing | Null if absent | Required |
| `regime` | Optional / usually missing | Null if absent | Required |
| `vol_regime` | Optional / usually missing | Null if absent | Required |
| `st_line` | Optional / usually missing | Null if absent | Required |
| `risk` | Not sent | Compute from payload if needed | Required |
| `tp_mode` | ✅ Pine | ✅ | ✅ Pine |
| `tp_scale` | ✅ Pine | ✅ | ✅ Pine |
| `r_multiples` | Not sent | Hardcode 1.0 / 2.0 / 3.0 in MVP | Required |

**Final MVP contract resolution:**
- `secret` must be sent by TradingView and validated against `config.yaml`.
- `event` is explicit in the request payload. MVP only accepts `event = "signal"`.
- `symbol` and `timeframe` are emitted directly by Pine (not normalized from `ticker`/`tf`).
- `bar_time` is required from Pine `time`; backend does not synthesize server-time values for idempotency.
- MVP does not apply staleness checks. Old `bar_time` values are still processed; staleness is deferred together with HTF verification.
- Backend validates the webhook against the **minimum wire contract**, then builds a normalized internal payload by filling defaults/derived fields (`source`, `version`, `exchange`, `trend`, `entry`) before verdict evaluation.

**MVP minimum webhook payload (TradingView wire contract):**
```json
{
  "event": "signal",
  "secret": "REPLACE_WITH_SECRET",
  "symbol": "BTCUSDT",
  "timeframe": "1h",
  "bar_time": 1730000000000,
  "action": "buy",
  "price": 79500.0,
  "score": 56,
  "tqi": 0.32,
  "tp_mode": "fixed",
  "tp_scale": 1.0,
  "sl": 77800.0,
  "tp1": 81200.0,
  "tp2": 83000.0,
  "tp3": 84900.0
}
```
BUY: entry=79500, SL=77800 (below entry), TP1/TP2/TP3=81200/83000/84900 (above entry) → valid long setup.

**MVP normalized payload (internal after validation + enrichment):**
```json
{
  "event": "signal",
  "source": "SATS",
  "version": "data-emitter-1.0",
  "symbol": "BTCUSDT",
  "exchange": "BINANCE",
  "timeframe": "1h",
  "bar_time": 1730000000000,
  "action": "buy",
  "price": 79500.0,
  "entry": 79500.0,
  "trend": "bullish",
  "score": 56,
  "tqi": 0.32,
  "tp_mode": "fixed",
  "tp_scale": 1.0,
  "sl": 77800.0,
  "tp1": 81200.0,
  "tp2": 83000.0,
  "tp3": 84900.0
}
```

#### Target Payload v1 (Phase sau Pine update)
```json
{
  "event": "signal",
  "secret": "REPLACE_WITH_SECRET",
  "source": "SATS",
  "version": "data-emitter-1.0",
  "symbol": "BTCUSDT",
  "exchange": "BINANCE",
  "timeframe": "1h",
  "bar_time": 1730000000000,
  "action": "buy",
  "price": 79500.5,
  "trend": "bullish",
  "score": 56,
  "tqi": 0.32,
  "er": 0.41,
  "rsi": 55.2,
  "vol_z": -0.8,
  "structure": 0.62,
  "momentum": 0.55,
  "regime": "mixed",
  "vol_regime": "normal",
  "entry": 79500.0,
  "sl": 77800.0,
  "tp1": 81200.0,
  "tp2": 83000.0,
  "tp3": 84900.0,
  "risk": 1700.0,
  "tp_mode": "fixed",
  "tp_scale": 1.0,
  "r_multiples": {
    "tp1": 1.0,
    "tp2": 2.0,
    "tp3": 3.0
  }
}
```

#### Bar Update Event (DEFERRED — Pine MVP không phát)
`bar_update` là điểm then chốt cho HTF verification. MVP không có event này nên HTF verifier bị defer. Phase sau: thêm `bar_update` cho `4h` và `1d` từ TradingView.

```json
{
  "event": "bar_update",
  "secret": "REPLACE_WITH_SECRET",
  "source": "SATS",
  "version": "data-emitter-1.0",
  "symbol": "BTCUSDT",
  "exchange": "BINANCE",
  "timeframe": "1h",
  "bar_time": 1730000000000,
  "close": 79500.5,
  "trend": "bullish",
  "tqi": 0.32,
  "er": 0.41,
  "rsi": 55.2,
  "vol_z": -0.8,
  "structure": 0.62,
  "momentum": 0.55,
  "regime": "mixed",
  "vol_regime": "normal",
  "st_line": 80750.0
}
```

### 3.2 Normalization

Convert variations to canonical:

```
BTCUSDT.P → BTCUSDT
BINANCE:BTCUSDT.P → BTCUSDT
1H / 60 / 1h → 1h
240 / 4H / 4h → 4h
D / 1D / 1d → 1d
```

Canonical timeframe set: `15m`, `30m`, `1h`, `4h`, `1d`

MVP Pine patch emits `symbol` and `timeframe` directly. Normalization only handles alias cleanup. `ticker`/`tf` → `symbol`/`timeframe` mapping is not MVP scope.

---

## 4. Database Schema

```sql
-- Every incoming payload, raw
CREATE TABLE raw_payloads (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    received_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    ingestion_source TEXT DEFAULT 'webhook',  -- 'webhook' or 'manual'
    event TEXT,
    symbol TEXT,
    tf TEXT,
    bar_time INTEGER,
    validation_status TEXT NOT NULL DEFAULT 'received',  -- received / invalid_json / invalid_secret / invalid_schema / accepted / duplicate
    error_code TEXT,
    payload_body TEXT NOT NULL  -- raw request body or manual input, even if JSON/schema is invalid
);

-- Dedup key is deterministic because `bar_time` is carried by TradingView payload.

-- Latest indicator state per symbol + timeframe
CREATE TABLE latest_states (
    symbol TEXT NOT NULL,
    tf TEXT NOT NULL,
    trend TEXT,
    tqi REAL,
    er REAL,
    rsi REAL,
    vol_z REAL,
    structure REAL,
    momentum REAL,
    regime TEXT,
    vol_regime TEXT,
    st_line REAL,
    bar_time INTEGER,
    last_event TEXT,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (symbol, tf)
);

-- Every signal decision with verdict + reasons
CREATE TABLE signal_decisions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    symbol TEXT NOT NULL,
    tf TEXT NOT NULL,
    action TEXT NOT NULL,
    bar_time INTEGER NOT NULL,
    entry REAL,
    sl REAL,
    tp1 REAL,
    tp2 REAL,
    tp3 REAL,
    price_at_decision REAL,
    current_r REAL,
    distance_to_tp1_r REAL,
    trend TEXT,
    tqi REAL,
    er REAL,
    rsi REAL,
    regime TEXT,
    vol_regime TEXT,
    score INTEGER,
    htf_snapshot_json TEXT,
    verdict TEXT NOT NULL,
    reasons_json TEXT,
    telegram_sent BOOLEAN DEFAULT FALSE,
    -- Market context at decision time
    funding_rate REAL,
    session TEXT,
    is_weekend BOOLEAN DEFAULT FALSE,
    position_size_recommendation REAL,
    UNIQUE(symbol, tf, action, bar_time)
);

-- Outcome tracking
CREATE TABLE signal_outcomes (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    decision_id INTEGER NOT NULL,
    hit_tp1 BOOLEAN DEFAULT FALSE,
    hit_tp2 BOOLEAN DEFAULT FALSE,
    hit_tp3 BOOLEAN DEFAULT FALSE,
    hit_sl BOOLEAN DEFAULT FALSE,
    max_favorable_r REAL DEFAULT 0.0,
    max_adverse_r REAL DEFAULT 0.0,
    realized_r REAL,
    closed_at DATETIME,
    FOREIGN KEY (decision_id) REFERENCES signal_decisions(id)
);

-- Indexes
CREATE INDEX idx_decisions_symbol_tf ON signal_decisions(symbol, tf);
CREATE INDEX idx_decisions_verdict ON signal_decisions(verdict);
CREATE INDEX idx_decisions_created ON signal_decisions(created_at);
CREATE INDEX idx_outcomes_decision ON signal_outcomes(decision_id);
```

---

## 5. Service Specifications

### 5.1 Webhook Handler

- Route: `POST /webhook/sats`
- Validate secret token from `config.yaml`
- Reject if secret missing/incorrect → 401
- Validate JSON schema → 422 if invalid
- **Always** log raw request body to `raw_payloads` before final validation outcome is returned
- If request body is parseable enough to extract fields, populate `event` / `symbol` / `tf` / `bar_time`; otherwise keep them null and store `validation_status` + `error_code`
- `bar_update` → store to `latest_states`, return 200
- `signal` → trigger full verdict pipeline
- **Idempotency:** `signal_decisions` has `UNIQUE(symbol, tf, action, bar_time)`. On duplicate key: skip verdict pipeline, mark raw payload `validation_status = 'duplicate'`, return `200` with `{"status": "duplicate", "id": <existing_id>}` — no Telegram spam on TradingView retry.

### 5.2 State Store

Performs upsert into `latest_states` for every `bar_update` and `signal` event.

State snapshot for verdict decision captured as JSON in `signal_decisions.htf_snapshot_json`.

**State merge semantics:**
- MVP: `signal` may upsert the latest snapshot because there is no continuous `bar_update` feed yet.
- Target v1: `bar_update` is authoritative for indicator state.
- Target v1: `signal` updates only columns present and non-null in the payload; missing/null values must preserve existing state (`COALESCE(existing, incoming)` is not enough in the wrong direction; preserve richer stored state).
- `last_event` tracks whether the current row was last refreshed by `signal` or `bar_update`.

### 5.3 HTF Deployment Model

MVP does not perform HTF verification. Backend verifies signals only from fields present in the payload. In MVP, `latest_states` is only a snapshot of received signals, not a continuous multi-timeframe state feed. A later phase adds `bar_update` to build true multi-TF state.

Rationale:
- Pine Script MVP không phát `bar_update` event → `latest_states` không có data để verify HTF.
- Backend tự fetch klines từ Binance để build HTF state → không còn là SATS authority (indicator state có thể khác Binance klines).
- MVP focus vào core verdict logic (entry quality, risk, regime indicators) trước. HTF verification là enhancement, không phải prerequisite cho verdict.
- Sau khi Pine Script update phát `bar_update`, bổ sung HTF verification dễ dàng hơn là refactor khi đang ổn định.

Phase sau (target v1): thêm `bar_update` events cho `4h` và `1d` từ TradingView, bổ sung HTF verifier vào pipeline.

Status: DEFERRED. MVP has no HTF verification.

Target v1 spec: HTF map `"1h": { confirm: ["4h"], macro: ["1d"] }` (không full matrix 15m-1d để giảm TradingView complexity). Precedence rules khi state missing/stale: missing confirm → `WATCH` max, missing macro → warning only, stale confirm → `BLOCKED`.

Target v1 staleness thresholds:

| Timeframe | Max Staleness |
|-----------|--------------|
| 15m       | 45 min       |
| 30m       | 90 min       |
| 1h        | 3 hours     |
| 4h        | 10 hours    |
| 1d        | 36 hours    |

### 5.4 Entry Quality Engine (No-Chase)

**Live price is a CORE dependency**, not optional enrichment. WebSocket must be connected for accurate verdict.

**Fallback behavior:**
- On WebSocket startup failure or disconnect → use `payload.price` (price at Pine alert time)
- When using fallback → verdict capped at `WATCH`, reason: `LIVE_PRICE_UNAVAILABLE`
- Backend must log when fallback is active
- Edge case: if BOTH ws fails AND `payload.price` is missing → block with reason `NO_PRICE_DATA`

**Reconnection:** Auto-reconnect on disconnect with exponential backoff (max 5 retries, 30s total wait).

```
current_r (BUY) = (live_price - entry) / (entry - sl)
current_r (SELL) = (entry - live_price) / (sl - entry)
```

Buckets (informational labels only — do NOT determine block/verdict):
| current_r range | Label |
|----------------|-------|
| < 0            | AGAINST_ENTRY |
| 0.00–0.30      | GOOD_ENTRY |
| 0.30–0.60      | LATE_ACCEPTABLE |
| 0.60–0.90      | LATE (watch) |
| > 0.90         | NO_CHASE |

**Block/quyết định verdict KHÔNG đến từ bucket label** — đến từ block rule #1 (`current_r > max_chase_r[tf]`) và rule #2 (`distance_to_tp1_r`). Bucket chỉ là informational display.

Example: 1h signal with `current_r = 0.85` → `current_r > 0.60` → **BLOCKED** (rule #1). Bucket label = "LATE (watch)" but verdict = BLOCKED. Remove "→ BLOCK" from bucket names to avoid confusion.

TP1 proximity:
```
distance_to_tp1_r = (tp1 - live_price) / (entry - sl)  [BUY]
distance_to_tp1_r = (live_price - tp1) / (sl - entry)  [SELL]
```
If `distance_to_tp1_r < 0.25` → BLOCK

### 5.5 Risk Engine

Check all:
- Entry, SL, TP1/TP2/TP3 exist and > 0
- TP1 is on correct side of entry
- SL is on correct side of entry
- RR to TP1 >= `min_tp1_r` (default 0.8)
- remaining distance to TP1 >= `min_remaining_to_tp1_r` (default 0.25R)

Max chase per TF:
| TF   | max_chase_r |
|------|-------------|
| 15m  | 0.50        |
| 30m  | 0.55        |
| 1h   | 0.60        |
| 4h   | 0.70        |
| 1d   | 0.80        |

Rule precedence:
- Check `distance_to_tp1_r < min_remaining_to_tp1_r` first. If true, verdict is `BLOCKED`.
- Check `current_r > max_chase_r[tf]` second. If true, verdict is `BLOCKED`.
- Entry-quality buckets are display labels only and do not block by themselves.

### 5.6 Verdict Engine

**Taxonomy chuẩn hóa (4 classes, bỏ C):**

| Internal Code | Telegram Label | Meaning |
|-------------|---------------|---------|
| `A_PLUS` | `A+ VALID` | Best-in-class signal |
| `A` | `A VALID` | Good signal, minor warnings |
| `WATCH` | `⚠️ WATCH` | Valid but not immediate entry |
| `BLOCKED` | `❌ BLOCKED` | Do not act |

Verdict taxonomy uses 4 classes only. `A_PLUS` and `A` cover quality range, `WATCH` covers valid-but-not-immediate entries, and `BLOCKED` covers hard blocks.

**Block conditions (any one triggers BLOCKED):**
1. `current_r > max_chase_r[tf]` (no-chase hard block)
2. `distance_to_tp1_r < min_remaining_to_tp1_r` (TP1 too close)
3. `live_price_unavailable` AND `payload.price` missing (ws down AND no price in payload → `NO_PRICE_DATA`)

**A+ conditions (ALL must be true):**
- Entry quality bucket = `GOOD_ENTRY` or `LATE_ACCEPTABLE`
- TP1 not too close (not blocked above)
- Risk engine checks pass
- If `regime` field present and `regime = choppy` → downgrade to A

**A conditions (ALL A+ minus regime):**
- Same as A+ but `regime = choppy` allowed with warning reason

**WATCH conditions:**
- Entry quality bucket = `WATCH_ONLY` (not blocked by rule 1)
- TP1 not blocked (passed rule 2)
- Risk checks pass
- Any other soft warning (e.g., `live_price_unavailable` with fallback used)
- If `regime` is absent in MVP payload, skip regime-based adjustments.

### 5.7 Deduper

Dedup key: `symbol + tf + action + bar_time`

Duplicate handling returns webhook response `{"status": "duplicate"}` and does not insert a new decision row.

Direction cooldown: No same-direction signal for same symbol within cooldown window (default 3 bars of source TF duration).

### 5.8 Market Context Service

**Live Price:**
- Connect to Binance WebSocket: `wss://stream.binance.com:9443/ws/btcusdt@trade`
- Cache last trade price in memory
- Reconnect automatically on disconnect

**Funding Rate:**
- Fetch from Binance REST: `GET /fapi/v1/fundingRate?symbol=BTCUSDT`
- Cache with 1-hour TTL
- If `funding_rate > 0.1%` (8h rate) → penalize BUY signals
- If `funding_rate < -0.1%` (8h rate) → penalize SELL signals

**Session Filter:**
```
UTC 00:00–08:00 → Asia session (weak, downgrade)
UTC 08:00–12:00 → London session (medium)
UTC 12:00–17:00 → NY session (strong, boost)
UTC 17:00–00:00 → Overlap/slowdown
```

**Weekend Filter:**
- Saturday 00:00 UTC to Sunday 20:00 UTC → weekend mode
- 15m/30m signals on weekend → `BLOCKED`
- 1h+ signals on weekend → downgrade `A+ → A`, `A → WATCH`, add reason `WEEKEND_MODE`

**All market context services are OPTIONAL ENRICHMENT:**
- If Binance REST fails → skip funding rate check, verdict core vẫn chạy
- If session detection fails → skip session filter, verdict core vẫn chạy
- If weekend check fails → skip weekend adjustment, verdict core vẫn chạy
- Core verdict logic must NOT depend on external API availability
- Market context only adds penalties/downgrades when data is available

### 5.9 Position Size Calculator

```
Account balance → from config.yaml
Risk% → from config.yaml (default 1%)
Risk Amount = Account Balance × Risk%

Position Size = Risk Amount / SL Distance in USDT
```

Verdict-adjusted sizing:
| Verdict | Risk Multiplier | Position % of full |
|---------|-----------------|-------------------|
| A+      | 1.0             | 100%             |
| A       | 0.75            | 75%              |
| WATCH   | 0.50            | 50%              |
| BLOCKED | 0.0             | 0% (skip)        |

### 5.10 Telegram Sender

Format message decision-first, reason-rich. Include:
- Verdict badge (emoji + class)
- Direction + TF + symbol
- Entry Quality badge
- Key indicators (TQI, ER, RSI) — if available in payload; null fields hidden
- Entry / SL / TP1 / TP2 / TP3
- Entry reasons (✅ / ⚠️ / ❌)
- Position size recommendation
- Session / weekend warning if applicable
- Funding rate if notable
- HTF trend context → display only when `bar_update` data exists (phase v1)

---

## 6. Web UI

### 6.1 Payload Input Page (`/`)

- Header: project name + nav links
- Textarea: paste JSON payload
- Button: "Verify & Send to Telegram"
- On submit:
  1. Validate JSON
  2. Run full verdict pipeline
  3. Display result card:
     - Verdict badge + class
     - All reasons with icons
     - Entry / SL / TP levels
     - Position size recommendation
     - "Sent to Telegram" status
- Sample payload button: pre-fills example JSON
- Error display for invalid JSON / validation failure

Manual input uses the same verdict pipeline as the webhook flow. Store `ingestion_source = "manual"` in `raw_payloads`, persist the resulting verdict in `signal_decisions`, and expose a single `"Verify & Send to Telegram"` action.

### 6.2 Dashboard Page (`/dashboard`)

**MVP Dashboard (Phase 1-2):**
- Total signals processed
- Verdict distribution (A+ / A / WATCH / BLOCKED) as bar chart
- Recent signals table: Timestamp, Symbol, TF, Action, Verdict, Entry, TQI, Telegram sent
  - Pagination (last 50), filter by verdict / TF / action
- Session & Market Context: current funding rate, current session, weekend status
- Latest received signal snapshot by TF (only when a signal for that TF exists; not a continuous HTF state feed in MVP)

**Phase sau (Dashboard v2):**
- Win rate (TP hit rate), Average R
- Win Rate over time (line chart), Avg R over time (line chart)
- Drawdown chart
- TP1/TP2/TP3/SL hit rates as percentage bars
- Regime Edge Table (9 cells)

### 6.3 REST API (`/api/`)

- `GET /api/signals` — list decisions (filter: verdict, tf, action, limit, offset)
- `GET /api/signals/{id}` — single decision detail
- `GET /api/stats` — aggregated stats for dashboard
- `GET /api/states` — latest states for all TFs
- `POST /api/decide` — run verdict pipeline on raw payload (for testing)

---

## 7. Configuration (`config.yaml`)

```yaml
app:
  host: "0.0.0.0"
  port: 8000
  debug: true

webhook:
  secret: "CHANGE_ME"

telegram:
  bot_token: "CHANGE_ME"
  chat_id: "CHANGE_ME"

trading:
  account_balance: 10000.0    # USDT for position sizing
  risk_percent: 1.0           # 1% risk per trade

symbols:
  BTCUSDT:
    enabled: true
    htf_map:
      "15m": { confirm: ["30m", "1h"], macro: ["4h"] }
      "30m": { confirm: ["1h"], macro: ["4h"] }
      "1h":  { confirm: ["4h"], macro: ["1d"] }
      "4h":  { confirm: ["1d"], macro: [] }
      "1d":  { confirm: [], macro: [] }
    max_chase_r:
      "15m": 0.50
      "30m": 0.55
      "1h":  0.60
      "4h":  0.70
      "1d":  0.80
    stale_after_minutes:
      "15m": 45
      "30m": 90
      "1h":  180
      "4h":  600
      "1d":  2160
    min_tp1_r: 0.8
    min_remaining_to_tp1_r: 0.25
    quality_thresholds:
      min_tqi: 0.20        # MVP active threshold
      min_er: 0.25         # target v1 only, when field is present
      min_structure: 0.45  # target v1 only, when field is present

market_filters:
  funding_rate_threshold: 0.001    # 0.1% 8h
  weekend_enabled: true
  session_filter_enabled: true
```

---

## 8. Project Structure

```
signal_backend/
├── main.py                      # FastAPI app, startup, lifespan
├── config.py                    # YAML config loader + pydantic model
├── config.yaml                  # All configuration
├── requirements.txt             # Dependencies
├── README.md                    # Setup instructions
│
├── db/
│   ├── database.py              # SQLite connection, session
│   └── migrations.py            # Schema creation, indexes
│
├── models/
│   └── schemas.py               # Pydantic models (request/response)
│
├── services/
│   ├── webhook_handler.py       # Route handler, event dispatcher
│   ├── validator.py             # Schema + secret validation
│   ├── normalizer.py            # Symbol/TF normalization
│   ├── state_store.py           # Read/write latest states
│   ├── htf_verifier.py          # HTF alignment scoring (DEFERRED to v1)
│   ├── entry_quality.py         # No-chase engine + live price + fallback
│   ├── risk_engine.py           # Risk/risk-reward checks
│   ├── verdict_engine.py        # A+/A/WATCH/BLOCKED classifier
│   ├── deduper.py               # Dedup (DB-level idempotency) + direction cooldown
│   ├── market_context.py        # Binance API: funding rate, session, weekend (optional enrichment)
│   ├── position_calculator.py   # Position size recommendation
│   ├── telegram_formatter.py    # Message formatting
│   └── telegram_sender.py       # Telegram Bot API calls
│
├── routers/
│   ├── ui.py                    # Web UI (Jinja2 templates)
│   ├── api.py                   # REST API endpoints
│   └── webhook.py               # POST /webhook/sats
│
├── templates/
│   ├── base.html
│   ├── index.html               # Payload input form
│   ├── result.html             # Verdict result display
│   └── dashboard.html          # Stats dashboard
│
└── static/
    ├── style.css
    └── chart.js                # Simple JS charts (Chart.js CDN)
```

---

## 9. Implementation Phases

### Phase 0: Lock MVP Emitter Contract
- Minimal Pine patch required: `event = "signal"`, `secret`, `bar_time = time`, `symbol`, `timeframe`
- Keep current SATS signal fields: `action`, `price`, `sl`, `tp1`, `tp2`, `tp3`, `score`, `tqi`, `tp_mode`, `tp_scale`
- Backend normalization: handles aliases (e.g. `BTCUSDT.P → BTCUSDT`, `1h → 1h`), does NOT normalize `ticker`/`tf` in MVP
- `ticker` and `tf` fields from Pine can be dropped in MVP — Pine patch emits `symbol`/`timeframe` directly
- `bar_update`, HTF fields (`er`, `rsi`, `vol_z`, `structure`, `momentum`, `regime`, `vol_regime`, `st_line`) and `r_multiples` stay deferred to target v1

### Phase 1: Core Infrastructure
- Project scaffold, config, database schema
- Pydantic models, SQLite setup
- State store service

### Phase 2: Web UI + Manual Input
- FastAPI Jinja2 setup
- Payload input page
- Verdict result display
- Manual "Verify & Send" flow

### Phase 3: Verdict Pipeline (Backend Brain)
- Validator + Normalizer
- HTF verifier deferred until Pine emits `bar_update`
- Entry Quality (with Binance WebSocket live price + fallback)
- Risk Engine
- Verdict Engine
- Deduper (DB-level idempotency, Direction cooldown tùy chọn)

### Phase 4: Telegram Integration
- TelegramFormatter
- TelegramSender
- Webhook → Telegram end-to-end

### Phase 5: Market Context
- Funding rate fetcher (Binance REST)
- Session detector
- Weekend filter
- Position size calculator

### Phase 6: Dashboard
- Dashboard MVP page: verdict distribution, recent signals, latest states, market context
- REST API: `/api/signals`, `/api/signals/{id}`, `/api/stats`, `/api/states`
- Prerequisite for analytics metrics: backend outcome tracker implemented (Phase 5 hoặc riêng)

---

## 10. Dependencies

```
fastapi>=0.110.0
uvicorn[standard]>=0.30.0
pydantic>=2.0
python-telegram-bot>=20.0
httpx>=0.27.0
websockets>=12.0
aiosqlite>=0.20
jinja2>=3.0
pyyaml>=6.0
python-multipart>=0.0.9
chart.js>=4.0  (CDN)
```

---

## 11. Security

- Webhook secret validated on every request
- No secrets in code — all in `config.yaml`
- Rate limit on webhook endpoint (100 req/min per IP)
- No external network required for verdict core — **with** deterministic fallback: if Binance WebSocket unavailable, use `payload.price` and cap verdict at `WATCH`. Market context (funding rate, session) is optional enrichment — backend continues without it.
- SQLite file outside web-accessible path

---

## 12. Quality Thresholds

Signals below active quality minimums are not blocked but receive `WATCH` verdict with warning reason `LOW_QUALITY`.

**MVP active thresholds:**

| Indicator | Min Value | MVP Status |
|-----------|-----------|------------|
| TQI       | 0.20      | Active |
| ER        | 0.25      | Deferred until field exists in payload |
| Structure | 0.45      | Deferred until field exists in payload |

Normative rule:
- MVP only enforces `tqi`.
- `er` and `structure` thresholds become active in target v1, and only when those fields are present in the normalized payload.
- Missing deferred fields must not downgrade an MVP signal by themselves.
