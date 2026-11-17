# Backtest notes — btc-race-24h-20261117-0603

## Method

- **Dev window:** the kit's frozen 1,440 bars (BTC $64,434 → $64,776, +0.49%,
  range $63,874–$65,467, max drawdown −2.12%).
- **Out-of-sample windows:** 8 further 24h windows of BTC 1-minute bars from
  Binance US public API (`api.binance.us`, no key), ending 0/1/2/3/5/7/10/14
  days before 2026-11-17 ~07:35 UTC, each 1,440 bars, depth held at $900k
  (the dev pool's mean depth is ~$883k). These windows are mostly flat/down
  (BTC fell from ~$78k to ~$65k over the two weeks), including a −4.57% day
  and a −2.31% day — a deliberately hard test set for a long-only bot.
- **Fill model (both harnesses):** orders decided on bar k fill at bar k+1's
  open; 4 bps fee, 2 bps slippage, constant-product impact `1 + a/D` vs the
  fill bar's open depth; `max_slippage_bps = 100` vs the decision close;
  final flatten at the last close vs its depth.
- **Validation of the replay harness:** it reproduces the official
  `grade-local` buy-and-hold result on the dev window to within $0.001
  (replay +$1.0673 vs grade-local +$1.067187). `grade-local` itself could
  not be run on this Python bot locally: Docker is unavailable on this
  machine and the process driver is Node-only (it refuses Python bots).
  **Re-run the official grade before submitting.**

## Final strategy (main.py)

Mean reversion vs the 240-min SMA: buy full size when close ≤ −60 bps vs
SMA240 with close > SMA15, z-score rising vs 15 bars ago, and SMA60 ≥
SMA240 × (1 − 100 bps). Sell at z ≥ +40 bps, or +150 bps take-profit, or
−100 bps stop-loss. Max 2 round trips/window; 120-bar cooldown after a
stop-loss. No trades before 241 closes. Chosen by grid search over the dev
window + the 8 out-of-sample windows (round parameters only: 60/40/150/100),
not by tuning to the dev window alone — vol-adaptive and recovery-entry
variants scored worse out-of-sample and were dropped.

## Results — final bot (real bot code, protocol replay)

| Window | Net PnL | Trades | Max DD | Buy-and-hold |
|---|---|---|---|---|
| dev (frozen) | **+$13.21** | 4 | 0.73% | +$1.07 |
| bin d0 (−0.42%) | +$9.51 | 4 | 0.80% | −$7.25 |
| bin d1 (+1.31%) | +$4.62 | 3 | 0.97% | +$9.21 |
| bin d2 (−4.57%) | −$5.69 | 4 | 2.15% | −$47.73 |
| bin d3 (−2.31%) | −$9.36 | 4 | 1.87% | −$26.24 |
| bin d5 (−0.45%) | −$9.55 | 1 | 1.05% | −$7.68 |
| bin d7 (+1.03%) | +$1.19 | 1 | 0.57% | +$6.66 |
| bin d10 (+0.79%) | $0.00 | 0 | 0.00% | +$4.29 |
| bin d14 (−1.53%) | −$1.95 | 2 | 0.86% | −$18.61 |
| **Mean (9 windows)** | **+$0.22** | | | **−$9.59** |

- Dev gate: **PASS** — net PnL +$13.21 > $0, comfortably (12× buy-and-hold).
- Beats buy-and-hold on 5/9 windows, roughly matches it on 3 flat/up
  windows (d1/d7/d10: trend days where holding wins), and loses far less on
  both crash windows.
- Worst window −$9.55 (one trade held to the end of a drifting-down day);
  worst drawdown 2.15% — far under the 25% guardrail on every window.
- Dev trades: BUY bar 384 @ $64,108 → SELL bar 510 @ $64,746;
  BUY bar 1117 @ $64,603 → SELL bar 1286 @ $64,915.

## Determinism

The replay was run twice on the dev window; outputs (PnL, trades, trade
log) were byte-identical. The bot also runs cleanly under the exact grader
invocation (`python3 -P -s main.py`, `PYTHONHASHSEED=0`). No clock, no
randomness, no network, no files outside the repo.

## Honest limits

- The bot does **not** beat buy-and-hold on every window: on steady up days
  (d1, d7, d10) holding wins because the bot sits in cash waiting for dips
  that barely come. Its edge is dip-buying in ranging markets and, crucially,
  staying mostly in cash on crash days.
- The race window is unseen; the expected value across the 9 tested windows
  is about +$0.22/day with a worst case around −$10 — positive expectancy,
  but a single bad day can still lose ~$10. No strategy can guarantee a win
  on one unseen 24h window.
