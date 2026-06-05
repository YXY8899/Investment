# Stock Portfolio Routine

This repo powers an automated daily stock monitoring routine for a personal US equity portfolio.

## What this repo does
- `portfolio.json` — list of current holdings (ticker, shares, cost basis, purchase date)
- `routine-prompt.md` — the full routine prompt to paste into claude.ai/code/routines
- `reports/` — historical daily reports appended by the routine on each run
- `.agents/skills/` — financial analysis skills used by the routine

## Routine schedule
Runs every weekday at 9 PM SGT (UTC+8) via claude.ai/code/routines.
Skips US market holidays automatically (list is embedded in the routine prompt).

## Key rules
- Never commit TELEGRAM_BOT_TOKEN or TELEGRAM_CHAT_ID — these live in the routine's cloud environment variables only
- `portfolio.json` is the single source of truth for holdings — edit it directly, never hardcode tickers elsewhere
- To add a stock: append an entry to `portfolio.json` with ticker, shares, avg_cost, bought — next run picks it up automatically
- To remove a stock: delete its entry from `portfolio.json`
- Skill files in `.agents/skills/` are read-only reference — do not modify them unless updating to a newer version

## Updating the routine prompt
The live prompt lives at claude.ai/code/routines. `routine-prompt.md` is the source of truth locally.
If you change the prompt here, also update it in the routines UI.

## Skills installed
- `us-stock-analysis` — fundamental + technical deep dive, Hold/Sell framework
- `market-news-analyst` — filters relevant news from noise, 48h lookback
- `earnings-calendar` — flags upcoming earnings dates within 30 days
- `technical-analyst` — RSI, MACD, MA crossovers, support/resistance
- `macro-regime-detector` — macro context (rate cycle, risk-on/off environment)

## Environment variables (set in routines UI, never here)
- `TELEGRAM_BOT_TOKEN` — from BotFather
- `TELEGRAM_CHAT_ID` — your numeric Telegram chat ID

## Model
Use `claude-opus-4-7` for the routine. It has the deepest financial reasoning and Adaptive Thinking mode.
