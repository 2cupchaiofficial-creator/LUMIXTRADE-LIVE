# LumixTrade (Aurum FX) — PRD / Memory

## Original Problem Statement
User imported a full-stack auto-trading bot (React + FastAPI + MongoDB + MT5 bridge v1.8.1) via ZIP and required deployment EXACTLY AS IS — no code/logic/config changes. Strategy files (strategy_v2.py, engine.py), DB schemas, API endpoints, and auth must be preserved.

## Current State (June 2026)
- Backend (FastAPI, port 8001) + Frontend (React, port 3000) running via supervisor. MongoDB connected.
- `.env` configured: fresh JWT_SECRET, ADMIN_EMAIL/ADMIN_PASSWORD set.
- Strategy: STRATEGY_VERSION=v2 (default), STRATEGY_CONSERVATIVE=true → conservative_config().
- Market data: MT5 bridge pushes candles to /api/bridge-candles (NO Twelve Data — user confirmed skip).
- Scheduler: APScheduler scans active bots every 3 min.

## Work Log
- 2026-06 (prev session): ZIP extracted, deps installed, services deployed, env vars set, admin seeded. Tested via curl/mongosh.
- 2026-06 (this session): DEEP AUDIT (analysis only). Reliability rated 58/100.
- 2026-06-11: ALL 7 AUDIT FIXES IMPLEMENTED (user-approved, safety features untouched):
  - P0-1: daily_loss_limit = % of equity (server.py gate #4; flat-USD fallback if no equity; matches frontend "DAILY LOSS %" label)
  - P0-2: Scalp activated in v2 — new `_setup_rsi_scalp` (RSI<35 buy / >65 sell + confirmation bar) as router fallback in ranging AND compression regimes; HTF hard-block preserved
  - P0-3: FIX #3b refined — blocks only when bar CLOSED beyond broken level (true breakout); sweeps (pierce + close back inside) now pass
  - P1-4: S/R gate ATR-scaled (0.5×ATR) instead of fixed 0.3% (`_sr_check` atr_v param)
  - P1-5: Asia enabled — DEFAULT_SESSIONS includes asia, startup backfill $addToSet on existing bots, current_session: 21-24 UTC = asia (was "off"); 0.5× lot via existing session_mult
  - P2-6: Scanner HTF gate blocks only DECISIVELY opposite trend (flat/None = neutral); htf_trend fetched once (dedupe)
  - P2-7: Scalp RR floor 1.3 (swing keeps 2.0), scalp_tp_atr 1.3, max_hold_scalp 30 min
  - Tests: /app/backend/tests/test_audit_fixes.py — 13/13 pass. test_risk_gates.py script ALL PASS. test_htf_confirmation failure verified PRE-EXISTING (fails on unmodified code too). HTTP legacy tests point to stale preview URL (pre-existing env issue).

## Audit Findings (key — full report delivered in chat)
1. Scalp mode dead code under v2: engine.py `_generate_scalp` / `enable_scalping_in_ranges` only used in v1 path; v2's only scalp = liquidity sweep reversal.
2. FIX #3b (strategy_v2.py L372-387) structurally blocks sweep-reversal setups (sweep requires breaking 20-bar extreme; gate blocks exactly that).
3. S/R gate `near_pct=0.003` (L454) not ATR-scaled → blocks most FX M15 trend signals (no neutral zone in 20-bar range).
4. Asia zero trades: DEFAULT_SESSIONS excludes "asia" (server.py L308); 21:00-24:00 UTC = "off" hard block (engine.py L154-165); triple penalty (conf −0.04, lot 0.5×, scalp 0.5×).
5. daily_loss_limit default $5 flat (server.py L223) → caps frequency at ~3 trades/day; should be % of equity.
6. Double HTF gating: strategy require_htf_alignment + scanner gate (L2241) treating "flat" as mismatch.
7. `_enforce_min_rr(2.0)` applied to scalps + 60-min max hold → scalp TPs unreachable, die by timeout.
8. `bar_close_only=True` is dead config — never referenced; scanner acts on forming bars.
9. `recent_win_rate` plumbed into adaptive_lot but always passed None (server.py L2272).

## Backlog (not yet approved/implemented)
- Implement dead `bar_close_only` flag (scanner still evaluates forming bars)
- Wire `recent_win_rate` into adaptive_lot (currently always None)
- Trade-frequency profiles (Conservative/Balanced/Aggressive)
- Per-gate "signals funnel" diagnostic dashboard

## Constraints
- STRICT: user is highly protective of codebase. No refactoring/improvements beyond explicitly approved fixes.
- Goal: 10-20 trades/day active sessions, 5-10 Asia, 45-55% WR, ≥1:2 RR (swings).

## Architecture Reference
- server.py (2818 lines): API + scanner `_scan_and_persist` (L2033) + scheduler (L2330) + gates.
- strategy_v2.py: v2 router (BOS retest / squeeze breakout / liquidity reversal), S/R gates, RR floor.
- engine.py: indicators, sessions, v1 engine (incl. dead scalp), calc_lot.
- risk_engine.py: adaptive_lot, volatility gate, slippage, heartbeat.
- marketdata.py: db.candles read/write (bridge-fed).
