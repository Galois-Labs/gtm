# source_selection.md — the discovery decision table (profile-driven lanes)

**The founder correction this file encodes:** FDA 510(k), SEC EDGAR, and
USAspending were *Galois Labs' own* ICP sources (regulated medtech / aero-defense)
that got fossilized into the generic flow. They are not defaults. A registry
recipe fires **only when the profile trait it detects is actually in the user's
ICP**. For most ICPs the primary lanes are the generic, agent-native ones below —
all still $0.

A pre-requisite for any discovery run: the profile already says **what the user's
company sells** (`identity.product_description`), **who the ICP is**
(`icp.segments`), and — new — **which adjacent customer profiles the user
confirmed** and **the per-segment discovery plan** (`discovery.*`, shape at the
bottom of this file). Selection is reading that plan, not guessing.

## How selection works

1. **Read `discovery.plan` from the profile.** Setup derived it at the interview
   accept gate: per segment → observable signals → chosen lanes (gates already
   checked, `gate_evidence` quoted). If the plan exists, run its lanes verbatim —
   the wave adds no new lanes on its own.
2. **Profile predates the plan (no `discovery` block)?** Derive a provisional plan
   from the table below — check every gate against `icp.segments[].keywords ∪
   pain_evidence` — show the user a 3-line summary (segment → lanes), and write it
   into the profile as the plan for this and future waves. Do not silently run
   Galois-shaped defaults.
3. **Free-first is trivially satisfied** — every lane in this file is $0. Paid
   calls (Apollo/Hunter) live in `/galois-enrich`, never here.
4. **Arm the multi-signal gate per segment.** Each plan entry names a
   `tier_a_pairing`: 2 lanes expected to overlap on the same companies. A company
   with 2+ **distinct source ids** in `sources` auto-proposes Tier A (the
   audit-proven 53%-vs-11% precision). Two hits from the *same* lane (two "top N"
   lists, or G2 + Capterra both stamping `review_directory`) count as **one**
   source — same lane, correlated evidence. That is deliberate.
5. **Search discipline carries over:** subagents return rows, the main session
   writes; ≤2 searches per subagent per company for per-company verification;
   list-level discovery searches are budgeted per lane below; trust **link titles
   only** per `skills/enrich/references/dork_rules.md`.

## The decision table

| lane id (= `sources` value) | gate (checkable condition on the profile) | emits (Companies columns) | cost |
|---|---|---|---|
| `web_search` | **always true** | name, domain, sometimes city/state; membership signal | $0 |
| `hiring_signal` | **always true** (needs ≥1 pain term — profile confirm guarantees it) | name, domain, city/state; **dated intent signal** | $0 |
| `csv_import` | a `*.csv` in the campaign folder, or `discovery.user_lists` non-empty | whatever the CSV holds | $0 |
| `<event-slug>` (event roster) | `discovery.events` non-empty | name, domain; met-in-person/exhibitor signal | $0 |
| `github_orgs` | dev-tools/technical-audience trait (below) | name, domain, city/state; dated OSS-activity signal | $0 |
| `pkg_registry` | same trait as `github_orgs` | name, domain; dated release signal | $0 |
| `hn_launch` | same trait as `github_orgs` | name; dated launch signal | $0 |
| `review_directory` | software-buyer trait (below) | name, often domain; category-membership signal | $0 |
| `funding_news` | funded-startup trait (below) | name, domain; dated funding signal | $0 |
| `yc_directory` | funded-startup trait | name, domain; batch signal | $0 |
| `openfda_510k` | **regulated-medtech gate** — see `registry_recipes.md` | name, city, state; dated clearance signal | $0 |
| `usaspending` | **US-gov-contractor gate** — see `registry_recipes.md` | name; dated federal-award signal | $0 |
| `sec_edgar` | **US-public-company gate** — see `registry_recipes.md`; low strength | name, state; dated filing signal | $0 |

Every lane maps onto the pinned Companies columns
(`name | domain | city | state | sources | signals`), stamps its lane id into
`sources`, and writes a **dated** `signals` line whenever the source carries a
date — rank (§5 of the skill) sorts on signal recency. Undated membership hits
get the fetch date: `listed in <where> (fetched <YYYY-MM-DD>)`.

---

## Universal lanes — primary for most ICPs

### `web_search` — agent web-search discovery. GATE: always true.

Category/directory/list dorking. Instantiate from segment keywords + `icp.geo`;
budget **≤6 searches per segment per wave**; fetch only listing pages, never each
company (per-company verification belongs to the tiering residue). Templates:

1. `top <category> companies <metro> <year>` — listicles, e.g. `top embedded-systems design firms San Diego 2026`
2. `"list of" <category> companies <geo>` — directory and wiki pages
3. `<category> "member directory" <association or geo>` — industry-association rosters
4. `"<standard or adjacent tool>" "case study" <category>` — customers of adjacent vendors, e.g. `"ISO 13485" "case study" contract manufacturer`
5. `intitle:"exhibitor list" <industry> <year>` — trade-show rosters even when the profile names no event

Emits: `sources=web_search`; `signals=listed in "<page title>" (fetched <date>)`.
Membership is a weak signal alone — it exists to pair with a dated lane.

### `hiring_signal` — job posts naming the pain. GATE: always true.

**The strongest generic intent signal for ANY vertical:** an employer paying a
salary to fight the pain the user's product solves is a lead. Instantiate from
`pain_evidence` terms + champion titles; **≤6 searches per segment per wave**.
Templates:

1. `site:boards.greenhouse.io "<pain term>"` (repeat for `jobs.lever.co`, `jobs.ashbyhq.com`)
2. `"<champion title>" "<pain term>" job <geo>`
3. `site:linkedin.com/jobs "<pain term>" "<metro>"` — link titles only, never scrape
4. `"we're hiring" "<pain term>" <category>`

The employer in the hit is the row. Prefer the board-scoped dorks (template 1)
— the open-web forms (2, 4) surface vendor content marketing more often than
postings; link-title discipline decides, and a search that returns zero
employers just logs thin. Emits: `sources=hiring_signal`;
`signals=hiring "<title>" naming <pain term>, posted <date>` (date from the
posting; if none, fetch date).

### `csv_import` — the user's own list. GATE: a CSV exists.

Any `*.csv` dropped in `~/GaloisGTM/<campaign>/` (or promised in
`discovery.user_lists`). Best-effort header map to `name/domain/city/state`,
`sources=csv_import`, carry any note as a `signals` line. **A user-curated list
is the strongest ICP prior** — Tier-B-or-better candidates, never residue to
demote.

### `<event-slug>` — conference/community rosters. GATE: `discovery.events` non-empty.

Only when the profile *names* events (e.g. `spacetech_expo`). Fetch the public
exhibitor/sponsor/speaker page, one fetch per event. Emits: `sources=<event-slug>`;
`signals=exhibitor at <event> <year>`. Audit humility stands: expo lists were
2.8% Tier-A / 78% junk **as a discovery source** — this lane is a warm/overlap
layer, never the backbone, and never a tiering source on its own.

## Technical-audience lanes

**GATE (shared):** ∃ segment s (core or confirmed adjacent) where
`s.keywords ∪ s.pain_evidence ∪ identity.product_description` intersects
`{sdk, api, developer, devops, open source, oss, cli, library, framework,
compiler, embedded software, infrastructure, platform engineering}` — i.e. the
buyer's engineers would touch the product.

### `github_orgs` — keyless org/repo discovery.

```
GET https://api.github.com/search/repositories?q="<segment keyword>"+in:name,description&sort=stars&order=desc&per_page=50
GET https://api.github.com/orgs/<owner-login>        # only owners with type=Organization
```
Quote multi-word keywords and skip readme matching — unquoted terms match
anywhere in any field, and star-sort then returns the world's biggest repos, not
the segment's (live-hit failure mode). Second call yields `name`, `blog`
(→ domain), `location` (→ city/state).
Unauthenticated limits: 10 search req/min, 60 core req/hr — one search per
segment, ≤40 org lookups per wave, sleep ~1s. Emits: `sources=github_orgs`;
`signals=OSS: <repo> <stars>★, pushed <date>`.

### `pkg_registry` — package-registry signals.

```
GET https://registry.npmjs.org/-/v1/search?text=keywords:<kw>&size=50
```
Publisher + `links.homepage` → name/domain. PyPI has no keyless search API — reach
Python publishers via `github_orgs` or `web_search site:pypi.org <keyword>`.
Emits: `sources=pkg_registry`; `signals=npm <package>, last publish <date>`.

### `hn_launch` — Show HN / launch lists.

```
GET https://hn.algolia.com/api/v1/search?query=<kw>&tags=show_hn&numericFilters=created_at_i><epoch-18mo-ago>
```
Keyless Algolia HN API. Emits: `sources=hn_launch`; `signals=Show HN: "<title>" <date>`.

## Software-buyer lanes

### `review_directory` — G2 / Capterra / Clutch category pages.

**GATE:** the ICP makes or is defined by software/services listed in a review
directory — segment terms intersect `{saas, software, app, platform, agency,
msp, consultancy, it services}`, or the profile names a G2/Capterra category.

```
fetch https://clutch.co/<directory>/<city>              # server-rendered, fetches clean
fetch https://www.capterra.com/<category-slug>-software/
```
G2 category pages often sit behind Cloudflare (403 on fetch) — degrade to
`web_search site:g2.com/products "<category>"`, link titles only. Clutch
category slugs vary (the DevOps directory lives at `it-services/devops/msp`,
not `it-services/devops`) — on a 404, locate the real category page with one
`web_search site:clutch.co "<category>"` and fetch the hit. One fetch per
category per wave. Emits: `sources=review_directory`;
`signals=listed in <site> "<category>" (fetched <date>)`. Which site goes in the
signal text, not the source id — G2+Capterra overlap is one source, correctly.

## Funded-startup lanes

**GATE (shared):** `funding ∈ icp.tier_rubric.signals`, or segment terms
intersect `{startup, seed, series a, series b, venture-backed, yc}`.

### `funding_news` — funding lists and round coverage.

`web_search`: `"<segment keyword>" ("raises" OR "seed round" OR "Series A") <year> site:techcrunch.com OR site:finsmes.com OR site:prnewswire.com`
Fetch the roundup/article, extract company + round + date. ≤4 searches per
segment. Emits: `sources=funding_news`; `signals=raised <round> <amount>, <date>`.
Crunchbase API is keyed — do not add it.

### `yc_directory` — YC company directory.

The directory app is client-rendered; go through search:
`web_search site:ycombinator.com/companies "<keyword>"`, then fetch each hit's
company page (server-rendered: website, one-liner, batch). ≤15 fetches per wave.
Emits: `sources=yc_directory`; `signals=YC <batch>`.

## Registry lanes — gated, not default

Full recipes + gates live in **`registry_recipes.md`** (moved out of the skill
body precisely so they read as conditional, not canonical). Summary:

- **`openfda_510k`** — GATE: geo covers US **and** a segment's terms intersect
  the medtech set (`medical device, medtech, 510(k), pma, de novo, fda, iso
  13485, iec 62304, mdr, ivd, diagnostic, clinical`). Strength high *when gated in*.
- **`usaspending`** — GATE: geo covers US **and** a segment's terms intersect the
  gov-contractor set (`defense, dod, government, federal, itar, cmmc, dfars,
  sbir, sttr, prime contractor, subcontractor, gsa`).
- **`sec_edgar`** — GATE: the profile explicitly targets US public companies
  **and** a filing event is in that segment's `observable_signals`. Low strength
  always — supplement, never a backbone.

Galois Labs' own plan legitimately gates in FDA + USAspending — that is why these
recipes existed. They are the worked example of the table, not its defaults.

## Killed-by-audit sources (kept, deprioritized, told plainly)

- **Hunter as primary email source:** 6 verified from 44 searches. Fallback tier
  in `/galois-enrich` only, confidence ≥80 or nothing. Never a discovery lane.
- **Apollo job-title intent sourcing:** 391 credits, 0 companies, 0 signals, 77%
  junk. The `hiring_signal` lane above replaces it at $0 — job posts found by
  search, not bought intent data.
- **Expo lists as discovery backbone:** 2.8% Tier-A. Roster lane is overlap-only.
- **SEC EDGAR as backbone:** filers skew far above the 11–500 band. Gated and
  low-strength.

When a low-strength lane is the only thing covering a segment, use it and say so
in the run report; otherwise the high-strength gated + universal lanes lead.

## The discovery-plan shape in `galois.profile.yaml`

Written by `/galois-setup` at the accept gate (interview proposes adjacent
profiles, user confirms; setup checks every gate and quotes the evidence). Read —
never re-derived when present — by `/galois-targets` §1.

```yaml
discovery:                     # schema_version 2 block
  adjacent_segments:           # inferred neighbors PROPOSED by the agent, confirmed one by one
    - key: contract_medtech_manufacturers
      inferred_from: medtech_qa            # names an icp.segments[].key
      rationale: "same 510(k) submission pain, one step down the supply chain"
      status: confirmed                    # proposed | confirmed | rejected — only confirmed enter plan
      keywords: [CDMO, contract manufacturing, ISO 13485]
      pain_evidence: ["510(k)", design history file]
  events: []                   # only if the user names them, e.g. [{slug: spacetech_expo, name: "Space Tech Expo", year: 2026}]
  user_lists: []               # CSVs the user says they will drop in the campaign folder
  plan:                        # ONE entry per confirmed segment (core + confirmed adjacent)
    - segment: medtech_qa      # icp.segments[].key or a confirmed adjacent key
      observable_signals:      # what in the public world betrays this segment having the pain NOW
        - "510(k) cleared in the last 18 months"
        - "job post naming submission/design-control grunt work"
      lanes:                   # ordered; every id from this file's table; gate must pass
        - id: openfda_510k
          gate_evidence: "keywords contain '510(k)', 'FDA'"   # REQUIRED on gated lanes: quote the trait
          params: {state: CA, since_year: 2024}
        - id: hiring_signal
          queries:
            - 'site:boards.greenhouse.io "510(k)" "quality engineer"'
        - id: web_search
          queries:
            - top medical device companies San Diego 2026
      tier_a_pairing: [openfda_510k, hiring_signal]   # the 2 lanes expected to overlap → auto-Tier-A
```

## The hard rule

Never run a lane whose gate does not match the profile. If no lane fits, web-search discovery always does.
