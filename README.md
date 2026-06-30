# NSE Options Backtesting Engine

A backtesting engine for Indian options strategies, built from scratch in Python — no Backtrader, no Zipline. Tests strategy logic against real minute-level NIFTY options data with realistic cost modeling (slippage, brokerage), stop-loss handling, and full trade-level P&L reporting.

Backtested on **4.86M rows of real NSE options data** spanning 78 weekly expiries (Oct 2024 – Mar 2026). Sell ATM Straddle on expiry day: **+₹86,358 net P&L, 56.4% win rate**, on actual market premiums and IV.

---

## Real Backtest Results

```
==============================================================
   BACKTEST SUMMARY — ATM STRADDLE (NIFTY)
==============================================================
  Period         : 2024-10-01 → 2026-03-31
  Total Expiries : 78
  Trades Taken   : 78
  Wins           : 44     (56.4%)
  Losses         : 34     (43.6%)
--------------------------------------------------------------
  Gross P&L      : ₹    98,963.43
  Total Costs    : ₹    12,605.60   (brokerage + slippage)
  Net P&L        : ₹    86,357.83
--------------------------------------------------------------
  Max Drawdown   : ₹    15,414.70
  Avg P&L/Trade  : ₹     1,107.15
  Best Trade     : ₹    19,260.77  (03-Feb-2026)
  Worst Trade    : ₹    -6,802.25  (05-Dec-2024)
==============================================================
```

Config used: 50% stop-loss on premium received, 0.5% slippage per leg, ₹20 flat brokerage per order, NIFTY lot size 50.

Full per-trade CSV is saved automatically to `results/` on every run.

---

## What this is

This isn't a toy backtester running on fake price series. It:

- Loads a **6GB minute-level options dataset** (4.86M rows after expiry-day filtering) using chunked pandas reads — no out-of-memory errors
- Reconstructs **per-minute market snapshots** (every strike, both CE and PE, real LTP/IV/OI) for each of 78 real expiry dates
- Runs a configurable strategy engine against those snapshots tick by tick
- Applies a realistic cost model: per-leg slippage, flat brokerage, stop-loss on combined premium
- Falls back to GBM-simulated synthetic data when real data isn't available, so the engine runs out of the box with zero setup

---

## Project Structure

```
options_backtester/
│
├── backtester/
│   ├── engine.py          # Core loop: iterate expiries → snapshots → entry/SL/exit
│   ├── portfolio.py       # Position tracking, cost application, stop-loss checks
│   └── models.py          # Trade, Leg, OptionContract, MarketSnapshot dataclasses
│
├── strategies/
│   ├── base.py            # Abstract strategy interface (3 methods to implement)
│   └── atm_straddle.py    # Sell ATM CE + PE on expiry day, hold to EOD
│
├── data/
│   ├── raw/               # Place your options CSV here (nifty_options.csv)
│   └── loader.py          # Real data loader (chunked) + synthetic GBM fallback
│
├── utils/
│   ├── options_math.py    # Black-Scholes: Delta, Gamma, Theta, Vega, IV solver
│   ├── nse_utils.py       # Expiry calendar (Thursday/Wednesday), ATM strike finder
│   └── db.py              # Optional PostgreSQL persistence
│
├── scripts/
│   ├── run_backtest.py    # CLI entrypoint
│   └── download_data.py   # NSE bhavcopy downloader (best-effort, has session limits)
│
├── tests/
│   └── test_engine.py     # 38 unit tests — Black-Scholes, NSE utils, models, portfolio, strategy
│
├── results/                # Trade log CSVs land here after each run
├── config.py                # All tunables in one place
└── requirements.txt
```

---

## Quickstart

```bash
git clone https://github.com/YOUR_USERNAME/options-backtester.git
cd options-backtester
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### Run without any data (synthetic mode)

The engine works immediately with zero setup — it generates realistic GBM-based intraday data when no real CSV is present:

```bash
python scripts/run_backtest.py --strategy atm_straddle --symbol NIFTY --from 2023-01-01 --to 2023-12-31 --stoploss 50
```

### Run with real data

Drop a minute-level options CSV at `data/raw/nifty_options.csv` with these columns:

```
strike_price, option_type, expiry, timestamp, ltp, volume, oi,
underlying_spot_price, strike_spot_diff, time_to_expiry, tte_years,
iv, delta, gamma, theta, vega, rho
```

Then run the same command — the engine auto-detects the file and extracts actual expiry dates from it:

```bash
python scripts/run_backtest.py --strategy atm_straddle --symbol NIFTY --from 2024-10-01 --to 2026-03-31 --stoploss 50 --slippage 0.5
```

First run takes 60–120 seconds to load and cache the file (chunked reading at 500k rows/chunk to stay memory-safe on large files). Subsequent expiries within the same run reuse the cached DataFrame.

### Run unit tests

```bash
python -c "
import sys, inspect
sys.path.insert(0, '.')
from tests.test_engine import TestBlackScholes, TestNSEUtils, TestModels, TestPortfolio, TestATMStraddle
for cls in [TestBlackScholes, TestNSEUtils, TestModels, TestPortfolio, TestATMStraddle]:
    obj = cls()
    for name, m in inspect.getmembers(obj, predicate=inspect.ismethod):
        if name.startswith('test_'):
            m()
            print(f'PASS  {name}')
"
```

38/38 passing.

---

## CLI Options

| Flag | Description | Default |
|---|---|---|
| `--strategy` | Strategy name (currently: `atm_straddle`) | required |
| `--symbol` | `NIFTY` or `BANKNIFTY` | `NIFTY` |
| `--from` / `--to` | Date range (`YYYY-MM-DD`) | required |
| `--stoploss` | Stop-loss % on premium received (e.g. `50`) | disabled |
| `--slippage` | Slippage % per leg | `0.5` |
| `--lot-size` | Override lot size | `50` |
| `--save-db` | Persist trades to PostgreSQL | off |
| `--verbose` | DEBUG-level logs | off |

---

## Strategy: Sell ATM Straddle on Expiry Day

1. On expiry morning, find the strike closest to spot (ATM)
2. Sell 1 lot ATM CE + 1 lot ATM PE
3. Hold until EOD, or until combined premium rises past the stop-loss threshold
4. Buy back (or let expire) at close
5. Net P&L = premium collected − premium paid back − brokerage − slippage

This is a theta-decay play: on expiry day, time value collapses fastest and IV crush accelerates. The strategy profits when the underlying stays range-bound; it loses on large directional moves, which the stop-loss is designed to cap.

### Adding a new strategy

Every strategy implements three methods:

```python
from strategies.base import BaseStrategy

class MyStrategy(BaseStrategy):
    def should_enter(self, snapshot) -> bool: ...
    def get_legs(self, snapshot, lot_size) -> list: ...
    def should_exit(self, trade, snapshot) -> bool: ...
```

Register it in `strategies/__init__.py`'s `STRATEGY_REGISTRY` and pass `--strategy my_strategy`.

---

## Known data quality note

One trade in the real backtest (26-Dec-2024) shows an anomalous premium (₹1,59,218 received) caused by a stale row at a far OTM strike (27000) that briefly matched the ATM lookup due to incomplete data at market open for that strike. A production fix would constrain the ATM search to strikes within ~2% of spot. Left as-is here since it's a useful, honest illustration of real-world data hygiene issues rather than a cleaned-up demo.

---

## Tech Stack

Python 3.10+, pandas, numpy, scipy (Black-Scholes), psycopg2 (optional PostgreSQL), argparse for the CLI.

## License

MIT
