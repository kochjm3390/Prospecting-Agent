---
name: daily-opportunity-scout-co-ut
description: >
  Daily AI opportunity scout for MMG Real Estate Advisors (Jason Koch). Surfaces the
  highest-probability next contacts in EXISTING multifamily investment sales across
  Colorado and Utah — primarily seller targets (owners likely to list), secondarily
  buyer matches (investors active now). Joins live event feeds (distress monitor,
  competitor-listing monitor, market briefing, CoStar sale-status-change monitor) to a
  monthly-refreshed master (ownership/units/year/contacts) + the active pipeline, scores
  each opportunity out of 100, applies exclusions, and appends a ranked top-10 to
  MMG_Opportunity_Log.xlsx in OneDrive.
schedule: Daily @ 6:00 AM MT (after the 5 AM briefing; re-run if distress/competitor feeds arrive later)
output: Email (ranked top-10) + appended rows in MMG_Opportunity_Log.xlsx
owner: jason.koch@mmgrea.com
---

# Daily Opportunity Scout — Colorado & Utah Multifamily

## Mission
Produce a ranked list of up to **10** opportunities Jason should pursue today:
- **~80% seller targets** (owners likely to list) and **~20% buyer matches** — held over a
  rolling 30-day window, not forced every single run.
- Scope is **existing multifamily investment sales only** in **CO and UT**. No new-development
  land, no other asset classes, no other states.

## Geographic mix (guideline, not a hard rule — quality wins)
- ~60% Denver Metro → Fort Collins corridor
- ~30% Colorado Springs metro
- ~30% Utah
(The percentages intentionally overlap; treat them as a steering bias, not a quota.)

## Property size guidance
- Ideal **30–200 units**. Go smaller the **newer** the construction (e.g., a 12–20 unit 2022
  build is fair game; a 12-unit 1960s walk-up generally is not).

---

## Data sources

### A. Live event feeds — read EVERY run (these are the daily edge)
| Feed | Where | Signal it provides |
|---|---|---|
| **Distress Monitor** (7 AM — see `distress-monitor-co-ut`) | CSV in OneDrive `_csv/Distress_CO_UT_YYYY-MM-DD.csv` + email subject `Distress Monitor — CO/UT — <date>` | lis pendens, CMBS special-servicing transfers, watchlists, near-term maturities, auctions → **seller distress (primary)** |
| **Competitor New-Listing Monitor** (7:15 AM) | Outlook email | new CO/UT MF listings from M&M, CBRE, W&D, NorthPeak, Pinnacle, JLL → deal velocity **and** other-broker exclusions |
| **Market Intelligence Briefing** (5 AM) | Outlook email | Treasury/cap markets, CO/UT legislation, macro → submarket momentum + timing context |
| **CoStar Sale Status-Change Monitor** (1st & 3rd Sun) | OneDrive xlsx | Active / Under-Contract / Sold / **Withdrawn** → withdrawn & expired = re-list targets; UC/Sold = exclude |

> Read Outlook with `outlook_email_search` then `read_resource` on the message URI. Match by
> subject prefix (e.g. "Market Intelligence Briefing —", the distress monitor's subject, the
> competitor monitor's subject). If the **distress monitor email is missing**, flag it in the
> output — it is the most important seller-signal feed (see SETUP.md → "Repair the distress feed").

### B. Monthly master — refreshed as CSV (slowly-changing; monthly is correct)
| File (CSV) | Use |
|---|---|
| CO / UT **All-Properties** | ownership entity, units, year built, **owner phone/email** (CoStar True Owner) |
| CO / UT **Solds** (2010/2016 → now) | exclude recently closed; hold-period (long hold w/o refi); comps |
| CO / UT **For-Sale** | currently listed → exclude / identify other-broker exclusives |
| **mLink master** (CRM) | buyer & owner contacts (name/phone/email); flag 1031 buyers |

> The `.xlsx` masters exceed the file-reader's conversion ceiling (HTTP 406). They MUST be
> provided as **CSV** on each monthly refresh — see SETUP.md → "CSV-ify the master files".

### C. Pipeline & distress workbook
- **Active pipeline** (`Adam Jason Pipeline.xlsx` / T100): exclusions (active MMG listings,
  under contract, SMAs presented) **and** the buyer pool (1031 buyers, committed equity).
- **CRED iQ distress export**: loan status, maturity date, rate, special servicer, borrower
  entity, LTV/DSCR.

---

## Scoring — out of 100 (four dimensions)

Score is guidance to rank; a clearly superior opportunity outranks a marginally higher number.

**1. Seller distress — high weight (0–35)**
- Lis pendens / foreclosure filed: +35
- Transferred to special servicer: +30
- CMBS watchlist or loan **maturity < 12 mo**: +25
- Loan maturity 12–24 mo at **above-market rate** / floating: +18
- **Withdrawn or expired** listing (failed sale): +15
- Long hold **>10 yr with no refinance**: +12
- **Out-of-state** ownership: +8

**2. Buyer urgency — high weight (0–35)**
- Active **1031** with identification clock running: +35
- Committed equity + mandate + deadline: +28
- **Assumable below-market debt** match to a known buyer: +20
- Stated acquisition appetite, no hard deadline: +12

**3. Submarket momentum — medium weight (0–15)**
- Rent growth / positive absorption, limited new supply, recent comp velocity.

**4. Deal velocity — medium weight (0–15)**
- Recent nearby trades, rising listing activity, price cuts signaling a motivated market.

Priority signals to always elevate: **lis pendens, CMBS alerts, maturing loans, long hold
without refinance, out-of-state ownership, assumable below-market debt, 1031 urgency.**

---

## Exclusions (drop the candidate)
- Already **under contract** or **recently closed** (check Solds + status-change + pipeline).
- Another broker is **confirmed exclusive** (check For-Sale list + competitor monitor).
- Currently an **active MMG listing / SMA presented** (check pipeline).
- Outside CO/UT, or not existing multifamily.
- Outside 30–200 units **unless** newer construction justifies going smaller.
- **De-dupe:** appeared in the log within the last ~30 days with no status change or new signal.

---

## Process each run
1. Load monthly master CSVs + pipeline + latest CRED iQ distress export.
2. Pull today's live feeds (Outlook distress / competitor / briefing) + latest status-change file.
3. Build the candidate pool:
   - Sellers: distressed owners (maturity ≤ ~18 mo, special servicing, lis pendens, watchlist),
     withdrawn/expired listings, long-hold-no-refi, out-of-state owners in momentum submarkets.
   - Buyers: 1031 / committed-equity contacts from pipeline + mLink with a current mandate.
4. Apply exclusions and size/geo filters.
5. Score 0–100; enrich contacts from master + mLink (never fabricate a phone/email — if not in
   the data, write "skip-trace needed" or "(in mLink)").
6. Rank; enforce the ~80/20 seller/buyer mix over the trailing 30 days of the log; steer geo mix.
7. De-dupe against the last ~30 days of the log.
8. Output + append.

## Output
**A. Email (ranked, up to 10).**
- Seller target: property address · units · year built · loan info (if it drives the reasoning) ·
  owner contact (name / phone / email) · score · recommended action · one-line reasoning.
- Buyer match: contact info · score · recommended action · one-line reasoning on **why active now**.
- If the distress feed was missing, say so at the top.

**B. Append to `MMG_Opportunity_Log.xlsx`** (one row per entry), columns in this exact order:
`Date, Rank, Contact Name, Company, Type, State, Submarket, Score, Property Address, Units,
Year Built, Loan Note, Owner Phone, Owner Email, Recommended Action, Reasoning, Status`
- `Type` = Seller | Buyer. `Status` starts as `New`.

## Guardrails
- Never invent owner names, phone numbers, emails, loan terms, or distress signals. Report only
  what the sources contain; mark gaps explicitly.
- Note the vintage of the distress data; re-confirm maturity-based signals before outreach if the
  export is older than ~60 days.
