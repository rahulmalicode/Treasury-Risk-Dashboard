# Treasury Risk Dashboard — Liquidity, Credit & Interest Rate Risk

A small end-to-end risk analysis project built around a mock bank balance sheet: SQL for data querying/aggregation, Excel for the risk model and reporting. Built to practice the kind of analysis a Treasury/CTC Risk team does day to day. LCR estimation, interest rate gap analysis, and credit concentration monitoring.

## Why I built this

I wanted a hands on project that covers Liquidity Risk, Interest Rate Risk, and Credit Risk together instead of just one in isolation, using SQL and Excel since those are the tools most Treasury/Risk teams actually work in day to day.

## What's in here

```
treasury-risk-dashboard/
├── data/
│   └── balance_sheet.csv          # synthetic loan/security/deposit book (212 instruments)
├── sql/
│   ├── schema.sql                 # table definition
│   ├── queries.sql                # 8 risk queries (gap, LCR, concentration, currency, etc.)
│   └── treasury.db                # SQLite DB, pre-loaded (schema + data already run)
├── excel/
│   └── Treasury_Risk_Dashboard.xlsx   # the risk model — 8 tabs, this is the main deliverable
├── docs/
│   └── Treasury_Risk_Summary_Memo.docx   # one-page stakeholder-facing write-up of the results
├── scripts/
│   ├── generate_data.py           # generates the synthetic balance sheet (data/balance_sheet.csv)
│   ├── build_excel.py             # builds the Excel workbook from the CSV
│   └── build_memo.js              # builds the Word memo
└── README.md
```

## The data

`data/balance_sheet.csv` is a synthetic balance sheet **not real bank data**. It's meant to look like what a Treasury team would actually pull from a core banking system: 212 instruments split across loans (~110), securities (~55, including government bonds flagged as HQLA-eligible), and retail + wholesale deposits (~47). Fields: instrument ID, type, counterparty, credit rating, currency, amount, interest rate, rate type (fixed/floating), origination date, maturity date, and an HQLA eligibility flag.

Reporting date used throughout the model: **31-Mar-2026**.

## SQL

`sql/schema.sql` creates the `balance_sheet` table. `sql/queries.sql` has 8 queries:

1. Maturity bucketing (0-30d / 31-90d / 91-365d / 1-3y / 3+y)
2. Gap summary, assets vs liabilities per bucket
3. HQLA total
4. Estimated 30 day net cash outflows (simplified LCR run off assumptions)
5. Top 10 counterparty exposures
6. Exposure by credit rating band
7. Rate sensitive book (for the NII shock calc)
8. Currency exposure summary

I used SQLite locally (`sql/treasury.db` is already built just open it in DB Browser for SQLite or run the queries directly), but the SQL is close to standard ANSI so it'll run on MySQL/Postgres with minor tweaks (mainly the `julianday()` date-diff function, which is SQLite-specific).

To rebuild the DB from scratch:
```bash
sqlite3 sql/treasury.db < sql/schema.sql
# then load data/balance_sheet.csv into the balance_sheet table
```

## Excel

`excel/Treasury_Risk_Dashboard.xlsx` is the main deliverable. Eight tabs:

| Tab | What it does |
|---|---|
| **Assumptions** | Reporting date, deposit run-off rates, inflow cap, rate shock size, market risk yield shock — edit these and everything downstream recalculates |
| **Raw Data** | The full 212-row balance sheet, with formula columns for side (asset/liability), days to maturity, maturity bucket, years to maturity, modified duration, and dollar duration |
| **Maturity Gap** | Assets vs. liabilities by time bucket, net gap, cumulative gap, chart. Negative gaps are flagged red |
| **LCR Calculation** | HQLA / 30-day net outflows, walked through step by step (HQLA → outflows → capped inflows → net outflow → LCR %) |
| **Concentration Risk** | Exposure by rating band + top 10 counterparties, with a flag if sub-investment-grade exposure crosses 25% |
| **Market Risk** | Duration-based mark-to-market price sensitivity of the securities book to a parallel yield shock, broken down by rating band |
| **Rate Sensitivity** | NII impact of a +100bps parallel rate shock on the floating-rate book repricing within a year |
| **Dashboard** | One-page summary — LCR, cumulative 1-year gap, NII impact, securities MTM impact, plus a few written observations |

Everything is a live formula (`SUMIFS`, `SUMPRODUCT`-style logic) pointing back at Raw Data and Assumptions — nothing is hardcoded, so changing an assumption (say, bumping the wholesale run-off rate from 25% to 40%) flows through the whole workbook.

## Risk summary memo

`docs/Treasury_Risk_Summary_Memo.docx` is a one-page write-up of the same results, framed for a non technical Treasury stakeholder rather than an analyst key metrics table, observations, assumptions/limitations, and recommendations. This is the kind of document that would actually get sent up or across a team, versus the workbook itself which stays with the analyst.

## Key results (base case)

- **LCR: ~321%** well above the 100% regulatory minimum, driven mostly by the government bond holdings counting as HQLA
- **Cumulative 1 year gap: -₹5.76B**  liabilities reprice faster than assets in the short buckets (deposits are short dated, loans are longer), which is typical for a deposit funded bank
- **NII impact of +100bps: -₹16.1M**  the book is mildly liability-sensitive, so rising rates hurt net interest income slightly in year one
- **Securities MTM impact of +50bps: -₹396.7M** portfolio modified duration of ~4.6 years means the bond book takes a real price hit under a rate shock, concentrated in the AAA/A rated holdings that make up most of the book
- **Credit concentration**: no single counterparty breaches a hard limit, but ~17% of the credit book is sub investment grade (BB/B)

## How I'd extend this

- Add a second scenario (rate shock of -100bps, or a deposit run off stress case) side by side with the base case
- Build the Basel III LCR properly instead of the simplified run off assumptions used here
- Replace the zero coupon duration proxy on the Market Risk tab with a full coupon bond duration calculation
- Pull the SQL queries into Excel via Power Query instead of copy pasting the top 10 counterparty table
- Add a NIM (net interest margin) trend view once there's more than one reporting period of data

## Notes

- All run-off/haircut/duration assumptions in `Assumptions` are simplified for this exercise and are **not** the actual Basel III LCR weights or a full bond-pricing model .I flagged this directly in the workbook and the memo so it's not mistaken for a regulatory grade model.
- `scripts/generate_data.py`, `scripts/build_excel.py`, and `scripts/build_memo.js` are just the tooling I used to generate the dataset and construct the workbook/memo the actual "deliverables" are the CSV, the SQL files, the xlsx, and the memo.
