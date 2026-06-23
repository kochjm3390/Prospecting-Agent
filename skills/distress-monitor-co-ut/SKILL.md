---
name: distress-monitor-co-ut
description: >
  Daily 7 AM MT distress intelligence for CO & UT commercial real estate (multifamily-first).
  TWO layers in one run: (A) public foreclosure / auction / trustee-sale / lis-pendens scan, and
  (B) CRED iQ loan-level distress (CMBS special-servicing, watchlist, near-term maturities). Emits
  a machine-readable CSV to OneDrive AND a skimmable email briefing. Supersedes
  `daily-foreclosure-auction-scan-co-ut` (which only scanned auction websites and stopped
  delivering). This is the primary seller-distress feed consumed by `daily-opportunity-scout-co-ut`.
schedule: Daily @ 7:00 AM MT
output: CSV to OneDrive (.../CoStar Exports/_csv/Distress_CO_UT_YYYY-MM-DD.csv) + email briefing
owner: jason.koch@mmgrea.com
supersedes: daily-foreclosure-auction-scan-co-ut
---

# Distress Monitor — Colorado & Utah (foreclosure + CRED iQ)

## Why this exists
The old `daily-foreclosure-auction-scan-co-ut` skill (a) stopped delivering — no briefing has
hit the inbox recently — and (b) only ever covered auction/foreclosure **websites**. It never
surfaced **CMBS special-servicing, lender watchlists, loan maturities, or lis pendens** — which
are the highest-value seller-distress signals (e.g., a maturing 7.3% floating CMBS portfolio).
This skill fixes delivery and adds the loan-level (CRED iQ) layer, and it outputs **structured
rows**, not just prose, so the opportunity scout can score them directly.

## Objective
One daily run producing two outputs:
- **A CSV** of all current CO/UT distress signals (the canonical artifact the scout reads).
- **An email briefing** (skimmable, MMG-branded) for Jason.

Multifamily-first; also retail, office, industrial, hospitality, land. Exclude 1–4 unit
residential unless part of a 5+ unit / plausibly-commercial portfolio.

---

## Layer A — Public foreclosure / auction / trustee / lis pendens
Scan every source, every run:
1. Auction.com — commercial filter, CO and UT
2. Xome — commercial/multifamily filter, CO and UT
3. Foreclosure.com — commercial section, CO and UT
4. LoopNet — auction listings filter, CO and UT
5. **CO county Public Trustee** sale calendars: Denver, Arapahoe, Adams, Jefferson, Douglas,
   Boulder, Larimer, Weld, El Paso, Pueblo, Mesa
6. **UT county Sheriff / trustee** sale calendars: Salt Lake, Utah, Davis, Weber, Washington,
   Cache, Summit
7. **Lis pendens** — county court records / CoStar "lis pendens" flag where available, same counties

For each item capture: address, asset type, units/SF, sale/auction date, opening bid/price,
trustee or platform, and a direct link. Flag any sale date **within 14 days** as TIME-SENSITIVE.

## Layer B — CRED iQ loan-level distress
Pull (or refresh) the CRED iQ export for **CO + UT, Apartment, units ≥ 30** (go smaller only for
newer construction). Capture these statuses:
- Transferred to **special servicer**
- On **watchlist** (note the WL code / reason)
- **Maturity within 18 months** (flag <12 mo as urgent), especially **above-market / floating** rate
- Payment default / forbearance / modification

For each loan capture: borrower entity, property, units, year built, lender / CMBS deal, special
servicer, loan amount, rate, **maturity date**, LTV, DSCR, and a one-line servicer note.

> CRED iQ is licensed data — keep the export in OneDrive; do not redistribute externally. There is
> no live API path here; refresh the export on a fixed cadence (≤ 30 days; weekly preferred).

---

## Output 1 — CSV (canonical; the scout reads this)
Write to `OneDrive/.../CoStar Exports/_csv/Distress_CO_UT_YYYY-MM-DD.csv`, one row per distinct
property, columns in this exact order:

`Scan Date, State, County, Submarket, Property Name, Address, City, Units, Year Built,
Owner/Borrower, Distress Signal, Loan Status, Lender/CMBS Deal, Special Servicer, Loan Amount,
Rate, Maturity Date, LTV, DSCR, Sale/Auction Date, Source, Source Link`

- `Distress Signal` = the single most actionable flag, e.g. `lis pendens filed 6/18`,
  `trustee sale 7/02`, `transferred to special servicer`, `CMBS maturity <12mo @7.3% floating`,
  `watchlist 1E low DSCR`.
- **De-dupe across A and B**: same address from a website AND CRED iQ = one row carrying both
  (combine signals; keep both source links).
- Sort: multifamily first, then by soonest sale/maturity date.

## Output 2 — Email briefing
Subject (pin this — the scout matches on it):
`Distress Monitor — CO/UT — <Mon DD, YYYY>`
Structure:
- Executive summary (3–5 most actionable bullets)
- TIME-SENSITIVE (sales/maturities within 14 days)
- Colorado — grouped by asset type; Utah — grouped by asset type
- Footer: CSV path, sources scanned, any that failed, CRED iQ export date

---

## Delivery hardening (the reason the old feed went dark)
- **Always send**, even with zero new items — include the source-check log so Jason knows it ran.
- Log every source failure in the footer; never drop a source silently.
- Write the CSV first; if the email fails, the CSV still exists for the scout.
- Keep the subject convention **exactly** as above so downstream automation can find it.

## Success criteria
- Every source attempted; failures logged, not hidden.
- No cross-source duplicates; every entry has a working link or servicer reference.
- CSV written to the OneDrive path; email delivered to jason.koch@mmgrea.com.
- CRED iQ layer present and its export date stated.

## Constraints
- **Never invent** a listing, loan, borrower, or distress signal. If a source is inaccessible,
  say so in the footer. Report only what the sources contain.
- Multifamily-first sizing aligns with the scout: 30–200 units ideal, smaller only when newer.
