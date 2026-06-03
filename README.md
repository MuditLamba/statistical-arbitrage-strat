# Crypto Statistical Arbitrage

Market-neutral cross-sectional strategy on Binance.US USDT pairs. Combines two anti-correlated signals (volume-confirmed reversal + short-horizon momentum) under global minimum variance weighting.

## Headline result

| Metric | Full | IS | OOS |
|---|---|---|---|
| Net Sharpe | **1.43** | 0.31 | 3.87 |
| Ann. return (net) | 19.5% | 4.7% | 47.8% |
| Ann. vol | 13.7% | 15.1% | 12.4% |
| Max drawdown | -33% | -33% | -10% |
| Beta to BTC | 0.006 | 0.008 | 0.002 |
| Alpha (annualized) | 19% | 4% | 48% |

- 20bps assumed transaction cost (market execution).
- 4h rebalance.
- IS = 2020-03-01 to 2023-12-31. OOS = 2024-01-01 onward.
- Committed weights: w_A ≈ 0.54, w_B ≈ 0.46.

The OOS Sharpe of 3.87 is not a forward expectation. The OOS window oversamples low-volatility periods (88% of OOS bars vs 65% full-sample). Forward Sharpe estimate is **1.0 to 1.5**, with the strategy positive in both low-vol (1.41) and high-vol (0.94) sub-samples.

## Strategy

### Signals built

| Signal | Mechanism | Status in v1 |
|---|---|---|
| A: Volume-confirmed XS reversal | Liquidity / uninformed flow overreaction | Committed (~54%) |
| B: Short-horizon XS momentum | Behavioral underreaction on large caps | Committed (~46%) |
| C: IVOL anomaly on residualized returns | Lottery / overpricing | Sensitivity only, dropped from v1 |
| D: BTC intraday seasonality (Mon+Tue 20 UTC) | Time-series calendar effect | Research stage |

### Universe

- Top 50 USDT pairs by trailing 30-day average daily dollar volume on Binance.US.
- A coin can enter at top 50, but stays in until top 70 to avoid boundary churn. We're optimizing to reduce t-costs.
- Monthly rebalance on universe membership. Portfolio rebalances per signal.
- Stablecoins, wrapped tokens, and Binance leveraged tokens excluded.
- Minimum 180 days of price history per coin.

### Portfolio construction

I committed to 6 weighting schemes: equal-weight, inverse-vol (eqvol), GMV, mean-variance optimal, sr-weighted (clipped meaning weights normalized), sr-weighted (unclipped so short negative signal). Three use sample mean (mu); three do not. The mu-based schemes all underperform OOS because Signal A's IS mean is negative (it's a regime-contingent signal that loses in high-vol periods, which dominate IS). GMV wins both on full-sample Sharpe and on robustness to mean-estimation error. Committed scheme is GMV on the A+B pair.

## Repo structure

```
cryptoStatArbProject/
├── data/                              raw + processed panels, signal artifacts
├── notebooks/
│   ├── infrastructure/
│   │   ├── DataPipeline.ipynb         pull OHLCV, build panels, build universe
│   │   └── ResidualReturns.ipynb      BTC-residualized returns (input to Signal C)
│   ├── signals/
│   │   ├── SignalA_VolumeReversal.ipynb
│   │   ├── SignalB_MomentumXS.ipynb
│   │   ├── SignalC_IVOLAnomaly.ipynb
│   │   └── SignalD_IntraDayMomentum_Research.ipynb
│   └── combined/
│       └── CombinedPortfolio.ipynb    final result, sensitivity analysis, conclusion
├── quantlib/                          shared utilities (returns, weights, stats)
├── PROJECT_PLAN.md                    full research log and rationale
└── README.md
```

### What each notebook does

**`infrastructure/DataPipeline.ipynb`** — pulls 4h OHLCV bars for ~193 USDT Binance.US pairs from 2019 via binance's API, excludes stablecoins, wrapped tokens, leveraged tokens, and aligns the per-coin DataFrames into panels: `px`, `ret`, `dvol`, `taker_buy_dvol`, `num_trades`, `high_px`, `low_px`. Builds a monthly-rebalanced universe matrix using rank-based liquidity filtering with our boundary rules described above. Saves everything as a single pickle (`data/binance_ohlcv_panel_4h.pkl`) consumed by every downstream notebook.

**`infrastructure/ResidualReturns.ipynb`** — strips BTC beta from each coin's return series using rolling 30-day and 60-day regression windows. Output is a residual returns panel that Signal C uses. The 30-day window is committed as primary based on a pre-registered sensitivity check.

**`signals/SignalA_VolumeReversal.ipynb`** — cross-sectional reversal on trailing 1-week return, with a BTC volatility regime gate (signal off when BTC 30d annualized vol > 0.6). Committed signal uses no volume gate. Standalone full Sharpe 1.17 net of slippage, trading costs, execution, etc. Regime-sensitive.

**`signals/SignalB_MomentumXS.ipynb`** — cross-sectional momentum on top-20 large cap coins, 1-week formation horizon with skip-1, 3-day rebalance. Standalone weak (full Sharpe 0.44 net) but anti-correlated with Signal A (-0.36 full), which makes it valuable in the combined book.

**`signals/SignalC_IVOLAnomaly.ipynb`** — long high-IVOL, short low-IVOL on top-30 universe, 3-day rebalance. Standalone full Sharpe 0.29 net. There is an IS/OOS sign flip in the IVOL-return relationship. Dropped from v1 because corr(C, A) ≈ 0 means it doesn't hedge the dominant signal, and corr(C, B) is +0.20 IS, so it correlates with B rather than diversifying.

**`signals/SignalD_IntraDayMomentum_Research.ipynb`** — research on BTC time-series seasonality (Mon+Tue 20 UTC bar). Gross SR ~1.4 full-sample with six consecutive positive years, but net-negative at 20bps because the strategy is highly turnover-intensive. Research stage; not in v1.

**`combined/CombinedPortfolio.ipynb`** — loads the three Tier 1 signal artifacts, computes the six committed weighting schemes, evaluates IS/OOS/full performance, runs a BTC factor regression on the committed book, and saves the v1 artifact. Includes sensitivity analysis (drop-C, drop-B, GMV vs equal vs eqvol on 2-signal subsets) and a vol-state diagnostic.

### `quantlib/`

Reusable functions, no notebook-specific logic. Returns construction, rank-demean-normalize for dollar-neutral weights, turnover, cost application, drawdown, Sharpe and other annualized stats, rolling beta, residual returns, factor regression, and the four weighting schemes (`optimal_weights`, `eqvol_weights`, `sr_weights`, `gmv_weights`).

## Running

1. Clone the repo to `~/Desktop/PythonJupyterCode/cryptoStatArbProject` (or update the `os.chdir` line at the top of each notebook to point to your install location).
2. Install dependencies: `pandas`, `numpy`, `matplotlib`, `python-binance`.
3. Launch Jupyter with the project root as the working directory.
4. Run order: `DataPipeline.ipynb` first (one-time, ~45 min to pull data), then signal notebooks in any order, then `ResidualReturns.ipynb` before `SignalC`, then `CombinedPortfolio.ipynb` last.

The `data/` folder ships with all processed pickles included, so re-running `DataPipeline.ipynb` is optional.

## Caveats

- **Cost model.** I assumed 20 bps as my cost per transaction (t-cost) to account for slippage and execution costs. Realized slippage on Binance.US thin-alt names is likely worse, so net Sharpe in production would fall. However, liquidity and volume in Binance global is far higher and would likely result in higher net SR given lower slippage. Tighter universe (top-20) or limit-order execution (~7bps commission only) would be our options if production costs exceed simulated assumed cost.
- **Survivorship bias.** Only currently-listed pairs were pulled. Coins delisted from Binance.US during the sample window are not represented.
- **Single venue.** Binance.US is roughly 1/1000th the liquidity of main Binance. The strategy logic transfers to global Binance, but cost assumptions would need re-calibration.

## References

- Fičura, M. & Colak, G. (2024). *Impact of Size and Volume on Cryptocurrency Momentum and Reversal.* SSRN 4378429.
- Liu, Y., Tsyvinski, A. & Wu, X. (2022). *Common Risk Factors in Cryptocurrency.* Journal of Finance.
- Ang, A., Hodrick, R. J., Xing, Y. & Zhang, X. (2006). *The Cross-Section of Volatility and Expected Returns.* Journal of Finance.
- Bali, T. G., Cakici, N. & Whitelaw, R. F. (2011). *Maxing Out: Stocks as Lotteries and the Cross-Section of Expected Returns.* Journal of Financial Economics.
