# ETF Arbitrage Backtesting System

A Python backtesting framework for a statistical arbitrage strategy between a synthetic ETF and QQQ.
## Strategy

The core idea is that QQQ should closely track a synthetic ETF constructed from its own top holdings. When they diverge, a mean-reversion trade is placed:

- **Construct** a synthetic ETF by rebasing 15 top QQQ stocks to 100 at t₀ and taking an equal-weighted average
- **Compute** the spread between the synthetic and real ETF (QQQ, also rebased)
- **Enter** when spread exceeds ±2.5× rolling standard deviation (20-day window)
- **Exit** when spread reverts to within 30% of the threshold, or stop-loss triggers at 2× entry spread

## Results (2020–2026)

| Metric | Value |
|---|---|
| Total Trades | 10 |
| Win Rate | 70% |
| Total Net P&L | +$10,144 |
| Sharpe Ratio | 0.70 |
| Max Drawdown | -$2,459 |

**Train/test split** (70/30) produced consistent Sharpe ratios of 0.71 vs 0.65, suggesting the strategy is not overfit.

## Key Learnings during writing the code

**Problem 1 — Spread was $600–$1000** because raw stock prices (AMZN ~$1,900, GOOGL ~$1,300) were being averaged directly against a rebased $100 ETF. Fix: rebase all stocks to 100 at t₀ before computing the synthetic.

**Problem 2 — 621 trades, $61k commission destroying $5k gross profit.** Threshold of 1.5× std was too sensitive, firing on noise every other day. Fix: raised to 2.5× std and added a 3-day minimum holding period.

**Problem 3 — Arbitrary position sizing** (10 shares vs 2 shares) made P&L metrics inconsistent. Fix: switched to fixed dollar notional ($5,000 per trade) so every trade risks the same amount regardless of price.

**Problem 4 — Stop-loss sign error for SHORT positions.** Multiplying a negative entry spread by 2.0 gave a condition that never triggered. Fix: `spread < -(abs(spread_at_entry) * 2.0)`.

## Project Structure

```
faang_arbitrage_backtest.py
├── fetch_data()             # yfinance download
├── build_synthetic_etf()    # rebase + equal-weight average
├── build_real_etf()         # rebase QQQ to 100
├── compute_spread()         # synthetic minus real
├── compute_dynamic_threshold() # rolling std * multiplier
├── Position                 # single trade tracker
├── BacktestEngine           # day-by-day simulation engine
│   ├── run()                # entry/exit logic loop
│   └── summary()           # performance metrics
├── plot_results()           # 4-panel matplotlib dashboard
└── run_train_test()         # 70/30 overfitting check
```

## Usage

```bash
pip install yfinance pandas numpy matplotlib
python faang_arbitrage_backtest.py
```

To run the train/test overfitting check, uncomment `run_train_test()` at the bottom of the file.

## Configuration

All parameters are in the `CONFIG` dictionary at the top of the file:

```python
CONFIG = {
    "threshold_mult":   2,   # higher = fewer, higher-conviction trades
    "threshold_window": 20,    # rolling window for volatility estimation
    "commission_pct":   0.005, # 0.5% per trade to test in harsh conditions (it is usually around 01.%)
    "dollar_notional":  10000,  # fixed $ exposure per trade
    "min_hold_days":    3,     # minimum days before exit allowed
}
```
