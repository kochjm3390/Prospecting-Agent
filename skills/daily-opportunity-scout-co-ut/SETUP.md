# Setup — making the daily scout durable

The scoring skill (`SKILL.md`) is only as good as the data flowing into it. Two layers age
very differently, and the daily decay you're worried about is a **feed** problem, not a
CoStar-login problem.

- **Slowly-changing master** (ownership, units, year built, owner phone/email): a **monthly**
  CoStar export is the correct cadence. It does not rot in 4–5 runs.
- **Fast-moving events** (distress, lis pendens, CMBS special servicing, new/withdrawn
  listings, maturities): these rot fast — and they already arrive **daily by email** plus the
  bi-weekly CoStar status-change file, all of which the scout reads live.

> Note on CoStar: there is no live API/credential path, and automated extraction violates
> CoStar's license. Distress data comes from **CRED iQ**, not CoStar. CoStar's role is the
> monthly ownership/listing master.

---

## Fix 1 — Repair the distress feed (highest leverage)  ✅ skill drafted: `distress-monitor-co-ut`
Verified findings:
- No `Foreclosure & Auction Briefing — CO/UT` email has landed recently (nothing since mid-May) —
  the old `daily-foreclosure-auction-scan-co-ut` skill **stopped delivering**. Only the 5 AM
  Market Intelligence Briefing is reliably arriving.
- That old skill also only scanned auction **websites** — it never covered CMBS special-servicing,
  watchlists, maturities, or lis pendens (the CRED iQ loan-level signals).

The replacement skill `skills/distress-monitor-co-ut/SKILL.md` fixes both. Remaining wiring:
- [ ] Deploy `distress-monitor-co-ut` in Cowork (daily 7 AM MT); retire the old skill.
- [ ] Confirm it actually **sends** to jason.koch@mmgrea.com and **writes the CSV** to
      `_csv/Distress_CO_UT_YYYY-MM-DD.csv` (check Sent/automation logs, not just the schedule).
- [ ] Set up the **CRED iQ export** (CO/UT, apartment, units ≥ ~30) on a ≤30-day refresh — weekly
      preferred — saved to OneDrive so Layer B has structured data.
- [ ] Subject is pinned to `Distress Monitor — CO/UT — <date>`; the scout matches on it. Do the
      same (confirm sending + pin subject) for the **Competitor New-Listing Monitor** (7:15 AM).

## Fix 2 — CSV-ify the master files (removes the 406 blocker)
The big `.xlsx` masters exceed the file-reader's text-conversion ceiling and return HTTP 406, so
ownership and contact data is currently unreadable by automation. On **each monthly CoStar
refresh**, also save CSV copies into a stable OneDrive folder:

```
OneDrive/.../CoStar Exports/_csv/
  CO_AllProperties_YYYY-MM.csv
  UT_AllProperties_YYYY-MM.csv
  CO_Solds_YYYY-MM.csv
  UT_Solds_YYYY-MM.csv
  CO_ForSale_YYYY-MM.csv
  UT_ForSale_YYYY-MM.csv
  mLink_master_YYYY-MM.csv
```
- One sheet per CSV; keep the CoStar "True Owner Name / Contact / Phone" columns.
- CSVs read reliably regardless of size. (Optional: also keep the `.xlsx` for humans.)

## Fix 3 — Scoring skill (done)
`SKILL.md` codifies the rubric, exclusions, geo/seller-buyer mix, de-dupe, and output/log
format so every run is identical no matter who or what triggers it. Wire it into the Cowork
scheduler (daily, ~6 AM MT) with write access to OneDrive so it can append to
`MMG_Opportunity_Log.xlsx`.

---

## Run architecture (chosen)
Cowork scheduler runs the skill daily and writes the log to OneDrive (next to the CoStar
databases). The scout reads feeds + CSV masters + pipeline, scores, emails the ranked top-10,
and appends rows. The `MMG_Opportunity_Log.xlsx` template (correct column order) is committed in
this repo at the project root as the starting point.
