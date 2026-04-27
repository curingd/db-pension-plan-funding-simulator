# DB Pension Plan Funding Simulator

A bootstrap-based simulator for UK defined benefit (DB) pension plans. Generates 10-year horizon distributions for the funding ratio (FR) and funding surplus/deficit (FS/D), and computes monthly value-at-risk (VaR) and conditional VaR (CVaR) statistics for plan portfolios using typical asset allocations, portfolio values and s179 liabilities from the Pension Protection Fund (PPF) universe in 2011 and 2025.

Built in Excel/VBA as a self-contained `.xlsm`. The model is designed to be transparent — every input, weight, and proxy series is visible on a worksheet — so methodological choices can be inspected, challenged, and modified without leaving the file.

## What it does

- **Reconstructs typical PPF plan asset allocations** for 2011 and 2025 from the Purple Book aggregate data, with a bottom-up disaggregation across cash, gilts (short/medium/long fixed and index-linked), corporate bonds, private debt, equities (UK/world/private), real estate, and hedge funds.
- **Models liabilities** as a weighted mix of nominal and index-linked gilts, with a parameterised maturity profile (immature, mature, overmature).
- **Bootstraps 10-year FR and FS/D distributions** using IID monthly resampling of historical asset and liability returns from June 2013 to December 2025 (144 monthly observations).
- **Reports portfolio statistics** — mean change, standard deviation, skewness, kurtosis, VaR, CVaR, and maximum drawdown — at user-specified confidence levels.
- **Scans VaR/CVaR across funding ratio levels** for sensitivity analysis.

## Methodological choices

### Asset class proxies

| Class | Proxy | Source |
|---|---|---|
| Cash | SONIA | Bank of England |
| Short/medium/long nominal gilts | BoE 3Y/10Y/20Y spot | Bank of England yield curves |
| Short/medium/long IL gilts | BoE 3Y/10Y/20Y implied real spot | Bank of England yield curves |
| Corporate bonds (IG) | iShares Core £ Corp Bond ETF (SLXX) | FMP, dividend-adjusted |
| Private debt | iShares Global HY Corp Bond ETF GBP-Hedged (GHYS.L), unsmoothed | FMP, dividend-adjusted |
| UK equities | iShares Core FTSE 100 UCITS ETF (CUKX) | FMP, dividend-adjusted |
| World equities | iShares MSCI World UCITS ETF (SWDA) | FMP, dividend-adjusted |
| Private equity | iShares Listed Private Equity UCITS ETF (IPRV), resmoothed | FMP, dividend-adjusted |
| Real estate | iShares UK Property UCITS ETF (IUKP), resmoothed | FMP, dividend-adjusted |
| Hedge funds | IQ Hedge Multi-Strategy Tracker (QAI) + (SONIA − EFFR) basis | FMP, dividend-adjusted, BoE, FRED |

### Smoothing and unsmoothing

For private debt, where the GHYS.L proxy already reflects daily-traded ETF pricing but the underlying private debt market is appraisal-based, the series is *unsmoothed* using the Geltner (1991) AR(1) inversion:

```
R_unsmoothed_t = (R_observed_t − α · R_observed_{t-1}) / (1 − α)
```

with default α = 0.6 for private debt.

For private equity, real estate, and hedge funds, where the ETF proxies are daily-traded but the equivalent private exposures exhibit appraisal smoothing, the inverse operation is applied (an MA(1)-style filter that adds autocorrelation):

```
R_smoothed_t = α · R_smoothed_{t-1} + (1 − α) · R_observed_t
```

with default α = 0.2.

All four α values are user-configurable from the Summary sheet for sensitivity analysis.

### Hedge fund GBP-hedge construction

QAI is a USD-denominated ETF tracking a hedge fund replication index. Returns are converted to a GBP-hedged equivalent using a static rate-differential adjustment:

```
R_GBP-hedged ≈ R_USD + (SONIA − EFFR)
```

This captures the bulk of the carry adjustment under a monthly-rolled forward hedge but does not reflect FX basis spread variation.

### Liability model

Liabilities are modelled as a mix of nominal and index-linked gilt returns, with short/medium/long durations corresponding to the proportions of retired/deferred/active members in schemes open (immature), closed to new members (mature) and closed to new benefit accrual (overmature). As with the asset allocation weights, liability weights can also be overridden.

## Limitations and caveats

A few are worth understanding before using the model:

- **The bootstrap is IID and discards autocorrelation.** Geltner unsmoothing and MA(1) resmoothing are applied to the underlying return series, which preserves their marginal monthly distributions. The bootstrap then resamples months independently, so the *persistence* of those returns is not preserved in the simulated 10-year paths. A block bootstrap (or stationary bootstrap) would address this, but is beyond the scope of this project due to the complexity of implementation and the sensitivity of the optimal block length to allocation adjustments. See Politis & White (2004) for more detail.
- **The 2025 cash allocation is reported as negative (−7.9%) in the PPF data**, reflecting unfunded LDI exposure on schemes' balance sheets. Modelling this as a literal short-cash position approximates but does not exactly replicate the LDI mechanics.
- **The QAI GBP-hedged construction omits FX basis variation.** For a more accurate hedged-return series, NDF or rolling forward returns net of basis would be required.
- **The September 2022 LDI gilt crisis is in-sample.** This is intentional (it is a real and recent stress event) but produces materially leptokurtic 10-year FR distributions for bond-heavy 2025 allocations. Comparisons with 2011 allocations should be interpreted with this in mind.

## Workbook structure

- **Summary** — primary control panel. Asset and liability presets via dropdown, manual override of asset class weights, Geltner α inputs, portfolio statistics display.
- **95% FS Monthly VaR/CVaR per FR** — VaR/CVaR scan across funding ratio levels for the 2025 overmature plan.
- **PPF 2011 & 2025 Allocations** — preset weights derived from the PPF Purple Book, with disaggregation notes.
- **Asset Class Returns & Weights** — historical monthly return series, smoothing/unsmoothing transforms, portfolio-level return aggregation, and base-period statistics.
- **(Bootstrap Simulation)** — 120-month IID resampling block plus 1,000-iteration result series. The `SimulationReturnsUpdate` macro recalculates the resample and pastes terminal FR and FS/D values to columns O and P.

## Usage

1. Open in Excel (macros must be enabled — file is `.xlsm`). LibreOffice and Google Sheets are not supported because the model uses VBA-driven preset toggles.
2. On the Summary sheet, select an asset preset (2011 Average / 2025 Average / Manual) and a liability preset (Immature / Mature / Overmature). Manual entry cells are gold-shaded.
3. Adjust Geltner α values if desired.
4. Run the `SimulationReturnsUpdate` macro by pressing the `UPDATE SIMULATIONS` button to regenerate the 10-year horizon distribution. Histograms and statistics on the Summary sheet update automatically.

## References

Geltner, D.M. (1991) 'Smoothing in appraisal-based returns', *Journal of Real Estate Finance and Economics*, 4(3), pp. 327–345.

Getmansky, M., Lo, A.W. and Makarov, I. (2004) 'An econometric model of serial correlation and illiquidity in hedge fund returns', *Journal of Financial Economics*, 74(3), pp. 529–609.

Pension Protection Fund (2025) *The Purple Book 2025: DB pensions universe risk profile*. Croydon: Pension Protection Fund. Available at: https://www.ppf.co.uk/-/media/PPF-Website/Public/Purple-Book-2025/Pension-Protection-Fund-Purple-Book-2025-accessible.pdf (Accessed: 26 April 2026).

Pension Protection Fund and The Pensions Regulator (2011) *The Purple Book: DB pensions universe risk profile 2011*. Croydon and Brighton: Pension Protection Fund and The Pensions Regulator. Available at: https://www.ppf.co.uk/sites/default/files/2024-12/purple_book_2011.pdf (Accessed: 26 April 2026).

Politis, D.N. and White, H. (2004) 'Automatic block-length selection for the dependent bootstrap', Econometric Reviews, 23(1), pp. 53–70.

## Author

David Curington · 2026

