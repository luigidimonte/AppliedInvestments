# AppliedInvestments

The **Index Life-Cycle Strategy** project studies three stages of S&P 500 additions. It tests whether we can anticipate additions, exit the positions around index implementation, and trade post-inclusion underperformance. The 1975–2025 history lets us compare periods, including whether the effect persisted in 2018–2025.

## Set up the Python kernel

From the repository root on Windows, create the project environment and install its dependencies.

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
```

In VS Code, open a notebook and choose **Select Kernel → Python Environments → `.venv`**. The interpreter is `.venv\Scripts\python.exe` under the project root.

## Run the data inventory

1. Place the 12 CSV files and the 1975 workbook in `Data/` at the repository root. The inventory accepts the descriptive CSV names and the original WRDS download names. If both copies are present, it uses the descriptive name. Git ignores this directory so the WRDS data stays local.
2. Open [01_inventory.ipynb](notebooks/01_inventory.ipynb) in VS Code and select the project kernel with the dependencies from `requirements.txt` installed.
3. Run all cells. The catalog explains what each file contains. The provisional additions register is available as `events`. Set `audit_monthly = True` to check the full monthly export for duplicate records.

The notebooks read the source files locally. Clear cell outputs before sharing a notebook.

## Prepare inputs for all three legs

Run [03_strategy_inputs.ipynb](notebooks/03_strategy_inputs.ipynb) to write the following CSV files to the Git-ignored `Data/prepared/` directory.

| File | Use |
| --- | --- |
| `strategy_events.csv` | Shared addition register with effective dates, announcement evidence, execution-close reference dates where supported, and monthly holding dates. Join by `permno` and `effective_date`. |
| `anticipation_candidates.csv` | Provisional month-end nonmember universe for 1975–2022, with CRSP size, rank, and a liquidity proxy. Join by `permno` and `formation_date`. |
| `anticipation_effective_date_labels.csv` | Separate labels for inclusion within three months of formation, based on effective dates. These support a pipeline check. A pre-announcement test needs verified announcement dates and coverage. |
| `inclusion_daily_windows.csv` | Daily prices, returns, volume, quotes, and the benchmark around additions. Set `build_daily_windows = True` to generate this file from the roughly 10 GB daily export. |
| `monthly_ff5_momentum.csv` | Monthly five-factor and momentum returns in decimal units for the post-inclusion tests. Keep `F-F_Research_Data_5_Factors_2x3_CSV.zip` and `F-F_Momentum_Factor_CSV.zip` in `Data/`. |

The [S&P notice seed](references/sp500_announcement_seed.csv) keeps the public source for each checked addition. Leave timing or migration fields blank when the notice does not establish them. Extend the anticipation universe through 2025 once recent membership and exits are reconciled. The full candidate model also needs point-in-time earnings, float, and MidCap 400 membership.

## Run the post-inclusion prototype

Open [02_post_inclusion.ipynb](notebooks/02_post_inclusion.ipynb) with the same Python kernel and run all cells. It runs the inventory and builds `portfolio_monthly`, the gross return series for the equal-weighted basket of recent additions from January 1975 through December 2025, along with monthly coverage counts.

For `alpha_summary`, place the [Kenneth French monthly factors archive](https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/ftp/F-F_Research_Data_Factors_CSV.zip) in `Data/` under its original filename. The notebook estimates a preliminary CAPM alpha for the full sample and each subperiod.

The gross hedged spread in `hedged_monthly` is long the CRSP value-weighted U.S. market and short the additions basket in equal dollar amounts. Returns are measured per dollar of short exposure. Run `03_strategy_inputs.ipynb` first to create `monthly_ff5_momentum.csv` for `multifactor_alpha_summary`. This table reports alpha, the remaining market beta, t-statistics, and approximate confidence intervals using 12 monthly Newey–West lags. The hedge is a market return proxy and the gross series precedes trading, financing, and stock-borrow costs.

## Team handoff

On a fresh clone, set up `.venv` and place the source files in `Data/`. Run the inventory to check coverage. Run `03_strategy_inputs.ipynb` to generate the shared tables. Enable `build_daily_windows` on the first run if the daily table is needed. The portfolio and gross hedge in `02_post_inclusion.ipynb` can run with the source files. Its multifactor section uses the prepared factor table.

| Workstream | First deliverable |
| --- | --- |
| **1. Anticipation** | Build a size-and-liquidity baseline on the 1975–2022 candidate universe. Follow the plan's 2000–2012 training window and test from 2013 onward wherever labels are available. Produce a dated list of candidates selected at each formation date. Use the effective-date labels to check the pipeline. Evaluate pre-announcement performance only where announcement dates and coverage are verified. |
| **2. Event verification and exit** | Start by sourcing the 2023–2024 announcements and reconciling 2023–2025 additions and removals. Record the source, announcement date, effective date, and timing in the shared S&P reference file. Test the closing-price proxy on events with confirmed timing. Use consistent dividend treatment when comparing stock and benchmark returns. Link the exit to positions selected by the anticipation model and define how to exit candidates that never join. |
| **3. Post-inclusion** | Use the gross hedged series and multifactor estimates in notebook 02 as the baseline. Check sensitivity to the hedge, holding period, event filters, and Newey–West lag choice. Add turnover and trading, financing, and borrow costs under stated assumptions. Show 2018–2025 separately. |

Use `permno` and `effective_date` to join event data. Use `permno` and `formation_date` to join anticipation features to labels. Keep each workstream in a separate notebook and edit the shared event references in one place.

### Which gaps block which tests?

| Missing input | Consequence |
| --- | --- |
| Announcement dates | Required for a pre-announcement test or an announcement-centered event study. Effective-date windows and the pooled post-inclusion test can start with verified effective dates. |
| Complete S&P 500 membership through 2025 | Required to extend the nonmember candidate universe beyond 2022. |
| S&P MidCap 400 membership history | Required to use MidCap membership as a predictor and to classify migrations reliably across the sample. The three baseline tests can start with a pooled sample. Keep unverified origins unknown in any migration comparison. |
| Quarterly report dates, float, and sector data | Required for the corresponding earnings, float, and sector inputs in the full screening model. The first model can use the size and liquidity fields already prepared. |
| Auction data, borrow fees, and benchmark weights | The daily close supports a proxy execution test. Use auction data to test the actual auction price, borrow-fee assumptions or observations to evaluate net short returns, and dated weights to test an index underweight. |

Commit the three notebooks with cell outputs cleared, `.gitignore`, `README.md`, `requirements.txt`, and `references/sp500_announcement_seed.csv`. The inventory reads this public S&P reference file. Git ignores `Data/` and `.venv/`.

To reproduce the same results, use the same WRDS exports and the same three Kenneth French factor archives. The factor archives used here report the September 2026 CRSP vintage. Each contributor creates their own `.venv` from `requirements.txt` and regenerates `Data/prepared/` by running notebook 03.

## Data acquisition checklist

### Priority inputs for the three legs

| Missing input | Fields and coverage to request | Where to look |
| --- | --- | --- |
| S&P 500 announcement dates for anticipation and the event study | Security identifier or historical ticker, addition or removal, public announcement date and time if available, effective date, and source. The prepared register has announcement dates for 19 of 1,220 candidate additions. All 31 candidates recorded for 2023–2024 still need an announcement date. | Free [S&P DJI announcements](https://www.spglobal.com/spdji/en/media-center/news-announcements/) for event-by-event verification. [Siblis Historical Component Changes](https://siblisresearch.com/data/historical-component-changes/) is a commercial source for bulk history. Confirm coverage before using it. |
| S&P 500 membership through 2025 and S&P MidCap 400 history | Index ID, security ID (`GVKEY` and `IID` or `PERMNO`), membership start and end dates. Request complete S&P 500 membership changes for 2023–2025 and MidCap 400 membership from its 1991 launch through 2025. The local removals file ends in August 2025. | [Compustat/CCM historical constituents on WRDS](https://wrds-www.wharton.upenn.edu/documents/402/CRSP-Compustat_Merged_Database_Data_Guide_9efDcmD.pdf) and [CRSP S&P 500 membership](https://wrds-www.wharton.upenn.edu/documents/397/CRSP_1925_historical_indexes_guide.pdf), depending on the subscription. Cross-check recent changes and documented migrations against S&P notices. |
| Quarterly earnings availability for anticipation | Add the quarterly earnings report date (`RDQ`) to `GVKEY` and `DATADATE` so it can be joined to the existing earnings data. Use historical vintages if testing what investors knew before a restatement. | [Compustat quarterly data on WRDS](https://wrds-www.wharton.upenn.edu/documents/403/CRSP_-_Compustat_Merged_Database_CCM_NiIeIWV.pdf). [SEC EDGAR](https://www.sec.gov/search-filings/edgar-application-programming-interfaces) provides dated 8-K, 10-Q, and 10-K filings for a public reconstruction. |
| Historical float and sector inputs for the full candidate screen | Dated float shares or a float factor for candidate securities, historical eligibility thresholds, and dated sector classifications and index weights if using sector balance. The current screen uses ordinary CRSP market capitalization. | [S&P float methodology](https://www.spglobal.com/spdji/en/documents/index-policies/methodology-sp-float-adjustment.pdf) and [U.S. index methodology](https://www.spglobal.com/spdji/en/methodology/article/sp-us-indices-methodology/) define the screen. Check historical coverage with S&P for its weights and IWF and with [FactSet Ownership on WRDS](https://wrds-www.wharton.upenn.edu/pages/about/data-vendors/factset/) for float-related data. The [CCM guide](https://wrds-www.wharton.upenn.edu/documents/402/CRSP-Compustat_Merged_Database_Data_Guide_9efDcmD.pdf) documents historical GICS. |
| Corporate events for the sample filters | Dates and security identifiers for IPOs, spin-offs, and acquisitions within the plan's ±100-day window. For M&A, record the target and acquirer. The CRSP delisting export already covers delisting dates and reasons. | [SDC M&A and New Issues on WRDS](https://wrds-www.wharton.upenn.edu/pages/data-announcements/retirement-of-refinitiv-sdc-platinum/), subject to access. Check public S&P notices and SEC filings for individual events. |

### Inputs for execution and additional tests

- **Closing auction execution.** Request timestamped closing trades, size, and sale condition codes if testing actual auction prices. [NYSE Daily TAQ on WRDS](https://wrds-www.wharton.upenn.edu/pages/about/data-vendors/nyse-trade-and-quote-taq/) provides historical trades and quotes. The daily CRSP export can serve as a closing-price proxy.
- **Costs and benchmark.** Request dated security-level borrow fees and availability for a short from [S&P Global Securities Finance on WRDS](https://wrds-www.wharton.upenn.edu/pages/about/data-vendors/markit/), subject to access. Use historical constituent weights for a benchmark underweight and a dividend-inclusive S&P 500 return series for a total-return comparison. Check [S&P index data](https://www.spglobal.com/spdji/en/methodology/article/sp-us-indices-methodology/) and [CRSP's S&P 500 total-return coverage](https://wrds-www.wharton.upenn.edu/documents/397/CRSP_1925_historical_indexes_guide.pdf).
- **Passive ownership mechanism.** This extension requires dated fund-level holdings and a classification of which funds track an index. [SEC 13F data](https://www.sec.gov/files/form_13f.pdf) reports institutional holdings. Fund strategy needs separate classification.
- **Full twelve-month outcomes for the 2025 cohort.** Extend CRSP returns through December 2026 once available. The calendar-time portfolio can already include these events through December 2025.
