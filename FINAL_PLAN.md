# Finalized Plan — Full Custom Auto-Buy Bot (red-team-corrected)

> **Chosen path (your decision, 2026-09-10):** build the full custom, automated,
> real-money CNC stock/ETF investing bot from [`prompt.md`](prompt.md) — *not* the
> stripped-down index-SIP version.
>
> This document keeps that goal, but **bakes in every technical correction** from the
> 5-agent adversarial review (so it actually runs on your hardware and broker), and
> **keeps the safety staging that is already in your own spec** (dry-run for months →
> tiny live). Nothing here is me overriding you — the guardrails below are *your*
> requirements (`prompt.md` §§20–26, 33, 42–45) plus fixes for things that were simply
> broken or impossible as written.
>
> ⚠ Re-verify broker/SEBI items before real money — the framework is new (in force
> since 1 Apr 2026) and still settling.

---

## 1. Ground rules for this build (non-negotiable, from your spec)

1. **Real orders come LAST.** We run `EXECUTION_MODE=DRY_RUN` for **months** before one
   rupee moves (your Stage 1→4 / Phase 5→7). A bug's cost in dry-run is zero.
2. **CNC delivery only. Auto-sell OFF.** EQ/ETF allowlist, hard risk filters, budget
   caps — enforced in code, at the DB, and rejected by default.
3. **Every order is idempotent and audited.** DB-level `UNIQUE(decision_id)` + Kite
   order `tag` + reconciliation + kill switch. A restart/retry physically cannot
   double-buy.
4. **We benchmark against Nifty the whole time** (your §27/§38). The bot must show
   whether it actually beats a simple index after costs — if it doesn't, that's a
   finding, not a failure.

These four are the safety net that makes "full auto-buy" responsible for a first-time
builder. They are not optional.

---

## 2. Red-team corrections that are now BAKED INTO the build

These are not scope choices — they are fixes for things that were wrong, broken, or
impossible as originally written. All are now part of the plan.

| Area | Was going to be… | Corrected to (final) |
|---|---|---|
| **Runtime/host** | Docker (Postgres+FastAPI) + in-process APScheduler | **Bare-metal PostgreSQL + Python venv + systemd services + systemd _timers_.** Heavy scoring job runs as a **one-shot process** with `MemoryMax=350M`. Reason: in-process peak ~1040 MB **OOM-kills** the scheduler on a ~950 MB box. |
| **OOM safety** | (none) | Install **`earlyoom`**; `shared_buffers=64MB`, `work_mem=4MB`, `maintenance_work_mem=32MB`, `max_connections=20`, `effective_cache_size=256MB`; 1 GB swap, `swappiness=10`. |
| **Instance survival** | Assume the free VPS just keeps running | **Upgrade OCI to Pay-As-You-Go** (still ₹0 within Always-Free limits) — idle reclamation hits your **AMD** micro too, and a monthly bot is idle by design. **+ nightly off-box `pg_dump` + Terraform** to rebuild fast. |
| **Prices** | Kite free plan / yfinance | Kite **free plan returns NO prices at all** (can't size a ₹5k order). Use **official NSE Bhavcopy/UDiFF** (via `eod2`/`NseIndiaApi`) for EOD prices, and **LIMIT orders with market-protection**. Consider ₹500/mo Kite data only later if you need intraday LTP. |
| **Fundamentals** | Scrape screener.in | **Scraping violates its ToS** (third-party C-MOTS data). No free programmatic feed exists. Pick one honestly (see §4): manual entry, XBRL parsing of BSE/NSE filings, or a paid feed. |
| **Auth flow** | "One-tap Telegram → firewalled `/callback`" | **Broken** — Kite redirects the *browser*, a firewalled server never gets the token. Expose **one hardened public HTTPS `/callback`** on 443 (independent of the order-placement static IP). Token **dies ~6 AM daily** → login on investment day + **retry so a missed login never silently skips the SIP**. |
| **LLM** | Gemini free primary + Groq fallback | **Cut from v1.** Gemini free **trains on inputs** (unsafe for holdings); Groq isn't a real fallback. Use **template-based decision explanations + rule-based news tags**. Re-add later, off the decision path, routed by data-sensitivity. |
| **Backtest** | Credible historical validation on free data | **Impossible for ₹0** (no point-in-time / survivorship-free data). Treat any backtest as **illustrative only**; rely on the spec's **months-long dry-run / forward paper-trading** as the real validation. |

**Regulatory (verified, ⚠ re-check):** a self-authored **white-box** bot for **personal
use**, well under **10 orders/sec**, needs **no exchange algo registration**. But Zerodha
auto-tags every API order as algo, it must stay personal-use, and **static IP + daily
2FA + kill switch are mandatory**.

---

## 3. Target architecture (corrected)

```text
                    YOU (phone) — occasional 2FA login + funding + reviewing reports
                       │  (browser 2FA on investment day)          ▲ Telegram reports/alerts
                       ▼                                           │
        ┌──────────────────────────── Oracle E2.1.Micro (PAYG, Reserved IP) ───────────────────────────┐
        │  systemd services:                                                                            │
        │    • FastAPI (uvicorn, 1 worker)  ── public HTTPS :443 /callback ONLY (Caddy TLS) ─────────── │
        │    • PostgreSQL 16 (BARE-METAL, tuned 64MB)                                                    │
        │  systemd TIMERS (each = short-lived one-shot process, MemoryMax, earlyoom guarding):           │
        │    • health (15m)  • price/EOD sync from NSE Bhavcopy (daily)  • corp-action scan (daily)      │
        │    • risk review (weekly)  • MONTHLY invest workflow  • nightly pg_dump → OFF-BOX              │
        │  Order path (Phase 6+): validate → risk → duplicate-check(UNIQUE) → EXECUTION_MODE gate →      │
        │                          Kite CNC order (tag) → verify → reconcile → audit_log → Telegram      │
        └───────────────────────────────────────────────┬───────────────────────────────────────────────┘
                     outbound orders from the RESERVED IP │  (whitelisted at Zerodha; 1 change/week)
                                                          ▼
                                                Zerodha Kite Connect (free personal execution API)
```

- **Prices** come from NSE Bhavcopy (free), stored with `source`/`fetched_at`/`data_as_of`.
- **Only :443 `/callback` is public**; SSH is 22; Postgres/app internals never exposed.
- **Backtesting & any future LLM batch run OFF this box** (your laptop / Colab / GH Actions).

---

## 4. The one decision you still owe reality: fundamentals data

Your 5-factor score needs fundamentals (ROE, ROCE, D/E, growth, margins…). There is **no
free, reliable, programmatic source** in 2026. Pick your poison — we design around it:

| Option | Cost | Effort | Quality | Note |
|---|---|---|---|---|
| **A. Manual entry** of a small watchlist's key ratios into the DB, refreshed quarterly | ₹0 | You, ~1–2 hrs/quarter | Good, but tiny universe | Fits a 3–5 stock, low-turnover, buy-and-hold plan surprisingly well. |
| **B. Parse BSE/NSE XBRL filings** yourself | ₹0 | **High** (real project) | Current-only, no history | Big effort for a beginner; brittle. |
| **C. Paid fundamentals API** (or Kite data ₹500/mo for prices + a fundamentals vendor) | ₹500+/mo | Low | Best | Only sensible once capital ≥ ~₹3–5L. |

**My recommendation for your stage: Option A** — manual quarterly entry of a small,
high-quality watchlist. It's ₹0, keeps the engine honest (real numbers you've eyeballed),
matches your 3–5 position target, and defers the paid feed until your capital justifies it.
Tell me A, B, or C and I'll build to it.

---

## 5. Roadmap (your Phases 0–7, corrected)

- **Phase 0 — Infra & safety:** verify box; 1 GB swap + swappiness; **earlyoom**;
  **upgrade to PAYG**; **Reserved** public IP; UFW 22+443; SSH keys only; **Terraform**
  the instance; nightly off-box backup wired.
- **Phase 1 — Data layer (bare-metal):** install Postgres (apt, tuned 64MB); schema
  (superset incl. `orders`, `risk_events`) + `UNIQUE(decision_id)` +
  `source/fetched_at/data_as_of`; migrations; restore-test a backup once.
- **Phase 2 — Data engine:** NSE Bhavcopy EOD prices + corporate actions (free);
  fundamentals per your §4 choice. Validation layer (missing-day / zero-price /
  split-jump detectors — silent bad data is the top failure mode).
- **Phase 3 — Investment engine (deterministic):** fundamental/valuation/growth/
  stability/dividend scores → composite 0–100 + **hard risk-filter vetoes** +
  **sector-aware** thresholds. **Template** explanations. No LLM.
- **Phase 4 — Validation:** an *illustrative* (heavily caveated) backtest **plus** a
  **forward paper-trading harness** and **live Nifty benchmark** (the real test).
- **Phase 5 — Dry run (months):** full monthly workflow at `EXECUTION_MODE=DRY_RUN`;
  hypothetical orders logged; Telegram reports. Audit every decision.
- **Phase 6 — Zerodha integration:** public HTTPS `/callback` login (browser 2FA);
  portfolio/funds sync; `validate_order()` allowlist (CNC + EQ/ETF); **DB duplicate
  protection**; reconciliation; **kill switch**; still `EXECUTION_MODE=DRY_RUN`.
- **Phase 7 — Limited live:** ⚠ re-verify Kite/SEBI; **whitelist the Reserved IP**
  (mind the 1-change/week cap); fund **₹10,000**; flip to live with **`auto_sell=false`**;
  start tiny; watch every order for months.

---

## 6. What stays true no matter what (accept these honestly)
- **No credible fundamental backtest for free** → dry-run/forward paper-trading *is* your
  validation. Don't trust backtest numbers as expected returns.
- **8–10% CAGR is a target, never a guarantee.** Individual years can be negative.
- **A daily 2FA login is unavoidable** on the day you invest — that's the compliant
  design, not a bug.
- **Reliable data eventually costs money** (Option C / ₹500 Kite data); budget for it as
  capital grows, don't pretend it's free forever.
- **The Nifty benchmark may win.** We keep it in view the whole time so you'll *know*.

---

## 7. Immediate next step
Per your own spec (§46: "do NOT jump directly into coding"), I'll start with **Phase 0
(infra)** the moment you're ready — and I'll drive it through the beginner
[`STEP_BY_STEP_GUIDE.md`](STEP_BY_STEP_GUIDE.md), which I'll update to this corrected
bare-metal + PAYG approach.

**To start, I need three things from you:**
1. **Windows or Mac?** (so I tailor the exact terminal steps)
2. **Fundamentals data — Option A, B, or C?** (§4; I recommend **A**)
3. Confirm you're OK **upgrading the Oracle account to Pay-As-You-Go** (stays ₹0 within
   limits; without it Oracle may delete your idle server).
</content>
