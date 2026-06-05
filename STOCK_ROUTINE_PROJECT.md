# Stock Portfolio Routine — Claude Code Project Brief

> Full context from planning session. Read this file before starting any work.
> Everything needed to build the routine is documented here.

---

## Project Goal

Build a fully cloud-native Claude Code **Routine** that:
1. Reads a portfolio of US stocks from a `portfolio.json` file in the repo
2. Runs every weekday at **9 PM SGT** (30 min before US market opens at 9:30 AM ET)
3. Pulls daily news + past price trends for each stock
4. Produces a **Hold / Sell recommendation with reasoning** for each position
5. Sends the formatted report to a **Telegram chat** via the Bot API

No local machine required. Runs entirely on Anthropic's cloud infrastructure.

---

## Recommended Model

Use **`claude-opus-4-7`** for this routine.

- Best reasoning depth for financial analysis — weighs conflicting signals (bullish news vs bearish technicals)
- **Adaptive Thinking** mode: automatically determines reasoning depth per stock
- 1M token context window — can process full SEC filings if needed
- Ideal for autonomous multi-step agentic workflows

If cost is a concern for large portfolios, fall back to **`claude-sonnet-4-6`** (~40% cheaper, slightly less nuanced).

---

## Repo Structure to Create

```
your-repo/
├── portfolio.json              ← Your holdings (edit this to add/remove stocks)
├── STOCK_ROUTINE_PROJECT.md    ← This file
├── CLAUDE.md                   ← Claude Code project instructions (see below)
└── .claude/
    └── skills/
        ├── us-stock-analysis/      ← From tradermonty/claude-trading-skills
        ├── market-news-analyst/    ← From tradermonty/claude-trading-skills
        ├── earnings-calendar/      ← From tradermonty/claude-trading-skills
        ├── technical-analyst/      ← From tradermonty/claude-trading-skills
        ├── macro-regime-detector/  ← From tradermonty/claude-trading-skills
        └── [finance_skills]/       ← From JoelLewis/finance_skills (math foundations)
```

---

## File 1 — portfolio.json

The routine reads this file at the start of every run. Edit it freely — no need to touch the routine prompt when your holdings change.

```json
{
  "holdings": [
    { "ticker": "AAPL",  "shares": 10, "avg_cost": 172.50, "bought": "2024-08-15" },
    { "ticker": "NVDA",  "shares": 5,  "avg_cost": 480.00, "bought": "2024-11-02" },
    { "ticker": "MSFT",  "shares": 8,  "avg_cost": 390.00, "bought": "2025-01-20" }
  ],
  "currency": "USD",
  "risk_tolerance": "moderate"
}
```

**To add a stock:** append a new object with `ticker`, `shares`, `avg_cost`, `bought`.
**To remove a stock:** delete its entry. Next run ignores it automatically.

---

## File 2 — Routine Prompt (paste into claude.ai/code/routines)

```
You are an autonomous portfolio monitoring analyst running a scheduled daily analysis.
Your job is to analyse each stock holding, then deliver a Hold/Sell report directly
to a Telegram chat using the Telegram Bot API. No human is watching the terminal.
Complete every step fully before exiting.

════════════════════════════════════════
STEP 1 — LOAD PORTFOLIO
════════════════════════════════════════
Read the file `portfolio.json` from the repository root.
Extract: ticker, shares, avg_cost (your purchase price), bought (date purchased).
If the file is missing or malformed, send a Telegram message:
  "⚠️ Portfolio routine failed: could not read portfolio.json"
Then exit.

════════════════════════════════════════
STEP 2 — RESEARCH EACH TICKER
════════════════════════════════════════
For EVERY ticker in the holdings list, gather all of the following via web search:

PRICE DATA
- Latest closing price
- % change over past 1 day, 5 days, 1 month
- 52-week high and 52-week low
- Current price vs 50-day moving average (above or below)
- Current price vs 200-day moving average (above or below)

TECHNICAL SIGNALS
- RSI (14-day): note if overbought (>70), neutral (30–70), or oversold (<30)
- MACD: note if bullish crossover, bearish crossover, or neutral
- Volume: note if today's volume is notably above or below average

FUNDAMENTAL SNAPSHOT
- Forward P/E ratio
- Revenue growth (most recent quarter YoY %)
- EPS surprise on last earnings (beat/miss/in-line)
- Next earnings date (if within 30 days, flag it)

NEWS (past 48 hours only)
- Up to 3 most market-relevant headlines with source name and date
- Focus on: earnings, guidance changes, analyst upgrades/downgrades,
  product launches, regulatory events, macro events affecting the sector

════════════════════════════════════════
STEP 3 — CALCULATE PORTFOLIO METRICS
════════════════════════════════════════
For each holding calculate:
- Current market value = shares × latest price
- Unrealised P&L in USD = (latest price − avg_cost) × shares
- Unrealised P&L % = ((latest price − avg_cost) / avg_cost) × 100
- Days held = today's date − bought date

Total portfolio:
- Sum of all current market values
- Sum of all unrealised P&L in USD
- Overall portfolio return %

════════════════════════════════════════
STEP 4 — GENERATE RECOMMENDATIONS
════════════════════════════════════════
For each ticker, produce a HOLD or SELL verdict using this logic:

Consider SELL if ANY of these are true:
- RSI > 75 AND price is at or above 52-week high
- Negative earnings surprise AND price dropped >5% on earnings day
- Analyst consensus downgraded to Sell/Underperform
- Significant negative news that materially changes the investment thesis
- Unrealised gain > 40% AND technicals show bearish reversal signals

Consider HOLD if:
- Trend remains positive (above 50-day MA)
- No material negative news
- Fundamentals intact
- No extreme overbought signal

Be decisive. Every ticker must get either HOLD or SELL. No "WATCH" or "NEUTRAL".

════════════════════════════════════════
STEP 5 — FORMAT THE TELEGRAM MESSAGE
════════════════════════════════════════
Compose the full report as a single plain-text string.
Use ONLY these characters for formatting: emojis, newlines, and dashes.
Do NOT use Markdown asterisks, underscores, or backticks — Telegram plain mode only.

Use this exact format:

📊 DAILY PORTFOLIO REPORT
{today's date, e.g. Mon 2 Jun 2026} — Pre-market

──────────────────────
{for each ticker repeat this block}

🔵 AAPL — HOLD   (or 🔴 AAPL — SELL)
Price: $XXX.XX  |  Your cost: $XXX.XX
P&L: +$XXX.XX (+X.X%)  |  Held: XX days
Trend: [1 line summary of recent price trend]
News: [Most important headline — Source, Date]
Why: [2 sentences max explaining the recommendation]
{if earnings within 30 days}: ⚠️ Earnings: [date]

──────────────────────

💼 PORTFOLIO SUMMARY
Total value:     $X,XXX.XX
Total P&L:       +/-$X,XXX.XX (+/-X.X%)
Market outlook:  [1 sentence on overall US market sentiment today]

{if any ticker is SELL}: 🚨 Action needed: [ticker list]
{if all HOLD}: ✅ No action needed today

════════════════════════════════════════
STEP 6 — SEND TO TELEGRAM
════════════════════════════════════════
Send the report using the Telegram Bot API.
The bot token and chat ID are available as environment variables:
  TELEGRAM_BOT_TOKEN
  TELEGRAM_CHAT_ID

Run this shell command:

curl -s -X POST \
  "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -d chat_id="${TELEGRAM_CHAT_ID}" \
  -d parse_mode="" \
  --data-urlencode text="<INSERT FULL REPORT TEXT HERE>"

If the curl command returns ok:false, log the error and retry once.
If it fails twice, exit without further retries.

Do not print the full report to the terminal — just confirm:
"Report sent to Telegram for {N} holdings — {date}"

════════════════════════════════════════
RULES
════════════════════════════════════════
- Complete ALL steps every run. Do not skip any ticker.
- If a data point is unavailable, write "N/A" — do not omit the field.
- Keep each ticker block under 100 words.
- Never output the TELEGRAM_BOT_TOKEN to the terminal or any file.
- This routine runs unattended. Do not ask for confirmation at any step.
```

---

## File 3 — CLAUDE.md (project instructions for Claude Code)

Create this file at the repo root. Claude Code reads it automatically on every session.

```markdown
# Stock Portfolio Routine

This repo powers an automated daily stock monitoring routine for a personal US equity portfolio.

## What this repo does
- `portfolio.json` — list of current holdings (ticker, shares, cost basis, purchase date)
- `.claude/skills/` — financial analysis skills that inform the daily routine
- The routine runs every weekday at 9 PM SGT via claude.ai/code/routines

## Key rules
- Never commit TELEGRAM_BOT_TOKEN or TELEGRAM_CHAT_ID — these live in the routine's cloud environment variables only
- portfolio.json is the single source of truth for holdings — edit it directly, never hardcode tickers elsewhere
- All skill files in .claude/skills/ are read-only reference — do not modify them unless updating to a newer version

## Skills installed
- us-stock-analysis — fundamental + technical deep dive, Buy/Hold/Sell framework
- market-news-analyst — filters relevant news from noise, 48h lookback
- earnings-calendar — flags upcoming earnings dates within 30 days
- technical-analyst — RSI, MACD, MA crossovers, support/resistance
- macro-regime-detector — macro context (rate cycle, risk-on/off environment)
- finance_skills (JoelLewis) — return calculations, valuation math, risk metrics

## Model
Use claude-opus-4-7 for the routine. It has the deepest financial reasoning and Adaptive Thinking mode.
```

---

## Skills to Install

### Source 1 — tradermonty/claude-trading-skills (1,600+ GitHub stars, actively maintained)

Install these 5 skills:

| Skill | Why it's needed |
|---|---|
| `us-stock-analysis` | Core framework: fundamentals, technicals, valuation, Hold/Sell output |
| `market-news-analyst` | Scans company news, filters signal from noise |
| `earnings-calendar` | Flags earnings dates — critical for timing sell decisions |
| `technical-analyst` | RSI, MACD, MA analysis, chart patterns, support/resistance |
| `macro-regime-detector` | Macro context so single-stock moves aren't misread |

```bash
git clone https://github.com/tradermonty/claude-trading-skills temp-skills
mkdir -p .claude/skills
cp -r temp-skills/.claude/skills/us-stock-analysis      .claude/skills/
cp -r temp-skills/.claude/skills/market-news-analyst    .claude/skills/
cp -r temp-skills/.claude/skills/earnings-calendar      .claude/skills/
cp -r temp-skills/.claude/skills/technical-analyst      .claude/skills/
cp -r temp-skills/.claude/skills/macro-regime-detector  .claude/skills/
rm -rf temp-skills
```

### Source 2 — JoelLewis/finance_skills

81 skills across 7 domain plugins. Gives Claude proper financial math — return calculations, time value of money, valuation models, risk metrics. Install the two most relevant plugins:

```bash
git clone https://github.com/JoelLewis/finance_skills temp-finance
bash temp-finance/install.sh --plugins wealth-management,trading-operations
rm -rf temp-finance
```

### Source 3 — quant-sentiment-ai/claude-equity-research (optional, institutional-grade)

Adds options flow analysis, insider transaction monitoring, and bull/bear/base scenario modelling.

```bash
# Install via Claude Code plugin marketplace
/plugin marketplace add quant-sentiment-ai/claude-equity-research
```

---

## Telegram Setup (one-time)

### Step 1 — Create your Telegram bot
1. Open [@BotFather](https://t.me/BotFather) in Telegram
2. Send `/newbot`
3. Give it a name and username (must end in `bot`)
4. Copy the **bot token** it returns

### Step 2 — Get your Chat ID
1. Message [@userinfobot](https://t.me/userinfobot) on Telegram
2. It instantly replies with your numeric Chat ID

### Step 3 — Add environment variables to the routine
At `claude.ai/code/routines` → Edit routine → Environment:

| Variable | Value |
|---|---|
| `TELEGRAM_BOT_TOKEN` | Token from BotFather |
| `TELEGRAM_CHAT_ID` | Your numeric Chat ID |

These are stored encrypted. Never commit them to the repo.

---

## Routine Configuration

| Setting | Value |
|---|---|
| **URL** | `claude.ai/code/routines` → New routine |
| **Model** | `claude-opus-4-7` |
| **Repository** | This repo |
| **Trigger type** | Schedule |
| **Schedule** | Weekdays, 9:00 PM (SGT / UTC+8) |
| **Environment vars** | `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID` |
| **Prompt** | Paste from File 2 above |

### CLI alternative
```bash
/schedule run portfolio analysis weekdays at 9:00 PM SGT
```

---

## How It Works End-to-End

```
9:00 PM SGT (weekdays)
       │
       ▼
Anthropic cloud triggers routine
       │
       ▼
Clones this repo → reads portfolio.json
       │
       ▼
For each ticker: web search (price, news, technicals, fundamentals)
       │
       ▼
Applies skill frameworks from .claude/skills/
       │
       ▼
Calculates P&L, generates Hold/Sell verdict per stock
       │
       ▼
Formats Telegram message
       │
       ▼
POST → api.telegram.org → Your Telegram chat
       │
       ▼
✅ Done — report in your phone
```

---

## Sample Telegram Output

```
📊 DAILY PORTFOLIO REPORT
Mon 2 Jun 2026 — Pre-market

──────────────────────

🔵 AAPL — HOLD
Price: $191.30  |  Your cost: $172.50
P&L: +$188.00 (+10.9%)  |  Held: 291 days
Trend: Above 50-day MA, RSI 54 (neutral)
News: Apple expands AI features in iOS 20 — Reuters, 1 Jun
Why: Momentum intact, no negative catalysts. Fundamentals stable.

──────────────────────

🔴 NVDA — SELL
Price: $1,102.40  |  Your cost: $480.00
P&L: +$3,112.00 (+129.6%)  |  Held: 215 days
Trend: RSI 78 (overbought), near 52-week high
News: Goldman downgrades NVDA to Neutral — Goldman Sachs, 2 Jun
Why: Extreme overbought + analyst downgrade. Profit-taking warranted.

──────────────────────

💼 PORTFOLIO SUMMARY
Total value:     $14,832.40
Total P&L:       +$5,232.40 (+54.5%)
Market outlook:  Cautious tone ahead of Fed minutes release.

🚨 Action needed: NVDA
```

---

## Updating Your Portfolio

When you buy a new stock:
```json
{ "ticker": "TSLA", "shares": 3, "avg_cost": 245.00, "bought": "2026-06-01" }
```
Add it to the `holdings` array in `portfolio.json`, commit, push. The next routine run includes it automatically.

When you sell: remove the entry from `portfolio.json` and commit.

---

## Future Enhancements (optional)

- **Second trigger:** Add an API trigger wired to a price alert service so the routine fires immediately on a >5% single-day drop
- **Weekly deep report:** Add a second Saturday routine using `us-stock-analysis` skill for a fuller fundamental review of each position
- **Historical log:** Have the routine append each report to a `reports/` folder in the repo for tracking recommendation history over time
- **InvestSkill integration:** Add `yennanliu/InvestSkill` for full HTML equity reports with interactive Chart.js charts saved to the repo

---

## Resources

| Resource | URL |
|---|---|
| Claude Code Routines docs | `code.claude.com/docs/en/routines` |
| tradermonty skills | `github.com/tradermonty/claude-trading-skills` |
| finance_skills | `github.com/JoelLewis/finance_skills` |
| claude-equity-research plugin | `github.com/quant-sentiment-ai/claude-equity-research` |
| InvestSkill | `github.com/yennanliu/InvestSkill` |
| Telegram BotFather | `t.me/BotFather` |
| Routine schedule | `claude.ai/code/routines` |