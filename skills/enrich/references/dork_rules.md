# LinkedIn-dork trust rules (contact-hunter subagents)

Ported verbatim-in-substance from the stall-fixed Bay-run gap-fill harvest prompt.
Every contact-hunter worker follows these exactly. They exist because prior batches
stalled from over-searching and because search engines confabulate titles that are not
in the actual results. Read the whole file before the first search.

## Persona priority (from the profile, not hardcoded)

Read `personas.buyer`, `personas.champion`, and `personas.exclude_tokens` from
`galois profile show --json`. The orchestrator injects them into your prompt. Rank the
people you keep in this order:

1. **BUYER** — the titles in `personas.buyer` (who signs). Founder / CEO / CTO / VP or
   Director of the relevant function, Head of <function>, Chief Engineer, and the like.
2. **CHAMPION** — the titles in `personas.champion` (who fights for it daily). Senior /
   staff / principal individual contributors and their line managers in the relevant
   function (engineering, quality, test, regulatory, and so on).
3. **EXCLUDE** — never an entry point: anything matching `personas.exclude_tokens`
   (default: recruiting, talent acquisition, sourcers, HR) plus sales / BD, marketing,
   admin / EA, students, interns.

Aim for 2-3 rows per company: one buyer plus one or two champions where findable.

## The search method

- **Restrict WebSearch to `linkedin.com`.** Set `allowed_domains: ["linkedin.com"]` on
  every call. Query shape: `"<Company>" <buyer/champion title terms OR'd together>`
  (build the OR list from the profile personas, e.g.
  `"Acme Surgical" VP Engineering OR Director Engineering OR CTO OR Quality OR Principal Engineer`).
- Try ONE search per company. If nothing usable, ONE broader fallback search. That is it.
- Disambiguate the company via its domain when the name is common.

## The trust rules (this is the whole point)

- **Trust the result LINK TITLES only** — the `Name - Title at Company` string that is the
  clickable title of a `linkedin.com/in/...` result.
- **IGNORE the search engine's prose summary. It confabulates** titles and affiliations that
  are not in the actual results. Never lift a name, title, or company from the snippet text.
- **Record a contact only when its name AND company are both visible in an actual `/in/`
  URL + link title.** No `/in/` result you can read = no contact.
- If you are not confident it is the right person at the right company, write `NONE` for
  that company rather than guess. A wrong contact is worse than a missing one.

## Slug normalization

Normalize every profile URL to `https://www.linkedin.com/in/<slug>/` (strip country
subdomains like `uk.` / `es.`, strip query strings, keep a trailing slash). Record the
`<slug>` (the path segment after `/in/`) as `linkedin_slug` in the output.

## Search budget and the anti-stall rule

- **Max 2 WebSearch calls per company. Hard cap.**
- **Do not loop on hard-to-find companies. Write `NONE` and move on.** This is critical:
  over-searching is what stalled prior batches. `NONE` is a valid, expected result.
- Write `NONE` rather than fire a third search or invent a plausible-looking person.

## The ddgs fallback lane (no host WebSearch tool)

Bare terminal or cron run with no host `WebSearch`? Degrade to `ddgs` — the free DuckDuckGo
wrapper (`pip install ddgs`, or `pip install 'galois-gtm[search]'`). This is the **same
public-search boundary**, just a different backend: link titles + hrefs off a SERP, never a
logged-in fetch of a `linkedin.com` page. Same ≤2 searches per company, same `NONE`-not-loop,
same trust rules.

**Command shapes** — site-restricted to `linkedin.com/in` so the SERP returns profiles:

```
# 1) buyer/champion pass: OR the profile's target titles together
ddgs text -q 'site:linkedin.com/in "Acme Surgical" ("Director of Quality" OR "VP Quality")' -m 8
# 2) broader fallback, ONLY if (1) returns nothing usable: drop the title clause
ddgs text -q 'site:linkedin.com/in "Acme Surgical"' -m 8
```

Power users with the optional CLI get those two queries, the ≤2-search cap, the slug
normalization, and the title-parsing contract below all encoded in one read-only command:

```
galois dork-linkedin "Acme Surgical" --domain acme.com --titles "Director of Quality,VP Quality"
```

It prints PersonSignal JSONL — one `{company_name, full_name, title, linkedin_slug}` per line
— ready to hand back exactly like a subagent's return.

**Title-parsing contract (what you may read).** Read the `title` and `href` fields ONLY. A
person is real only when `href` is an actual `linkedin.com/in/<slug>` URL; take the name and
title from the `title` string (`"Name - Title at Company | LinkedIn"`), normalize the slug per
*Slug normalization* above, and drop anything you cannot tie to an `/in/` href.

**The `body` snippet is BANNED.** ddgs also returns a `body` field — the engine's prose
summary. Never read it and never lift a name, title, or affiliation from it: it is the same
confabulation class as the SERP prose the trust rules already forbid. `title` + `href` only.

**The merged-titles quirk.** ddgs occasionally concatenates several result titles into ONE
`title` line — e.g. `"Jane Doe - Eng at Acme | LinkedInJohn Smith - VP at Acme | LinkedIn"`,
where the next name butts straight against the previous `| LinkedIn`. One result still carries
exactly ONE `/in/` href, so it identifies exactly one person. Split the line on the
`| LinkedIn` / `- LinkedIn` seam, keep only the entry whose name matches the slug, and drop the
rest — they have no verified `/in/` URL of their own (`NONE`, never a guessed name↔slug
pairing). `galois dork-linkedin` does exactly this in code.

ddgs not installed = the fallback lane is logged as skipped and never guessed.

## Output shape (PersonSignal JSONL, one object per line)

Emit rows the CLI ingests directly. No email at harvest time; email is a later reveal step.

```json
{"company_name": "Acme Surgical", "full_name": "Maria Chen", "title": "Director of Quality", "linkedin_slug": "maria-chen-8821"}
```

- `company_name` and `full_name` are required; `title` and `linkedin_slug` where found.
- Companies that returned `NONE` produce no row (report the count instead).
- Reply to the orchestrator with: companies processed, contacts found, companies returning
  `NONE`.
