# Closing Auction Imbalances: Strategy Test + Closing Cost Alert

**Gator Quant Hacks 2026 · Systematic Trading track · Data: Databento only**

## The problem

Every day at 4:00 PM, US stocks set their official closing price in an auction. Index funds and
other benchmarked investors *must* trade there, and when their orders don't balance, the
exchange publishes an **imbalance** from about 3:50 PM. Too many buyers pushes the close up; too
many sellers pushes it down. Funds forced to trade on the crowded side quietly pay for it.

We asked two questions:
1. **Can a trader profit by taking the other side** of big imbalances and exiting at the next open?
2. **Can we warn a fund, at 3:55 PM, how much the close will cost them?**

## Data

- Nasdaq closing-auction imbalance messages, 3:45–4:01 PM ET (Databento `XNAS.ITCH`, schema `imbalance`)
- Official daily open/close/volume (Databento `EQUS.SUMMARY`, `ohlcv-1d`)
- 100 large Nasdaq-listed stocks, Oct 2024 – Sep 2026 (~49,500 stock-days)
- **Year 1 (Oct 2024 – Sep 2025) = train. Year 2 (Oct 2025 – Sep 2026) = test, used once.**

All returns are **market-adjusted** (stock move minus the average of all 100 stocks that night),
so a market-wide rally is never counted as skill.

## How to reproduce

```bash
pip install databento pandas pyarrow python-dotenv matplotlib streamlit
# put DATABENTO_API_KEY=... in a .env file
python download_data.py          # check cost only
python download_data.py --go     # download (~$30)
python build_table.py            # -> data/features.parquet (one row per stock per day)
jupyter notebook 01_check_effect.ipynb   # research on year 1 only
python backtest.py               # frozen rules, train + test results
python models.py                 # push model (train on year 1, score on year 2)
python make_charts.py            # final figures
streamlit run cost_alert.py      # demo app
```

| File | What it does |
|---|---|
| `download_data.py` | Downloads imbalance messages and daily bars from Databento |
| `build_table.py` | Shrinks ~500 MB of per-second data into one row per stock per day |
| `01_check_effect.ipynb` | Year-1 research: does the effect exist, where, and which filters help |
| `backtest.py` | Runs the frozen strategy on both years; Sharpe, deflated Sharpe, drawdown |
| `models.py` | Predicts the 3:55 PM → close price push; tested on year 2 |
| `make_charts.py` | Figures for the write-up |
| `cost_alert.py` | Streamlit app: "your closing order will cost about X bps today" |

## Part 1: The trading strategy

**Rules (chosen on year 1, then frozen):** at 3:55 PM, take stocks whose imbalance is at least
**6.69% of normal daily dollar volume** (year-1 95th percentile). Skip any whose imbalance
**flipped direction** since 3:50; keep only those whose imbalance **grew**. Trade against the
imbalance at the close, exit at the next open. Cost: 3 bps per round trip.

| | Train (rules chosen here) | **Test (never seen)** |
|---|---|---|
| Trades | 769 | 701 |
| Net profit per trade | 9.75 bps | **4.33 bps** |
| Sharpe | 2.48 | **0.84** |
| t-stat | 2.41 | **0.83** |
| Max drawdown | −469 bps | **−842 bps** |
| Deflated Sharpe probability (20 trials) | 0.76 | **0.11** |

**Verdict: the strategy did not pass the out-of-sample test.** Its test-year profit came almost
entirely from one night: on **5 March 2026** a single trade (MRVL) gained ~1,330 bps, most
likely an after-hours earnings move unrelated to the auction. Without that night the test year
is roughly flat.

What we learned along the way:
- **Raw backtests were mostly market beta.** Before market-adjusting, the strategy looked far
  better; most of that was the market rising overnight.
- **Single-stock event risk dominates.** On 46 test days only one stock qualified, so one
  earnings surprise could swing the year.
- **The filter helps in both years:** test-year profit per trade 4.33 vs 3.68 bps without it,
  and the worst drawdown fell from −1,405 to −842 bps.
- **Sell-side imbalances reversed more than buy-side** (test: +18.5 vs −11.4 bps per trade).
  This was noticed after the test, so it is a hypothesis for new data, not a result.

## Part 2: The Closing Cost Alert

The **push** (3:55 PM → 4:00 PM) is measured *before* the close, so after-hours earnings news
cannot contaminate it. It held up in both years.

| Biggest 10% of imbalances | Train | **Test (never seen)** |
|---|---|---|
| Extra cost paid by crowded-side orders | +2.63 bps (t = 5.6, n = 2,356) | **+2.07 bps (t = 3.2, n = 2,109)** |
| Push model gets the direction right | 57% | **57%** |
| Correlation, predicted vs actual push | 0.13 | **0.11** |

- On a **$10M** closing order on the crowded side, that's about **$2,000 per order**, and large
  funds place closing orders across many stocks every day.
- The effect is concentrated in the largest imbalances; small imbalances carry no signal, so the
  app only alerts above a size threshold.
- The model is deliberately simple (linear, three inputs known at 3:55 PM). Its two size inputs
  overlap, so individual coefficients aren't meaningful on their own; only the combined
  prediction is.

The app shows, for any stock and date: imbalance side and size, predicted push, whether *your*
order is on the crowded side, the expected cost in bps and dollars, and the most crowded closes
across all 100 stocks.

## Limitations

- Stock list is today's large Nasdaq names (mild survivorship bias); NYSE stocks not included.
- Holidays and half-days excluded.
- No earnings calendar in our permitted data, so earnings nights could not be filtered out.
- 3:55 PM decision time and 3 bps cost are assumptions; real auction order-entry cutoffs and
  costs depend on order type and broker.
- The push model is a simple linear model; five-minute price moves are noisy, so it is useful on
  average, not as a guarantee for any single day.

## What we'd do next

Cap the weight of any single trade, exclude known earnings nights, test the sell-side asymmetry
on fresh data, and add NYSE closing-auction data.
