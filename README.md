# Fund Reporting Pipeline – Robeco 3D ETFs (unofficial replication)

A small replication of a monthly client-reporting workflow using public data:
raw prices → data quality checks → clean price table → performance → reconciliation
against published figures → Excel deliverable.

**Report date:** 30 September 2026  
**Built with:** Python, SQL (SQLite), Excel

*For learning purposes only. Not affiliated with or endorsed by Robeco.*

---

## Business question

A relationship manager needs monthly performance figures for a fund that are accurate,
traceable to source, and consistent with what the asset manager publishes.
This project builds that process end to end and documents every data issue found on the way.

## Pipeline

```
Yahoo Finance (Xetra closing prices)
        │  
        ▼
prices_raw          raw prices, stored unchanged
        │  quality checks against the official Xetra trading calendar
        ▼
prices_clean        one row per fund per trading day, gaps forward-filled and flagged
        │
        ▼
month_end → performance (1M, 3M, YTD, 1Y, since first price)
        │
        ▼
reconciliation against Robeco's published figures → Excel workbook
```

## Funds

| Ticker | Fund | Role |
|---|---|---|
| 3DGE | Robeco 3D Global Equity UCITS ETF EUR(H) Acc | Main fund (performance + reconciliation) |
| 3DCE | Robeco 3D EUR Enhanced Index Credits UCITS ETF EUR Acc | Bond fund, under 12 months of history |
| 3DCH | Robeco 3D Global Enhanced Index Credits UCITS ETF EUR(H) Acc | Bond fund, under 12 months of history |
| EUNL | iShares Core MSCI World | Market reference only (not a benchmark) |

## Data quality checks

Each fund is checked against the **official Xetra trading calendar** from its first price.
An independent calendar matters: a day missing for every fund would be invisible if the
calendar were built from the data itself.

| Check | Result | Impact | Action |
|---|---|---|---|
| Missing trading days | 24 Oct 2025: no price for any fund | None (not a month-end) | Forward-filled, flagged `is_filled = 1` |
| Missing trading days | 6 Mar 2026: no price for EUNL only | None (reference series, not a month-end) | Forward-filled, flagged |
| Prices on closed days | None | – | – |
| Duplicates | None | – | – |
| Outliers (> 4 standard deviations of the fund's own daily returns) | 5 days | 4 match market moves; 3DGE on 13 May 2025 (+3.7% vs +0.6% market) is likely a lagged price in a newly listed ETF, not verified | Documented; none fall on a month-end |

Outlier thresholds are fund-specific: a single fixed cut-off would be too loose for bond funds
and too strict for equity funds.

## Performance (3DGE, as of 30 Sep 2026)

| 1M | 3M | YTD | 1Y | Since first price |
|---|---|---|---|---|
| −0.65% | +2.76% | +11.67% | +17.19% | +35.79% cumulative (+24.18% annualised) |

3DCE and 3DCH were launched in 2026. They have no YTD or 1Y figure and, with less than
12 months of history, their performance would not be shown in a client-facing report under MiFID rules.

## Reconciliation with published figures

| Period | Published (Robeco) | Calculated | Difference |
|---|---|---|---|
| 1M | −1.14% | −0.65% | +0.49 |
| 3M | +2.06% | +2.76% | +0.70 |
| YTD | +11.92% | +11.67% | −0.25 |
| 1Y | +15.83% | +17.19% | +1.36 |
| Since inception (ann.) | +25.29% | +24.18% | −1.11 |

**Why the numbers differ**

- The differences go in **both directions**, which points to a timing difference rather than
  an error in the method. A formula error or a missing fee would push every row the same way.
- This project uses **Xetra closing prices** (last trade at 17:30 CET). Robeco's figures are
  "net of fees based on transaction prices" (stated on the fund factsheet), which reflect
  markets that close later, such as US equities.
- Supporting evidence: on 30 Sep 2026, 3DCE's Xetra close was 4.9603 while its official
  fund price was 4.94, a gap of about 0.4% on a single day.
- The since-inception row also has a different start date: Robeco measures from 1 May 2025;
  the first Xetra price available here is 2 May 2025.
- This explanation is the most likely cause but is **not fully proven**: daily NAV history
  was not available. With NAV data, the same pipeline should reconcile to near zero.

## Benchmark: why EUNL is not used as one

The official benchmark for 3DGE is the **MSCI World Index (net return, hedged into EUR)**.
Daily data for this hedged index is not freely available, so EUNL (unhedged MSCI World)
is used only as a market reference in the outlier check.

Using it as a benchmark would give the wrong conclusion:

| YTD to 30 Sep 2026 | Return | 3DGE vs. this comparator |
|---|---|---|
| 3DGE (published) | +11.92% | – |
| Official benchmark, hedged (published) | +11.14% | **+0.78 (outperformed)** |
| EUNL, unhedged | +15.53% | −3.61 (appears to underperform) |

The gap between the two comparators is a **currency effect**: EUNL carries USD exposure,
3DGE is hedged to EUR. Robeco's September commentary notes that the US dollar strengthened
that month, consistent with EUNL rising (+1.87%) while 3DGE fell (−0.65%).
With hedged index data, or a hedged ETF proxy such as iShares IBCH (ISIN IE00B441G979),
the pipeline could run a proper fund-vs-benchmark comparison.

## Excel deliverable

`output/fund_report_2026-09_final.xlsx`

| Sheet | Contents |
|---|---|
| Data | Month-end prices and returns; fund names added with XLOOKUP |
| Lookup | Fund master data (name, asset class, ISIN) |
| Monthly pivot | Monthly returns by fund. Grand totals switched off, because summing monthly returns understates compounded returns |
| Performance | Period returns from SQL |
| Reconciliation | Calculated vs published; differences above 1 percentage point highlighted |
| Checks | YTD recalculated in Excel with SUMIFS and checked against the SQL result |

## From data to factsheet

At an asset manager, the output of this pipeline would feed a reporting tool such as
Vermilion or Seismic, where templates apply rules such as hiding performance for funds with
less than 12 months of history and adding country-specific disclaimers.
This project covers the data, checks and calculations behind that step.

## Limitations

- Exchange closing prices, not NAV
- No daily data for the official hedged benchmark
- Report date is hard-coded in several queries; a production version would parameterise it
- Outlier on 13 May 2025 explained but not verified

## How to run

```bash
pip install -r requirements.txt
```

Open `fund_reporting_pipeline.ipynb` and run all cells.
The first run downloads prices and saves a snapshot to `data/raw/`; later runs reuse it.

---

Author: Navat Azat 