# ICT Order Block Retest — Strategy Specification

> Source-of-truth design document for the Pine Script strategy (with an eventual
> Python port in mind). This captures the decisions made during the planning
> phase. No trade logic should diverge from this spec without updating it here.

---

## 1. Objective

Build a TradingView **Pine Script strategy** that:

1. Programmatically identifies **ICT Order Blocks (OBs)** on a higher (context)
   timeframe.
2. Waits for price to **retest** an unmitigated OB on a lower (execution)
   timeframe, then enters with clearly defined **entry / stop / target** levels.
3. **Visually marks** all relevant levels (zones, proximal/distal/50% lines,
   entry/stop/TP labels, legend) so the rationale behind every trade is visible.

The detection logic is to be kept as small, pure, well-isolated functions so the
strategy can later be **ported to Python** (likely pandas/vectorized) with
minimal friction.

---

## 2. Core Concept: What is an Order Block

An **Order Block** is the last opposing candle before an aggressive,
displacement-driven move in the opposite direction.

- **Bullish OB** — the last *down* (bearish) candle before a strong *up* move.
- **Bearish OB** — the last *up* (bullish) candle before a strong *down* move.

### 2.1 Key levels of a single OB

| Level | Bullish OB | Bearish OB | Use |
|---|---|---|---|
| **OB High** | top of range | top of range | edge of zone |
| **OB Low** | bottom of range | bottom of range | edge of zone |
| **Open / Close (body)** | core zone | core zone | optional body-only mode |
| **50% Mean Threshold** | `(high+low)/2` | `(high+low)/2` | **default entry trigger** |
| **Proximal line** | OB High | OB Low | edge nearest price on retest |
| **Distal line** | OB Low | OB High | far edge → stop placement |

### 2.2 Contextual validators (what makes an OB "real")

A lone candle is not enough. An OB is only valid when the move *away* from it is
true displacement:

1. **Fair Value Gap (FVG) / Imbalance** — required. 3-candle imbalance where
   candle 1 and candle 3 ranges do not overlap.
2. **Break of Structure (BOS)** — required. The displacement breaks a recent
   swing high (bullish) / swing low (bearish). Needs swing (pivot) detection.
3. **ATR strength filter** — optional, default **off**. Displacement candle(s)
   exceed N× ATR.
4. **Liquidity sweep** — optional/future. OB forms right after a stop run.

---

## 3. Architecture: Multi-Timeframe (Path A) — LOCKED

**Path A:** the script runs on the **lower (execution) timeframe chart** and
pulls the **higher (context) timeframe** candles via `request.security()`.

- HTF defines the order blocks.
- LTF times the retest entry and executes trades (realistic strategy fills).

### 3.1 Multi-timeframe rules (anti-repaint, non-negotiable)

1. **`lookahead = barmerge.lookahead_off`** on every `request.security` call.
2. **Closed HTF bars only** — reference the HTF series at a confirmed offset
   (e.g. `[1]`) so an OB is only acted on once its HTF candle has closed.
   Guarantees live == backtest (no repaint).
3. **HTF ≥ chart TF** — input guard must warn/error if context TF < execution TF.
4. **`barmerge.gaps_off`** — HTF value held constant across the LTF bars inside
   the HTF period, so zones stay fixed while LTF bars travel through them.
5. **Detection on HTF, execution on LTF** — swings/FVG/BOS/OB on the HTF series;
   retest state machine + `strategy.entry/exit` on native LTF bars.

### 3.2 Timeframe pairing input (LOCKED)

| Option | Context (HTF) | Execution (LTF) |
|---|---|---|
| **Default** | 1H | 15m |
| Option 2 | 30m | 15m |
| Option 3 | 30m | 5m |

Two-timeframe (context + execution) ships first. A third "bias" timeframe
(e.g. 4H bias → 1H zones → 15m entry) is **deferred** to a later iteration.

---

## 4. The Retest Mechanic (heart of the strategy) — LOCKED

We **never** enter on the candle that forms the OB. Identification and execution
are distinct events; the gap between them is the retest.

### 4.1 OB lifecycle (per-OB state machine)

```
FORMED ──> ARMED ──> ACTIVE ──> DEAD
```

1. **FORMED** — OB candle + displacement (FVG + BOS) confirmed on HTF. Draw the
   zone. *No trade.*
2. **ARMED** — price moved away; zone is fresh (unmitigated). Pending setup.
3. **RETEST (trigger)** — price re-enters the zone on the LTF:
   - **Passive**: limit fill at the configured retest level.
   - **Confirmation**: enter on an LTF confirmation candle closing in the trade
     direction.
4. **ACTIVE** — position open. Stop at distal edge (+buffer); target at R / structure.
5. **DEAD / MITIGATED** — retired when a **body closes through the distal edge**,
   the OB is consumed, or it expires.

### 4.2 Retest rules (LOCKED)

- **Retest trigger level** — input dropdown, **default = 50% Mean Threshold**;
  alternatives: `Proximal Edge`, `Distal Edge`.
- **Freshness** — only ARMED (unmitigated) OBs are eligible. No re-arming after
  a trade.
- **Expiry** — OB dies after **N HTF candles** (default **10**) *or* on an
  **opposing BOS**, whichever comes first. Expiry counted in HTF terms so it is
  meaningful across timeframes.
- **One-shot** — each OB produces at most one trade.
- **Mitigation** — first *tap* triggers the retest entry but does not by itself
  kill the zone; a **body close through the distal edge** kills it.

---

## 5. Trade Execution Model

- **Entry** — at the retest trigger level (default 50% mean threshold), either
  passive limit or LTF confirmation candle (input).
- **Stop** — beyond the **distal** edge (below OB low for longs / above OB high
  for shorts) + small buffer.
- **Target** — configurable **R multiple** (default **2R**), with an optional
  **structure-based** target (prior swing liquidity / opposing FVG).
- **Risk** — defined by OB height (entry at proximal/50%, stop at distal), giving
  a clean, self-contained R.

---

## 6. Inputs (planned)

| Input | Default | Notes |
|---|---|---|
| Timeframe pairing | 1H / 15m | dropdown (see §3.2) |
| OB zone definition | Full range | toggle: Full range / Body only |
| Retest trigger level | 50% Mean Threshold | dropdown (50% / Proximal / Distal) |
| Entry style | Passive limit | toggle: Passive / Confirmation candle |
| Swing pivot lookback | 5 / 5 | left / right |
| Require FVG | true | displacement filter |
| Require BOS | true | displacement filter |
| ATR strength filter | off | optional, N× ATR |
| OB expiry (HTF candles) | 10 | + opposing BOS |
| Stop buffer | small | beyond distal edge |
| Target model | 2R | R multiple and/or structure |
| Visualization toggles | on | boxes / lines / labels / legend |

---

## 7. Visualization (requirement: "see the logic")

- **Boxes** for OB zones (color by direction + state).
- **Lines** for proximal, distal, and 50% mean threshold.
- **Labels** for entry, stop, and TP at trade time.
- **On-chart legend** explaining colors/states.

---

## 8. Build Phases

| Phase | Deliverable |
|---|---|
| **0 — Spec lock** | This document. ✅ |
| **1 — Structure primitives** | swing high/low, FVG, BOS detection |
| **2 — OB identification** | last opposing candle before qualifying displacement → FORMED zones w/ levels |
| **3 — Zone lifecycle** | ARMED tracking, mitigation, expiry, drawing (state machine) |
| **4 — Retest + execution** | retest detection, `strategy.entry/exit`, stop at distal, R/structure targets |
| **5 — Visualization & legend** | clean entry/stop/TP labels + legend |
| **6 — Backtest tuning** | inputs, defaults, sanity checks across instruments/TFs |

---

## 9. Python Port Notes

- Keep **all detection logic** (swings, FVG, BOS, OB selection) as pure functions
  with explicit inputs/outputs.
- Pine is bar-by-bar/series-based; Python will likely vectorize with pandas.
- The per-OB **state machine** (FORMED→ARMED→ACTIVE→DEAD) is explicit and testable
  in both environments.

---

_Status: Phase 0 complete. Ready to begin Phase 1 (structure primitives)._
