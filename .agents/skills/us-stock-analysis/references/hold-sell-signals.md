# Hold/Sell Signal Reference

This file defines the scoring signals used by the daily routine to produce HOLD or SELL verdicts.
Edit this file to tune signal definitions or thresholds without touching the routine prompt.

---

## Sell Signals (S1–S7)

| Signal | Condition |
|---|---|
| S1 | RSI > 75 AND price at or above 52-week high |
| S2 | Negative earnings surprise AND price dropped > 5% on earnings day |
| S3 | Analyst consensus is "sell" or "strongSell" |
| S4 | High-impact negative news that materially changes the investment thesis |
| S5 | Unrealised gain > 40% AND MACD shows bearish crossover |
| S6 | Price fell > 8% in the past 5 days with no recovery — half-weight if confirmed sector-wide contagion from peer earnings/guidance (not company-specific) |
| S7 | Insider sell flag: C-suite/director sold > $1,000,000 in past 30 days (discretionary only — exclude RSU sell-to-cover) |

### Sell Confidence Thresholds
- 3+ signals → SELL, High Confidence
- 2 signals  → SELL, Medium Confidence
- 1 signal   → SELL, Low Confidence (use judgement — do not sell automatically)

---

## Hold Signals (H1–H9)

| Signal | Condition |
|---|---|
| H1 | Price above 50-day MA |
| H2 | Price above 200-day MA |
| H3 | RSI between 30 and 70 (healthy momentum range) |
| H4 | MACD neutral or bullish crossover |
| H5 | Analyst consensus is "buy", "strongBuy", or "hold" |
| H6 | Upside to analyst mean price target > 10% from current price |
| H7 | No high-impact negative news in past 48 hours |
| H8 | Business quality / moat assessment intact (from fundamental review) |
| H9 | Insider buy flag: C-suite/director bought > $500,000 in past 30 days (discretionary only) |

### Hold Confidence Thresholds
- 8–9 signals → HOLD, High Confidence
- 6–7 signals → HOLD, Medium Confidence
- 1–5 signals → HOLD, Low Confidence

---

## Stop-Loss Formula (every HOLD)

```
raw_stop  = max(currentPrice × 0.92, fiftyDayMA × 0.98)
stop_loss = min(raw_stop, currentPrice × 0.99)
```

- Floor: 8% drawdown from current price
- Anchor: just below the 50-day MA (which may be higher than the floor)
- Hard cap: always strictly below current price (prevents stop above entry)

## Take-Profit (every HOLD)

Use the analyst consensus mean price target.
- If current price is below the mean target: "Consider taking profit above $XXX (analyst mean target)"
- If current price already exceeds the mean target: use the analyst high target instead

---

## Rules

- Every ticker must receive exactly one verdict: HOLD or SELL. No "WATCH" or "NEUTRAL".
- Signals are scored independently — no signal cancels another.
- S6 half-weight applies only when the sector-wide drop is confirmed by a peer company's earnings miss or guidance cut. Company-specific bad news does not qualify.
- S7 and H9 ignore automatic RSU sell-to-cover transactions (routine tax withholding on vesting). Only flag discretionary trades.
