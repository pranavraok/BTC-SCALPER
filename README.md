# ScalperPro v24 - How it Works (End to End)

This document explains the full flow of the app in plain language, including how prices, setups, recommendations, and outcomes are produced and stored.

## 1) What the app does
- Shows live prices and charts for BTC, Gold (XAU/USD), and Silver (XAG/USD).
- Calculates multi-timeframe signals and generates Buy/Sell setup levels.
- Creates at most one active trade recommendation per market at a time.
- Tracks when price reaches the entry and then monitors targets or stop loss.
- Stores all important data in Supabase.

## 2) Live price sources
- BTC price: Binance ticker API.
- Gold/Silver price: TwelveData price endpoint (primary), CDN fallback.

These live prices are the inputs for the algo and also used as the displayed live price.

## 3) Chart sources
- BTC chart: BINANCE:BTCUSDT.
- Gold chart: OANDA:XAUUSD.
- Silver chart: OANDA:XAGUSD.

Charts are visual only. The algo uses the data sources in Section 2 and Section 4.

## 4) Candles and indicators
- BTC candles: Binance klines (5m, 15m, 1h).
- Gold candles: TwelveData XAU/USD.
- Silver candles: Gold candles scaled by the current gold/silver ratio.

Indicators used:
- EMA, RSI, MACD, Bollinger Bands, ATR, volume ratio, swing S/R.
- Signals are blended into a buy vs sell probability, a confidence score, and a clarity label.

## 5) Buy/Sell setup levels
- The app continuously updates Buy/Sell setups from live data.
- Levels shown in the Buy/Sell setup cards always change as new data arrives.

These are informational and update every refresh.

## 6) What counts as a recommendation
A recommendation is created only when the algo is strong enough and the entry is near.

Reliability gates:
- Confidence >= 65.
- Dominant side probability >= 60.
- Clarity is Clear or Very clear.
- Entry is close to current price (within ENTRY_PCT = 0.12%).
- Target 1 must still be reachable.

If all gates pass and no active recommendation exists, the app stores a new recommendation in Supabase with outcome = watch.

## 7) Watch vs Pending vs Outcomes
- watch: setup is valid, but entry has NOT been touched yet.
- pending: entry has been touched on a completed candle, trade is live.
- t1, t2, sl: targets or stop loss hit.
- expired: recommendation aged out or moved away without triggering.

Only one active recommendation (watch or pending) is shown at a time in the Trade Recommendation area.

## 8) Entry and rapid moves
Entry is considered touched when a completed candle crosses the entry price:
- Buy: candle low <= entry
- Sell: candle high >= entry

Rapid moves count because the check uses completed candle high/low, not just a live tick.

## 9) Validity window and staleness
- A watch recommendation expires after 3 hours (REC_VALID_MS).
- If the price feed is stale for more than 10 minutes, the record is flagged as stale and is not expired until data resumes.

## 10) Stop-loss cooldown
After a stop-loss (sl) outcome in a direction, new recommendations in the same direction are blocked for 30 minutes.
This cooldown is derived from Supabase history (no localStorage).

## 11) Supabase storage
All important data is stored in Supabase table scalper_signals.

Core columns:
- id, market, dir, price, entry, t1, t2, s1, s2
- buy_p, sell_p, conf, clarity
- ts, ts_ms, outcome, dp, entered

Extended columns (now used):
- valid_until_ms
- entry_touched_ms
- data_source
- chart_source
- session_score
- sess_line
- algo_version
- last_price
- last_price_at
- stale

The app reads and updates these fields on every refresh.

## 12) UI behavior summary
- Buy/Sell setup cards always update with live data.
- Trade Recommendation shows only the single best active recommendation.
- Open Position card only shows when entry is executed (pending + entered = true).
- History shows only resolved trades (t1, t2, sl, expired).

## 13) Timers
- BTC refresh: every 30 seconds.
- Gold/Silver refresh: every 60 seconds.
- History UI refresh: every 30 seconds.
- Outcome checks: every 5 minutes (uses completed candles).

## 14) Key constants
- ENTRY_PCT = 0.0012 (0.12% near-entry window)
- REC_VALID_MS = 3 hours
- DATA_STALE_MS = 10 minutes
- SL_COOLDOWN_MS = 30 minutes

If you want these tuned, adjust the constants in index.html.

## 15) Common reasons a recommendation does not show
- Confidence or clarity is below the minimum.
- Price is too far from entry (outside 0.12%).
- There is already an active recommendation (watch or pending).
- The data feed is stale (no recent live price updates).

---
If you want any behavior changed, update the constants or the gating rules in index.html.
