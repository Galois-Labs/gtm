# registry_recipes.md — gated registry lanes (openFDA / USAspending / SEC EDGAR)

These three recipes **moved here from the targets skill body** so they read as
what they are: conditional lanes for specific ICP traits, not the default
discovery flow. They were Galois Labs' *own* ICP sources (regulated medtech,
aero-defense, US-gov work) — useful to any user whose profile shares those
traits, useless to everyone else. **Check the GATE first, every time.** The gate
is a checkable condition on `icp.segments` (core + confirmed
`discovery.adjacent_segments`); when setup wrote the lane into `discovery.plan`
it quoted the matching trait as `gate_evidence` — verify it is still true against
the current profile before running (profiles drift).

Shared mechanics: keyless, $0, run with `curl`/`python3` urllib in the main
session (Desktop: chat web-fetch). Hold hits in memory — dedupe/merge in the
Companies tab does the writing. Exception-isolate per source: a 403, rate-limit,
or empty result logs one line and never crashes the wave. Map onto the pinned
columns (`name | domain | city | state | sources | signals`), always with the
trigger's **date** in `signals` so rank can sort by recency. `--since 1d` delta
pulls pass the window into each recipe's date filter.

---

## `openfda_510k` — FDA 510(k) clearances

**GATE (must pass before this recipe runs):** `icp.geo` covers the US **and**
∃ segment s ∈ (core ∪ confirmed adjacent) such that
`s.keywords ∪ s.pain_evidence` intersects
`{medical device, medtech, 510(k), 510k, pma, de novo, fda, iso 13485,
iec 62304, mdr, ivd, diagnostic, clinical}`.
No intersection → this lane does not exist for this user.

Regulatory-clearance triggers. When gated in, the strongest free trigger source
we have measured: **40% Tier-A at $0** on the Galois profile.

```
GET https://api.fda.gov/device/510k.json?search=state:CA+AND+decision_date:[20240101+TO+20260705]&limit=100&skip=0
```

Per result → `name`=applicant, `city`, `state`; `sources`=`openfda_510k`;
`signals`=`510(k) <k_number> <device_name> cleared <decision_date>`.
Page with `skip += 100` (`skip+limit ≤ 26000`), sleep ~0.35s between pages
(~40 req/min unkeyed). Params from the plan entry: `state`, `cities`,
`since_year`.

## `usaspending` — federal award search

**GATE (must pass before this recipe runs):** `icp.geo` covers the US **and**
∃ segment s such that `s.keywords ∪ s.pain_evidence` intersects
`{defense, dod, government, federal, federal contract, itar, ear, cmmc, dfars,
sbir, sttr, prime contractor, subcontractor, gsa}`.

Federal-award triggers (defense / aero / hardware / research contractors). The
classic `tier_a_pairing` partner for `openfda_510k` on a med-device maker that
also holds a federal award.

```
POST https://api.usaspending.gov/api/v2/search/spending_by_award/
{"filters":{"time_period":[{"start_date":"2024-01-01","end_date":"<today>"}],
 "place_of_performance_locations":[{"country":"USA","state":"CA"}],
 "keywords":["avionics","DO-254"],"award_type_codes":["A","B","C","D"]},
 "fields":["Award ID","Recipient Name","Award Amount","Start Date"],"limit":100,"page":1}
```

Per award → `name`=Recipient Name; `sources`=`usaspending`;
`signals`=`federal award <Award ID> $<Award Amount> <Start Date>`. Page politely.
Params from the plan entry: `state`, `agency`, `keywords`, `max_pages` — the
`keywords` come from the segment, never from this file.

## `sec_edgar` — SEC filings

**GATE (must pass before this recipe runs), both conditions:**
(a) the profile **explicitly targets US public companies** — some segment's
`keywords ∪ pain_evidence` intersects `{public company, nyse, nasdaq, ticker,
10-K, 8-K, S-1, investor relations}`; **and** (b) a filing-type event is listed
in that segment's `observable_signals` in `discovery.plan` (i.e. a filing
actually means something to this user's motion).

**Low strength always** — reporting filers skew far above the 11–500 employee
band. A supplement for a filing/funding trigger, never the discovery backbone;
when it is the only lane covering a segment, say so in the run report.

```
-H "User-Agent: galois-gtm <user email from profile>"      # REQUIRED — SEC blocks anonymous UAs
GET https://www.sec.gov/files/company_tickers.json          # CIK map
GET https://data.sec.gov/submissions/CIK{cik:010d}.json     # per-company
```

Per filer → `name`, `state`, hardware SICs first; `sources`=`sec_edgar`;
`signals`=`SEC <form> filed <date>`. ≤10 req/s allowed; be gentler. Params:
`state`, `sic`, `limit`, `max_fetch`.

---

## The hard rule

Never run a lane whose gate does not match the profile. If no lane fits, web-search discovery always does.
