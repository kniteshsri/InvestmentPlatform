# Automated Long-Term Investment Platform
## Oracle Cloud VPS + Zerodha Kite Connect + Python + PostgreSQL + n8n

> **Project objective:** Build a compliant, low-frequency, long-term investment automation platform for Indian equities. This is **not** an options, futures, intraday, or high-frequency trading bot.

---

## 1. Important Disclaimer and Design Principles

### Investment objective
- Initial investment: **₹10,000**
- Monthly contribution: **₹5,000**
- Gradually increase contributions over time
- Target: long-term wealth creation
- Desired long-term return objective: approximately **8–10%+ CAGR**, without any guarantee
- Investment horizon: **5–10+ years**

### Explicitly excluded
- Options
- Futures
- Intraday trading
- Margin/leverage
- MIS orders
- Speculative penny stocks
- High-frequency trading

### Core principle

> Buy fundamentally strong businesses at reasonable valuations and allow long-term compounding.

No software, AI model, or automation can guarantee an 8–10% minimum return. The system must optimize for disciplined, risk-adjusted long-term investing.

---

# 2. Target Architecture

```text
                         USER
                    Laptop / Phone
                          │
                     HTTPS / VPN
                          │
                          ▼
              ┌────────────────────────┐
              │ Oracle Cloud Free VPS  │
              │ Reserved Public IP     │
              │                        │
              │  Docker Compose        │
              │                        │
              │ ┌────────────────────┐ │
              │ │ Dashboard / API    │ │
              │ │ FastAPI            │ │
              │ └─────────┬──────────┘ │
              │           │            │
              │ ┌─────────▼──────────┐ │
              │ │ Investment Engine  │ │
              │ ├────────────────────┤ │
              │ │ Risk Engine        │ │
              │ ├────────────────────┤ │
              │ │ Portfolio Engine   │ │
              │ ├────────────────────┤ │
              │ │ Zerodha Executor   │ │
              │ └─────────┬──────────┘ │
              │           │            │
              │ ┌─────────▼──────────┐ │
              │ │ PostgreSQL         │ │
              │ └────────────────────┘ │
              │                        │
              │ ┌────────────────────┐ │
              │ │ n8n Scheduler      │ │
              │ └────────────────────┘ │
              └────────────┬───────────┘
                           │
                  Whitelisted Static IP
                           │
                           ▼
                  Zerodha Kite Connect
                           │
                           ▼
                    Zerodha Account
```

## Why Oracle VPS?

Your home broadband does not need a static IP.

The production automation server should originate API order requests from a stable public IP. The Oracle VPS should be configured with a **Reserved Public IP** where supported and required by the broker's current API rules.

Your home connection is used only for:
- SSH administration
- Dashboard access
- Reviewing reports
- Emergency pause/stop

---

# 3. Recommended Deployment Strategy

Do not immediately enable real-money automation.

Use four stages:

## Stage 1 — Development

Build locally/on Oracle:
- Database
- Stock screening
- Fundamental scoring
- Portfolio allocation
- Corporate action monitoring

**Order execution disabled.**

## Stage 2 — Backtesting

Test using historical data.

Validate:
- CAGR
- XIRR
- Maximum drawdown
- Sharpe ratio
- Benchmark comparison
- Transaction costs
- Portfolio turnover

## Stage 3 — Dry Run

Run the automation on a schedule.

It should:
- Generate investment decisions
- Generate hypothetical orders
- Store results

But:

**DO NOT SEND ORDERS TO ZERODHA.**

Run for at least several months if possible.

## Stage 4 — Limited Production

Enable only:
- CNC delivery equity
- ETFs
- Strict monthly budget

Start with:
- ₹10,000 initial capital
- ₹5,000/month

---


---

# 3A. Actual Oracle Cloud VPS Configuration (User Environment)

This implementation is tailored to the following existing Oracle Cloud VPS.

> **Security note:** The instance OCID and other OCI identifiers are intentionally not repeated here because they are not required for day-to-day deployment and should not be unnecessarily copied into project repositories.

| Configuration | Current Value |
|---|---|
| Oracle Region | `ap-hyderabad-1` (Hyderabad) |
| Availability Domain | AD-1 |
| Fault Domain | FD-1 |
| Public IP | `68.233.112.199` |
| SSH Username | `ubuntu` |
| Operating System | Canonical Ubuntu 22.04 |
| Shape | `VM.Standard.E2.1.Micro` |
| OCPU | 1 |
| Memory | 1 GB |
| Network Bandwidth | 0.48 Gbps |
| Storage | Block volume only |
| Launch Mode | PARAVIRTUALIZED |
| In-transit Encryption | Enabled |
| Capacity Type | On-demand |

## Critical Capacity Assessment

The current instance has only **1 GB RAM** and **1 OCPU**.

Therefore, the original architecture containing all of the following simultaneously is **too heavy** for this VM:

- PostgreSQL
- FastAPI
- n8n
- Multiple Python worker processes
- Redis
- React dashboard
- LLM inference
- Multiple Docker containers

The production architecture must be optimized for a micro-instance.

### Recommended architecture for this VPS

```text
Oracle VM.Standard.E2.1.Micro
1 OCPU / 1 GB RAM
          │
          ├── Docker Compose
          │
          ├── PostgreSQL
          │
          ├── FastAPI Investment Engine
          │
          └── Lightweight Scheduler (cron/APScheduler)
```

### Do NOT run initially

- Local LLMs
- Ollama
- Open WebUI
- Heavy vector databases
- Elasticsearch
- Kafka
- Redis unless strictly required
- Multiple AI agents running continuously
- n8n together with unnecessary services

For this 1 GB VM, use **cron or APScheduler instead of n8n initially**. n8n can be moved to another server later if workflow complexity increases.

---

# 3B. Recommended Resource Allocation

Approximate target allocation:

| Service | Expected Memory Target |
|---|---:|
| Ubuntu OS | 200–350 MB |
| PostgreSQL (tuned) | 150–250 MB |
| FastAPI + Python | 100–200 MB |
| Scheduler | <50 MB |
| Docker overhead | Variable |
| Safety buffer | 200+ MB |

This is tight. Avoid running multiple containers with unnecessary background processes.

---

# 3C. Swap Configuration

Because the VM has only 1 GB RAM, configure a small swap file as an emergency buffer.

> Swap is not a substitute for RAM. It prevents immediate out-of-memory failures but will be slower than physical memory.

Check existing swap:

```bash
swapon --show
free -h
```

Create a 1 GB swap file if no swap exists:

```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Persist after reboot:

```bash
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
```

Verify:

```bash
free -h
swapon --show
```

Set conservative swappiness:

```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-investment-platform.conf
sudo sysctl --system
```

---

# 3D. PostgreSQL Memory Tuning for 1 GB RAM

Do not use default production PostgreSQL tuning intended for larger servers.

Suggested starting point:

```conf
shared_buffers = 128MB
effective_cache_size = 256MB
work_mem = 4MB
maintenance_work_mem = 32MB
max_connections = 20
```

The database workload is expected to be low because this system performs:

- Scheduled research
- Monthly investment decisions
- Periodic portfolio monitoring
- Audit logging

It does not require high transaction throughput.

---

# 3E. Docker Strategy for the Current VPS

Use only essential services.

Recommended initial `docker-compose.yml`:

```yaml
services:

  postgres:
    image: postgres:16-alpine
    container_name: investment-postgres
    restart: unless-stopped
    env_file:
      - .env
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    mem_limit: 300m
    networks:
      - investment-network

  backend:
    build: ./backend
    container_name: investment-backend
    restart: unless-stopped
    env_file:
      - .env
    depends_on:
      - postgres
    mem_limit: 350m
    networks:
      - investment-network

volumes:
  postgres_data:

networks:
  investment-network:
    driver: bridge
```

Do not expose PostgreSQL publicly.

The scheduler can initially run:

- Inside the FastAPI application using APScheduler, OR
- As host-level cron jobs calling API endpoints/scripts

This avoids an additional n8n container on a memory-constrained VM.

---

# 3F. ARM/x86 Compatibility

The current Oracle shape `VM.Standard.E2.1.Micro` is an x86-based micro instance.

Therefore, standard `amd64` Docker images should generally work without ARM-specific image selection.

Verify architecture:

```bash
uname -m
```

Expected architecture should be checked before deployment.

---

# 3G. Oracle Network Security

The Oracle VM has a public IP, so secure both:

1. Oracle Cloud VCN Security Lists / Network Security Groups
2. Ubuntu UFW firewall

Recommended inbound access:

| Port | Purpose | Recommendation |
|---|---|---|
| 22 | SSH | Restrict to your IP where practical |
| 80 | HTTP | Only if required for certificate setup |
| 443 | HTTPS | Public if dashboard is enabled |
| 5432 | PostgreSQL | **Never public** |
| 8000 | FastAPI | **Never public directly** |
| 5678 | n8n | Avoid exposing on this VM |

Recommended production pattern:

```text
Internet
   │
   ▼
Nginx / Caddy (443)
   │
   ├── FastAPI (internal)
   └── Dashboard (internal)
```

---

# 3H. Public IP and Zerodha Execution

Current observed VPS public IP:

```text
68.233.112.199
```

Before using this IP for broker-side configuration:

1. Verify from inside the VPS:

```bash
curl -4 ifconfig.me
```

2. Confirm that the OCI public IP is **Reserved** rather than Ephemeral.
3. Verify the latest Zerodha/Kite requirements for API order placement.
4. Use the production VPS outbound IP only after confirming it is stable.
5. Never hard-code the IP into application source code.

If the IP assignment is Ephemeral, convert the production architecture to use a Reserved Public IP before enabling live execution.

---

# 3I. Recommended Service Schedule

To conserve CPU and memory, avoid continuously running heavy analysis.

Use scheduled jobs:

| Job | Frequency |
|---|---|
| Health check | Every 15 minutes |
| Portfolio sync | Daily |
| Price update | Daily / end-of-day |
| Corporate action scan | Daily |
| Fundamental data update | Weekly |
| Portfolio risk review | Weekly |
| Monthly investment allocation | Monthly |
| Quarterly results review | Quarterly |
| Full portfolio report | Monthly |

The server should perform intensive analysis only when needed.

---

# 3J. Revised Technology Decisions

Based on the actual 1 GB Oracle VPS:

## Use

- Ubuntu 22.04
- Docker
- Docker Compose
- PostgreSQL 16 Alpine
- Python 3.12
- FastAPI
- SQLAlchemy
- APScheduler or cron
- Telegram notifications
- Nginx/Caddy only when dashboard is exposed

## Avoid initially

- n8n on this VM
- Local AI/LLM models
- Ollama
- Open WebUI
- Kubernetes
- Redis
- Celery
- Heavy React build services
- Elasticsearch

## Future Upgrade Path

If additional compute becomes necessary:

```text
Current Oracle Micro VPS
        │
        ├── Zerodha Execution
        ├── PostgreSQL
        └── Lightweight API
                 │
                 ▼
Optional External/Separate Compute
        │
        ├── Backtesting
        ├── LLM analysis
        ├── Large-scale research
        └── Advanced dashboards
```

The order execution component should remain lightweight and reliable.

---

# 3K. Updated Implementation Priority

### Step 1 — Immediately

Verify:

```bash
uname -m
free -h
df -h
swapon --show
curl -4 ifconfig.me
docker --version
```

### Step 2

Configure:

- Swap
- Firewall
- SSH hardening
- Automatic security updates

### Step 3

Deploy only:

- PostgreSQL
- FastAPI backend
- Lightweight scheduler

### Step 4

Implement:

- Database schema
- Portfolio tracking
- Investment scoring
- Dry-run mode

### Step 5

Only after successful testing:

- Official Zerodha API integration
- Current static-IP requirement verification
- Authentication workflow
- Limited production execution

---


# 4. Oracle Cloud VPS Setup

## 4.1 Check Server Resources

SSH into the Oracle server:

```bash
ssh ubuntu@YOUR_SERVER_IP
```

Check architecture:

```bash
uname -m
```

Check memory:

```bash
free -h
```

Check disk:

```bash
df -h
```

Check CPU:

```bash
lscpu
```

Check outbound public IP:

```bash
curl -4 ifconfig.me
```

Save this IP.

---

## 4.2 Verify Public IP Type

In Oracle Cloud Console:

1. Open **Compute**
2. Select **Instances**
3. Select your VM
4. Open **Primary VNIC**
5. Open IPv4 addresses

Determine whether the IP is:

- Ephemeral
- Reserved

Prefer a **Reserved Public IP** for production stability.

### Important

Do not assume an IP is permanent merely because it has not changed after a reboot. Verify its assignment type in OCI.

---

# 5. Operating System Preparation

Update Ubuntu:

```bash
sudo apt update
sudo apt upgrade -y
```

Install common utilities:

```bash
sudo apt install -y \
  curl \
  wget \
  git \
  vim \
  htop \
  ufw \
  ca-certificates \
  gnupg \
  lsb-release
```

Set timezone:

```bash
sudo timedatectl set-timezone Asia/Kolkata
```

Verify:

```bash
timedatectl
```

---

# 6. Firewall Configuration

Allow SSH:

```bash
sudo ufw allow OpenSSH
```

Enable firewall:

```bash
sudo ufw enable
```

Check:

```bash
sudo ufw status
```

Initially avoid exposing:
- PostgreSQL port 5432
- Redis port
- n8n directly
- Internal APIs directly

Use a reverse proxy and authentication for web services.

---

# 7. Install Docker

Install Docker:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Allow current user to run Docker:

```bash
sudo usermod -aG docker $USER
```

Log out and reconnect.

Verify:

```bash
docker --version
docker compose version
```

Enable Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

---

# 8. Project Directory Structure

Create the project:

```bash
mkdir -p ~/investment-platform
cd ~/investment-platform
```

Recommended structure:

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
├── n8n/
│   └── workflows/
│
├── data/
│   ├── imports/
│   └── exports/
│
├── reports/
│
└── scripts/
    ├── backup.sh
    └── healthcheck.sh
```

---

# 9. Environment Variables

Create `.env`:

```bash
nano .env
```

Example:

```env
# Database
POSTGRES_DB=investment_db
POSTGRES_USER=investment_user
POSTGRES_PASSWORD=CHANGE_THIS_TO_A_LONG_RANDOM_PASSWORD

# Application
APP_ENV=production
APP_SECRET=CHANGE_THIS_TO_A_RANDOM_SECRET

# Zerodha
KITE_API_KEY=
KITE_API_SECRET=
KITE_ACCESS_TOKEN=

# Safety
EXECUTION_MODE=DRY_RUN
MONTHLY_BUDGET=5000
MAX_SINGLE_ORDER=2500
MAX_STOCK_ALLOCATION_PERCENT=30

# Notifications
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=
```

Generate secure secrets:

```bash
openssl rand -base64 32
```

## Never commit `.env`

Create `.gitignore`:

```gitignore
.env
__pycache__/
*.pyc
data/
reports/
*.db
```

---

# 10. PostgreSQL Database

Use PostgreSQL for:

- Portfolio holdings
- Transactions
- Investment decisions
- Fundamental metrics
- Corporate actions
- Dividends
- Audit logs
- Order history

## Core Tables

### companies

```sql
CREATE TABLE companies (
    id SERIAL PRIMARY KEY,
    symbol VARCHAR(50) UNIQUE NOT NULL,
    company_name VARCHAR(255),
    sector VARCHAR(100),
    industry VARCHAR(100),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### portfolio_holdings

```sql
CREATE TABLE portfolio_holdings (
    id SERIAL PRIMARY KEY,
    company_id INTEGER REFERENCES companies(id),
    quantity NUMERIC NOT NULL,
    average_buy_price NUMERIC,
    invested_amount NUMERIC,
    current_value NUMERIC,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### transactions

```sql
CREATE TABLE transactions (
    id SERIAL PRIMARY KEY,
    company_id INTEGER REFERENCES companies(id),
    transaction_type VARCHAR(10),
    quantity NUMERIC,
    price NUMERIC,
    total_amount NUMERIC,
    order_id VARCHAR(100),
    transaction_date TIMESTAMP,
    status VARCHAR(50)
);
```

### investment_decisions

```sql
CREATE TABLE investment_decisions (
    id SERIAL PRIMARY KEY,
    company_id INTEGER REFERENCES companies(id),
    score NUMERIC,
    decision VARCHAR(50),
    reason TEXT,
    execution_mode VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### dividends

```sql
CREATE TABLE dividends (
    id SERIAL PRIMARY KEY,
    company_id INTEGER REFERENCES companies(id),
    ex_date DATE,
    dividend_per_share NUMERIC,
    received_amount NUMERIC,
    received_date DATE
);
```

### audit_logs

```sql
CREATE TABLE audit_logs (
    id SERIAL PRIMARY KEY,
    event_type VARCHAR(100),
    details JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

# 11. Docker Compose Stack

For the current 1 GB Oracle Micro instance, prefer the lightweight compose configuration in **Section 3E**. The following is a broader reference architecture for future upgrades:

```yaml
services:

  postgres:
    image: postgres:16
    container_name: investment-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - investment-network

  backend:
    build: ./backend
    container_name: investment-backend
    restart: unless-stopped
    env_file:
      - .env
    depends_on:
      - postgres
    networks:
      - investment-network

  n8n:
    image: n8nio/n8n:latest
    container_name: investment-n8n
    restart: unless-stopped
    volumes:
      - n8n_data:/home/node/.n8n
    networks:
      - investment-network

volumes:
  postgres_data:
  n8n_data:

networks:
  investment-network:
    driver: bridge
```

Start:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Logs:

```bash
docker compose logs -f
```

---

# 12. Investment Strategy Engine

The strategy must use multiple independent factors.

## Fundamental Score — 30%

Evaluate:

- Revenue growth
- Profit growth
- EPS growth
- ROE
- ROCE
- Debt/equity
- Free cash flow

Example:

```text
Revenue Growth       5 points
Profit Growth        5 points
ROE                  5 points
ROCE                 5 points
Debt                 5 points
Free Cash Flow       5 points
```

---

## Valuation Score — 25%

Evaluate:

- P/E
- PEG
- Price/Book
- EV/EBITDA
- Historical valuation
- Sector comparison

---

## Growth Score — 20%

Evaluate:

- Industry growth
- Earnings momentum
- Competitive advantage
- Market opportunity

---

## Financial Stability — 15%

Evaluate:

- Interest coverage
- Debt trends
- Cash reserves
- Operating margins

---

## Dividend Quality — 10%

Evaluate:

- Dividend consistency
- Dividend growth
- Payout ratio
- Free cash flow coverage

---

# 13. Composite Investment Score

```text
Fundamental Strength       30%
Valuation                  25%
Growth Potential           20%
Financial Stability        15%
Dividend Quality           10%
--------------------------------
TOTAL                     100%
```

Decision:

| Score | Decision |
|---|---|
| 85–100 | Strong Buy |
| 75–84 | Buy |
| 65–74 | Accumulate |
| 50–64 | Watch |
| Below 50 | Avoid |

The score should never be the sole decision mechanism.

Add hard filters for:

- Corporate governance concerns
- Extreme debt
- Promoter pledging
- Accounting red flags
- Illiquidity

---

# 14. Portfolio Construction Rules

Initial capital is small.

## ₹10,000 Initial Portfolio

Target:

- 3–5 positions maximum
- Prefer ETFs if diversification is difficult

Example allocation framework:

```text
Core Index / ETF        40%
Quality Growth          30%
Value / Opportunity     20%
Cash Reserve            10%
```

These percentages are configurable and are not mandatory investment advice.

---

# 15. Monthly ₹5,000 Workflow

Every month:

```text
Scheduled Date
      │
      ▼
Check Monthly Budget
      │
      ▼
Fetch Current Portfolio
      │
      ▼
Update Fundamentals
      │
      ▼
Calculate Stock Scores
      │
      ▼
Calculate Valuation
      │
      ▼
Risk Checks
      │
      ▼
Portfolio Allocation
      │
      ▼
Generate Orders
      │
      ▼
DRY RUN / LIVE?
      │
      ├── DRY RUN → Save hypothetical orders
      │
      └── LIVE → Zerodha API
                    │
                    ▼
                Verify Orders
                    │
                    ▼
                Update Database
                    │
                    ▼
                Send Notification
```

---

# 16. Risk Management Rules

These rules must execute BEFORE any order.

## Hard Limits

```text
Monthly budget:               ₹5,000
Maximum single order:         configurable
Maximum stock allocation:     30%
Maximum sector allocation:    40%
Leverage:                     0%
Options:                      BLOCKED
Futures:                      BLOCKED
Intraday:                     BLOCKED
MIS orders:                   BLOCKED
```

## Execution Allowlist

Only allow:

```text
Exchange: NSE/BSE
Product: CNC
Instrument type: EQUITY / approved ETF
```

Everything else should be rejected by code.

Example validation:

```python
def validate_order(order):
    if order.product != "CNC":
        raise Exception("Only CNC orders allowed")

    if order.instrument_type not in ["EQ", "ETF"]:
        raise Exception("Instrument blocked")

    if order.amount > MAX_SINGLE_ORDER:
        raise Exception("Maximum order amount exceeded")

    return True
```

---

# 17. Zerodha Integration

## Important Security Rule

Never:

- Store Zerodha password
- Automate browser login using scraping
- Attempt to bypass 2FA
- Circumvent authentication requirements

Use only official Zerodha APIs and supported authentication flows.

---

## Required Components

Create:

```text
ZerodhaAuthManager
        │
        ├── authentication/session handling
        ├── token validation
        └── session expiry detection

ZerodhaPortfolioService
        │
        ├── holdings
        ├── positions
        └── funds

ZerodhaExecutionService
        │
        ├── order validation
        ├── place order
        ├── order status
        └── reconciliation
```

---

# 18. Authentication Design

The system should minimize human interaction but must respect broker authentication requirements.

Workflow:

```text
Authentication Required
       │
       ▼
Secure Official Login Flow
       │
       ▼
Generate Authorized Session
       │
       ▼
Store Token Securely
       │
       ▼
Automation Continues
```

When a session expires:

```text
Detect Expiration
       │
       ▼
Stop Order Execution
       │
       ▼
Send Notification
       │
       ▼
User Completes Official Authentication
       │
       ▼
Resume Automation
```

Never attempt to bypass mandatory re-authentication.

---

# 19. Static IP Configuration

Before enabling LIVE execution:

1. Determine Oracle VPS outbound public IPv4.
2. Confirm it is stable/reserved.
3. Check the latest Zerodha static-IP requirements.
4. Configure the broker's permitted IP settings.
5. Verify connectivity.
6. Test in dry-run mode.
7. Enable limited production execution.

Do not whitelist an IP until you are certain it belongs to the production execution server.

---

# 20. Order Execution Safety

Every order requires:

```text
Investment Decision
       │
       ▼
Portfolio Allocation
       │
       ▼
Risk Validation
       │
       ▼
Order Validation
       │
       ▼
Duplicate Check
       │
       ▼
Budget Check
       │
       ▼
Execution Mode Check
       │
       ▼
Broker Submission
       │
       ▼
Order Status Verification
       │
       ▼
Database Reconciliation
```

## Idempotency

Store a unique decision ID.

Never place the same logical order twice because:

- Server restarted
- API timed out
- Network failed
- Workflow retried

---

# 21. Sell Strategy

Selling should be much stricter than buying.

Possible sell triggers:

1. Business fundamentals deteriorate significantly.
2. Debt increases unsustainably.
3. Corporate governance concerns emerge.
4. Investment thesis is invalidated.
5. Competitive advantage deteriorates.
6. Portfolio concentration becomes excessive.

Do NOT automatically sell simply because:

- Price falls 5%
- Price falls 10%
- Market crashes
- Short-term news is negative

Long-term investments should tolerate normal market volatility.

---

# 22. Corporate Action Monitoring

Track:

- Dividends
- Stock splits
- Bonus shares
- Buybacks
- Rights issues
- Mergers
- Demergers

Important:

A stock split is NOT automatically a buy signal.

Analyze corporate actions alongside:

- Earnings growth
- Revenue growth
- Cash flow
- Valuation
- Business quality

---

# 23. Dividend Strategy

Calculate:

```text
Dividend Yield
Dividend Growth
Dividend Payout Ratio
Free Cash Flow Coverage
Dividend Consistency
```

Avoid dividend traps.

A high dividend yield alone is insufficient.

---

# 24. Automation Workflows (APScheduler/cron First, n8n Optional Later)

## Workflow A — Monthly Investment

Schedule:

```text
Monthly
   │
   ▼
Check Budget
   │
   ▼
Call Investment API
   │
   ▼
Generate Decisions
   │
   ▼
Risk Check
   │
   ▼
Execute / Dry Run
   │
   ▼
Send Telegram Report
```

## Workflow B — Weekly Portfolio Health

```text
Weekly
  │
  ▼
Fetch Portfolio
  │
  ▼
Update Market Data
  │
  ▼
Check Risk Thresholds
  │
  ▼
Generate Report
```

## Workflow C — Quarterly Results

```text
Quarterly
   │
   ▼
Fetch Updated Financials
   │
   ▼
Recalculate Scores
   │
   ▼
Compare Previous Score
   │
   ▼
Alert Material Deterioration
```

## Workflow D — Dividend Monitoring

```text
Daily / Scheduled
       │
       ▼
Check Corporate Actions
       │
       ▼
Dividend Detected?
       │
   ┌───┴────┐
   │        │
  YES       NO
   │        │
   ▼        ▼
Record     End
   │
   ▼
Notify User
```

---

# 25. Notification System

Recommended first option: Telegram. This is lightweight and suitable for the current 1 GB VPS.

Notifications:

### Successful Investment

```text
Monthly Investment Completed

Budget: ₹5,000
Invested: ₹4,850
Cash Remaining: ₹150

Orders: 3
Execution Status: SUCCESS
```

### Risk Alert

```text
PORTFOLIO ALERT

Company: XYZ
Previous Score: 82
Current Score: 61

Reason:
Debt and cash-flow deterioration detected.

Action:
Under review.

No automatic action taken.
```

### Authentication Required

```text
ZERODHA AUTHENTICATION REQUIRED

Order execution is paused.

Complete the official authentication process.

No pending orders will be submitted automatically until authorization is restored.
```

---

# 26. Emergency Kill Switch

Create a database setting:

```text
AUTOMATION_STATUS = ACTIVE
```

Possible values:

```text
ACTIVE
PAUSED
EMERGENCY_STOP
DRY_RUN
```

## ACTIVE

Automation may execute validated orders.

## PAUSED

Research continues.

No new orders.

## EMERGENCY_STOP

Immediately block:

- Buy orders
- Sell orders
- Rebalancing

## DRY_RUN

Generate and log orders but never submit them.

---

# 27. Dashboard Requirements

Build later using React or a simple FastAPI-compatible frontend.

Show:

## Portfolio

- Total invested
- Current value
- Profit/loss
- XIRR
- CAGR
- Dividend income

## Holdings

- Company
- Quantity
- Average price
- Current price
- Allocation
- Fundamental score
- Valuation score

## Automation

- Current mode
- Last execution
- Next scheduled investment
- API health
- Authentication status

## Performance

Compare against:

- Nifty 50
- Relevant broad-market benchmark

Important:

The system must demonstrate whether active stock selection is actually outperforming a simple benchmark after costs.

---

# 28. Backtesting

Before real execution, test strategy using historical data.

Measure:

```text
CAGR
XIRR
Maximum Drawdown
Sharpe Ratio
Sortino Ratio
Win Rate
Portfolio Turnover
Transaction Costs
Benchmark Relative Return
```

Test across:

- Bull markets
- Bear markets
- Sideways markets
- High-interest-rate periods
- Market crashes

Avoid survivorship bias.

Avoid look-ahead bias.

Historical financial data must only be available to the strategy at the point it would have actually been published.

---

# 29. Backup Strategy

Create PostgreSQL backups.

Example:

```bash
#!/bin/bash

DATE=$(date +%Y-%m-%d)

docker exec investment-postgres \
pg_dump \
-U investment_user \
investment_db \
> ~/investment-platform/backups/investment-$DATE.sql
```

Create backup directory:

```bash
mkdir -p ~/investment-platform/backups
```

Automate using cron or n8n.

Retention suggestion:

- Daily: 7 days
- Weekly: 4 weeks
- Monthly: 12 months

Encrypt backups if storing them externally.

---

# 30. Monitoring and Health Checks

Monitor:

- Server uptime
- Disk usage
- Memory
- Docker containers
- Database availability
- API availability
- Scheduled workflow status

Example health endpoint:

```text
GET /health
```

Expected:

```json
{
  "status": "healthy",
  "database": "connected",
  "execution_mode": "DRY_RUN"
}
```

---

# 31. Security Checklist

- [ ] SSH key authentication
- [ ] Disable password SSH login
- [ ] Firewall enabled
- [ ] Database not publicly exposed
- [ ] Strong PostgreSQL password
- [ ] `.env` excluded from Git
- [ ] API secrets encrypted at rest where practical
- [ ] HTTPS for dashboard
- [ ] Audit logs enabled
- [ ] Rate limiting enabled
- [ ] Emergency kill switch tested
- [ ] Automated backups configured
- [ ] Production and development environments separated

---

# 32. Implementation Roadmap

## Phase 0 — Infrastructure

- [ ] Verify Oracle VPS
- [ ] Verify architecture (ARM/x86)
- [ ] Verify RAM/storage
- [ ] Verify public IP type
- [ ] Install Docker
- [ ] Configure firewall

## Phase 1 — Foundation

- [ ] Create Git repository
- [ ] Docker Compose
- [ ] PostgreSQL
- [ ] FastAPI
- [ ] Environment management
- [ ] Health checks

## Phase 2 — Data

- [ ] Company universe
- [ ] Historical prices
- [ ] Financial statements
- [ ] Corporate actions
- [ ] Dividend data

## Phase 3 — Investment Intelligence

- [ ] Fundamental scoring
- [ ] Valuation scoring
- [ ] Dividend scoring
- [ ] Portfolio allocation
- [ ] Risk engine

## Phase 4 — Backtesting

- [ ] Historical simulation
- [ ] Benchmark comparison
- [ ] Cost modeling
- [ ] Bias detection
- [ ] Strategy tuning

## Phase 5 — Automation

- [ ] n8n workflows
- [ ] Monthly investment scheduler
- [ ] Portfolio monitoring
- [ ] Notifications

## Phase 6 — Zerodha Integration

- [ ] Official API setup
- [ ] Authentication flow
- [ ] Portfolio synchronization
- [ ] Order validation
- [ ] Dry-run execution

## Phase 7 — Production

- [ ] Confirm current broker requirements
- [ ] Confirm static/reserved IP
- [ ] Whitelist required IP
- [ ] Test with minimal amount
- [ ] Enable limited CNC execution
- [ ] Monitor carefully

---

# 33. Final Production Rules

The production system MUST enforce:

```text
NO OPTIONS
NO FUTURES
NO INTRADAY
NO LEVERAGE
NO MARGIN
NO MIS
ONLY CNC DELIVERY
ONLY APPROVED EQUITIES / ETFs
STRICT MONTHLY BUDGET
ORDER DUPLICATE PROTECTION
EMERGENCY STOP
AUDIT LOGGING
```

---

# 34. Recommended Starting Configuration

```yaml
investment:
  initial_capital: 10000
  monthly_budget: 5000
  investment_horizon_years: 10

execution:
  mode: DRY_RUN
  allowed_product: CNC
  allowed_instruments:
    - EQUITY
    - ETF

risk:
  max_single_stock_percent: 30
  max_sector_percent: 40
  max_single_order_amount: 2500
  allow_leverage: false

automation:
  monthly_investment: true
  weekly_monitoring: true
  quarterly_fundamental_review: true
  auto_sell: false

notifications:
  telegram: true
  email: optional
```

Initially, keep `auto_sell: false`.

Automated selling should only be considered after the strategy has been extensively validated.

---

# 35. Definition of Success

The project is successful when:

1. The system runs reliably on Oracle Cloud.
2. Portfolio data is automatically synchronized.
3. Companies are evaluated using transparent criteria.
4. Monthly investment allocation is generated automatically.
5. Risk controls prevent prohibited order types.
6. Duplicate orders cannot occur.
7. Dry-run mode produces auditable decisions.
8. Backtesting demonstrates the strategy's characteristics.
9. Production execution follows all current broker and regulatory requirements.
10. The user only needs to intervene for mandatory authentication, funding, exceptional events, or emergency decisions.

---

# Final Principle

> Automation should remove repetitive work, not remove risk awareness.

The goal is not to create a machine that blindly buys and sells stocks.

The goal is to build a disciplined investment operating system that:

- Researches systematically
- Invests consistently
- Controls risk
- Tracks dividends and corporate actions
- Avoids emotional decisions
- Minimizes unnecessary trading
- Provides transparent reasoning
- Respects broker and regulatory requirements
- Allows long-term compounding
