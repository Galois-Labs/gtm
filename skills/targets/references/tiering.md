# tiering.md — the residue rubric (profile-interpolated)

The multi-signal gate (`score run` / `gate multisignal`) already set the strong
accounts to Tier A deterministically. This rubric is what a `tier-verifier`
subagent applies to the **residue only** — companies the gate left at tier NULL
(one signal, or fit-without-a-trigger). It is an ICP-**fit** judgment, not a
signal count. It also catches the gate's blind spot: a real ICP company buried in
the residue that deserves promotion.

## The rubric template (fill from the profile, once, per run)

Build this block from the profile's `icp.*` fields and paste it verbatim into
every tier subagent prompt. The bracketed slots interpolate:

```
ICP: a company that {icp.segments[].pain_evidence / keywords, summarized into one
"what they build + why they buy" sentence per segment} in {icp.geo.locations},
sized {icp.size_bands}.

THE DECISIVE QUESTION: "{icp.decisive_question}"
Answer it first for every company. It is the one-line tiebreaker; when in doubt,
it decides the tier.

- Tier A: {icp.tier_rubric.tier_a} — clear ICP fit AND the decisive question is a
  confident YES. (Most Tier A is already set by the 2+ source gate; only assign A
  here to rescue a buried, obviously-fitting company — see promotions below.)
- Tier B: {icp.tier_rubric.tier_b} — adjacent or single-signal: fits the segment
  and the decisive question is YES, but with one signal or a harder entry.
- Tier C: {icp.tier_rubric.tier_c} — fit-only or NOT ICP: the decisive question is
  NO, or they sell TO the ICP rather than being it. Parked, never deleted.

EXCLUDE outright ({icp.exclude} + constitution.suppression): competitors, current
customers, and anything under the size floor — tier C with recommend=skip.
```

### Worked instantiation (Galois Altium-wedge, for shape only)

This is what the template looks like filled — do not hardcode it; regenerate from
whatever profile is loaded:

```
ICP: a company that DESIGNS its own electronic hardware, PCBs, embedded systems,
avionics, or instrumentation AND operates in a regulated vertical (aero/defense,
space, medtech, automotive) with heavy compliance burden (DO-254/DO-178C/DO-160/
AS9100/MIL-STD-461/810/IEC 62304/FDA 21 CFR 820/ISO 13485/ITAR). 11-500 employees.

THE DECISIVE QUESTION: "does this company DESIGN its own electronics?"
PCB fabricators and EMS/assembly houses are Tier C regardless of how
aerospace-adjacent they sound. Do not conflate "makes boards" with "designs boards."

- Tier A: clearly designs own electronics + regulated vertical + heavy compliance.
- Tier B: some hardware design but mostly mechanical, OR a test/instrumentation
  vendor that itself designs hardware, OR a large prime where 2-week entry is hard.
- Tier C: NOT ICP — pure machining/sheet-metal/materials/coatings/fasteners/
  connectors-only/distributors/logistics/services/consulting/staffing/academia/
  test-equipment rental/pure EMS or PCB-fab. They sell TO the ICP or do not design PCBs.
```

## Known traps (from `icp.known_traps` + the standing defaults)

Interpolate `icp.known_traps` first, then always append these audit-proven ones:

- **PCB fabricators / EMS / assembly houses tagged A or B → demote to C.** "Makes boards" ≠ "designs boards."
- **Connector / cable / material / coating shops tagged A or B → demote to C.** They sell to the ICP.
- **A real designer buried in B or C → promote.** The audit's canonical miss was space-electronics designers (satellite-avionics houses whose sites read as generic "engineering services" at first glance) wrongly demoted. If the decisive question is a confident YES, rescue it.
- **Same-name confusion → disambiguate via domain, never by name alone.**
- **Software company that sounds hardware-adjacent → Tier C.** "X-for-hardware" SaaS, autonomy/simulation/EDA software. Fits the segment's customers, is not the segment. Judge by what they themselves design.
- **Large primes → technically A, treat as B** (hard entry inside the window); note it in the reason.

## Search discipline (the stall fix)

- **Obvious cases get zero searches.** Clearly EMS, PCB-fab, machining, materials, distribution, services, academia → Tier C from the description alone.
- **One WebSearch only for a genuinely ambiguous name** — `<name> <domain> products` — to confirm what they actually design. Restrict to the result **link titles**; ignore the search engine's prose summary (it confabulates).
- **Hard cap ≤2 searches per company; write the tier from the description rather than loop.** Over-searching hard-to-classify companies is exactly what stalled 33 agents. A confident-enough tier now beats a perfect tier never.
- Tier from **cache first**: description + segment + signals already in the KB carry most of the decision at zero web cost.

## Output — one `decisions.jsonl` line per company

Write only companies you actually re-tiered or confirmed. Either shape is accepted
by `galois decisions apply`; use `company_key` = domain (preferred) or exact name:

```jsonl
{"company_key": "acmesurgical.com", "action": "confirm",  "new_tier": "B", "note": "designs own Class II device electronics; single 510(k) signal + fit", "agent": "tier-verifier"}
{"company_key": "ttm.com",         "action": "demote",   "new_tier": "C", "note": "PCB fabricator, does not design boards (decisive=NO)", "agent": "tier-verifier"}
{"company_key": "astrafab.example",   "action": "promote",  "new_tier": "A", "note": "buried space-electronics designer; decisive=YES, rescued from C", "agent": "tier-verifier"}
```

Compact `kind`-shape is equivalent: `{"kind":"tier","company":"ttm.com","value":"C","reason":"...","agent":"tier-verifier"}`.

Rules for the line:
- `new_tier` / `value` must be `A`, `B`, or `C` (anything else is rejected as an error).
- `company_key` resolves by domain, lowercased name, `name_normalized`, or numeric id — prefer domain to dodge same-name collisions.
- Always include a terse one-line `note`/`reason` (it lands in the audit sidecar) and set `agent` for traceability.
- Do **not** emit A for a routine residue company on a single signal — that is the multi-signal gate's job; reserve a hand-set A for a rescued buried designer.
