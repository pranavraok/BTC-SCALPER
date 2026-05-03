# ScalperPro v24 — How It Works

This document explains the full flow of the app in plain language, covering how prices, charts, signals, trade recommendations, and outcomes are produced and stored.

---

## 1. What the App Does

- Shows live prices and charts for **BTC**, **Gold (XAU/USD)**, and **Silver (XAG/USD)**.
- Calculates multi-timeframe signals and generates **Buy/Sell setup levels**.
- Creates at most **one active trade recommendation per market** at a time.
- Tracks when price reaches the entry and then monitors targets or stop loss.
- Stores all important data in **Supabase**.
- On **Saturday and Sunday**, Gold and Silver show a weekend-closed popup before opening the dashboard.

---

## 2. Live Price Sources

| Market | Primary Source | Fallback |
|---|---|---|
| BTC | Binance ticker API | — |
| Gold (XAU/USD) | TwelveData price endpoint | CDN (fawazahmed0 currency API) |
| Silver (XAG/USD) | TwelveData price endpoint | CDN (fawazahmed0 currency API) |

These live prices are the inputs for the algo and also used as the displayed live price.

---

## 3. Chart Sources

| Market | Chart Symbol |
|---|---|
| BTC | `BINANCE:BTCUSDT` |
| Gold | `OANDA:XAUUSD` |
| Silver | `OANDA:XAGUSD` |

Charts are **visual only**. The algo uses the data sources in Sections 2 and 4.

---

## 4. Candles and Indicators

### Candle Sources

| Market | Source | Timeframes |
|---|---|---|
| BTC | Binance klines | 5m, 15m, 1h |
| Gold | TwelveData XAU/USD klines | 15m, 1h |
| Silver | Gold candles scaled by live gold/silver ratio | 15m, 1h |

### Indicators Used

- EMA, RSI, MACD, Bollinger Bands, ATR, volume ratio, swing support/resistance

Signals are blended into:
- A **Buy vs Sell probability**
- A **confidence score** (0–100)
- A **clarity label** (Balanced / Moderate / Clear / Very clear)

---

## 5. Separate Algorithms for Crypto vs Metals

BTC and Gold/Silver use **different algo parameters** because they behave differently.

### BTC Algorithm (`runAlgoBTC`)

| Parameter | Value |
|---|---|
| Timeframes | 5m + 15m + 1H |
| EMA periods | 9, 21, 50, 200 |
| MACD periods | 12 / 26 / 9 (standard) |
| Volume scoring | Yes — 10% weight |
| Session weight | 8% |
| Trend weight | 35% |
| RSI weight | 22% |

### Gold/Silver Algorithm (`runAlgoMetal`)

| Parameter | Value |
|---|---|
| Timeframes | 15m + 1H only (5m metals is too noisy) |
| EMA periods | 20, 50, 100, 200 (slower) |
| MACD periods | 15 / 30 / 9 (tuned for metals) |
| Volume scoring | No — metals volume from API is unreliable |
| Session weight | 13% (London/NY open drives gold moves) |
| Trend weight | 30% |
| RSI weight | 30% |

---

## 6. Buy/Sell Setup Levels (Informational)

- The app continuously updates Buy/Sell setups from live data.
- Levels in the Buy/Sell setup cards update on every refresh.
- These are **informational** — they are not the same as a trade recommendation.

### Color Coding

| Color | Meaning |
|---|---|
| Green | Buy at Price / Take Profit levels |
| Red | Sell at Price / Stop Loss levels |

---

## 7. What Counts as a Recommendation

A recommendation is only created when the algo is strong enough **and** the entry is near the current price.

### Reliability Gates (all must pass)

| Gate | Minimum |
|---|---|
| Confidence score | >= 65 |
| Dominant side probability | >= 60% |
| Clarity label | Clear or Very clear |
| Entry distance from price | Within 0.12% (`ENTRY_PCT`) |
| Target 1 | Must still be reachable |
| Active recommendation | None already exists |

If all gates pass, a new recommendation is stored in Supabase with `outcome = watch`.

---

## 8. Recommendation Lifecycle

```
Created → watch → pending → t1 / t2 / sl / expired
```

| Status | Meaning |
|---|---|
| `watch` | Setup is valid, entry has NOT been touched yet |
| `pending` | Entry was touched on a completed candle — trade is now live |
| `t1` | Take Profit 1 was hit |
| `t2` | Take Profit 2 was hit |
| `sl` | Stop Loss was hit |
| `expired` | Recommendation aged out or price moved away without triggering |

Only one active recommendation (`watch` or `pending`) is shown at a time in the Trade Recommendation area.

---

## 9. Entry Touch Logic

Entry is considered triggered when a **completed candle** crosses the entry price:

| Direction | Entry Triggered When |
|---|---|
| Buy | Candle low <= entry price |
| Sell | Candle high >= entry price |

Rapid moves count because the check uses completed candle high/low, not just a live tick.

---

## 10. Validity Window and Staleness

- A `watch` recommendation expires after **3 hours** (`REC_VALID_MS`).
- If the price feed is stale for more than **10 minutes** (`DATA_STALE_MS`), the record is flagged as `stale` and is not expired until data resumes.

---

## 11. Stop-Loss Cooldown

After a stop-loss (`sl`) outcome in a direction, new recommendations in the **same direction are blocked for 30 minutes** (`SL_COOLDOWN_MS`).

This cooldown is derived from Supabase history — no localStorage is used.

---

## 12. Weekend Market Closure (Gold/Silver)

When a user opens Gold or Silver on **Saturday or Sunday**, a popup appears explaining:

- The metals market is closed on weekends.
- Prices shown are from Friday's close — signals are not live.
- Trading resumes on **Monday at approximately 6:00 AM IST**.

The user can choose to **Go Back** or **Continue Anyway** (signals shown as reference only).

BTC is unaffected — crypto trades 24/7.

---

## 13. Supabase Storage

All important data is stored in Supabase table: **`scalper_signals`**

### Core Columns

| Column | Description |
|---|---|
| `id` | Unique record ID |
| `market` | btc / gold / silver |
| `dir` | BUY or SELL |
| `price` | Price at time of recommendation |
| `entry` | Entry level |
| `t1`, `t2` | Take Profit 1 and 2 |
| `s1`, `s2` | Stop Loss 1 and 2 |
| `buy_p`, `sell_p` | Buy/Sell signal strength (%) |
| `conf` | Confidence score |
| `clarity` | Balanced / Moderate / Clear / Very clear |
| `ts`, `ts_ms` | Timestamp |
| `outcome` | watch / pending / t1 / t2 / sl / expired |
| `dp` | Decimal precision |
| `entered` | Whether entry was triggered |

### Extended Columns

| Column | Description |
|---|---|
| `valid_until_ms` | Expiry timestamp for watch status |
| `entry_touched_ms` | Timestamp when entry was first touched |
| `data_source` | Which price API was used |
| `chart_source` | Which chart symbol was used |
| `session_score` | Active session weight at time of signal |
| `sess_line` | Session activity summary text |
| `algo_version` | Algorithm version tag |
| `last_price` | Most recent price seen |
| `last_price_at` | Timestamp of last price update |
| `stale` | True if data feed is stale |

---

## 14. UI Behavior Summary

| Area | When it Updates |
|---|---|
| Buy/Sell setup cards | Every refresh (live data) |
| Trade Recommendation | Shows only the single best active recommendation |
| Open Position card | Only shows when entry is executed (`pending` + `entered = true`) |
| History | Shows only resolved trades (`t1`, `t2`, `sl`, `expired`) |

---

## 15. Refresh Timers

| Task | Interval |
|---|---|
| BTC price + signals | Every 30 seconds |
| Gold/Silver price + signals | Every 60 seconds |
| History UI refresh | Every 30 seconds |
| Outcome checks (targets/SL) | Every 5 minutes (uses completed candles) |

---

## 16. Key Constants

These constants are defined in `index.html` and can be adjusted:

| Constant | Value | Meaning |
|---|---|---|
| `ENTRY_PCT` | `0.0012` | Entry must be within 0.12% of current price |
| `REC_VALID_MS` | 3 hours | How long a watch recommendation stays active |
| `DATA_STALE_MS` | 10 minutes | How long before price data is considered stale |
| `SL_COOLDOWN_MS` | 30 minutes | Cooldown after a stop loss in the same direction |

---

## 17. Common Reasons a Recommendation Does Not Show

- Confidence or clarity is below the minimum threshold.
- Price is too far from entry (outside 0.12%).
- There is already an active recommendation (`watch` or `pending`).
- The data feed is stale — no recent live price updates.
- Stop-loss cooldown is active for that direction.

---

## 18. Modifying Behavior

To change any behavior, update the constants or gating rules in `index.html`.
To change algo weights or indicator parameters, edit `runAlgoBTC()` (for BTC) or `runAlgoMetal()` (for Gold/Silver).
