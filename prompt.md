# Build a Fully Automated Long-Term Indian Equity Investment Platform

I want to build a **fully automated, low-frequency, long-term investment platform for my Zerodha account**.

The purpose is **wealth creation and compounding**, not trading.

The platform should automate research, portfolio analysis, investment allocation, monitoring, and—where officially supported and compliant—delivery-based equity order execution through Zerodha.

---

# 1. My Investment Profile

## Capital

* Initial capital available now: **₹10,000**
* Monthly investment initially: **₹5,000**
* Increase the monthly contribution gradually as my income and confidence in the system increase.
* Investment horizon: **minimum 5 years, preferably 10+ years**

## Return Objective

I would like to target approximately **8–10%+ long-term CAGR**, with unlimited upside.

However:

* Do NOT guarantee an 8–10% minimum return.
* Clearly explain that equity returns are uncertain.
* Accept that individual years can produce negative returns.
* Optimize for long-term risk-adjusted returns rather than a guaranteed annual return.
* Compare performance against appropriate benchmarks.

The system's objective is:

> **Consistent long-term compounding with controlled risk, low turnover, and minimal human intervention.**

---

# 2. Absolutely No Trading

This is an **investment system**, not a trading bot.

DO NOT use:

* Options
* Futures
* Intraday trading
* MIS
* NRML
* Margin
* Leverage
* Short selling
* High-frequency trading
* Scalping
* Momentum trading as the primary strategy
* Speculative penny stocks
* Borrowed money

Only consider:

* Delivery-based equities
* Suitable ETFs
* Other long-term instruments explicitly permitted by the strategy and Zerodha

The system should be optimized for **buy-and-hold investing**.

---

# 3. Zero-Touch / Minimal Human Intervention Requirement

A major requirement is:

> **I do not want to repeatedly log in to Zerodha whenever the system needs to buy, sell, rebalance, or monitor my portfolio.**

The desired experience is:

```text
Initial Setup
      ↓
Official Zerodha Authentication
      ↓
Configure Investment Rules
      ↓
Automation Starts
      ↓
Research
      ↓
Portfolio Analysis
      ↓
Investment Decision
      ↓
Risk Validation
      ↓
Zerodha API
      ↓
Delivery Order
      ↓
Portfolio Reconciliation
      ↓
Notification
```

I should NOT have to manually:

* Search for stocks
* Calculate quantities
* Place routine monthly orders
* Monitor every order
* Rebalance manually
* Log in repeatedly just to check the portfolio

However, **do not bypass Zerodha authentication, 2FA, TOTP, security controls, or regulatory requirements**.

If Zerodha requires periodic human authentication, the system should:

1. Detect session expiration.
2. Stop order execution safely.
3. Notify me.
4. Provide the official authentication path.
5. Resume automation after valid authorization.

The objective is:

> **No routine human intervention, while remaining completely compliant with Zerodha, NSE and SEBI requirements.**

---

# 4. My Existing Infrastructure

I already have an **Oracle Cloud Free Tier VPS** and want to use it instead of purchasing another VPS.

## Current Oracle VPS

```text
Region:              ap-hyderabad-1
Availability Domain: AD-1
Fault Domain:        FD-1

Public IP:
68.233.112.199

Username:
ubuntu

Operating System:
Canonical Ubuntu 22.04

Shape:
VM.Standard.E2.1.Micro

CPU:
1 OCPU

Memory:
1 GB RAM

Network:
0.48 Gbps

Storage:
Block storage

Launch Mode:
PARAVIRTUALIZED

In-transit Encryption:
Enabled
```

The VPS was launched on:

```text
22 June 2025
```

Do not unnecessarily expose or store sensitive OCI identifiers such as the full instance OCID in application source code or public repositories.

---

# 5. Important VPS Constraint

The Oracle VPS has only:

* 1 OCPU
* 1 GB RAM

Therefore, **do not design an unnecessarily heavy production stack**.

The production architecture should initially use:

```text
Oracle VPS
│
├── Docker
├── PostgreSQL
├── FastAPI / Python
├── Lightweight Scheduler
├── Zerodha Execution Service
└── Telegram Notifications
```

Avoid initially running:

* Ollama
* Local LLMs
* Open WebUI
* Elasticsearch
* Kafka
* Kubernetes
* Redis unless genuinely necessary
* Celery
* Multiple AI agents running continuously
* Heavy n8n workflows

Use **cron or APScheduler** for scheduled jobs initially.

If more compute is required later, move research/backtesting/LLM workloads to another machine or cloud service.

---

# 6. Oracle VPS Networking

My home internet connection does **not** have a static IP.

That is acceptable.

The production order execution should originate from the Oracle VPS.

First verify:

```bash
curl -4 ifconfig.me
```

and confirm the returned address.

The currently known public IP is:

```text
68.233.112.199
```

Determine whether the Oracle public IP is:

* Reserved
* Ephemeral

For production API execution, use a stable/reserved public IP where required.

Before live execution:

1. Verify the current Zerodha Kite Connect static-IP requirements.
2. Verify Oracle's IP assignment.
3. Confirm the VPS outbound IP.
4. Configure the permitted/whitelisted IP according to the current Zerodha process.
5. Test API connectivity.
6. Run dry-run execution.
7. Only then enable real-money execution.

Do not assume current broker requirements remain unchanged; verify them against official documentation before deployment.

---

# 7. Security Requirements

Never:

* Store my Zerodha password
* Store plaintext credentials unnecessarily
* Automate browser login
* Scrape Zerodha's website for authentication
* Bypass 2FA
* Circumvent broker security
* Attempt to create an indefinitely authenticated session by hacking around expiration

Use the **official Zerodha Kite Connect API and authentication mechanism**.

Secrets must be stored using environment variables or a secure secret-management approach.

Never commit:

```text
.env
API keys
API secrets
access tokens
database passwords
SSH private keys
```

to Git.

---

# 8. Production Technology Stack

Use:

### Infrastructure

* Oracle Cloud Free Tier
* Ubuntu 22.04
* Docker
* Docker Compose

### Backend

* Python
* FastAPI
* SQLAlchemy
* PostgreSQL

### Scheduling

Initially:

* Cron
* APScheduler

Potential future:

* n8n on another machine/server if required

### Notifications

Initially:

* Telegram

Potential future:

* Email
* Mobile notifications

### Frontend

Only build a dashboard after the core investment engine works.

Potential:

* React
* FastAPI backend

---

# 9. Recommended Directory Structure

Create:

```text
investment-platform/
│
├── docker-compose.yml
├── .env
├── .gitignore
├── README.md
│
├── backend/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app/
│       ├── main.py
│       ├── config.py
│       │
│       ├── api/
│       ├── database/
│       ├── models/
│       ├── services/
│       │
│       ├── engines/
│       │   ├── fundamental.py
│       │   ├── valuation.py
│       │   ├── dividend.py
│       │   ├── portfolio.py
│       │   └── risk.py
│       │
│       └── brokers/
│           └── zerodha.py
│
├── database/
│   └── init.sql
│
├── scripts/
│   ├── backup.sh
│   └── healthcheck.sh
│
├── data/
├── reports/
└── backups/
```

---

# 10. Database

Use PostgreSQL.

Store:

* Companies
* Historical financial data
* Fundamental metrics
* Valuation metrics
* Portfolio holdings
* Transactions
* Orders
* Dividends
* Corporate actions
* Investment decisions
* Risk alerts
* Audit logs
* Benchmark performance

Core tables:

```text
companies
portfolio_holdings
transactions
orders
investment_decisions
fundamentals
valuations
dividends
corporate_actions
risk_events
audit_logs
```

Every automated decision must be auditable.

For each investment decision store:

```text
Timestamp
Company
Score
Decision
Allocation
Reason
Fundamental metrics
Valuation metrics
Risk checks
Execution mode
Order ID
Final result
```

---

# 11. Investment Universe

Focus on Indian listed companies.

Primary universe:

* NSE
* BSE where appropriate
* Large-cap
* High-quality mid-cap
* Broad-market ETFs
* Dividend-paying quality companies

Avoid:

* Penny stocks
* Illiquid stocks
* Companies with serious governance concerns
* Excessive debt
* Unsustainable businesses
* Pump-and-dump candidates

---

# 12. Fundamental Analysis

Evaluate:

* Revenue growth
* Profit growth
* EPS growth
* ROE
* ROCE
* Debt/equity
* Interest coverage
* Free cash flow
* Operating margin
* Net margin
* Book-value growth
* Cash-flow consistency

Example preference:

```text
ROE > 15%
ROCE > 15%
Positive FCF
Sustainable revenue growth
Sustainable profit growth
Manageable debt
```

These must NOT be treated as rigid universal rules.

The system must account for sector differences.

For example, acceptable leverage for a bank is fundamentally different from acceptable leverage for a manufacturing company.

---

# 13. Valuation Engine

Evaluate:

* P/E
* Forward P/E where reliable
* PEG
* P/B
* EV/EBITDA
* Historical valuation
* Sector valuation
* Earnings growth relative to valuation

Classify each stock:

```text
Undervalued
Fairly Valued
Slightly Expensive
Overvalued
```

Do not buy an excellent company blindly at any price.

Preferred hierarchy:

```text
High quality + undervalued
        ↓
High quality + fairly valued
        ↓
High quality + reasonable correction
```

---

# 14. Investment Score

Build an explainable score from 0–100.

Initial framework:

```text
Fundamental Strength     30%
Valuation                25%
Growth Potential         20%
Financial Stability      15%
Dividend Quality         10%
```

Score interpretation:

```text
85–100   Strong Buy
75–84    Buy
65–74    Accumulate
50–64    Watch
<50      Avoid
```

However, the score must NOT override hard risk filters.

A stock with a high score must still be rejected if it has severe:

* Governance issues
* Accounting red flags
* Excessive debt
* Illiquidity
* Other material risks

---

# 15. Dividend / Passive Income Strategy

I want the system to consider passive income.

Track:

* Dividend yield
* Dividend growth
* Dividend history
* Payout ratio
* Free-cash-flow coverage
* Dividend consistency

Do NOT simply select the highest dividend yield.

Detect dividend traps.

A company with:

```text
Very high dividend yield
+
Falling share price
+
Weak cash flow
+
Deteriorating business
```

should NOT automatically receive a high dividend score.

Create a:

```text
Dividend Quality Score
```

---

# 16. Stock Splits and Corporate Actions

Track:

* Stock splits
* Bonus shares
* Dividends
* Rights issues
* Buybacks
* Mergers
* Demergers

Important:

> A stock split itself does not create economic value.

Never use a stock split as a standalone buy signal.

Instead evaluate whether the company also demonstrates:

* Earnings growth
* Revenue growth
* Improving cash flow
* Business expansion
* Strong balance sheet
* Sustainable competitive advantage

---

# 17. Portfolio Construction

With ₹10,000 starting capital, avoid excessive diversification.

Initial target:

```text
3–5 positions maximum
```

If individual stock prices make diversification inefficient, consider suitable broad-market ETFs.

Possible conceptual allocation:

```text
40% Core index / ETF
30% Quality growth
20% Attractive valuation opportunities
10% Reserve
```

These are configurable starting assumptions, not guaranteed optimal allocations.

As the portfolio grows:

```text
₹10k–₹50k       3–5 holdings
₹50k–₹2L        5–8 holdings
₹2L–₹5L         8–12 holdings
₹5L+            10–15 holdings
```

Do not mechanically force these numbers if portfolio risk would increase.

---

# 18. Monthly ₹5,000 Investment Automation

Every month:

```text
₹5,000 Budget
      ↓
Fetch Portfolio
      ↓
Update Financial Data
      ↓
Score Existing Holdings
      ↓
Scan Investment Universe
      ↓
Identify Opportunities
      ↓
Calculate Portfolio Allocation
      ↓
Risk Validation
      ↓
Generate Orders
      ↓
Check Execution Mode
      ↓
DRY RUN / LIVE
      ↓
Execute CNC Orders
      ↓
Verify Order Status
      ↓
Reconcile Portfolio
      ↓
Send Report
```

Do not automatically invest the same ₹5,000 equally every month.

Allocation should depend on:

* Existing holdings
* Valuation
* Portfolio concentration
* Market conditions
* Investment score
* Available opportunities

---

# 19. Market Correction Strategy

Do not panic sell.

Use correction levels as opportunities for research.

Example:

```text
0–10% market correction
→ Normal monthly investment

10–15%
→ Consider increasing allocation if quality opportunities exist

15–25%
→ Deploy part of reserve

25%+
→ Aggressively evaluate high-quality companies
```

Do not automatically buy merely because the market has fallen.

Distinguish:

```text
Market-wide correction
vs
Sector-specific problem
vs
Company-specific fundamental deterioration
```

---

# 20. Automated Sell Strategy

Selling should be more restrictive than buying.

Potential sell conditions:

* Investment thesis invalidated
* Significant fundamental deterioration
* Unsustainable debt
* Serious governance concern
* Structural business decline
* Excessive portfolio concentration
* Extreme valuation combined with deteriorating fundamentals

Do NOT sell solely because:

* Stock fell 5%
* Stock fell 10%
* Market crashed
* Technical indicator turned bearish
* Short-term news was negative

Initially:

```text
auto_sell = false
```

Generate sell recommendations and notifications first.

Only enable automated selling after extensive backtesting and validation.

---

# 21. Zerodha Order Safety

The execution engine must allow only approved order types.

Conceptually:

```text
Allowed:
CNC / Delivery
Equity
Approved ETFs
```

Blocked:

```text
MIS
NRML
F&O
Options
Futures
Margin
Leverage
Short selling
```

Before every order:

```text
Budget Check
      ↓
Instrument Check
      ↓
Product Check
      ↓
Position Limit Check
      ↓
Sector Limit Check
      ↓
Risk Check
      ↓
Duplicate Check
      ↓
Execution
```

---

# 22. Duplicate Order Protection

This is mandatory.

If:

* API times out
* VPS restarts
* Network fails
* Workflow retries
* Container restarts

the system must NOT accidentally submit the same order twice.

Use:

* Unique decision ID
* Unique execution ID
* Order reconciliation
* Broker order-status verification
* Retry limits
* Database transaction locking where appropriate

---

# 23. Failure Handling

If Zerodha API is unavailable:

```text
Detect Failure
      ↓
Do NOT blindly retry
      ↓
Check whether order was actually accepted
      ↓
Reconcile broker state
      ↓
Retry only if safe
      ↓
Otherwise stop
      ↓
Notify user
```

Never create duplicate orders because of an uncertain API response.

---

# 24. Authentication Failure

If authentication expires:

```text
Detect Expiration
      ↓
Disable Live Execution
      ↓
Continue Research
      ↓
Notify User
      ↓
User completes official authentication
      ↓
Validate session
      ↓
Resume execution
```

No authentication bypass is permitted.

---

# 25. Emergency Kill Switch

Implement:

```text
ACTIVE
PAUSED
DRY_RUN
EMERGENCY_STOP
```

### ACTIVE

Validated transactions may execute.

### PAUSED

No new transactions.

Research and monitoring continue.

### DRY_RUN

Generate hypothetical orders only.

### EMERGENCY_STOP

Immediately block:

* Buy
* Sell
* Rebalance

Portfolio holdings remain untouched.

---

# 26. Backtesting

Before live money:

Test the complete strategy against at least **5–10 years of historical data**, where reliable data is available.

Measure:

```text
CAGR
XIRR
Maximum Drawdown
Sharpe Ratio
Sortino Ratio
Volatility
Portfolio Turnover
Transaction Costs
Benchmark Performance
```

Test:

* Bull markets
* Bear markets
* Sideways markets
* Major corrections
* High-interest-rate environments

Avoid:

* Look-ahead bias
* Survivorship bias
* Data leakage
* Using future financial information

The backtest must simulate information availability as it existed at the time.

---

# 27. Benchmarking

Compare the portfolio against suitable benchmarks, such as:

* Nifty 50
* Nifty 500
* Relevant ETF/index

The system should answer:

> "Did this automated stock-selection strategy actually outperform simply investing in a broad-market index after costs?"

If it does not, the system should be capable of shifting more capital toward a passive index/ETF approach.

---

# 28. Performance Tracking

Dashboard should show:

## Portfolio

```text
Total Invested
Current Value
Absolute Gain/Loss
CAGR
XIRR
Dividend Income
```

## Individual Holdings

```text
Company
Quantity
Average Price
Current Price
Allocation %
Profit/Loss
Fundamental Score
Valuation Score
Dividend Score
```

## Automation

```text
Execution Mode
Last Run
Next Run
Zerodha Authentication Status
API Health
Server Health
```

---

# 29. Notifications

Use Telegram initially.

### Monthly investment

```text
Monthly Investment Completed

Budget: ₹5,000
Invested: ₹X,XXX
Cash Reserve: ₹XXX

Orders:
Company A: ₹X
Company B: ₹X
ETF: ₹X

Portfolio XIRR: X.XX%
```

### Fundamental warning

```text
PORTFOLIO ALERT

Company: XYZ

Score:
82 → 61

Reason:
Debt increased and free cash flow deteriorated.

Action:
Under review.

No automatic sale executed.
```

### Authentication

```text
ZERODHA AUTHENTICATION REQUIRED

Automated execution has been paused.

Complete the official authentication process.
```

---

# 30. Automation Schedule

Use lightweight scheduled jobs.

```text
Every 15 minutes:
    Server health check

Daily:
    Portfolio synchronization
    Corporate action monitoring

Weekly:
    Portfolio risk analysis
    Fundamental monitoring

Monthly:
    ₹5,000 investment workflow

Quarterly:
    Full fundamental review

Annually:
    Performance and strategy review
```

Do not run heavy computation continuously.

---

# 31. Current VPS Memory Strategy

Because the Oracle VPS has only 1 GB RAM:

Configure approximately:

```text
PostgreSQL:
128 MB shared_buffers

max_connections:
20

work_mem:
4 MB

maintenance_work_mem:
32 MB
```

Use a **1 GB swap file** as an emergency memory buffer.

Check:

```bash
free -h
swapon --show
```

Create swap if required:

```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Persist:

```bash
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
```

Set conservative swappiness:

```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-investment-platform.conf
sudo sysctl --system
```

---

# 32. Docker Architecture for Current VPS

Initially use only:

```text
PostgreSQL
FastAPI
```

and run the scheduler inside FastAPI or through host cron.

Example:

```yaml
services:

  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      - postgres_data:/var/lib/postgresql/data
    mem_limit: 300m

  backend:
    build: ./backend
    restart: unless-stopped
    env_file:
      - .env
    depends_on:
      - postgres
    mem_limit: 350m

volumes:
  postgres_data:
```

Do not expose PostgreSQL publicly.

Do not expose FastAPI directly to the Internet.

Use a reverse proxy later if a dashboard is exposed.

---

# 33. Oracle Firewall

Secure both:

* OCI VCN security rules
* Ubuntu UFW

Initially allow:

```text
SSH: 22
HTTPS: 443
```

Only expose HTTP 80 if needed for certificate setup.

Do NOT expose:

```text
5432 PostgreSQL
8000 FastAPI
5678 n8n
```

publicly.

---

# 34. SSH Hardening

Use SSH keys.

Disable password login after confirming key access works.

Example:

```text
PasswordAuthentication no
PermitRootLogin no
```

Test SSH access in a separate session before closing the current session.

---

# 35. Data Source Strategy

Separate:

### Market price data

Use reliable market-data providers/API sources.

### Fundamental data

Use reliable financial statement sources.

### Corporate actions

Use authoritative corporate-action data.

### News

Use reputable sources.

The system must record:

```text
Source
Timestamp
Data Date
```

Never let an LLM invent financial numbers.

LLMs may summarize and reason over verified data, but financial calculations must use structured numerical data.

---

# 36. AI Usage

Use AI primarily for:

* Annual report summarization
* Earnings-call analysis
* Qualitative business analysis
* Risk-factor extraction
* News classification
* Explaining investment decisions

Do NOT let an LLM directly decide:

```text
BUY
SELL
```

without passing through deterministic:

* Financial filters
* Valuation rules
* Risk rules
* Portfolio limits

The final execution decision must be generated by a deterministic investment/risk engine.

---

# 37. Example Decision Pipeline

```text
Raw Financial Data
       ↓
Data Validation
       ↓
Fundamental Engine
       ↓
Valuation Engine
       ↓
Dividend Engine
       ↓
Growth Engine
       ↓
AI Qualitative Analysis
       ↓
Composite Score
       ↓
Portfolio Optimization
       ↓
Risk Engine
       ↓
Execution Rules
       ↓
Zerodha API
```

---

# 38. Passive vs Active Decision Rule

The platform must continuously compare active stock selection with passive investing.

If active investing consistently underperforms the benchmark after:

* Costs
* Taxes
* Slippage
* Turnover

then recommend increasing the allocation to broad-market ETFs/index funds.

The system should not become emotionally attached to active stock picking.

---

# 39. Tax Awareness

Track transactions for future tax reporting.

Store:

```text
Buy date
Buy price
Quantity
Sell date
Sell price
Realized gain
Holding period
Dividend income
Corporate actions
```

The system should distinguish:

* Short-term capital gains
* Long-term capital gains
* Dividend income

Tax calculations must be based on the current Indian tax rules at the time of reporting and should be validated against authoritative sources.

---

# 40. Backup

Perform automated PostgreSQL backups.

Maintain:

```text
Daily backups:
7 days

Weekly backups:
4 weeks

Monthly backups:
12 months
```

Never keep the only copy of the database on the VPS.

---

# 41. Health Monitoring

Monitor:

```text
CPU
RAM
Disk
Docker
PostgreSQL
FastAPI
Zerodha API
Scheduler
Last successful execution
```

Health endpoint:

```text
GET /health
```

Example:

```json
{
  "status": "healthy",
  "database": "connected",
  "execution_mode": "DRY_RUN",
  "zerodha_session": "valid"
}
```

---

# 42. Development Phases

## Phase 0 — Infrastructure

1. Verify Oracle VPS
2. Verify public IP
3. Verify Reserved/Ephemeral IP
4. Configure swap
5. Configure firewall
6. Harden SSH
7. Install Docker

## Phase 1 — Database

Build:

* PostgreSQL
* Schema
* Migrations
* Backup

## Phase 2 — Data Engine

Build:

* Company universe
* Market data
* Fundamental data
* Corporate actions
* Dividend data

## Phase 3 — Investment Engine

Build:

* Fundamental score
* Valuation score
* Growth score
* Dividend score
* Risk score
* Composite score

## Phase 4 — Backtesting

Validate:

* CAGR
* XIRR
* Drawdown
* Benchmark performance
* Costs
* Robustness

## Phase 5 — Dry Run

Generate hypothetical monthly orders.

No real money.

## Phase 6 — Zerodha Integration

Implement:

* Official authentication
* Portfolio sync
* Funds check
* Order generation
* Order execution
* Order reconciliation

## Phase 7 — Production

Start with:

```text
₹10,000 initial capital
₹5,000/month
CNC only
Auto-buy enabled
Auto-sell disabled
```

After sufficient validation, reconsider automated selling.

---

# 43. Production Acceptance Criteria

Do not consider the project complete until:

* [ ] Oracle VPS is secured
* [ ] Stable public IP is confirmed
* [ ] PostgreSQL is working
* [ ] Automated backups work
* [ ] Investment engine works
* [ ] Risk engine works
* [ ] Dry-run mode works
* [ ] Duplicate-order protection works
* [ ] Zerodha API authentication works
* [ ] Portfolio reconciliation works
* [ ] Prohibited order types are technically blocked
* [ ] Emergency stop works
* [ ] Notifications work
* [ ] Backtesting is completed
* [ ] Benchmark comparison is implemented
* [ ] Current Zerodha API requirements are verified
* [ ] Static-IP requirements are satisfied where applicable
* [ ] Live execution is tested with minimal capital

---

# 44. Final Desired User Experience

Ultimately I want the system to behave like this:

```text
                    ME
                     │
          Initial Configuration
                     │
                     ▼
             Investment Rules
                     │
                     ▼
          ┌─────────────────────┐
          │ Automated Platform  │
          └──────────┬──────────┘
                     │
       ┌─────────────┼──────────────┐
       ▼             ▼              ▼
   Research       Portfolio       Risk
       │             │              │
       └─────────────┼──────────────┘
                     ▼
              Decision Engine
                     │
                     ▼
              Zerodha API
                     │
                     ▼
              CNC Investment
                     │
                     ▼
             Portfolio Update
                     │
                     ▼
               Notification
```

My normal interaction should be limited to:

```text
Occasional authentication
        +
Funding the investment account
        +
Receiving notifications
        +
Reviewing monthly reports
        +
Emergency intervention if required
```

I should **not** have to manually perform routine buy/sell operations.

---

# 45. Most Important Requirements

Treat the following as non-negotiable:

1. **Long-term investment only.**
2. **No options or other derivatives.**
3. **No intraday trading.**
4. **No leverage or margin.**
5. **CNC/delivery only.**
6. **₹10,000 initial capital.**
7. **₹5,000 monthly contribution initially.**
8. **Target 8–10%+ long-term CAGR, never guaranteed.**
9. **Consider dividends as a passive-income component.**
10. **Track corporate actions, including stock splits.**
11. **Stock splits must never be treated as value creation by themselves.**
12. **Use fundamental + valuation + growth + dividend + risk analysis.**
13. **Avoid excessive diversification.**
14. **Automate routine portfolio management.**
15. **Minimize Zerodha login requirements without bypassing authentication.**
16. **Use official Zerodha APIs only.**
17. **Use my existing Oracle Cloud Free Tier VPS.**
18. **Optimize for 1 OCPU / 1 GB RAM.**
19. **Use a stable/reserved public IP where required for API order execution.**
20. **Never store or automate my Zerodha password.**
21. **Use dry-run mode before real-money execution.**
22. **Backtest before deployment.**
23. **Benchmark against passive index investing.**
24. **Implement an emergency kill switch.**
25. **Implement duplicate-order protection.**
26. **Keep complete audit logs.**
27. **Do not allow an LLM to bypass deterministic risk controls.**
28. **Do not guarantee investment returns.**
29. **Never blindly average down a deteriorating business.**
30. **If active investing fails to justify itself against passive investing, say so clearly.**

---

# 46. What I Want You to Deliver

Do NOT jump directly into coding.

First produce:

### A. Architecture

Complete production architecture tailored to my Oracle `VM.Standard.E2.1.Micro`.

### B. Investment Strategy

Detailed rules for stock selection, valuation, dividends, portfolio allocation and risk.

### C. Security Architecture

Zerodha authentication, API secrets, IP configuration, firewall and failure handling.

### D. Database Design

Complete schema and relationships.

### E. Automation Design

Exact monthly, weekly, daily and quarterly workflows.

### F. Backtesting Design

Historical data requirements and methodology.

### G. Deployment Guide

Exact Oracle VPS commands, Docker configuration and production setup.

### H. Zerodha Integration

Use only the current official Kite Connect API and current Zerodha documentation.

### I. Dry-Run System

Implement the ability to run the entire strategy without placing real orders.

### J. Production Rollout

Provide a safe staged process for enabling real-money execution.

### K. Monitoring

Dashboard, Telegram alerts, logs, backups and health checks.

---

# Final Instruction

Build this as a **long-term automated investment operating system**, not a trading bot.

The system should remove repetitive manual work while retaining strict financial, technical, regulatory and security controls.

The first objective is **not maximum return**.

The first objective is:

> **Build a reliable, transparent, low-cost, low-maintenance investment system that can consistently deploy ₹5,000/month into high-quality long-term investments while minimizing unnecessary human intervention and avoiding unnecessary trading.**

Only after the system is proven through historical testing, dry-run operation and controlled deployment should real-money automated execution be enabled.
