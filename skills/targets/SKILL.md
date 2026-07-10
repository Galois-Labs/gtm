---
name: targets
description: >-
  Discover the day's target companies straight into the workbook's Companies tab
  and tier them into a trigger-sorted slice of 10-20. Run the profile's discovery
  plan — gate-matched lanes only: web-search and hiring-signal discovery for any
  vertical, registry pulls (openFDA / USAspending / SEC EDGAR) solely when the
  ICP trait matches, plus any CSV the human dropped in the folder — dedupe in the
  Companies tab itself, apply the 2+ lane multi-signal gate as a tier-A proposal,
  rank by trigger recency, then subagent-tier the single-signal residue
  (subagents RETURN rows, the main session writes). Use to build or refresh the
  target list before /galois-enrich.
when_to_use: The profile is confirmed and you need a tiered, trigger-sorted slice of companies in the Companies tab before enrichment — a full build or a delta pull of new triggers. Run once per daily slice.
argument-hint: "[--since 1d | <segment-key>]"
allowed-tools: Bash, Read, Agent, WebSearch, WebFetch
---

> If a structured question tool is available (`AskUserQuestion` on Claude Code — one question per call; `ask_user_question` on Codex — at most 3 per turn, 2–3 options plus free text, pause the auto-resolve countdown), use it. Otherwise restate each question **verbatim**, then lettered options with the recommended one marked, and accept a letter, "recommended", or the user's own answer. Never present options without restating the question (the spec-kit #1147 lesson). On Claude Desktop, at most 2 questions per message. Targets rarely asks anything — only which campaign folder (if more than one exists) and, on a pre-v2 profile, the single discovery-plan confirmation in §1.


# Targets — discover + tier the daily slice into the Companies tab

**The workbook IS the state.** This wave reads `~/GaloisGTM/<campaign>/gtm.xlsx`,
discovers companies by running the lanes in the profile's **discovery plan**
(gate-matched only — a registry fires solely when the ICP trait matches; for most
ICPs the backbone is web-search and hiring-signal discovery), dedupes and merges
them **inside the Companies tab** (the tab is the index — there is no database),
proposes a tier, ranks by trigger recency, tiers the residue with subagents that
return rows, writes the Companies tab back, and closes with one line. No `galois`
CLI, no sqlite, no ingest ceremony. Everything is idempotent over the tab:
re-running merges rather than duplicates; a delta pull only tiers the net-new
residue.

The whole flow is single-writer: **you** (the main session) perform every
workbook read and write. Subagents never open the file — they receive rows in
their prompt and return rows as text (§6).

## 0. Workbook + profile gate — never tier without a confirmed profile

**Locate the campaign.** Scan `~/GaloisGTM/*/gtm.xlsx`. Each campaign folder holds
`gtm.xlsx` + `galois.profile.yaml` side by side. If exactly one campaign exists,
use it. If several, ask which. If none, tell the user to run `/galois-setup` and
stop — setup creates the folder, the workbook, and the profile.

**Check the dependency once per session:**

```
python3 -c "import openpyxl" || pip install --user openpyxl
```

If install is impossible, fall back to the CSV form of the workbook —
`<campaign>/sheets/Companies.csv`, `Suppressed.csv`, `Budget.csv` — same headers,
same conventions, same logic below (read the CSV, merge, write it back).

**Read the profile** (`galois.profile.yaml`, plain YAML — Read tool or `python3`/
`yaml`). Gate on it:
- File missing, or `meta.confirmed_by_user` is not `true`: the interview was never
  accepted. Send the user to `/galois-setup` and **stop**. Nobody gets a target
  list without a confirmed profile.
- Confirmed: print a 2-line banner (`Running as <identity.company> · dial
  <claims.caution_dial> · Tier-A = 2+ lanes · profile v<meta.profile_version>`)
  and proceed with zero further questions.

Keep these ICP fields in hand — they are the raw material for lane selection
(§1) and the residue rubric (§6): `icp.segments` (each `{key, keywords,
pain_evidence}`), `icp.geo` (`{mode, locations, exclusions}`), `icp.size_bands`,
`icp.decisive_question`, `icp.known_traps`, `icp.tier_rubric`, `icp.exclude`, and
`constitution.suppression.domains` — plus, on a schema-v2 profile, the whole
`discovery` block (`adjacent_segments`, `events`, `user_lists`, `plan`): it is
the entire input to §1.

## 1. Read the discovery plan — the profile decides the lanes

**Step 1 is a read, not a choice.** The lanes for this wave live in the profile:
`discovery.plan` — one entry per confirmed segment (core + confirmed
`discovery.adjacent_segments`), each `segment → observable_signals → lanes`, with
`gate_evidence` quoted on every gated lane and a `tier_a_pairing` naming the
2-lane overlap expected to arm the Tier-A gate. Setup derived and confirmed it at
the interview accept gate; **run it verbatim** — add no lanes, and never fall
back to Galois-shaped defaults (openFDA / USAspending / SEC EDGAR are gated lanes
like any other; they exist for this user only if their `gate_evidence` matches
the ICP).

**Profile predates the plan (no `discovery` block)?** Derive it now, in-session,
per `references/source_selection.md`: check every lane gate in its decision table
against each segment's `keywords ∪ pain_evidence` (quote the matching trait as
`gate_evidence` on any gated lane that passes), pick the segment's observable
signals and a `tier_a_pairing`, then confirm with the user in **ONE question** —
a read-back of one line per segment:

> `<segment>` → lanes: `web_search, hiring_signal[, <gated lanes whose trait matched>]`. Run this plan? (yes = I save it to the profile / edit it)

On yes, write the `discovery` block into `galois.profile.yaml` (canonical shape
at the bottom of `source_selection.md`) so every future wave just reads it; on
edits, apply them, then write. Never run a wave off an unconfirmed guess, and
never ask again once the plan is in the profile.

`references/source_selection.md` also holds the honest **killed-by-audit**
humility (why SEC EDGAR is low-strength/supplement-only, why Hunter, bought
intent, and expo lists are not discovery backbones). Free-first is trivially
satisfied — every lane is $0; paid calls live in `/galois-enrich`, never here.

## 2. Discover — run ONLY the plan's gate-matching lanes

Walk `discovery.plan` segment by segment and run **exactly the lanes it lists — a
lane whose gate does not match the profile does not run, full stop.** Hold every
hit **in memory** — do not touch the workbook yet (dedupe in §3 does the
writing). Isolate per lane: a 403, a rate-limit, or an empty result **logs a
one-line note and never crashes the wave** — the run degrades to whatever lanes
answered. Every lane maps onto the pinned Companies columns
(`name | domain | city | state | sources | signals`), stamps **its lane id into
`sources`** (source column = lane name, always), and carries the trigger's date
in `signals` so rank (§5) can sort by recency.

**The universal lanes are the backbone for most ICPs — first-class, full
subagent treatment:**

**1 · `web_search` — category/directory/list discovery.** Fan out one subagent
per segment (the degradation contract and concurrency ceiling in §6 /
`references/orchestration.md` govern discovery fan-out exactly as they govern
tiering). Each subagent gets its plan entry's queries — or instantiates the five
templates in `source_selection.md` from segment keywords + `icp.geo` — runs
**≤6 searches for its segment**, fetches only listing pages (never each company;
per-company verification belongs to the §6 residue), trusts **link titles only**,
and **returns candidate rows as text**: `name | domain | city | state` plus
`signals=listed in "<page title>" (fetched <YYYY-MM-DD>)`. The main session
stamps `sources=web_search` and stages the rows — subagents never open the
workbook.

**2 · `hiring_signal` — job posts naming the pain.** Same fan-out, same caps
(≤6 searches per segment per wave; rows returned as text; main session writes).
Queries from the plan entry or the `source_selection.md` templates
(Greenhouse/Lever/Ashby dorks plus champion-title searches built from
`pain_evidence`). The employer in the hit is the row: `sources=hiring_signal`;
`signals=hiring "<title>" naming <pain term>, posted <date>`. This is the
strongest generic intent signal for **any** vertical — an employer paying a
salary to fight the pain the user's product solves is a lead.

**3 · `csv_import` — the human's own list.** Any `*.csv` in
`~/GaloisGTM/<campaign>/` (or promised in `discovery.user_lists`). Read it in the
main session, best-effort map headers to `name`/`domain`/`city`/`state`, tag
`sources`=`csv_import`, carry any note as a `signals` line. A user-curated list
is the strongest ICP prior — Tier-B-or-better candidates, never residue to
demote.

**Other gated lanes in the plan** (`<event-slug>` rosters, `github_orgs`,
`pkg_registry`, `hn_launch`, `review_directory`, `funding_news`, `yc_directory`):
run each per its recipe and stated budget in `references/source_selection.md` —
fetch/API lanes run in the main session; search-shaped lanes may share the
discovery fan-out above.

**If your plan includes `registry_recipes.md` lanes** (`openfda_510k`,
`usaspending`, `sec_edgar`), follow that file: re-verify the lane's
`gate_evidence` against the current profile before running (profiles drift),
then run the recipe in the main session. That file is the whole registry story —
no registry call appears anywhere else in this skill.

`--since 1d` (delta pull): pass the recency window into each lane's date filter
(registry date params, posted-since on hiring searches, fetch-date lists) so only
net-new triggers come back; the merge in §3 folds them into existing rows.

**Keep the lane ledger as you go** — one line per lane, ran or not:
`ran: <lane_id> — <N> rows` / `skipped: <lane_id> — <reason>`. A gated lane from
the decision table that never made the plan is skipped as
`gate not matched — <plain-ICP-terms reason>` (e.g.
`skipped: openfda_510k — gate not matched: ICP is not regulated medtech`);
a plan lane that failed operationally states the failure
(`skipped: csv_import — no CSV in folder`, `skipped: sec_edgar — API 403,
degraded`). §7's wave summary reads this ledger verbatim.

## 3. Dedupe + merge IN the Companies tab (the tab is the index)

There is no database — **the Companies tab is the dedupe index, re-read fresh every
wave.** Load it and build the index before adding anything:

```
python3 - <<'EOF'
import openpyxl, re
WB="/Users/<you>/GaloisGTM/<campaign>/gtm.xlsx"
wb=openpyxl.load_workbook(WB); ws=wb["Companies"]
hdr=[c.value for c in ws[1]]                       # the header row IS the contract
def col(name): return hdr.index(name)             # column positions by header name
def ndom(d):  d=(d or "").strip().lower(); d=re.sub(r"^https?://","",d); d=d.replace("www.",""); return d.split("/")[0]
def nname(n): return re.sub(r"\b(inc|llc|corp|ltd|co|gmbh|company)\b|[^a-z0-9]","",(n or "").lower())
by_dom, by_name = {}, {}
for i,row in enumerate(ws.iter_rows(min_row=2), start=2):   # i = real sheet row number
    d=ndom(row[col("domain")].value); n=nname(row[col("name")].value)
    if d: by_dom[d]=i
    if n: by_name.setdefault(n,i)                 # domainless fallback index
# also load Suppressed once:
sup={ (r[0].value or "").strip().lower() for r in wb["Suppressed"].iter_rows(min_row=2) if r[1].value=="domain" }
EOF
```

Then for each company discovered in §2:

1. **Suppression check FIRST.** If its normalized domain is in the Suppressed tab
   (scope=`domain`) or in `constitution.suppression.domains` from the profile,
   **drop it — suppressed keys never (re)enter the workbook.** A `domain`-scope
   key with no dot in it is a company name (registry rows often have no domain
   yet) — match it against the normalized company name instead.
2. **Match by domain, then by normalized name.** Prefer domain (dodges same-name
   collisions per the tiering traps). A name-only match is provisional — if a
   domain resolves later it may split into its own row.
3. **Hit an existing row → MERGE in place** at that row number:
   - `sources`: **union** the comma list (this is what arms the 2+ gate on a
     re-pull — `web_search` + a later `hiring_signal` becomes two lanes, in any
     vertical; `openfda_510k` + `usaspending` the same way when those lanes are
     gated in).
   - `signals`: append any new dated trigger not already present.
   - **Never touch the human-owned columns** (`TIER`, `SELECT`) or a non-blank
     `notes`; leave `tier`/`rank` for §4–§5 to refresh.
4. **No match → stage a new row** with the pinned columns filled, `tier`/`TIER`/
   `SELECT` blank.

Keep the staged new rows and the merged row-numbers in memory; §4–§6 finish the
machine columns before the single write in §7.

## 4. Multi-signal gate — the deterministic Tier-A proposal

Pure instruction, no code oracle: **any company whose `sources` cell now holds 2+
distinct lane ids → propose `tier` = `A`.** Multi-signal means 2+ distinct
**lanes** — which now works for any vertical (`web_search` + `hiring_signal`
arms it exactly as `openfda_510k` + `usaspending` did on the Galois profile);
two hits from the *same* lane count as one, per `source_selection.md`. These are
the strong accounts; they **skip the residue verify entirely** (§6). Set the
lowercase `tier` only — never write or overwrite the uppercase human `TIER`.
Everything left with a **single** lane id (and blank `tier`/`TIER`) is the
**residue** for §6.

**If fewer than 3 companies clear the gate, say so plainly and move on.** That is
the gate being strict over thin lane overlap (a seed-only tab or one dead lane
can leave A at 0–1), not a stall. Never hand-promote a single-source
company to A to fill it — the residue rubric (§6) still rescues a genuinely buried
designer, and rank (§5) means a fresh-trigger Tier-B outranks a stale Tier-A, so
the queue never starves. Report it in one line and continue.

## 5. Rank — trigger recency dominant

Write the `rank` column so downstream skills read head-of-queue order straight off
the tab. Sort every non-suppressed row by, in order: **(a) freshest `signals` date,
descending — dominant; (b) effective tier A > B > C** (effective tier = `TIER` if
non-blank else `tier`); **(c) source count, descending.** Stamp `rank` = 1, 2, 3, …
down that sort — `rank` 1 is head-of-queue. A fresh-trigger Tier-B row *should*
outrank a stale Tier-A row; that is the intended behavior, not a bug.

## 6. Tier the residue with subagents — they RETURN rows, you write

The residue is every row left at single-source with blank `tier` **and** blank
`TIER`. Tier it by ICP **fit**, not signal count. Build the rubric block from the
profile once and paste it into every subagent prompt — the A/B/C definitions, the
decisive question, the trap list, and the search discipline are in
`references/tiering.md`; the fan-out mechanics, chunk sizes, and the exact
return-line shape are in `references/orchestration.md`.

**The concurrency rule, stated hard: subagents NEVER open, read, or write the
workbook.** You pass each chunk's residue rows (domain, name, segment, description,
signals) into the subagent prompt as text; the subagent **returns verdict lines as
text**; you parse them and write the `tier` column yourself, single-threaded,
**checkpointing the workbook after each chunk** (a stall loses at most one chunk).

**The degradation contract, verbatim:**

> If your platform supports parallel subagents, fan out at 8–12 (Claude Code) or ≤6 (Codex), chunks of 4–6. If not, process the same chunks sequentially, checkpointing to the workbook after each chunk.

Chunk sizes split by whether the chunk touches the web:
- **No-web (description-only judgment): chunks of 40–50.** Most residue tiers off
  the cached description + segment + signals with zero searches.
- **Web (a genuinely ambiguous name or a suspected buried ICP company): chunks of
  4–6.** One WebSearch per company, `<name> <domain> products`, trust **link titles
  only**; **≤2 searches per company hard cap; write the tier from the description
  rather than loop** — over-searching is what stalled 33 agents in the audit.

Each subagent returns one line per company it actually re-tiered (the JSONL shape
in `references/tiering.md`, e.g.
`{"company_key":"astrafab.example","new_tier":"A","note":"buried space-electronics designer; decisive=YES"}`).
A company it cannot classify within budget gets **no line** (stays blank, surfaces
next wave) — write NONE rather than loop. Parse the returned lines and set `tier`
on the matching row (`company_key` resolves by domain, else normalized name); a
rescued buried designer may get `A`, but never emit `A` for a routine single-signal
row — that is the gate's job (§4). Then write the workbook (this is the checkpoint)
before launching the next chunk.

## 7. Write-back + wave log + outro

**Before the first write of the wave, say:** *"If gtm.xlsx is open in Excel/Numbers,
close it now — Excel's next save would silently overwrite what I'm about to write."*

Write discipline (**load and mutate the existing workbook in place** — never rebuild
a sheet, or you drop the A1 notes and frozen header setup wrote): copy `gtm.xlsx` →
`gtm.prev.xlsx` (one-generation undo), save to `gtm.tmp.xlsx`, then atomic
`os.replace("gtm.tmp.xlsx","gtm.xlsx")`.

```
python3 - <<'EOF'
import os, shutil, datetime, openpyxl
WB="/Users/<you>/GaloisGTM/<campaign>/gtm.xlsx"
shutil.copyfile(WB, WB.replace("gtm.xlsx","gtm.prev.xlsx"))
wb=openpyxl.load_workbook(WB)
# ... update merged rows by number; append staged new rows; write tier/rank ...
b=wb["Budget"]                                    # append one wave-log row (row4+ block)
b.append([datetime.datetime.now().isoformat(timespec="seconds"),
          "targets", "<surface>", <rows_added>, <rows_changed>, 0, "<one-line note>"])
tmp=WB.replace("gtm.xlsx","gtm.tmp.xlsx"); wb.save(tmp); os.replace(tmp, WB)
EOF
```

Targets spends **$0** — the wave-log `credits_spent` is always 0 (no paid call
lives on this path; Apollo/Hunter are `/galois-enrich`).

**Close with the mandatory wave summary** — the one-line outro shape every skill
uses, plus targets' **lane ledger line** (from §2) naming which lanes ran AND
which were skipped, with the reason:

> Wave done: +<N> companies, <M> auto-tiered A (2+ lanes), <K> need your TIER call — edit anything in gtm.xlsx; the next wave reads whatever is there.
> Lanes: ran <lane_id> (<n> rows) · <lane_id> (<n> rows); skipped <lane_id> — <reason>; <lane_id> — <reason>.

Skipped-clause format, one per skipped lane:
`<lane_id> — gate not matched: <plain-ICP-terms reason>` for a gated lane whose
trait is absent (e.g. `openfda_510k — gate not matched: ICP is not regulated
medtech`), or the operational reason otherwise (`csv_import — no CSV in folder`,
`sec_edgar — API 403, degraded`). Lanes skipped for an identical reason may be
grouped (`openfda_510k, usaspending — gate not matched: ICP is neither medtech
nor a gov contractor`). Never omit a skipped registry silently — the ledger is
how the user sees the gates working.

Then hand off: the Tier A/B rows with fresh triggers (top of the `rank` order) are
the input to `/galois-enrich`. Tier C is parked in the tab, never deleted — a future
trigger promotes it at $0. UPPERCASE columns (`TIER`, `SELECT`) are the human's; the
next wave reads their edits as current truth and never overwrites them.

---

*Optional deterministic engine (power users only): the legacy `galois` CLI + sqlite
KB survive in the repo's `src/` as a faster batch engine for heavy registries. No
skill requires it; this workbook flow is complete without it.*
