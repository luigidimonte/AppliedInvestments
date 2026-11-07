# AppliedInvestments

This project studies the **S&P 500 index effect**. The research question is:

> *Did the S&P 500 additions and deletions effect disappear, and is there a tradable version after costs?*

The headline analysis has two parts. A daily event study measures market-adjusted cumulative abnormal returns (CARs) around additions and deletions. Calendar-time portfolios go long deletions and short additions, and their Fama–French five-factor plus momentum alphas are reported before and after trading and borrowing costs. The **core sample is 1976–2022**, where the S&P 500 membership data are complete. Events from 2023–2025 rely on provisional, manually collected membership changes and appear only as a separately labelled robustness check.

The earlier work on predicting additions with a logit model (anticipation) is kept as an optional appendix. It is not part of the headline results.

## Notebooks

| Notebook | Role |
| --- | --- |
| [01_inventory.ipynb](notebooks/01_inventory.ipynb) | Data catalogue and **event register**: one table of additions and one of deletions, with the flags that define the main samples. Appendix A at the end builds the anticipation features and labels. |
| [02_post_inclusion.ipynb](notebooks/02_post_inclusion.ipynb) | **Headline analysis**: event study, calendar-time portfolios, costs, robustness checks, and the count of specifications. The filename is kept from the earlier prototype, whose content this notebook replaces. |
| [03_strategy_inputs.ipynb](notebooks/03_strategy_inputs.ipynb) | Optional supporting tables written to `Data/prepared/`: an additions register with S&P notice evidence, the anticipation candidate universe, daily windows around additions, and the factor CSV. Notebook 02 does not need it. |
| [04_anticipation_baseline.ipynb](notebooks/04_anticipation_baseline.ipynb) | Appendix B.1: anticipation logit, 2000–2012 training and out-of-sample test. |
| [05_anticipation_extended.ipynb](notebooks/05_anticipation_extended.ipynb) | Appendix B.2: walk-forward anticipation portfolio, 2000–2025. |
| [06_anticipation_long_short.ipynb](notebooks/06_anticipation_long_short.ipynb) | **Strategy**: long predicted additions, short predicted discretionary deletions (stocks only, no derivatives), exit at the index trade date. Logits fixed on 1976–2009, out-of-sample test 2010–2022, gross and net alphas. Runs notebook 01 first, reads the daily Parquet copy, writes nothing to disk; about one minute. |
| [07_event_backtest.ipynb](notebooks/07_event_backtest.ipynb) | **Event backtest without forecasting**: buy every main addition and short every main deletion 5, 3 or 1 sessions before the index trade date, close at its close. Per-trade and calendar-time results for 1976–2022 by sub-period, 2023–2025 as a provisional check, gross and net of spreads and borrow fees. Writes nothing to disk; under half a minute. |

## Set up the Python kernel

From the repository root on Windows, create the project environment and install its dependencies.

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

In VS Code, open a notebook and choose **Select Kernel → Python Environments → `.venv`**. The interpreter is `.venv\Scripts\python.exe` under the project root.

Notebook 02 reads the daily CRSP file faster from a Parquet copy in `Data/parquet/crsp_daily/`, which requires `pyarrow`. `pyarrow` is not listed in `requirements.txt`. Without the Parquet copy, the notebook scans the roughly 10 GB daily CSV in chunks, which is much slower.

## Run the analysis

1. Place the WRDS CSV exports, the 1975 membership workbook, the two 2023–2025 CSV files, and the Kenneth French archives `F-F_Research_Data_5_Factors_2x3_CSV.zip` and `F-F_Momentum_Factor_CSV.zip` in `Data/` at the repository root. The inventory accepts the descriptive CSV names and the original WRDS download names. Git ignores this directory so the WRDS data stay local.
2. Run [02_post_inclusion.ipynb](notebooks/02_post_inclusion.ipynb) from top to bottom. It runs notebook 01 first, needs no prepared files, and writes nothing to disk. A full run takes about half a minute with the Parquet copy.
3. To inspect the samples, run [01_inventory.ipynb](notebooks/01_inventory.ipynb) on its own. `additions_register` and `deletions_register` hold the events and flags.

The notebooks read the source files locally. Clear cell outputs before sharing a notebook.

## Design

**Event register (notebook 01).** Additions and deletions come from the membership spells. For an addition, `effective_date` is the first session in the index. For a deletion, it is the last session in the index. In the historical export, 96% of deletion end dates are followed by a replacement addition on the next session, which confirms this convention. The 2023–2025 removals file reports S&P's effective date, the first session *out* of the index. That date is moved back one session so that both sources use the same convention. The 1975 workbook supplies spells that the historical export omits: one core deletion, Ingersoll-Rand in 2009, and one 2024 removal missing from the 2023–2025 file. All flags use only information available at the event date:

- **Spin-offs and merger successors** (additions): no CRSP monthly row in the three calendar months before the effective month. Excluded from the main sample.
- **Delisting-related deletions**: a CRSP delisting before the deletion or at most 21 sessions after it, covering acquisitions, mergers, and failures. Excluded from the main sample, which keeps deletions whose stock continues to trade. The register reports counts for 5- and 63-session windows.
- **Spin-off removals** (deletions): the same CRSP-history proxy applied to deletions. It identifies 19 spin-offs that S&P added on the distribution date and removed a few days later, all in 2015–2022. This flag was added after the data were inspected. It defines a robustness check only.

Main samples, 1976–2022: **1,013 additions** and **481 deletions**.

**Event study (notebook 02).** Day 0 is the **index trading date**, the close at which index funds trade. For an addition, this is the session before the effective date; for a deletion, the last session in the index. The abnormal return is the CRSP daily total return minus the S&P 500 return, and CARs are summed over [−60, −1], [0, +20], [+1, +60], and [+1, +250]. Delisting returns are included, and the abnormal return is zero after delisting. t-statistics are clustered by effective month. Sub-periods are 1976–1989, 1990–1999, 2000–2009, and 2010–2022, and a regression tests whether 2010–2022 differs from 1990–1999. Robustness checks centre additions on the effective date, use price returns on both sides, and apply the design to 2023–2025.

**Calendar-time portfolios (notebook 02).** Each event is held for 12 months starting in the month after its effective month, so no position uses the effective-month close. The portfolios are equal weighted, using CRSP monthly returns including delisting returns. The strategies are long deletions, short additions (`RF − R_additions`), and the long–short spread. Alphas use the Fama–French five factors and momentum, with Newey–West standard errors and 12 lags.

**Costs and robustness (notebook 02).** Trading costs are applied to actual monthly turnover. The cost per side is either half the quoted CRSP bid–ask spread from the month before the trade, or an assumed 0.2%, 0.5%, or 1.0%. The short leg pays an assumed borrow fee of 0.5%, 1%, or 2% per year. Robustness checks are value weighting, holding periods of 6 and 24 months, dropping the five largest events (by size before the event), an assumed −30% for missing non-merger delisting returns, and 2023–2025.

## Main results

The figures below come from the run of 7 November 2026. Alphas are monthly percentages, with t-statistics in parentheses.

| | 1976–1989 | 1990–1999 | 2000–2009 | 2010–2022 |
| --- | --- | --- | --- | --- |
| Additions, CAR [−60, −1] | 2.1% (2.5) | 8.4% (6.8) | 11.3% (8.3) | 9.0% (8.7) |
| Additions, CAR [+1, +60] | 4.3% (4.8) | −4.2% (−2.8) | −2.3% (−1.7) | 0.2% (0.2) |
| Deletions, CAR [−60, −1] | 1.9% (0.9) | −11.6% (−2.3) | −33.0% (−4.9) | −10.7% (−4.8) |
| Deletions, CAR [+1, +60] | 0.9% (0.3) | 12.8% (3.3) | 27.4% (4.1) | 0.6% (0.2) |
| Long–short spread, gross alpha | −0.51 (−0.51) | 3.33 (2.76) | 1.38 (1.53) | 0.04 (0.11) |
| Long–short spread, net of estimated spreads | −0.94 (−0.94) | 2.71 (2.22) | 1.15 (1.20) | 0.02 (0.06) |

- **The post-event effect has disappeared.** The reversal of additions and the rebound of deletions after the index trading date were large in the 1990s and 2000s and are absent in 2010–2022. The decline in the deletion rebound from 1990–1999 to 2010–2022 is significant: −12.3 percentage points over [+1, +60] (t = −2.6) and −21.6 percentage points over [+1, +250] (t = −2.2).
- **The pre-event run-up of additions persists.** It is about 9% over 60 sessions in 2010–2022, but it mixes announcement, anticipation, and selection on past returns, and it cannot be traded without knowing the additions in advance.
- **No tradable version survives in the recent period.** Over 1976–2022, the long–short alpha net of estimated spreads is 0.35% per month (t = 0.87). In 2010–2022 the gross alpha is zero, and the break-even trading cost is 0.08% per side. Every robustness check gives the same pattern. The provisional 2023–2025 spread alpha is −0.42% per month (t = −0.48).
- **Multiple testing.** Notebook 02 reports **297 estimates**: 89 with |t| > 2 and 39 with |t| > 3. Of those 39, 38 are event-study means and one is a negative net alpha. No portfolio alpha has t > 3. All specifications were fixed before the results were computed, except the spin-off removal check, which is labelled as post hoc.

## Data limits

- **Announcement dates are mostly missing.** Verified dates exist only for the 19 additions of 2025 in the [S&P notice seed](references/sp500_announcement_seed.csv). Windows are centred on the index trading date, and the pre-event window mixes announcement and anticipation effects. Dates from the provisional additions file are not used.
- **2023–2025 membership is provisional.** The additions and removals files were collected manually. The removals file ends in August 2025 and omits at least one 2024 removal. The [December 5, 2025 announcement](https://www.spglobal.com/spdji/en/documents/indexnews/announcements/20251205-1480992/1480992_december2025rebalance.pdf) confirms three removals effective December 22, 2025, that the file does not contain. Results for this period are a robustness check only.
- **Pre-1990 dates are imprecise.** Before about 1990, membership changes are recorded on a weekly grid, and some exits are recorded weeks after the CRSP delisting. S&P also did not pre-announce changes before October 1989.
- **Restated Compustat is not used** in the headline analysis. Compustat enters only the anticipation appendix, where it is the current, restated vintage and not point-in-time data.
- **Event types are proxies.** The export does not classify events, and S&P MidCap 400 membership is not available, so migrations cannot be identified.
- **Costs are approximate.** Quoted spreads cover only 15% of traded weight in 1976–1989. Gaps are filled with the median quoted spread among that month's trades, or with an assumed 0.5% per side when no trade has a quote. Quoted spreads leave out price impact at index-event closes. Borrow fees are assumptions.
- **Unresolved records.** Ingersoll-Rand (PERMNO 12431, now Trane Technologies) exits the index on June 30, 2009, according to the 1975 workbook, but no later spell records its return to the index. Nine deleted securities have CRSP monthly rows without returns for 78 position-months, and those months are left out of the portfolio average.

## Reproducibility

To reproduce the results, use the same WRDS exports and the same Kenneth French factor archives (September 2026 CRSP vintage). Each contributor creates their own `.venv` from `requirements.txt`. Notebook 02 does not depend on `Data/prepared/`. Notebook 03 regenerates those files for the appendix when `save_prepared = True`.

Commit the notebooks with cell outputs cleared, `.gitignore`, `README.md`, `requirements.txt`, and `references/sp500_announcement_seed.csv`. Git ignores `Data/` and `.venv/`.

## Data that would strengthen the study

| Missing input | Use | Where to look |
| --- | --- | --- |
| S&P 500 announcement dates | Announcement-centred windows that separate the announcement effect from anticipation and from the trading-date effect. | [S&P DJI announcements](https://www.spglobal.com/spdji/en/media-center/news-announcements/) for event-by-event verification. [Siblis Historical Component Changes](https://siblisresearch.com/data/historical-component-changes/) is a commercial source for bulk history. |
| Complete S&P 500 membership for 2023–2025 | Turning the 2023–2025 check into part of the core sample. | [CRSP S&P 500 membership](https://wrds-www.wharton.upenn.edu/documents/397/CRSP_1925_historical_indexes_guide.pdf) or Compustat index constituents on WRDS, cross-checked against S&P notices. |
| S&P MidCap 400 membership history | Separating migrations from other additions and deletions. | Compustat/CCM historical constituents on WRDS, depending on the subscription. |
| Closing auction prices, borrow fees, and trade-level costs | Measuring execution at the index trading close and replacing the cost and borrow assumptions. | [NYSE Daily TAQ on WRDS](https://wrds-www.wharton.upenn.edu/pages/about/data-vendors/nyse-trade-and-quote-taq/) and [S&P Global Securities Finance on WRDS](https://wrds-www.wharton.upenn.edu/pages/about/data-vendors/markit/), subject to access. |
| Corporate-event data (SDC M&A, spin-offs) | Classifying events directly instead of with CRSP proxies. | [SDC on WRDS](https://wrds-www.wharton.upenn.edu/pages/data-announcements/retirement-of-refinitiv-sdc-platinum/), subject to access. |
