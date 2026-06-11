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
- 2026-06 (this session): DEEP AUDIT (analysis only, NO code changes) of signal starvation. Reliability rated 58/100.

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

## User-Proposed Fix Sequence (awaiting user decision — DO NOT implement until approved)
1. Daily loss limit → % of equity (2-3%)
2. Resolve FIX #3b ↔ sweep contradiction
3. ATR-scale near_pct in S/R gate
4. HTF: dedupe gates, "flat" = neutral
5. Scalp RR floor 1:1–1.5
6. Asia: default sessions + fix 21-24 UTC hole + revive v1 scalp engine in v2
7. Implement bar_close_only

## Constraints
- STRICT: user is highly protective of codebase. No refactoring/improvements beyond explicitly approved fixes.
- Goal: 10-20 trades/day active sessions, 5-10 Asia, 45-55% WR, ≥1:2 RR (swings).

## Architecture Reference
- server.py (2818 lines): API + scanner `_scan_and_persist` (L2033) + scheduler (L2330) + gates.
- strategy_v2.py: v2 router (BOS retest / squeeze breakout / liquidity reversal), S/R gates, RR floor.
- engine.py: indicators, sessions, v1 engine (incl. dead scalp), calc_lot.
- risk_engine.py: adaptive_lot, volatility gate, slippage, heartbeat.
- marketdata.py: db.candles read/write (bridge-fed).
