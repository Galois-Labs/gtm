# Org-mapping rules (org-mapper subagents)

The Tier-0 buying-map contract. Every org-mapper worker follows this file exactly; the main
session pastes it **verbatim** into every org-mapper prompt. **Read the whole file before the first
lookup.** It exists so the map is a decision aid, not a headcount census, and so an inferred edge
never gets mistaken for a public fact.

## 1. What this is

A **buying map for ONE decision** — who signs (economic buyer), who champions it, who to talk to
first — covering the **5-15 relevant people** at a company, never the full HR chart. You are not
building an org chart; you are answering "who do we approach, in what order, and who reports to
whom **where a public source says so**."

**Inferred edges are routing metadata.** A reporting edge you deduced from titles (rather than read
on a public page) may shape *who gets drafted first and in what register*, but the messages skill
will **refuse it in message copy**. Flag every edge honestly: `stated` (a public source says it) or
`inferred` (you deduced it). A wrong `stated` flag is the one error that can leak an unverified
reporting claim into a message.

## 2. Budget and the public-search boundary

- **≤4 total WebSearch+WebFetch actions per company. Hard cap.** The `$0`-lookup sources in §3 run
  FIRST and cost nothing against this budget; spend the 4 only after.
- **NONE-not-loop.** An unmappable company gets `role=unknown` rows or nothing — never a 5th
  lookup. Over-searching is the failure mode; `NONE` is a valid, expected result.
- **The public-search boundary (same as the dork lane).** `linkedin.com` is **SERP-link-titles-only**
  via WebSearch, under the dork trust rules (trust the `Name - Title` link title, ignore the prose
  snippet, `/in/` hrefs only). **NEVER WebFetch a `linkedin.com` URL.** Non-LinkedIn public pages —
  the company's own site, ATS JSON, gov registries, press — **ARE** fetchable.

## 3. `$0`-lookup recipes (run first, no budget)

**(a) Title inference over the pasted harvested People rows.** Every harvested row already carries a
title. Run the §5 rubric on it: seniority, function, org_role, and inferred reporting edges — zero
lookups. This is the substrate the rest fuses onto.

**(b) Registry person-name reuse.** The `/galois-targets` registry lines are already in the Companies
row's `signals`/`sources`. Reuse the NAMED people at 0 lookups:

- **openFDA 510(k) `contact` / correspondent** = a NAMED RA/QA person → `function=regulatory`,
  `org_role` per personas, `reports_to_basis` may be `stated` with `evidence` = the `api.fda.gov`
  URL already in the signals line.
- **SEC Form D "related persons"** = officers / directors → `function=executive`, `evidence` = the
  EDGAR URL.
- **CA SoS Statement of Information officers** → `evidence` = the `bizfileonline` URL.

These are `new_person=true` rows (name + title only, no linkedin, no email) unless they match a
harvested row.

## 4. Budgeted recipes, in value order

**(a) ATS reports-to mining — ONLY when a `hiring_signal` line already names the ATS board.**
Piggyback; **never guess a slug**. 1 fetch:

```
GET https://boards-api.greenhouse.io/v1/boards/{co}/jobs?content=true
GET https://api.lever.co/v0/postings/{co}?mode=json
```

Regex the posting content for `report(s|ing) (directly )?to (the )?<Title>` → a **STATED** edge
(the open role → that title), `evidence` = the posting URL. `team of N` / `join our N-person QA
team` → notes-grade context. The posting's own title also proves the FUNCTION exists and who runs
it.

**(b) Company `/team` | `/about` | `/leadership` page — 1 fetch.** WebFetch `https://<domain>/team`,
fall back to `/about`, then `/leadership` — pick ONE best guess (or 1 `site:<domain> team OR
leadership` search, counted against budget). Names + titles on the company's **own** page are
**STATED**, `evidence` = the page URL; the page's ordering / grouping is a stated hierarchy hint.

**(c) Press-release quote titles — 1 search.** `"<company>" appoints OR names OR "said" "<function>"`.
A quote attribution ("said Jane Doe, VP Engineering at Acme") is **STATED** title evidence,
`evidence` = the article URL. Check the date — the stale-title rule (§6) downgrades press older than
~18 months.

**(d) `theorg.com` — ONE cheap try.** WebFetch `https://theorg.com/org/<kebab-name>`. 404 / empty =
move on immediately, **never a second guess**.

## 5. Title-inference rubric (the fallback when nothing is stated)

**Seniority** (1 most senior):

| band | titles |
|---|---|
| 1 | founder · CEO · president · C-suite (any Chief X) · owner |
| 2 | VP · SVP · EVP · Vice President |
| 3 | Director · Head of |
| 4 | Manager · Lead · Supervisor |
| 5 | Senior / Staff / Principal individual contributor |

An unmapped title → seniority **blank** and `org_role=unknown`. Most-senior rule wins ties: a
"Senior Director" is 3 (director), not 5.

**Function** ∈ `engineering | quality | regulatory | operations | executive | other` by token table:
quality = QA / quality / QMS; regulatory = RA / regulatory / regulatory affairs / compliance;
engineering = engineer / software / hardware / mechanical / electrical / R&D / CTO / technical /
design; operations = operations / supply chain / manufacturing / production / procurement;
executive = CEO / president / founder / CFO / GM; else `other`.

**org_role:**
- `economic_buyer` = the highest-seniority person matching `personas.buyer` in the relevant
  function (**exactly one per company** where possible; demote the rest to influencer).
- `champion` = a `personas.champion` match.
- `influencer` = an adjacent-function senior person (director+).
- `blocker` = procurement / legal / finance / compliance-gatekeeper tokens.
- else `unknown`.
- `personas.exclude_tokens` people (recruiting / talent / HR) are **never emitted**.

**reports_to inference:** a rank-N person reports to the nearest rank N-1 in the same function (else
the function head, else rank 1) → `basis=inferred`, `evidence=title-inference`. **A stated edge
ALWAYS wins over an inferred one.**

**approach_order:** 1 = best champion, 2 = economic buyer, then influencers by seniority. A blocker
**never gets 1**. Ties break toward the person with a harvested linkedin slug (DM-reachable now).

## 6. Failure-mode checks (run before emitting — Bay-trap style)

- **Same-name person.** The evidence page must tie the person to THIS company (its domain, or an
  unambiguous company name on the page). If it does not, **drop the person** — a fact about the
  other "Acme" is not this Acme's.
- **Title inflation.** In a ≤20-person company a "VP" may be an IC. Cap seniority at the company's
  size band (use `icp.size_bands` context if pasted) and note it.
- **Stale titles.** Press older than ~18 months downgrades to inferred-grade; `basis` stays
  `stated` only if the company's own **current** page confirms it.
- **Parent / subsidiary confusion.** An edge must live inside the workbook company. **Never map
  parent-company executives onto the subsidiary row**, and a team page on the parent's domain maps
  only the people explicitly placed in the subsidiary.

## 7. Optional Apollo lane (Tier-1, OFF by default)

> **OPTIONAL — UNVERIFIED ON OUR PLAN. Off by default. Ask first, every wave it would run.**

Our verified-endpoints memory covers `org/match` + `people/match` + the `mixed_people/search` 403
only; this lane is **documentation until a probe confirms it**. If the cohort turns it on:

- **Endpoint:** `POST https://api.apollo.io/api/v1/mixed_people/api_search` — note `api_search`, NOT
  `/api/v1/mixed_people/search` (403 API_INACCESSIBLE on our plan; that warning in the reveal
  section stays untouched). Per Apollo's docs it needs a **MASTER** API key, consumes **NO** export
  credits, and returns people with titles/seniorities but **NO emails** — exactly a roster.
- **Body:** `{"q_organization_domains_list":["<domain>"],"person_seniorities":["owner","founder","c_suite","vp","director","manager"],"per_page":25}`. Read only name, title, seniority, and Apollo's
  seniority/department labels.
- **Binding constraints:** (a) **main-session `curl` only** — API keys never reach a subagent
  prompt; (b) **ZERO reveal credits** — never call `people/match`, never request or accept an email
  field, never write `email`/`email_source`; (c) **ask-first** every wave ("Apollo roster lane is
  optional and unverified on our plan — try it for these N companies? It should cost 0 credits; if
  the plan refuses with 403 we log one line and continue keyless"); (d) results **fuse like any
  keyless source with `evidence=apollo_api_search`** and are **inferred-grade for the copy gate** —
  an API roster is never a stated public edge, so nothing from this lane can ever be referenced in
  message copy; (e) failure = one plain line ("apollo api_search refused on this plan — org map
  stays keyless"), no retry loop, wave continues.

## 8. Output shape

Emit the JSONL the org-mapper agent def specifies — one `org_person` object per mapped person and
exactly one `org_summary` object per company, in a fenced block, with a plain summary line per
company above it. `company_name` + `full_name` required; `evidence` required on every `org_person`;
`reports_to_basis=stated` requires a URL evidence; `new_person=true` only for someone absent from
the pasted People rows. **Companies with nothing mappable report `NONE` — no rows.**
