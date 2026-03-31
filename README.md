# IMC Prosperity 4 Backtester

In-house backtester for [IMC Prosperity 4](https://prosperity.imc.com/) algorithms, forked from [jmerle's Prosperity 3 backtester](https://github.com/jmerle/imc-prosperity-3-backtester).

## Installation

### Prerequisites
- Python 3.9+
- [uv](https://docs.astral.sh/uv/) (recommended) or pip

### Setup
```sh
# Clone the repo
git clone https://github.com/jinfelixx/prosperity4-backtester.git
cd prosperity4-backtester

# Option A: using uv (recommended)
uv venv
uv sync

# Option B: using pip
python -m venv .venv
# Windows: .\.venv\Scripts\activate
# macOS/Linux: source .venv/bin/activate
pip install -e .
```

> **Corporate network note:** if `uv sync` fails with SSL errors, use `uv pip install -e . --system-certs` instead.

## Running the Backtester

```sh
# With uv:
uv run prosperity4bt <path to trader.py> <round> [options]

# With pip install:
prosperity4bt <path to trader.py> <round> [options]

# Or as a module (no install needed, from repo root):
python -m prosperity4bt <path to trader.py> <round> [options]
```

### Examples
```sh
# Run on all days in round 0
prosperity4bt trader.py 0

# Specific day
prosperity4bt trader.py 1-0

# Multiple rounds
prosperity4bt trader.py 0 1 2

# Multiple specific days
prosperity4bt trader.py 1-0 1-1 2-0
```

### Options
| Flag | Description |
|---|---|
| `--match-trades all` | (default) Match market trades with prices equal to or worse than your quotes |
| `--match-trades worse` | Only match market trades with prices strictly worse than your quotes |
| `--match-trades none` | Do not match market trades against your orders |
| `--print` | Print trader's stdout output while running |
| `--merge-pnl` | Merge profit and loss across multiple days |
| `--out <file>` | Save output log to a specific file |
| `--no-out` | Skip saving output log |
| `--vis` | Open results in the visualizer when done |
| `--no-progress` | Hide the progress bar |
| `--original-timestamps` | Preserve original timestamps instead of merging across days |
| `-v, --version` | Show version number |

## Adding New Products

When a new round is announced, three things may need updating: **position limits**, **data files**, and optionally **conversion observations**.

### 1. Set position limits

Edit `prosperity4bt/data.py` and add entries to the `LIMITS` dict:

```python
LIMITS: dict[str, int] = {
    "PRODUCT_A": 50,
    "PRODUCT_B": 100,
    # Add new products here as rounds are announced
}
```

Position limits are enforced before order matching. If your aggregate orders for a product would exceed the limit (long or short), **all** orders for that product are cancelled.

### 2. Add round data

Create a directory `prosperity4bt/resources/roundN/` and place the CSV files from IMC's data capsule:

```
prosperity4bt/resources/
├── round0/
│   ├── prices_round_0_day_0.csv
│   └── trades_round_0_day_0.csv
├── round1/
│   ├── prices_round_1_day_0.csv
│   ├── prices_round_1_day_-1.csv
│   ├── trades_round_1_day_0.csv
│   └── trades_round_1_day_-1.csv
```

**File formats** (semicolon-delimited):

`prices_round_N_day_D.csv`:
```
day;timestamp;product;bid_price_1;bid_volume_1;bid_price_2;bid_volume_2;bid_price_3;bid_volume_3;ask_price_1;ask_volume_1;ask_price_2;ask_volume_2;ask_price_3;ask_volume_3;mid_price;profit_and_loss
```

`trades_round_N_day_D.csv`:
```
timestamp;buyer;seller;symbol;currency;price;quantity
```

`observations_round_N_day_D.csv` (comma-delimited, only if the round has conversion products):
```
timestamp,bidPrice,askPrice,transportFees,exportTariff,importTariff,sugarPrice,sunlightIndex
```

You can also extract data from submission logs using the built-in parser:
```sh
python -m prosperity4bt.parse_submission_logs <log_file> <round> <day>
```

### 3. Conversion observations (if applicable)

If a round introduces a product with conversions, update the observation mapping in `prosperity4bt/runner.py` in the `prepare_state()` function. The product name for `conversionObservations` must match the product that supports conversions:

```python
state.observations = Observation(
    plainValueObservations={},
    conversionObservations={"YOUR_CONVERSION_PRODUCT": conversion_observation}
)
```

## Order Matching

Orders placed by `Trader.run()` at a given timestamp are matched in two phases:

1. **Order depth** (priority): your orders match against the resting book snapshot.
2. **Market trades** (fallback): if unfilled, your orders optionally match against recorded market trades.

Market trades are matched at **your order's price**, not the trade's price. This is consistent with the official Prosperity environment.

Limits are enforced **before** matching. If your aggregate buy (sell) orders for a product would push your position beyond the limit if all filled, all orders for that product are cancelled.

## Environment Variables

During backtests, two environment variables are set:
- `PROSPERITY4BT_ROUND` — the round number
- `PROSPERITY4BT_DAY` — the day number

These do **not** exist in the official submission environment. Do not depend on them in code you submit to IMC.

## Known Limitations

- **No market impact**: your orders do not affect future book states. The book at T+1 is a static recording regardless of what you did at T.
- **No bot reaction modeling**: bots in the real environment may react to your resting orders within a tick. The backtester cannot simulate this.
- **Conversions**: must be manually configured per product (see above).
- **Observation fields**: the `ConversionObservation` fields (`sugarPrice`, `sunlightIndex`, etc.) may change in Prosperity 4. Update `prosperity4bt/data.py` (`ObservationRow`) and `prosperity4bt/runner.py` (`prepare_state()`) if IMC introduces new fields.
