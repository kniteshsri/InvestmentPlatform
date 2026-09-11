# Step-by-Step Beginner's Guide

> Written for someone **new to investing and new to computers**. No step assumes you
> already know something — if a word looks technical, it's explained the first time it
> appears, and there's a glossary at the bottom.
>
> Read alongside [`SUGGESTED_PLAN.md`](SUGGESTED_PLAN.md) (the "what and why") — this
> file is the "how, in order."

---

## First, three honest truths (please read)

1. **This is a real, advanced project.** Professionals build things like this. You
   *can* do it, but it takes weeks-to-months of small steps. That slow pace is not a
   problem — it's actually **required**, because the plan says we run in "pretend
   mode" for months before any real money. So going slow *is* the correct strategy.
2. **You do NOT need to become a programmer.** Think of it as a team of two:
   - **Your job:** create accounts, click buttons, add money, and **copy-paste
     commands I give you**, then tell me what appeared on the screen.
   - **My job (Claude):** write all the actual code, explain each piece in plain
     words, and fix any error you paste to me.
3. **Two golden rules that keep you safe:**
   - 🛑 **No real money for a long time.** We stay in "dry-run" (practice) mode until
     everything is proven. Real rupees come last, and start tiny (₹10,000).
   - 🆘 **When anything looks confusing or shows an error, don't guess — copy the
     whole message and send it to me.** That's how we fix things.

---

## The big picture, explained like you're new (because you are — and that's fine)

Imagine you're hiring a very disciplined, tireless assistant to invest ₹5,000 for you
every month. To do that, the assistant needs a few things:

| The techie word | What it really is (plain English) |
|---|---|
| **VPS / server** (your Oracle machine) | A small computer that lives in the cloud and **never sleeps**. You already rented one for free. Your assistant "lives" here. |
| **Terminal** | A text window where you type commands to talk to that computer (instead of clicking). Looks intimidating, is actually just typing. |
| **SSH** | The secure way to connect *from your laptop* into that cloud computer. Like a locked tunnel. |
| **Docker** | A tool that installs complicated apps cleanly, in one click, without a mess. |
| **PostgreSQL (a database)** | A smart filing cabinet that remembers every price, decision, and order forever. |
| **FastAPI (the backend)** | The "brain" — the program that thinks: *which stock, how much, is it safe?* |
| **Scheduler** | An alarm clock that wakes the brain on a schedule (e.g., once a month). |
| **Zerodha Kite API** | The official, legal doorway that lets a program place orders in *your* Zerodha account. |
| **Gemini (the free AI)** | A reading assistant that skims long company reports and explains them in plain English. It never decides trades. |
| **Telegram** | A chat app the system uses to **text you** ("Done! Invested ₹4,850") or ask you to log in. |
| **Dry-run mode** | **Practice mode.** The system does everything *except* spend money — it writes down what it *would* buy. This is where we live for months. |

That's the whole system. Everything below is just building these pieces one at a time.

---

## The map: 8 milestones (don't do them all at once)

You are here → **Milestone 0**. We finish one milestone, breathe, then start the next.

```text
M0  Learn the basics + set up YOUR laptop           ← start here (this week)
M1  Create the accounts you'll need
M2  Safely connect into your Oracle cloud computer
M3  Prepare the cloud computer (safety + tools)
M4  I build the "skeleton" app; you run it and see it say "healthy"
M5  Add data + the scoring brain
M6  DRY-RUN for several months (practice, no money)   ← we stay here a long time
M7  Backtest (test the strategy on old data)
M8  Go live with a tiny amount (₹10,000), watch closely
```

**Only Milestones 0–3 are things you do mostly by yourself (with me guiding).**
From Milestone 4 onward, I write the code and you mostly run what I hand you.

---

# MILESTONE 0 — Learn the basics & set up your laptop (this week)

Goal: be able to open a "terminal" and connect to things. No investing yet.

### Step 0.1 — Figure out what laptop you have
- **Windows** or **Mac**? You'll use slightly different apps. Tell me which — I'll
  tailor the exact clicks.

### Step 0.2 — Open a "terminal" once, just to see it
A terminal is just a window where you type commands.
- **Windows:** click Start, type **`PowerShell`**, press Enter. A dark window opens.
- **Mac:** press `Cmd + Space`, type **`Terminal`**, press Enter.

Type this and press Enter (it just shows today's date — a harmless test):
```bash
date
```
✅ **You should see:** today's date printed. That's it — you've used a terminal. 🎉

### Step 0.3 — Install two free tools on your laptop
Don't worry about what they do yet; we'll use them soon.
1. **A code editor — VS Code** (free): download from the official Microsoft site.
   It's where we'll view the project files together.
2. **Git** (free): a tool that saves versions of our work. (On Mac it may already be
   there. On Windows, download "Git for Windows".)

> 🆘 If any install asks confusing questions, screenshot it and send it to me. Don't
> guess.

**End of Milestone 0.** Tell me: *Windows or Mac?* and *"terminal test worked."*
Then we go to Milestone 1.

---

# MILESTONE 1 — Create your accounts

You'll need a few free logins. Do these in order. **Write every password down safely**
(a password manager or a physical notebook kept private).

### Step 1.1 — Zerodha account (the broker)
- If you don't have a **Zerodha Demat/trading account**, open one at zerodha.com.
  This is a regulated financial account — it needs your PAN, bank, etc. It can take a
  day or two to be approved. **This is the only account tied to real money.**

### Step 1.2 — Zerodha **Kite Connect** developer access (the API doorway)
- Separate from your normal login. You sign up as a "developer" to get an
  **API key** and **API secret** (two secret codes that let our program talk to your
  account). Per our plan, the **execution (order-placing) API is free for personal
  use**.
- ⚠ **Verify at signup** exactly what's free vs the optional ₹500/month data plan —
  we are **not** taking the data plan. Follow only the official Zerodha screens.
- 🔒 When you get the API key + secret, **do not paste them into chat or any file
  yet.** We'll store them safely later. Just keep them private.

### Step 1.3 — Google **Gemini** API key (the free AI)
- Go to Google's AI developer site and create a **free API key** for Gemini. No credit
  card needed for the free tier. Keep this key private too.

### Step 1.4 — **Telegram** bot (how the system texts you)
- Install Telegram (phone or desktop). In Telegram, search for **`@BotFather`** (the
  official bot that makes bots), start it, and follow its prompts to create a new bot.
  It gives you a **bot token** (another secret). Keep it private.

✅ **End of Milestone 1:** you have four things saved privately — Zerodha login, Kite
API key+secret, Gemini API key, Telegram bot token. **Don't share them with me or
anyone.** We'll load them into the server the safe way.

---

# MILESTONE 2 — Connect into your Oracle cloud computer

Your cloud computer's address is **`68.233.112.199`**, username **`ubuntu`**.

### Step 2.1 — Find your Oracle SSH key
When your Oracle server was created (22 June 2025), Oracle gave you a **private key
file** (a small file that acts like a house key). Find it on your laptop — it's often
named something like `ssh-key-....key` or `oracle_private.key`.

> 🆘 Can't find it? That's common. Tell me — there's a recovery path in the Oracle
> console (we can add a new key). Don't panic.

### Step 2.2 — Connect (the "SSH tunnel")
In your terminal, type this — replacing `PATH_TO_YOUR_KEY` with where that key file
is:
```bash
ssh -i PATH_TO_YOUR_KEY ubuntu@68.233.112.199
```
- The first time, it asks "are you sure you want to connect?" — type `yes`.
- ✅ **You should see:** the prompt changes to something like
  `ubuntu@your-server:~$`. **You are now inside the cloud computer.** 🎉
- To leave and come back to your own laptop, type `exit`.

> 🆘 If it says "permission denied" or "bad permissions," paste the exact message to
> me — it's usually a one-line fix about the key file.

✅ **End of Milestone 2:** you can get in and out of your server. That's a big one.

---

# MILESTONE 3 — Prepare the server (safety first)

Now we make the server secure and install the tools. **I'll give you the exact
commands** — you paste them one block at a time and tell me what you see. Here's the
order we'll follow (from our plan, `SUGGESTED_PLAN.md` §7 Phase 0):

1. **Check the server's health** — confirm memory, disk, and its public address.
2. **Add "swap"** — a 1 GB emergency memory cushion (because the server only has
   1 GB of real memory).
3. **Turn on the firewall** — only allow the doors we need (SSH + secure web).
4. **Lock down login** — use the key only, disable password login.
5. **Make the IP permanent (Reserved)** — required for placing orders from
   1 April 2026, and free on Oracle. (Some clicking in the Oracle website.)
6. **Install Docker** — the one-click app installer.

> You won't understand every command, and that's OK — each one is standard and safe,
> and I'll say in plain words what it does before you run it. Nothing here touches
> money.

✅ **End of Milestone 3:** a secure server with Docker installed, ready for our app.

---

# MILESTONE 4 — I build the skeleton; you press "start"

Here the balance flips: **I write the code.** You'll:
1. Copy the project files onto the server (I'll give exact commands).
2. Put your secret keys into one protected file called `.env` (never shared, never
   uploaded — I'll show you how).
3. Run **one** command to start everything:
   ```bash
   docker compose up -d
   ```
4. Open the health check and confirm it says it's alive:
   ```bash
   curl http://localhost:8000/health
   ```
   ✅ **You should see** something like:
   ```json
   { "status": "healthy", "database": "connected", "execution_mode": "DRY_RUN" }
   ```
   Notice `"DRY_RUN"` — that's your safety guarantee: **practice mode, no real money.**

✅ **End of Milestone 4:** the assistant is "alive" and safely in practice mode.

---

# MILESTONE 5 — Data + the scoring brain (I build, we test together)

I'll add, piece by piece (from `SUGGESTED_PLAN.md` §7 Phases 2–3 & 5):
- **Data collectors** — pull free stock prices (`yfinance`) and company fundamentals
  (`screener.in`) into your database.
- **The scoring engine** — the deterministic "brain" that rates each company 0–100 on
  fundamentals, valuation, growth, stability, and dividends, with hard safety filters.
- **The free AI helper (Gemini)** — reads long reports and writes plain-English
  summaries. **It never decides trades** — the scoring brain does.

You'll test each piece by running small commands and seeing sensible output. If a
number looks wrong, we investigate together.

---

# MILESTONE 6 — DRY-RUN for several months (the important, patient part)

Now the assistant runs its **full monthly routine** — but in **practice mode**:
- Once a month it: checks the ₹5,000 budget → scores stocks → applies risk checks →
  decides what it *would* buy → **writes it down** → **texts you a report on
  Telegram** — **without spending a rupee.**
- Your job: read the monthly Telegram reports and sanity-check them. Do the picks make
  sense? Is it behaving calmly in market ups and downs?

> ⏳ We stay here for **several months on purpose.** This is where you build trust in
> the system and catch mistakes while they're free. Patience here is the whole point.

---

# MILESTONE 7 — Backtest (test the strategy on history)

We run the strategy against **5–10 years of past data** (on your laptop or a free
Google Colab page, *not* the little server) to answer one honest question:

> "Would this strategy actually have beaten simply buying a Nifty 50 index — **after
> costs**?"

If yes, great. If no, the plan says we shift more toward a simple index fund. **The
system is designed to tell you the truth, even if the truth is "just buy the index."**

---

# MILESTONE 8 — Go live with a tiny amount (only when everything above passed)

Only after dry-run and backtest look good:
1. Confirm the current Zerodha rules (⚠ verify — they change).
2. Fund the account with your **₹10,000**.
3. Switch **one setting** from `DRY_RUN` to live — with `auto_sell = false` (it can
   buy, but it will **never sell on its own** yet).
4. The **once-a-month one-tap login**: on investment day, the system texts you a Zerodha
   login link; you tap it, do the normal 2FA login **once**, and it does the rest.
5. Watch the first few months very closely.

✅ **End of Milestone 8:** a working, disciplined, mostly-hands-off monthly investor —
built by you.

---

## What to do RIGHT NOW (your first 3 actions)

1. Reply telling me: **Windows or Mac?**
2. Do **Step 0.2** (open a terminal, type `date`) and tell me it worked.
3. Start **Step 1.1** (open/confirm your Zerodha account) — this one can take a couple
   of days for approval, so kicking it off early saves time.

Then I'll walk you through the next steps, one small block at a time. 🙂

---

## Mini-glossary (bookmark this)

- **Command** — a line of text you type into the terminal to make the computer do
  something.
- **Server / VPS** — your always-on cloud computer.
- **SSH** — the secure way to connect into it from your laptop.
- **Terminal / PowerShell** — the text window where you type commands.
- **Docker** — installs complex apps cleanly in one step.
- **Database (PostgreSQL)** — where all data and decisions are stored.
- **Backend / FastAPI** — the "brain" program.
- **API** — a legal, official way for two programs to talk (e.g., our app ↔ Zerodha).
- **API key / secret / token** — secret passwords for programs. **Never share them.**
- **`.env` file** — a protected file on the server holding those secrets. Never
  uploaded anywhere.
- **Dry-run** — practice mode; no real money moves.
- **CNC / delivery** — buying shares to *keep* (the only kind we do). No fancy trading.
- **Backtest** — testing a strategy on old data before risking money.
- **Reserved IP** — a permanent address for your server (needed to place orders).

> 🆘 Golden rule again: confused or seeing red text/an error? **Copy it all, send it to
> me.** We fix it together.
</content>
