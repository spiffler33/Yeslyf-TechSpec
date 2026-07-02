# 11. Market Data Feed — MF NAVs, Indices, Scheme Metadata

> Status: **was flagged as missing from the architecture doc.** This document closes that gap.
> Researched: July 2026. Pricing for Indian data vendors is almost entirely **quote-based** — every figure not explicitly sourced is marked "quote".

---

## 1. Overview & role in Yesly

Yesly (SEBI-registered RIA; React Native/Expo app + FastAPI backend on AWS Mumbai) needs a market-data feed for three product surfaces:

| Surface | Screen / engine | What it consumes |
|---|---|---|
| **Portfolio dashboard** | F03 | Latest NAV per held scheme → current value; NAV history → XIRR inputs |
| **Allocation-gap report** | F06 | Scheme metadata (category, sub-category, asset class) to bucket holdings; benchmark index levels for comparison |
| **Market-regime rebalancing overlay** | Investment engine | Nifty 50 P/E and P/B daily values + long history (to compute percentile), India VIX daily close |
| **Curated product shelf** | Investment engine | Scheme master: category, expense ratio (TER), risk-o-meter band, AUM, exit load — for index funds, flexi-cap, ELSS, liquid, gold ETF, REITs, international MF |
| **Goal calculators** | Planning flows | CPI inflation (RBI/MOSPI), long-run return assumptions |

**Important scoping note — XIRR is not bought, it's built.** XIRR is just Newton–Raphson over the user's transaction cash flows plus current valuation. The feed's only job is to supply the *NAV* (and, later, prices/index levels). No vendor sells "XIRR as a service" that we need; the computation lives in the FastAPI backend.

The tech spec names **CMOTS**, **Accord Fintech**, and **Value Research** as candidate vendors. All three are real, established Indian data vendors; all three are quote-based. This doc evaluates them against the free/official alternatives (AMFI, NSE/NSE Indices, RBI/MOSPI) and makes a build-vs-buy call.

---

## 2. What data is actually needed

| Data item | Used by | Update frequency needed | Candidate source (v1 → later) |
|---|---|---|---|
| Latest NAV, all MF schemes | F03 valuation, XIRR | Daily (published ~9–11 PM IST) | **AMFI NAVAll.txt (free)** → vendor API |
| NAV history per scheme | XIRR back-fill, charts | One-time backfill + daily append | AMFI history portal / mfapi.in (dev) → **Accord/CMOTS (prod)** |
| Scheme master (name, ISIN↔AMFI code map, category, plan, option) | F06 bucketing, product shelf | Weekly (new schemes/mergers) | Partially derivable from AMFI file → **vendor scheme master (buy)** |
| Expense ratio (TER), exit load, AUM, risk-o-meter | Product shelf | Monthly (TER changes; risk-o-meter monthly per SEBI) | **Vendor only** (Accord ACE MF / CMOTS / Value Research); AMC sites otherwise |
| Fund ratings (star ratings) | Product shelf (optional) | Monthly | **Value Research license** (their core IP) — optional, not required for v1 |
| Nifty 50 / benchmark index closes | F06 benchmark comparison | Daily EOD | niftyindices.com reports (manual/free) → **NSE Data & Analytics EOD license** for in-app display |
| Nifty 50 P/E, P/B daily + full history | Regime overlay percentile | Daily EOD | niftyindices.com "P/E, P/B & Div Yield" historical report (free download, no API) |
| India VIX daily close + history | Regime overlay | Daily EOD | nseindia.com historical-VIX report (free CSV, no official public API) |
| CPI inflation | Goal calculators | Monthly | **RBI DBIE / MOSPI eSankhyiki (free, official)** |
| Stock prices (direct equity) | Post-v1.5 only | EOD (delayed OK) | **NSE Data & Analytics paid EOD feed** — do not scrape |
| Gold ETF / REIT prices | Product shelf valuation | Daily EOD | These trade on-exchange → same as stock prices (NSE EOD license) |

Key insight: **MF NAVs and scheme identity are free and official (AMFI). Everything qualitative about a scheme (TER, category richness, ratings) is vendor territory. Everything exchange-traded (index levels, VIX, ETF/REIT prices) is NSE-licensed territory once it appears inside a commercial app.**

---

## 3. Vendor deep-dives

### 3.1 CMOTS Internet Technologies (cmots.com / apidatafeed.com)

- **Who**: Mumbai-based market-data vendor for the Indian capital market, ~30 yrs old; states it is **empanelled with BSE/NSE/MCX/NCDEX as a data vendor** — this matters because it means their exchange data is legitimately licensed for redistribution (you still sign downstream paperwork).
- **Products**: "API DataFeed" (productised at apidatafeed.com) — financial APIs across Equity, Derivatives, Mutual Funds, Commodity, Currency, IPO, News, Corporate Announcements, Company Fundamentals, Indices.
- **MF coverage**: fund/scheme masters, scheme profile, category master (e.g., a Category API returning `SchemeClassCode` + `SchemeClass`), NAV latest + history, fund-wise and scheme-wise data.
- **API style**: REST endpoints returning **JSON, XML or CSV**; frequency tiers of **delayed / EOD / historical** (real-time available for exchange data under the appropriate exchange licensing).
- **Licensing**: subscription per API module/category; you register and get credentials after a commercial agreement. Their public docs are thin — no sample payloads, auth details, or rate limits published; you get full docs after sales contact.
- **Pricing**: **quote-based, not published.** Anecdotally Indian fintechs pay per-module annual subscriptions; treat any number as unknown until quoted.
- **Fit for Yesly**: strongest as a one-stop shop once Yesly needs MF *and* index/EOD equity data under one contract; also the vendor most commonly used by broker/fintech front-ends for corporate data pages.

### 3.2 Accord Fintech (accordfintech.com)

- **Who**: Navi Mumbai data vendor; best known for **ACE Equity** (fundamentals terminal) and **ACE MF / ACE MF Nxt** (mutual-fund research database) — widely used inside AMCs, wealth firms and media.
- **MF dataset (ACE MF)**: covers **all ~42 AMCs, 4,000+ schemes**; per-scheme: full factsheet data, **NAVs since inception**, portfolios (asset-wise and sector-wise holdings), fund-manager details, dividends, **AMFI codes / SEBI category mapping**, risk & rolling-return analytics, half-yearly accounts. This is exactly the "scheme master + qualitative metadata" bucket (TER, category, exit load, AUM, risk band) Yesly's product shelf needs.
- **Delivery**: **data feed via FTP (file drops) and API**; also packaged desktop/web applications (ACE MF Nxt, ACE Reports) and a distributor back-office suite (ACE MF Suite). For a backend like Yesly's, the raw feed (FTP/CSV or API) is the relevant product, ingested nightly into Postgres.
- **Licensing**: annual data-licensing agreement, scoped by dataset (MF only vs equity too), delivery mode, and redistribution rights (internal analytics vs end-user display are priced differently).
- **Pricing**: **quote-based, not published.** Third-party listings (IndiaMART) show the products but no rates.
- **Fit for Yesly**: **best-fit MF-only vendor.** If v1/v1.5 is MF-centric, an ACE MF feed (scheme master + NAV history + TER/risk-o-meter + portfolios) is likely the single most useful paid dataset, and it avoids paying for exchange data Yesly doesn't need yet.

### 3.3 Value Research (valueresearchonline.com)

- **Who**: India's oldest independent fund research house (ratings since 1993); its star ratings and fund categorisation are the de-facto standard cited by ET/Mint and used by Bloomberg; SEBI has referenced its rating methodology.
- **Data products**: consumer site/app plus B2B analytics ("Value Research Analytics Pro", launched 2019, aimed at advisors/researchers) and bespoke **data licensing** to banks, media and fintechs (ratings, categories, returns, fund analytics). There is **no public self-serve API**; licensing is a direct commercial negotiation.
- **Warning**: listings like the "Value Research Online" API on RapidAPI are **unofficial scrapers of their website — do not use**; a SEBI-registered RIA consuming scraped third-party IP is a legal and reputational risk.
- **Licensing/pricing**: **quote-based; negotiated per use-case** (which fields, display vs internal use, attribution). Ratings are their core IP and priced accordingly.
- **Fit for Yesly**: buy **only if** the product shelf wants to display third-party star ratings/analyst opinions. For raw NAV/scheme-master data, Accord/CMOTS are cheaper substitutes. Note an RIA curating its own shelf may deliberately *avoid* showing third-party ratings to keep advice provenance clean — decide product-first.

---

## 4. Free / official sources

### 4.1 AMFI — the official MF NAV source (free)

- **Latest NAVs (all schemes)**: `https://www.amfiindia.com/spages/NAVAll.txt` — plain-text, **semicolon-delimited**, grouped by scheme type and fund house, refreshed daily. SEBI/AMFI norms require AMCs to publish daily NAV to AMFI **by ~9–11 PM IST** on business days (Yesly should poll ~11 PM and again early morning for stragglers/backdated corrections).
- **Format** (header + example row):

```text
Scheme Code;ISIN Div Payout/ ISIN Growth;ISIN Div Reinvestment;Scheme Name;Net Asset Value;Date

119551;INF209KA12Z1;INF209KA13Z9;Aditya Birla Sun Life Banking & PSU Debt Fund - DIRECT - IDCW;103.2716;01-Jul-2026
```

  Interleaved with unnumbered section lines like `Open Ended Schemes(Debt Scheme - Banking and PSU Fund)` and fund-house name lines — the parser must treat any line without semicolons as a category/AMC header. Fields sometimes contain `N.A.` and blank ISINs; NAV date can lag (liquid/overseas funds).
- **History**: `http://portal.amfiindia.com/DownloadNAVHistoryReport_Po.aspx?mf=<amcCode>&frmdt=<dd-MMM-yyyy>&todt=<dd-MMM-yyyy>` — **max ~90 days per request**, so the initial backfill is a chunked crawl per AMC (slow but scriptable; be polite with rate limits).
- **What it gives / doesn't**: gives NAV, scheme code, ISINs, scheme name, and a coarse category from the section headers. Does **not** give TER, exit load, risk-o-meter, AUM, benchmark, or clean SEBI category taxonomy — that's the vendor gap.
- **Licensing**: public data published by the industry body for dissemination; standard practice is to attribute "NAV data: AMFI". No fee. (There is no formal API/SLA — it's a file on a website; build retries and checksum/row-count sanity checks.)

### 4.2 mfapi.in — free JSON NAV history (dev/prototyping only)

- Community-run free API over AMFI data. Endpoints:
  - `GET https://api.mfapi.in/mf` — all schemes (code + name)
  - `GET https://api.mfapi.in/mf/search?q=<text>` — search
  - `GET https://api.mfapi.in/mf/{schemeCode}` — **full NAV history since inception**
  - `GET https://api.mfapi.in/mf/{schemeCode}/latest`
- Example response:

```json
{
  "meta": {
    "fund_house": "UTI Mutual Fund",
    "scheme_type": "Open Ended Schemes",
    "scheme_category": "Other Scheme - Index Funds",
    "scheme_code": 120716,
    "scheme_name": "UTI Nifty 50 Index Fund - Direct Plan - Growth"
  },
  "data": [
    { "date": "01-07-2026", "nav": "182.4501" },
    { "date": "30-06-2026", "nav": "181.9902" }
  ],
  "status": "SUCCESS"
}
```

- Site advertises no auth/no keys and ~6x-daily refresh from AMFI.
- **Caveats**: it is an **unofficial hobby project — no SLA, no contract, no uptime guarantee, could disappear or change format**. Fine for prototyping and for one-time history backfill cross-checks; **do not put it in the production request path** of a regulated RIA app. Production should ingest AMFI directly into Yesly's own Postgres (which also makes the app independent of any third party for its hottest read path).

### 4.3 NSE / NSE Indices — index levels, Nifty P/E–P/B, India VIX

- **Nifty P/E, P/B, Div Yield**: official source is **niftyindices.com → Reports → Historical Data → "P/E, P/B & Div Yield"** — free CSV download per index and date range, history back to ~1999. **No official public REST API**; the pragmatic pattern is a scheduled job that downloads the daily report (or the daily index snapshot file) and appends to Yesly's own table, from which the percentile is computed.
- **⚠ The April 2021 methodology break**: NSE switched Nifty P/E from **standalone** to **consolidated** earnings in April 2021 — published P/E dropped overnight (~40 → ~32). A naive percentile over the full 1999–2026 series mixes two regimes. Options: (a) compute percentile only on post-Apr-2021 data (short window), (b) use P/B (methodology unchanged, and empirically a better mean-reversion signal) as primary with P/E as secondary, or (c) apply a documented adjustment to pre-2021 P/E. **Recommendation: lead with P/B percentile + post-2021 P/E; document the choice in the investment-methodology note (SEBI RIA record-keeping).**
- **India VIX**: official historical report at `https://www.nseindia.com/reports-indices-historical-vix` (CSV download). No official public API; nseindia.com's undocumented JSON endpoints (used by libraries like `nsepy`/NseIndiaApi) sit behind aggressive bot protection (cookies/Akamai) and their use in a commercial product is **against NSE's website terms** (personal/non-commercial only). For a once-a-day EOD value this is a manual-ish scheduled fetch of the official report, or part of the paid vendor feed.
- **Index closes for benchmark comparison (F06)**: same historical-data reports give daily closes free. **BUT** — see §7: *displaying* NSE index values inside a commercial app is redistribution and technically requires an agreement with **NSE Data & Analytics Ltd** (formerly DotEx), even for EOD. Delayed/EOD display licenses are far cheaper than real-time. Budget for this at the point benchmarks go live in-app.
- **Stock prices (if/when direct equity arrives)**: free "bhavcopy" EOD files exist in NSE archives, but NSE's copyright terms restrict content use to **personal, non-commercial or educational** purposes; commercial use requires the paid EOD/Historical subscription from NSE Data & Analytics (contact: marketdata@nse.co.in). Unofficial scraping libraries (`nsepy`, `nsetools`, etc.) carry the same legal exposure — **flagged: do not build a regulated product on scraped exchange data.** Alternative: get equities data through an empanelled vendor (CMOTS/Accord) who wraps the exchange licensing.

### 4.4 RBI / MOSPI — macro inputs for goal calculators (free, official)

- **RBI DBIE** (`https://data.rbi.org.in/DBIE/`): CPI inflation, repo rate, G-sec yields, FX — downloadable series; the revamped portal exposes data programmatically.
- **MOSPI eSankhyiki** (`https://esankhyiki.mospi.gov.in`): official CPI (note the **new 2024=100 base series**), IIP, GDP; MOSPI has published **open-data APIs** and even an MCP pilot (github.com/nso-india/esankhyiki-mcp) over ~25 datasets including inflation.
- Usage: monthly batch pull of CPI YoY into a config table feeding goal-calculator defaults. Free, attribution to source is good practice.

---

## 5. Build-vs-buy recommendation

### v1 (MF-only portfolio + shelf) — **build on AMFI, buy one scheme-master license**

1. **NAV pipeline (build, free)**: nightly Lambda/ECS cron ~11 PM + 6 AM IST → fetch `NAVAll.txt` → parse → upsert into `scheme_nav` (Postgres). One-time history backfill via AMFI 90-day portal crawl (use mfapi.in only to cross-check). XIRR computed in-house from user transactions + these NAVs. Effort: ~1–2 dev-weeks including parser hardening and reconciliation checks.
2. **Scheme master + metadata (buy)**: license **Accord ACE MF data feed** (or CMOTS MF APIs as the comparison quote) for scheme master, SEBI category, TER, exit load, AUM, risk-o-meter, portfolios. This is the piece that is genuinely painful to build (TER/risk-o-meter aren't in any single free feed) and it powers both F06 bucketing and the curated shelf. Weekly/monthly refresh is enough.
3. **Regime overlay (build, free-ish)**: daily job pulling niftyindices P/E–P/B report + NSE VIX report into internal tables; percentile logic in-house; P/B-first per §4.3. These values drive *internal* engine decisions — if they are only inputs to advice (not displayed as quotes in-app), licensing exposure is minimal; the moment you *display* index levels, trigger the NSE Data EOD display license (§7).
4. **Macro (build, free)**: monthly RBI/MOSPI CPI pull.
5. **Do NOT buy in v1**: real-time anything; equity tick/EOD feeds; Value Research ratings (decide product-first).

### v1.5+ (benchmarks displayed in-app, ETFs/REITs valued, direct equity)

- Sign **NSE Data & Analytics EOD/delayed display license** (or, usually simpler/cheaper at small scale, take index + ETF/REIT EOD prices via an **empanelled vendor — CMOTS or Accord** — under their redistribution umbrella; get both quotes).
- Consolidate: at this point one vendor contract (CMOTS or Accord) covering MF + indices + EOD equity likely beats stitching AMFI + niftyindices scrapes + separate exchange paperwork.
- Add **Value Research** only if third-party ratings become a product requirement.

---

## 6. Pricing snapshot (July 2026)

| Source | Model | Price |
|---|---|---|
| AMFI NAVAll.txt + history portal | Public file | **Free** |
| mfapi.in | Free community API | **Free** (no SLA; not for prod) |
| niftyindices.com historical reports | Public downloads | **Free** (display licensing separate) |
| NSE historical VIX report | Public download | **Free** (same caveat) |
| RBI DBIE / MOSPI eSankhyiki | Official open data | **Free** |
| CMOTS API DataFeed (MF module, indices, EOD equity) | Annual subscription per module | **Quote-based — not published** |
| Accord Fintech ACE MF feed (FTP/API) | Annual data license | **Quote-based — not published** |
| Value Research data/ratings license | Bespoke B2B license | **Quote-based — not published** |
| NSE Data & Analytics EOD/delayed display license | Exchange fee schedule + vendor agreement | **On application** (delayed/EOD ≪ real-time; real-time also incurs per-subscriber/display fees) |

Rule of thumb from the Indian fintech market: MF-only data licenses from Accord/CMOTS are typically low-to-mid lakhs ₹/year depending on fields and redistribution scope — **treat as unverified until quoted; get quotes from at least two of the three.**

---

## 7. Compliance & licensing notes

1. **Exchange data is licensed, full stop.** Any NSE/BSE-originated data (prices, index values, VIX) shown to end users in a commercial app = "redistribution/display" and requires an agreement with **NSE Data & Analytics Ltd** (and/or BSE's market-data arm), even for delayed/EOD. Real-time additionally has per-user display fees; **non-display (algo/engine) usage has its own policy and fee schedule** — relevant if the rebalancing engine consumes index data at scale.
2. **Vendor route wraps this**: buying via an empanelled vendor (CMOTS/Accord) means the vendor holds the exchange vendor license, but Yesly still typically signs an end-use/sub-license schedule declaring display vs non-display and user counts. Ask vendors explicitly what exchange paperwork lands on Yesly.
3. **AMFI NAV data** is public industry-body data; attribute it. AMC-published TER/factsheet data is public but vendor-normalized versions are the vendor's licensed compilation.
4. **Value Research ratings are copyrighted IP** — display requires a license + attribution ("Ratings by Value Research"); scraping their site (or using RapidAPI wrappers) is not a defensible position for a SEBI-registered RIA.
5. **Attribution/branding**: NSE index names ("Nifty 50") are trademarks of NSE Indices Ltd; licensed display normally comes with prescribed attribution text.
6. **SEBI RIA angle**: the RIA rules require documented, auditable basis for advice. The regime overlay's data lineage (source, methodology, the 2021 P/E break handling) should be written into the investment-policy note; using licensed/official sources strengthens that audit trail vs scraped data.
7. **Scraping nseindia.com is a real risk**: terms restrict to personal/non-commercial use; NSE has pushed takedowns of scraper libraries before, and the endpoints break routinely (bot protection). Don't put it in prod.

---

## 8. Gotchas & effort

- **AMFI parser edge cases**: section-header lines without semicolons; `N.A.` NAVs; blank ISINs; duplicate scheme names across plans; NAV dates that lag (overseas/liquid funds publish late or backdated) — build idempotent upserts keyed on (scheme_code, date) and a daily reconciliation count vs previous day.
- **ISIN ↔ AMFI code ↔ RTA folio mapping** is the classic Indian MF data headache (CAS files speak ISIN/RTA codes, AMFI speaks scheme codes). The vendor scheme master largely exists to solve this — verify the mapping table is in the quoted dataset.
- **History backfill** via AMFI's 90-day window is a multi-day polite crawl across ~45 AMC codes × ~25 years; run once, store forever.
- **Nifty P/E series has a structural break (Apr 2021, standalone→consolidated)** — percentile logic must handle it (P/B-first recommended).
- **No official API for niftyindices/NSE reports** — scheduled report downloads are semi-brittle (URL/format changes); wrap with alerting, and plan to move to a vendor feed at v1.5.
- **mfapi.in is not infrastructure** — no SLA; mirror, don't depend.
- **Vendor sales cycles**: CMOTS/Accord/Value Research quotes typically take 2–6 weeks incl. NDA and sample-data evaluation; start conversations one quarter before the feature needs the data.
- **Effort estimate**: AMFI pipeline + XIRR ≈ 1–2 dev-weeks; vendor feed ingestion (FTP/API → Postgres) ≈ 1 dev-week once contract signed; regime-overlay data jobs ≈ 2–4 dev-days.

---

## 9. Sources

- CMOTS: https://www.cmots.com/ · https://www.cmots.com/products/api-datafeed · https://www.cmots.com/services/market-data-provider · http://www.apidatafeed.com/product/mutual-fund/master/category/167
- Accord Fintech: https://www.accordfintech.com/ace-mf-nxt · https://www.accordfintech.com/ace-report · https://www.accordfintechnxt.com/
- Value Research: https://www.valueresearchonline.com/ · https://www.valueresearchonline.com/about-us/ (unofficial scraper to avoid: https://rapidapi.com/sahmad98/api/value-research-online-for-finance-market)
- AMFI: https://www.amfiindia.com/net-asset-value/nav-download · https://www.amfiindia.com/spages/NAVAll.txt · http://portal.amfiindia.com/DownloadNAVHistoryReport_Po.aspx · parser references: https://github.com/AmruthPillai/AMFI-API · https://pkg.go.dev/github.com/lordofsati/amfi
- mfapi.in: https://www.mfapi.in/ · https://www.mfapi.in/docs/
- NSE / NSE Indices: https://www.niftyindices.com/reports/historical-data · https://www.nseindia.com/reports-indices-historical-vix · https://www.nseindia.com/static/market-data/real-time-data-subscription · https://www.nseindia.com/static/market-data/eod-historical-data-subscription · https://www.nseindia.com/static/market-data/nse-data-policy · https://nsearchives.nseindia.com/web/sites/default/files/inline-files/Non_Display_Policy.pdf · https://www.nseindia.com/static/nse-copyright
- Nifty P/E 2021 methodology change: https://primeinvestor.in/nifty-pe-ratio/ · https://www.samco.in/knowledge-center/articles/nifty-50-pe-ratio/ · https://freefincal.com/download-nifty-historical-data-price-total-returns-pe-pb-div-yield/
- RBI / MOSPI: https://data.rbi.org.in/DBIE/ · https://esankhyiki.mospi.gov.in/ · https://www.mospi.gov.in/cpi · https://github.com/nso-india/esankhyiki-mcp
