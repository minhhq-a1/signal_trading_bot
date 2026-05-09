# SATS BTC Signal System — Optimized Solution Context

## 1. Executive Summary

Mục tiêu không nên là tiếp tục tối ưu Pine Script để tự quyết định điểm vào lệnh. Hướng tối ưu nhất là tách hệ thống thành 3 lớp rõ ràng:

```text
TradingView SATS Indicator = Data Emitter / Sensor
Backend = Stateful Multi-Timeframe Decision Engine
Telegram Bot = Messenger / Alert Delivery
```

Indicator chỉ gửi dữ liệu theo đúng logic SATS: trend, signal gốc, TQI, ER, RSI, Vol Z, structure, momentum, regime, entry, SL, TP. Backend mới là nơi verify đa timeframe, đánh giá điểm vào, lọc no-chase, xếp hạng setup, chống trùng alert, ghi log, và gửi kết quả cuối sang Telegram.

Đây là hướng tối ưu hơn việc ép Pine Script xử lý toàn bộ vì Pine bị hạn chế về state, debug, logging, replay, multi-timeframe verification và decision logic phức tạp.

---

## 2. Core Principle

### 2.1 Indicator is a Sensor

Indicator không nên quyết định lệnh cuối cùng.

Indicator chỉ trả lời:

```text
Tôi đang thấy thị trường như thế nào?
```

Nó gửi:

- Symbol / exchange / timeframe
- Bar time / close price
- SATS trend
- Raw BUY/SELL signal nếu có
- TQI và các component
- Regime / volatility regime
- Entry / SL / TP
- Score theo logic gốc
- TP mode / R multiples

### 2.2 Backend is the Brain

Backend trả lời:

```text
Signal này có đáng gửi cho trader không?
Nếu có thì chất lượng setup là gì?
Nếu không thì vì sao bị block?
```

Backend thực hiện:

- Validate payload
- Normalize symbol/timeframe
- Store latest state per symbol/timeframe
- Verify multi-timeframe alignment
- Check no-chase / entry quality
- Check risk/reward
- Check stale data
- Check duplicate signal
- Classify verdict
- Format Telegram message
- Log all raw/valid/blocked signals

### 2.3 Telegram is the Messenger

Telegram không chỉ gửi BUY/SELL. Telegram phải gửi:

- Verdict: A+ / A / B / WATCH / BLOCKED
- Direction
- Timeframe source
- Entry / SL / TP
- Multi-timeframe context
- Entry quality
- Reasons
- Warnings
- Suggested action style

---

## 3. Target Architecture

```text
TradingView SATS Data Emitter
        ↓
Webhook Receiver
        ↓
Payload Validator + Normalizer
        ↓
State Store by Symbol + Timeframe
        ↓
Multi-Timeframe Verifier
        ↓
Entry Quality / No-Chase Engine
        ↓
Risk Engine
        ↓
Decision Engine
        ↓
Dedupe / Cooldown / Priority Engine
        ↓
Telegram Formatter
        ↓
Telegram Bot
        ↓
Signal Database + Replay / Review Layer
```

---

## 4. Components

## 4.1 TradingView SATS Data Emitter

### Purpose

The Pine indicator should only emit data. It should not be heavily modified to become the decision engine.

### Recommended Pine Direction

Use a clean SATS-based indicator that:

- Preserves original SATS trend logic
- Emits JSON webhook alerts
- Sends enough market state for backend verification
- Avoids over-optimization in Pine
- Avoids timeframe-specific overfitting
- Avoids excessive score/regime/cooldown logic inside Pine

### Alert Types

The system should support at least two alert event types.

### A. `bar_update`

A regular state update on confirmed bar close.

This can be used by backend to keep latest state for each timeframe.

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
  "trend": "bearish",
  "tqi": 0.32,
  "er": 0.41,
  "rsi": 38.5,
  "vol_z": -0.8,
  "structure": 0.62,
  "momentum": 0.55,
  "regime": "mixed",
  "vol_regime": "normal",
  "st_line": 80750.0
}
```

### B. `signal`

A raw SATS BUY/SELL signal.

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
  "action": "sell",
  "price": 79500.5,
  "trend": "bearish",
  "score": 56,
  "tqi": 0.32,
  "er": 0.41,
  "rsi": 38.5,
  "vol_z": -0.8,
  "structure": 0.62,
  "momentum": 0.55,
  "regime": "mixed",
  "vol_regime": "normal",
  "entry": 79500.5,
  "sl": 81200.0,
  "tp1": 77800.0,
  "tp2": 76100.0,
  "tp3": 74400.0,
  "risk": 1699.5,
  "tp_mode": "fixed",
  "r_multiples": {
    "tp1": 1.0,
    "tp2": 2.0,
    "tp3": 3.0
  }
}
```

---

## 4.2 Webhook Receiver

### Responsibilities

- Receive TradingView webhook payload
- Validate secret
- Validate schema
- Reject invalid/malformed payloads
- Normalize symbol and timeframe
- Store latest state
- Trigger verifier for signal events
- Log raw payload

### Recommended Stack

For simplicity and reliability:

```text
Python FastAPI
SQLite for MVP
Redis optional for low-latency state
Telegram Bot API
```

Python is recommended because rule engine, replay, and analytics will be easier to evolve.

---

## 4.3 Payload Validator / Normalizer

### Required Validations

Payload should be rejected or marked invalid if:

- Secret is missing or incorrect
- Event is unknown
- Symbol missing
- Timeframe missing
- Bar time missing
- Numeric fields are invalid
- Action is not buy/sell for signal events
- Entry/SL/TP are missing for signal events
- Risk is zero or invalid
- Timeframe is unsupported

### Normalization

Convert these variations into canonical values:

```text
BTCUSDT.P → BTCUSDT
BINANCE:BTCUSDT.P → BTCUSDT
1H / 60 / 1h → 1h
240 / 4H / 4h → 4h
D / 1D / 1d → 1d
```

Canonical timeframe set:

```text
15m
30m
1h
4h
1d
```

---

## 4.4 State Store

Backend must store latest indicator state per symbol and timeframe.

Example:

```json
{
  "BTCUSDT": {
    "15m": {
      "trend": "bearish",
      "tqi": 0.28,
      "er": 0.33,
      "regime": "mixed",
      "updated_at": "2026-05-08T10:15:00Z"
    },
    "30m": {
      "trend": "bearish",
      "tqi": 0.35,
      "er": 0.39,
      "regime": "mixed",
      "updated_at": "2026-05-08T10:30:00Z"
    },
    "1h": {
      "trend": "bearish",
      "tqi": 0.42,
      "er": 0.41,
      "regime": "mixed",
      "updated_at": "2026-05-08T10:00:00Z"
    },
    "4h": {
      "trend": "bearish",
      "tqi": 0.47,
      "er": 0.50,
      "regime": "trending",
      "updated_at": "2026-05-08T08:00:00Z"
    },
    "1d": {
      "trend": "mixed",
      "tqi": 0.30,
      "er": 0.27,
      "regime": "mixed",
      "updated_at": "2026-05-08T00:00:00Z"
    }
  }
}
```

### Stale State Rules

Each timeframe has a maximum allowed staleness.

Recommended initial values:

```yaml
stale_after:
  15m: 45m
  30m: 90m
  1h: 3h
  4h: 10h
  1d: 36h
```

If required higher timeframe state is stale, downgrade verdict or block.

---

## 5. Decision Model

## 5.1 Verdict Classes

The backend should not return only BUY/SELL. It should classify setup quality.

```text
A+ VALID
A VALID
B WATCH
C WEAK
BLOCKED
```

### A+ VALID

High-quality trade candidate.

Typical requirements:

- Raw signal confirmed on source timeframe
- Source timeframe trend agrees with signal
- Higher timeframe aligned
- Macro timeframe not strongly opposing
- Entry not chased
- TP1 not too close
- Regime not choppy
- TQI/ER/structure context acceptable
- No duplicate or cooldown conflict

### A VALID

Good trade candidate, but with one or two minor warnings.

### B WATCH

Not immediate entry, but worth monitoring.

### C WEAK

Low confidence. Usually do not trade, but not fully invalid.

### BLOCKED

Do not send as actionable signal.

Reasons may include:

- Price already moved too far from entry
- TP1 too close
- HTF strongly against signal
- Payload stale
- Duplicate signal
- Choppy regime + weak quality
- Invalid risk
- Cooldown active

---

## 5.2 Source-Timeframe Rules

### 15m Signal

15m should not decide major trades alone.

```text
15m VALID only if:
- 30m same direction or neutral
- 1h same direction
- 4h not strongly opposite
- Entry quality <= 0.50R from entry
```

If 15m conflicts with 1h/4h:

```text
Verdict: WATCH or BLOCKED
Label: Counter-trend scalp only
```

### 30m Signal

```text
30m VALID only if:
- 1h same direction or mixed
- 4h not strongly opposite
- Entry quality acceptable
- Regime not choppy
```

### 1h Signal

1h is the main execution timeframe.

```text
1h VALID if:
- 4h same direction or mixed
- 1d not strongly opposite
- Entry quality <= 0.60R
- TP1 not too close
- Regime not choppy unless quality is high
```

### 4h Signal

```text
4h VALID if:
- 1d same direction or mixed
- Entry quality <= 0.70R
- Risk/reward acceptable
```

### 1d Signal

1d mainly defines macro bias.

```text
1d signal = macro regime update
Do not spam Telegram
Use it as filter for lower timeframe signals
```

---

## 6. Entry Quality / No-Chase Engine

This is mandatory.

Signal direction can be correct while entry quality is bad. The backend must classify current price relative to entry/SL/TP.

### Formula

For BUY:

```text
current_r = (current_price - entry) / (entry - sl)
```

For SELL:

```text
current_r = (entry - current_price) / (sl - entry)
```

### Entry Quality Buckets

```text
current_r < 0:
  AGAINST_ENTRY

0.00R – 0.30R:
  GOOD_ENTRY

0.30R – 0.60R:
  LATE_ACCEPTABLE

0.60R – 0.90R:
  WATCH_ONLY

> 0.90R:
  NO_CHASE
```

### TP1 Proximity Rule

Block if price is too close to TP1.

For BUY:

```text
distance_to_tp1_r = (tp1 - current_price) / (entry - sl)
```

For SELL:

```text
distance_to_tp1_r = (current_price - tp1) / (sl - entry)
```

Recommended:

```text
distance_to_tp1_r < 0.25R → BLOCK or WATCH_ONLY
```

---

## 7. Multi-Timeframe Verification

## 7.1 Alignment Matrix

For a SELL signal:

```text
Aligned:
- Same timeframe trend = bearish
- HTF trend = bearish or mixed
- Macro trend != strongly bullish

Against:
- HTF trend = bullish with good TQI
- Macro trend = bullish with high TQI
```

For a BUY signal:

```text
Aligned:
- Same timeframe trend = bullish
- HTF trend = bullish or mixed
- Macro trend != strongly bearish

Against:
- HTF trend = bearish with good TQI
- Macro trend = bearish with high TQI
```

## 7.2 HTF Mapping

```yaml
htf_map:
  15m:
    confirm: ["30m", "1h"]
    macro: ["4h"]
  30m:
    confirm: ["1h"]
    macro: ["4h"]
  1h:
    confirm: ["4h"]
    macro: ["1d"]
  4h:
    confirm: ["1d"]
    macro: []
  1d:
    confirm: []
    macro: []
```

## 7.3 HTF Scoring

Recommended alignment scoring:

```text
+2 if confirm timeframe same direction
+1 if confirm timeframe mixed
-2 if confirm timeframe opposite with TQI >= 0.35

+1 if macro timeframe same direction or mixed
-2 if macro timeframe opposite with TQI >= 0.40
```

Use this score as part of verdict classification.

---

## 8. Regime Logic

Regime should not blindly block all trades. It should influence verdict.

### Recommended Rules

```text
Trending:
  No penalty

Mixed:
  Slight penalty
  Require better entry quality
  Downgrade A+ to A if HTF is not aligned

Choppy:
  Block weak signals
  Only allow if:
    - HTF strongly aligned
    - Entry quality is GOOD_ENTRY
    - TQI/structure context is acceptable
```

### Initial Thresholds

```yaml
quality_thresholds:
  min_tqi:
    15m: 0.20
    30m: 0.20
    1h: 0.20
    4h: 0.18
    1d: 0.15
  min_er:
    15m: 0.25
    30m: 0.25
    1h: 0.25
    4h: 0.22
    1d: 0.20
  min_structure:
    default: 0.45
```

---

## 9. Risk Engine

Risk engine checks whether signal is tradeable from risk perspective.

### Required Checks

- Entry exists
- SL exists
- TP1/TP2/TP3 exist
- Risk > 0
- TP1 is on correct side
- SL is on correct side
- Reward-to-risk to TP1 >= minimum
- Price not beyond TP1
- No-chase not violated

### Recommended Rules

```yaml
risk_rules:
  min_tp1_r: 0.8
  min_remaining_to_tp1_r: 0.25
  max_chase_r:
    15m: 0.50
    30m: 0.55
    1h: 0.60
    4h: 0.70
    1d: 0.80
```

---

## 10. Dedupe / Cooldown / Priority

### Duplicate Protection

Prevent repeated Telegram messages from the same signal.

Dedup key:

```text
symbol + timeframe + action + bar_time
```

### Direction Cooldown

Avoid repeated same-direction spam.

Example:

```yaml
cooldown:
  same_symbol_timeframe_signal: 3 bars
  same_symbol_any_tf_action: 15 minutes
```

### Priority Handling

If multiple signals arrive close together:

```text
4h signal has higher priority than 1h
1h signal has higher priority than 30m
30m signal has higher priority than 15m
```

If 15m and 1h conflict:

```text
prefer 1h/4h context
downgrade 15m to WATCH or BLOCKED
```

---

## 11. Telegram Output

Telegram message should be decision-first and reason-rich.

### A Valid Example

```text
🔴 BTCUSDT SELL — 1H

Verdict: A VALID
Entry Quality: GOOD_ENTRY (+0.18R)
Trend: 1H Bearish | 4H Bearish | 1D Mixed
Regime: Mixed / Normal Vol
TQI: 0.32 | ER: 0.41 | RSI: 38.5
Score: 56

Entry: 79,500
SL: 81,200
TP1: 77,800
TP2: 76,100
TP3: 74,400

Why:
✅ 1H raw signal confirmed
✅ 4H aligned bearish
✅ Price not chased
⚠️ 1D mixed, reduce size
⚠️ Regime mixed

Suggested Action:
Consider partial size. Do not chase if price moves beyond +0.60R.
```

### Blocked Example

```text
⚠️ BTCUSDT SELL blocked — 1H

Reasons:
❌ Price already moved +0.82R from entry
❌ TP1 too close: 0.12R remaining
⚠️ 4H trend mixed
```

### Watch Example

```text
🟡 BTCUSDT BUY watch — 30M

Reason:
Signal is valid by SATS, but 1H trend is mixed and price is already +0.54R from entry.

Suggested Action:
Wait for pullback near entry or next confirmed signal.
```

---

## 12. Database / Logging

Logging is mandatory for optimization.

### Store Every Payload

Do not only store sent alerts.

Store:

- raw bar updates
- raw signals
- validated signals
- blocked signals
- Telegram-sent alerts
- reasons
- state snapshot at decision time
- later outcome

### Suggested Tables

```text
raw_payloads
latest_states
signal_decisions
telegram_messages
signal_outcomes
```

### `signal_decisions` Fields

```text
id
created_at
symbol
timeframe
action
bar_time
entry
sl
tp1
tp2
tp3
price_at_decision
current_r
distance_to_tp1_r
trend
tqi
er
rsi
regime
htf_snapshot_json
verdict
score
reasons_json
telegram_sent
```

### Outcome Tracking

Later, track:

```text
hit_tp1
hit_tp2
hit_tp3
hit_sl
max_favorable_r
max_adverse_r
outcome_after_4h
outcome_after_24h
```

---

## 13. Replay / Review Engine

This is what makes the system optimizable.

After collecting data, replay decisions with different rules:

```text
What if max_chase_r = 0.5 instead of 0.6?
What if 15m requires 1h alignment?
What if mixed regime downgrades instead of blocks?
What if choppy blocks all signals?
```

### Metrics

Track:

```text
Win rate
Avg R
Max drawdown
TP1 hit rate
TP2 hit rate
TP3 hit rate
Blocked-good-signal rate
Sent-bad-signal rate
Timeframe-specific edge
Regime-specific edge
Long vs short edge
```

---

## 14. Configuration-Driven Rules

Rules should live in `config.yaml`, not be hardcoded.

Example:

```yaml
symbols:
  BTCUSDT:
    enabled_timeframes: ["15m", "30m", "1h", "4h", "1d"]

    htf_map:
      "15m":
        confirm: ["30m", "1h"]
        macro: ["4h"]
      "30m":
        confirm: ["1h"]
        macro: ["4h"]
      "1h":
        confirm: ["4h"]
        macro: ["1d"]
      "4h":
        confirm: ["1d"]
        macro: []
      "1d":
        confirm: []
        macro: []

    max_chase_r:
      "15m": 0.50
      "30m": 0.55
      "1h": 0.60
      "4h": 0.70
      "1d": 0.80

    min_remaining_to_tp1_r: 0.25

    regime_policy:
      trending: allow
      mixed: downgrade
      choppy: strict

    stale_after_minutes:
      "15m": 45
      "30m": 90
      "1h": 180
      "4h": 600
      "1d": 2160
```

---

## 15. Security / Reliability

### Webhook Security

- Use a secret token in payload
- Reject requests without correct secret
- Optional: IP allowlist if infrastructure supports it
- Rate limit endpoint
- Log invalid attempts

### Reliability

- Respond quickly to TradingView webhook
- Do not block request while sending Telegram if possible
- Use queue for Telegram send if needed
- Retry Telegram send on failure
- Log failures

### Idempotency

Use dedupe key:

```text
symbol + timeframe + action + bar_time
```

This prevents duplicate Telegram messages if TradingView retries.

---

## 16. Implementation Roadmap

## Phase 1 — Pine Data Emitter

Deliverables:

- Clean SATS Data Emitter Pine
- JSON webhook alert for bar_update
- JSON webhook alert for signal
- Minimal dashboard
- No heavy decision logic inside Pine

Goal:

```text
TradingView reliably sends clean structured data.
```

## Phase 2 — Backend MVP

Deliverables:

- FastAPI webhook endpoint
- Payload validation
- SQLite storage
- Latest state table
- Raw signal log
- Simple Telegram send

Goal:

```text
End-to-end pipeline works.
```

## Phase 3 — Multi-Timeframe Verifier

Deliverables:

- HTF state lookup
- Stale state check
- No-chase engine
- Risk checks
- Verdict classification
- Reasons list
- Telegram verdict message

Goal:

```text
Backend decides VALID / WATCH / BLOCKED with reasons.
```

## Phase 4 — Logging + Replay

Deliverables:

- Signal decision logs
- Outcome tracking
- Replay script
- Metrics report

Goal:

```text
Optimize rules from data, not from visual guessing.
```

## Phase 5 — Production Hardening

Deliverables:

- Queue/retry Telegram sends
- Better error handling
- Deployment config
- Monitoring/log rotation
- Backups
- Admin commands for Telegram bot

Goal:

```text
System can run continuously.
```

---

## 17. Initial Recommendation

Start with:

```text
Symbol: BTCUSDT
Timeframes: 15m, 30m, 1h, 4h, 1d
Main execution timeframe: 1h
Primary confirm timeframe: 4h
Macro bias timeframe: 1d
```

Initial verdict rules:

```text
1h signal + 4h aligned + no-chase good = A/A+
1h signal + 4h mixed + no-chase good = B/A depending quality
1h signal + 4h opposite strong = BLOCKED
15m signal against 1h/4h = WATCH or BLOCKED
Price > 0.6R from entry on 1h = WATCH/BLOCKED
Distance to TP1 < 0.25R = BLOCKED
```

---

## 18. Final Answer

The optimized solution is:

```text
SATS Indicator should become a clean data emitter.
Backend should be a stateful, deterministic, multi-timeframe verifier.
Telegram should only receive ranked verdicts with reasons.
Every signal, including blocked signals, must be logged.
Replay engine must be used to improve rules using real collected data.
```

This is the best architecture because it avoids overfitting Pine Script, preserves the original SATS logic, enables real multi-timeframe validation, supports no-chase/risk checks, gives useful Telegram alerts, and creates a data loop for systematic improvement.
