# Suggested Implementation Plan — Free-Tier Edition

> A planning layer on top of [`prompt.md`](prompt.md) and [`README.md`](README.md).
> Nothing here overrides your non-negotiables (§45 of `prompt.md`). It adds a
> concrete, **₹0-recurring-cost** path, resolves two contradictions hidden in the
> spec, and recommends a specific free-tier AI model.
>
> Researched: 2026-09-10. Broker/API rules change — re-verify the items marked **⚠ VERIFY** before live execution.

---

## 0. TL;DR — the three things that actually matter

1. **You can run almost the entire platform for ₹0/month.** Zerodha now makes the
   **order-execution + account-read APIs free for personal use**. You only pay
   Zerodha (₹500/mo) if you want *their* live/historical market-data feed — and you
   don't need it, because free data sources cover prices and fundamentals.
2. **For the AI (LLM) part, use Google Gemini Flash's free tier as primary,
   Groq as fallback.** Your 1 GB VPS *cannot* self-host an LLM (per your own §5),
   so a free *hosted* API is the correct answer. Your design only calls the LLM
   occasionally (quarterly/annual summaries, news) and never for math or the final
   buy/sell — so free daily caps (~1,500 req/day) are far more than enough.
3. **The real blocker to "zero-touch" isn't cost — it's Zerodha's daily token.**
   The Kite access token **expires every morning (~6 AM)** and re-issuing it needs
   a 2FA/TOTP login. Fully scripting that = storing your password + TOTP seed =
   exactly what your own §7 forbids. The clean, compliant fix is below (§4): for a
   *monthly* bot you only need to authenticate **once, on investment day**, via a
   one-tap Telegram link. That is the "occasional authentication" you already
   accepted in §44.

---

## 1. The free-tier stack (target: ₹0/month recurring)

| Layer | Choice | Cost | Notes |
|---|---|---|---|
| VPS | Oracle `VM.Standard.E2.1.Micro` (you have it) | **Free** (Always Free) | 1 OCPU / 1 GB RAM. x86. |
| Static/Reserved IP | Oracle **Reserved Public IP** | **Free** in Always Free tier | Required for order placement from 1 Apr 2026 (§4). Convert ephemeral → reserved now. |
| DB | PostgreSQL 16-alpine (Docker) | **Free** | Tuned for 1 GB (`shared_buffers=128MB`). |
| Backend | Python 3.12 + FastAPI + SQLAlchemy | **Free** | |
| Scheduler | APScheduler (in-process) or host cron | **Free** | No n8n on this box (your §5). |
| **Broker execution + account reads** | **Kite Connect "Personal" (free)** | **Free** | Order placement, holdings, positions, orders, funds. ⚠ VERIFY at signup. |
| Market prices (EOD) | `yfinance` (`.NS`/`.BO`) and/or `jugaad-data` | **Free** | Daily/EOD only — matches your daily schedule. |
| Fundamentals | `screener.in` (scrape/cache) + AnnualReport/BSE filings | **Free** | Cache hard in Postgres; scraping is fragile. |
| Corporate actions | NSE/BSE announcements + screener | **Free** | |
| **AI / LLM** | **Google Gemini Flash free tier** (primary), **Groq** (fallback) | **Free** | See §3. |
| Notifications | Telegram Bot API | **Free** | |
| Backtesting / heavy research | **Off the VPS** (your laptop / Google Colab / GitHub Actions) | **Free** | See §5. Keeps the 1 GB box lean. |
| TLS / reverse proxy (only if dashboard exposed) | Caddy (auto-HTTPS) | **Free** | Build dashboard last. |

**Optional paid add-on (skip for now):** Kite Connect **data plan ₹500/month** adds
Zerodha's own live-streaming + historical candle data (historical is bundled into
this since Feb 2025, no longer a separate charge). Only buy it if free data sources
prove insufficient for your universe. **⚠ VERIFY** current price at signup — public
sources show both an older ₹2,000 and a revised ₹500 figure.

> Net: the platform's steady-state cost is **₹0/month** on the free path. The only
> thing you *must* have (a static IP) Oracle gives you for free.

---

## 2. Where I'd change / sharpen your spec (suggestions "on top")

These are additions, not contradictions of your requirements.

1. **Split the broker layer into "free execution + free reads" vs "paid data."**
   Your docs assume you'll take Zerodha's data feed. You don't have to. Use Kite
   only for what's uniquely Zerodha's (placing CNC orders, reading *your* holdings/
   funds) and get prices/fundamentals from free sources. This is the single biggest
   cost lever and it's fully compliant.
2. **Move backtesting and any LLM-batch work off the 1 GB VPS.** pandas backtests
   over 5–10 years of data will OOM a 1 GB box. Your own §5/§46 already says "move
   research/backtesting/LLM to another machine" — make that concrete: run backtests
   on your laptop or a **free Google Colab / GitHub Actions** run, commit results,
   and keep the VPS strictly for *execution + storage + scheduling*.
3. **Treat the daily access-token as a first-class design constraint, not a
   footnote.** (§4 below.) Neither doc addresses that the token dies daily; it's the
   thing most likely to break "zero-touch" in practice.
4. **Idempotency should be enforced by the database, not just app logic.** Add a
   `UNIQUE` constraint on a `decision_id` (and pass Kite's `tag` on each order) so a
   restart/retry physically cannot double-submit. Your §22 asks for this — make it a
   DB constraint + Kite order `tag`, not just a code check.
5. **Every external data row needs `source`, `fetched_at`, `data_as_of`.** Your §35
   wants this; bake it into the schema from day one so backtests can honor
   point-in-time availability (no look-ahead bias, §26).
6. **One kill switch, checked in exactly one place.** `EXECUTION_MODE` +
   `AUTOMATION_STATUS` read from DB at the *single* choke point right before any
   `place_order`. Everything upstream can run freely; only that gate submits.
7. **Start the schema with `orders` and `risk_events` tables too** (your `prompt.md`
   §10 lists them; `README.md` §10 omits them). Use the `prompt.md` list — it's the
   superset.

---

## 3. The AI model decision (your explicit ask)

**Recommendation: Google Gemini Flash (free tier) as the default provider, with an
abstraction layer so Groq / OpenRouter / Mistral are drop-in fallbacks.**

Why this is the right call for *your* system specifically:

- **You can't self-host.** A 1 GB / 1 OCPU box cannot run Ollama / a local LLM — you
  said so in §5. So "free tier" here means a **free *hosted* API**, not a local model.
- **Your LLM workload is tiny and bursty, not a hot loop.** Per your §36, the LLM only
  does *qualitative* work — summarize an annual report, digest an earnings call,
  classify news, explain a decision in plain English. That's a handful of calls per
  week, spiking at quarterly results. Free daily caps dwarf that.
- **The free model never touches money-critical logic.** Your §35–§36 forbid the LLM
  from doing financial math or issuing the final BUY/SELL. The deterministic
  scoring/risk engine decides; the LLM only writes prose. So a free model's
  occasional imperfections can't cause a bad trade — which removes the usual reason
  to pay for a premium model.

Free-tier options compared (as of 2026 — limits shift, **⚠ VERIFY** at signup):

| Provider | Free limits (approx) | Best for | Watch-outs |
|---|---|---|---|
| **Google Gemini Flash** ⭐ primary | ~1,500 requests/day, ~15 RPM, ~250k TPM, no card | Long documents (big context), structured JSON output | Google trimmed free limits in late 2025; daily reset is midnight PT |
| **Groq** (Llama 3.3 70B etc.) ⭐ fallback | ~30 RPM, ~1,000 req/day | Very fast responses | Lower daily cap; model availability changes |
| **OpenRouter** (free models) | ~20 RPM, ~50 free req/day (until $10 credit) | One API, many models | Low daily cap on pure-free |
| **Mistral** (free mode) | Free, no card | EU option, decent quality | Rate limits vary |
| Cohere | ~1,000 calls/month | — | **Non-commercial only** (fine for personal use, but noted) |

**Design point:** put a thin `LLMClient` interface in `services/` with one method
(`summarize`, `classify`) and select the provider by env var
(`LLM_PROVIDER=gemini`). If Gemini rate-limits you on a heavy quarterly day, the
job falls back to Groq automatically. This keeps you free *and* resilient, and lets
you swap providers without touching the engines.

> Reminder from your own spec, worth repeating: **never let the LLM invent a number.**
> Feed it only already-fetched, DB-stored figures; it reasons/summarizes, the
> deterministic engine computes and decides.

---

## 4. The daily-token problem and the compliant zero-touch pattern

**Fact:** A Kite Connect `access_token` is valid for **one trading day** and expires
around **6 AM** the next morning. Re-issuing it requires the official login flow with
**2FA/TOTP**. Scripting that end-to-end (Selenium + `pyotp`) means storing your
password and TOTP seed on the server and automating a browser login — which is
**exactly what `prompt.md` §7 and `README.md` §17 forbid**, and it's real account risk.

**Reconcile it like this (fits your §3/§44 "occasional authentication"):**

- Your bot is a **monthly** investor, plus light daily read-only sync. You do **not**
  need a fresh token every single day for the core job — you need one **on the day
  the monthly order runs.**
- On investment day, the scheduler sends a **Telegram message with the official
  Kite login link**. You tap it once, complete Zerodha's real 2FA/TOTP login, and the
  redirect lands a `request_token` on your FastAPI `/callback` endpoint. The server
  exchanges it for the day's `access_token` (SHA-256 checksum of
  `api_key + request_token + api_secret`) and proceeds fully automatically from there:
  score → risk → allocate → place CNC orders → reconcile → report.
- On non-investment days, if the token is stale, portfolio "sync" simply uses the last
  known holdings snapshot (or is skipped) rather than forcing a login. Read-only
  freshness is not worth compromising your security posture.
- On expiry mid-run: detect the `TokenException`, **stop order submission**, fire the
  "authentication required" Telegram alert (your §24/§29), resume after you re-auth.

That's ~**one tap, once a month** (plus any ad-hoc rebalance day) — genuinely
zero-touch for routine operation, with **no stored password and no bypassed 2FA.**

> If you later decide you want unattended daily token refresh, that's a conscious
> security trade-off to make with eyes open — it is out of scope for the compliant
> free-tier build described here, and I'd advise against it for a real-money account.

---

## 5. Compute placement (respecting 1 GB RAM)

```text
┌───────────────────────── Oracle Micro VPS (1 GB) — keep LEAN ─────────────────────────┐
│  Docker: PostgreSQL (mem_limit 300m) + FastAPI (mem_limit 350m) + APScheduler          │
│  Jobs:  health (15m) · portfolio sync (daily) · price/EOD update (daily) ·             │
│         corp-action scan (daily) · risk review (weekly) · MONTHLY invest workflow      │
│  Broker: Kite execution + reads (free) from the RESERVED IP                            │
│  Out:   Telegram alerts                                                                 │
└────────────────────────────────────────────────────────────────────────────────────────┘
        ▲ commit results / configs                     │ occasional 1-tap auth (Telegram)
        │                                               ▼
┌──────────────── Off-box, FREE (laptop / Colab / GitHub Actions) ───────────────┐
│  Backtesting (5–10y, pandas) · LLM batch summaries · large research sweeps       │
│  Produces: validated params, scorecards, reports  →  committed to repo / DB      │
└──────────────────────────────────────────────────────────────────────────────────┘
```

Rule of thumb: if a job needs pandas over years of data, or many LLM calls in a
batch, it runs **off** the VPS. The VPS does small, scheduled, deterministic work.

---

## 6. Data-source strategy (free, with the caveats)

| Need | Free source | Reliability | Mitigation |
|---|---|---|---|
| Your holdings, positions, funds, order status | **Kite (free personal API)** | Authoritative | This is the source of truth for *your account*. |
| EOD prices, adjusted history | `yfinance` (`SYMBOL.NS`), `jugaad-data` | Good for EOD; unofficial | Cache in `prices` table; daily job; retry/backoff. |
| Fundamentals (P&L, ratios, ROE/ROCE, D/E) | `screener.in` (scrape) | Good coverage; **scraping is fragile & ToS-sensitive** | Cache hard; refresh weekly/quarterly only; store `source`/`data_as_of`. |
| Corporate actions (splits, bonus, dividends) | NSE/BSE announcements, screener | Decent | Cross-check before acting; never treat a split as value (your §16). |
| News (for LLM classification) | Public RSS / reputable sources | Varies | LLM classifies; humans/engine decide. |

Caveats you should accept up front:
- Free/unofficial data can break or lag. That's fine for a **monthly** buy-and-hold
  system with daily EOD needs — it is **not** fine for intraday (which you've excluded
  anyway). Build every fetch with cache-first + graceful degradation.
- If free fundamentals ever become the weak link, the ₹500/mo Kite data plan (or a
  paid fundamentals API) is the escalation — a deliberate, later choice, not day one.

---

## 7. Recommended build order (maps to your Phases 0–7, with the free-tier picks)

**Phase 0 — Infra & security (VPS)**
- `uname -m`, `free -h`, `df -h`, `curl -4 ifconfig.me`; create 1 GB swap, `swappiness=10`.
- **Convert the public IP to Reserved** in OCI (free) — needed for order placement.
- UFW (22, 443 only) + OCI security list; SSH keys only, disable password login.
- Install Docker + Compose.

**Phase 1 — Foundation**
- Docker Compose: Postgres (tuned) + FastAPI. `.env` (git-ignored) + `settings.yaml`.
- Schema from `prompt.md` §10 (superset incl. `orders`, `risk_events`) + `source`/
  `fetched_at`/`data_as_of` columns + `UNIQUE(decision_id)`.
- `/health` endpoint; automated `pg_dump` backup script + off-VPS copy.

**Phase 2 — Data engine (free sources)**
- Company universe (NSE list), `yfinance`/`jugaad-data` prices, `screener.in`
  fundamentals, corporate actions. Everything cached; daily/weekly jobs.

**Phase 3 — Investment intelligence (deterministic)**
- Fundamental (30) · Valuation (25) · Growth (20) · Stability (15) · Dividend (10)
  → composite 0–100 with your decision bands. **Hard risk filters** that can veto any
  score (governance, extreme debt, pledging, illiquidity). Sector-aware thresholds
  (your §12 point about banks vs manufacturers).

**Phase 4 — Backtesting (OFF the VPS, free)**
- Point-in-time data only; CAGR/XIRR/maxDD/Sharpe/Sortino/turnover/costs vs Nifty 50
  & Nifty 500. Test bull/bear/sideways/crash regimes. **Passive-vs-active verdict**
  as a first-class output (your §27/§38).

**Phase 5 — AI layer (free LLM)**
- `LLMClient` abstraction; `LLM_PROVIDER=gemini` default, Groq fallback. Summaries,
  news classification, decision explanations. Never math, never final BUY/SELL.

**Phase 6 — Dry run (months, no real money)**
- Full monthly workflow in `EXECUTION_MODE=DRY_RUN`: generate & store hypothetical
  orders, Telegram the report. Run for several months; audit every decision.

**Phase 7 — Zerodha integration → limited production**
- Free personal execution API; the §4 one-tap auth pattern; portfolio reconciliation;
  order `validate_order()` allowlist (CNC + EQ/ETF only); DB-level duplicate
  protection; **⚠ VERIFY** static-IP whitelisting + current Kite requirements; then
  live with ₹10k + ₹5k/month, `auto_sell=false`.

---

## 8. Open decisions for you (small, but I'd pin them now)

1. **Primary free LLM:** go with **Gemini Flash** as I recommend, or prefer Groq
   (faster, lower daily cap) / self-host later on a bigger box? (Default: Gemini.)
2. **Take the ₹500/mo Kite data plan, or stay fully free on data?** (Default: stay
   free with `yfinance`/`jugaad-data`/`screener.in`; escalate only if data quality
   bites.)
3. **Daily token stance:** accept the compliant **once-a-month one-tap auth** (my
   recommendation), or do you want to discuss unattended refresh (I'd advise against)?
4. **Dashboard now or later?** Your spec says later — I agree; ship the engine + dry
   run first, add a read-only FastAPI/HTMX page before React (lighter on 1 GB).

---

## 9. Honest risk register

- **Data fragility:** free scrapers break; ToS on scraping `screener.in` is a real
  consideration. Cache-first design + willingness to fall back to a paid feed later.
- **Free LLM limits change:** providers cut free tiers with little notice (Gemini did
  in late 2025). The multi-provider abstraction is your insurance.
- **Broker rules change:** static-IP mandate (1 Apr 2026), fees, and auth flow can
  shift — **⚠ VERIFY** against official Kite docs before every go-live step.
- **1 GB ceiling:** avoid feature creep on the VPS; anything heavy goes off-box.
- **No guaranteed returns:** the 8–10% CAGR is a target, not a promise — your §1 and
  the passive-vs-active check keep the system honest about whether stock-picking is
  even earning its keep after costs.

---

## Sources

- [Free LLM APIs compared 2026 — OpenRouter](https://openrouter.ai/blog/tutorials/free-llm-apis-compared/)
- [Gemini API free-tier limits 2026 — TokenMix](https://tokenmix.ai/blog/gemini-api-free-tier-limits)
- [Zerodha: Free personal APIs from Kite Connect — Z-Connect](https://zerodha.com/z-connect/updates/free-personal-apis-from-kite-connect)
- [Zerodha makes trading API free for personal use; bundles historical data — Marketcalls](https://www.marketcalls.in/fintech/zerodha-makes-trading-api-free-for-personal-use-bundles-historical-data-with-connect-api.html)
- [Revising Kite Connect fees ₹2000 → ₹500/month — Kite forum](https://kite.trade/forum/discussion/15015/revising-kite-connect-fees-from-2000-to-500-per-month)
- [Zerodha: static IP requirement (effective 1 Apr 2026)](https://support.zerodha.com/category/trading-and-markets/general-kite/kite-api/articles/static-ip)
- [Kite Connect user/session docs (access token daily expiry)](https://kite.trade/docs/connect/v3/user/)
- [Oracle Cloud Always Free resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)
- [jugaad-data (free NSE historical data)](https://github.com/jugaad-py/jugaad-data)
- [Screener.in (Indian fundamentals)](https://www.screener.in/)
</content>
</invoke>
