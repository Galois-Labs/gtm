# The question set — Q1.1 to Q5.4 (the shipped script)

Ask with this wording. One question at a time; exactly one `(Recommended)` option per MC, listed first. Write each accepted answer immediately: `galois profile set <path> <value> --source asked`. Field paths and enums: `profile_template.yaml`.

**Notation:** `[MC]` multiple-choice, inference or house default pre-selected as `(Recommended)` first option; `[MC-multi]` multi-select; `[FREE]` short free text, pre-filled where inferable; `[CONFIRM]` one-tap yes/edit. Every MC carries the built-in "Other" free-text escape.

**Count:** 20 scripted items; Q5.4 is a one-tap confirm, Q2.1a is skipped when nothing adjacent is honestly inferable, and the Q4.2/Q4.3 skip rules drop at least one channel question in most flows, so 18 real questions is still the hard max (Mode B, blank slate) — when Q2.1a fires it counts as exactly ONE question however many profiles it proposes. Escape-hatch core: **Q1.1, Q1.2, Q2.1, Q2.3, Q3.1, Q4.1, Q5.1, Q5.3** — the rest defaulted with `--source default` and listed in `meta.defaulted_fields`. The escape hatch drops Q2.1a (no adjacent proposals); the discovery-plan derivation below still runs on the core segments — it costs zero questions.

**Order rationale:** concrete before abstract (identity → targeting → personas → channel/voice → policy); channels before voice because length bands depend on channel; the dial late because by then the user has seen what verified-vs-inferred looks like in their own answers; budgets last because defaults are sane.

---

## ROUND 1 — Product, offer, sender

**Q1.1 Product one-liner** `[FREE, pre-filled]` → `identity.one_liner`
"Here's how I'd describe what you sell: '{inferred}'. Edit so a stranger in your buyer's seat gets it."
Downgrade to `[CONFIRM]` when the site/doc gave a high-confidence tagline.

**Q1.2 The offer** `[MC + FREE]` → `offer.type`, `offer.price_band`, `offer.artifact`
"What is this outreach selling?"
- Paid pilot `(Recommended)`
- Demo
- Free audit/report artifact
- Intro call
- Custom
Follow-ups (same turn where the surface allows free text): price band if paid; the low-friction giveaway artifact if any (e.g. a 48h gap-report). Enum values: `paid_pilot | demo | free_audit | call | waitlist`.

**Q1.3 Provable proof points** `[FREE-multi]` → `claims.assertable`
"List the concrete, TRUE claims I may assert about you (numbers, named customers, certs, pilots). If it's not on this list, messages won't claim it. 'None yet' is fine."
This list is the self-claims whitelist at dial 0–1.

**Q1.4 Sender identity** `[MC]` → `voice.sender`, plus `identity.sender.{name,title,linkedin_url}`
- You, the founder, first person `(Recommended for <50-person cos — founder-to-peer converts best)` → `founder_first_person`
- Named teammate → `named_teammate`
- Team/brand voice → `brand_voice`

## ROUND 2 — ICP rubric & tiers (the targeting core)

**Q2.1 Dream customer + segments** `[FREE → structured]` → `icp.segments`
"Describe your dream customer company: industry, what they build, the pain that makes them buy. Name 1-3 real companies that fit."
Structure the answer into segments — `{key, weight, keywords: [], pain_evidence: []}` (the `segments.py` shape; `pain_evidence` holds standards/regulations/tool names that mark the pain) — and read them back for confirmation before writing.

**Q2.1a Adjacent profiles** `[MC-multi, agent-proposed]` → `icp.segments` (append, flagged) + `discovery.adjacent_segments`  *(schema_version 2)*
Asked immediately after the Q2.1 read-back is accepted. Counts as ONE question however many profiles it proposes. SKIP if no adjacent profile is honestly inferable — never invent one to fill the slot.
Propose 1-2 adjacent customer profiles you can infer — from the product's mechanics (who else touches the same workflow), who-else-has-this-pain reasoning (one step up or down the supply chain; the same pain in a neighboring vertical), and, in Mode C, site evidence (case-study logos, integrations, who the docs address). One-line rationale each, then a per-profile decision:
"Companies adjacent to that profile likely share the same pain. Based on what you sell, I'd also target:
- **{adjacent profile 1}** — {one-line rationale}
- **{adjacent profile 2}** — {one-line rationale}
Take each: yes / no / edit."
Write rule: each ACCEPTED profile is appended to `icp.segments` as `{key, weight, keywords: [], pain_evidence: [], inferred_adjacent: true}` (weight below the core segment it derives from) AND recorded in `discovery.adjacent_segments` as `{key, inferred_from: <core segment key>, rationale, status: confirmed, keywords, pain_evidence}`. REJECTED proposals are recorded there too with `status: rejected`, so re-runs and drift check-ins never re-propose them. Never fold these into a batched confirm — each proposal gets its own yes/no/edit, whatever its confidence.

**Q2.2 Tier-A signals** `[MC-multi]` → `icp.tier_rubric.signals`
"Which signals make an account top-priority?"
- Hiring for roles your product replaces/assists `(Recommended)`
- Regulatory/compliance events (filings, certs)
- Recent funding/award
- Specific tech in use
- Event/expo attendance
- Buyer-title intent
Enum tokens: `hiring_intent, regulatory_event, funding, tech_in_use, expo_attendance, buyer_intent`.

**Q2.3 Tier definitions** `[MC]` → `icp.tier_rubric`
"Default rubric: **A** = 2+ independent signals verified from 2+ sources (this multi-signal gate tested at **53% precision vs 11%** single-signal in our runs); **B** = 1 verified signal + segment fit; **C** = segment fit only, parked."
- Use this `(Recommended)`
- Stricter (A = 3+)
- Looser
- Custom
Write as `icp.tier_rubric.tier_a '{min_signals: 2, min_sources: 2}'` etc.

**Q2.4 Hard exclusions** `[FREE, pre-filled]` → `icp.exclude`, `constitution.suppression`
"Who must never be contacted? Competitors, current customers, specific domains, size floors."
Pre-fill from any customer list found. Suppression domains/people go to `constitution.suppression.{domains,people}`.

**R2 additions** (ride the round-2 checkpoint, not standalone MC — these carried most of the Bay-run tiering precision):
- The **decisive question** → `icp.decisive_question`: elicit the one-line tiebreaker shaped like "does this company DESIGN its own electronics?"
- **Known traps** → `icp.known_traps`: e.g. "PCB fabricators are C regardless of how aerospace-adjacent they sound."
Propose both from the segment read-back; the user edits or accepts.

**R2 derivation — THE DISCOVERY PLAN** *(agent work, ZERO questions — runs the moment segments are final: Mode B right after the Round-2 checkpoint; Modes A/C at draft time)* → `discovery.plan`, `discovery.events`, `discovery.user_lists`  *(schema_version 2)*
For EVERY segment (core + confirmed adjacent), derive one plan entry:
1. **Observable signals** — 2-4 lines: what in the public world betrays this segment having the pain NOW (a job post naming it, a category/directory listing, a dated trigger event).
2. **Lanes** — select from the decision table in `skills/targets/references/source_selection.md`, keeping only lanes whose GATE matches this segment's `keywords ∪ pain_evidence`; on every gated lane, quote the matching trait as `gate_evidence`. Registry lanes fire ONLY on a trait match (medtech → `openfda_510k`; US-gov contractors → `usaspending`; public-co filing events → `sec_edgar`); for most ICPs the plan is `web_search` + `hiring_signal` plus whatever trait lanes gate in. Instantiate each lane's `queries`/`params` from the segment's own keywords, pain terms, and `icp.geo` — never from another company's plan.
3. **`tier_a_pairing`** — name the 2 lanes expected to overlap on the same companies (this arms the 2+-distinct-source Tier-A gate).
Events the user names anywhere (e.g. when picking Q2.2's expo option) → `discovery.events`; CSVs they promise to drop in the campaign folder → `discovery.user_lists`. The plan is never asked as a question: it renders in the SUMMARY & COMMIT screen as one `segment → lanes` line per segment and is locked by the same final confirm. Canonical yaml shape: bottom of `source_selection.md` — do not fork it. The hard rule carries: never plan a lane whose gate does not match the profile; if no trait lane fits, `web_search` + `hiring_signal` always do.

## ROUND 3 — Personas, geography, size

**Q3.1 Title map** `[FREE → structured, pre-filled from `titles_library.md` for the chosen vertical]` → `personas.buyer`, `personas.champion`, `personas.exclude_tokens`
"Inside a target company: who signs (**buyer** titles), who fights for it day-to-day (**champion** titles), who is never the entry point (**exclude**, e.g. HR/recruiting)?"
Present the seed lists as a pre-filled draft; the user prunes/adds; read back before writing. A hiring req is a COMPANY signal; the contact is the engineering chain found by title, never whoever posted the job.

**Q3.2 Geography** `[MC + FREE, pre-filled]` → `icp.geo`
- Specific metro(s) — name them `(Recommended — tight metros compound via events + referrals)` → `mode: metros`
- Country-wide → `mode: country`
- Global English → `mode: global_english`
- Custom → `mode: custom`
Locations as full strings, e.g. `"San Diego, California, US"`.

**Q3.3 Size bands** `[MC]` → `icp.size_bands`
- 11-500 employees `(Recommended: founder-reachable, has budget)` → `'["11,20", "21,50", "51,100", "101,200", "201,500"]'`
- 1-50
- 50-1000
- Custom ranges
Stored in Apollo range format.

## ROUND 4 — Channels then voice (channels first: length bands depend on channel)

**Q4.1 Channel priority** `[MC]` → `channels.priority`  *(always asked — policy field)*
"Our run data: verified-email coverage caps around **~32%** of contacts while LinkedIn coverage was **~100%**, and DMs perform as well or better for technical personas."
- LinkedIn-DM-first, email second `(Recommended)` → `'["linkedin_dm", "email"]'`
- Email-first → `'["email", "linkedin_dm"]'`
- Email-only → `'["email"]'`
- DM-only → `'["linkedin_dm"]'`
Nuance to offer when the ICP is dev-tools/IC-heavy: champions may flip email-first via `personas.channel_sequence`.

**Q4.2 LinkedIn setup** `[MC]` → `channels.linkedin.*`  *(SKIP if Q4.1 = email-only)*
"Do you have LinkedIn Premium or Sales Navigator, and is the account established (6+ months, active)?"
- Sales Nav, established → `plan: sales_nav`
- Premium, established → `plan: premium`
- Free or new account → `plan: free`
- No LinkedIn → `plan: none`, `enabled: false`
Established default: 15-20 personalized connects/day, ~80/week, manual sends only. New/low-activity: cap ~20/day + 30-day warm-up suggestion. Free tier is told plainly: personalized-note invites cap at ~5/month, so the DM-first motion effectively requires Premium/Sales Nav.

**Q4.3 Email setup** `[MC]` → `channels.email.*`  *(SKIP if Q4.1 = DM-only)*
- Dedicated cold domain, warmed, SPF+DKIM+DMARC `(Recommended)` → `warmed: true`
- Primary domain (warning: never send cold from the domain your deals live on)
- Nothing yet (we'll build the queue; you send once warmed) → `warmed: false`
- No email channel → `enabled: false`
Caps: 20-25/day/inbox mature, 5-10/day during warm-up.

**Q4.4 Voice** `[grouped MC + FREE, pre-filled from their copy]` → `voice.*`  *(banned words part always asked — policy field)*
- **Register** `[MC]` → `voice.register`: Peer-to-peer technical, terse, zero marketing language `(Recommended for engineer/skeptical audiences)` → `peer_technical` · Warm consultative → `consultative` · Formal → `formal` · Describe your own → `custom`.
- **Banned words/styles** `[FREE, pre-filled]` → `voice.banned_words`, `voice.banned_styles`: "Recommended for skeptical technical buyers: **'AI', hype adjectives, em dashes, exclamation marks** — prune or add."
- **Length bands** `[MC]` → `voice.length_bands`: DM note ≤300 chars, DM ≤60 words, email `75-120w (Recommended) · <75w · 120-180w`.
- **Sign-off** `[FREE, pre-filled from sender identity]` → `voice.signoff`.

## ROUND 5 — Claims policy (THE CAUTION DIAL), CTA, budgets, constitution

**Q5.1 THE CAUTION DIAL** `[MC]` → `claims.caution_dial`  *(always asked — policy field)*
"One knob controls how messages treat inferred vs verified info, 0-3:
- **0 — Locked.** Every claim traceable to a verified source; no inference; safest, most generic personalization.
- **1 — Careful. (Recommended)** Verified claims plus clearly-safe inferences (they're hiring test engineers → test work exists).
- **2 — Confident.** Confident inferences allowed with hedged phrasing ('looks like', 'if I'm reading this right').
- **3 — Adventurous.** Bold personalization from inference, sharper hooks; you accept an occasional miss.
The dial also sets fact-check strictness: at **0-1** a hard fact-check gate **blocks** unverified claims before export; at **2-3** the fact-check still runs but **flags instead of blocks**."
Write only the integer. `claims.factcheck_mode` is DERIVED by code (0-1 → block, 2-3 → flag) — never asked, never set.

**Q5.2 CTA policy** `[MC + FREE, pre-filled]` → `cta.default`, `cta.segmented`
Default ask:
- Low-friction artifact offer — "want the report?" `(Recommended: highest reply for cold technical audiences)` → `type: artifact_offer`
- 15-min call → `type: call`
- Reply-to-gauge-interest → `type: reply_gauge`
- Custom → `type: custom`
Then: "Any segment where the CTA must differ?" Worked example: ITAR-likely aero/defense gets a US-person, on-prem-aware CTA; medtech gets the standard one. → `cta.segmented '[{segment: aerospace_defense, condition: itar_likely, cta: "..."}]'`.

**Q5.3 Budget caps** `[grouped MC + FREE]` → `budget.*`  *(always asked — policy field)*
- **Apollo credits/run:** Free-endpoints-only · ≤100 `(Recommended)` · ≤500 · Custom → `budget.apollo_credits_per_run`
- **Hunter:** Free ~25 finds/mo `(Recommended — it will NOT cover a big list; our run resolved 6/44)` · Paid (type cap) → `budget.hunter_finds_per_month`
- **Sends/day:** ≤25 email + ≤20 DM `(Recommended)` · Custom → `budget.sends_per_day '{email: 25, dm: 18}'`
- **Max contacts/run:** 200 `(Recommended)` · Custom → `budget.max_contacts_per_run`

**Q5.4 Safety-rail constitution** `[CONFIRM]` → `constitution`
One confirm screen: `verified_emails_only · fact_check_before_export (mode per dial) · linkedin_manual_only + public-search-only · no_pattern_guessing · suppression honored`. "These are on by default and enforced every run."
`verified_emails_only` and `linkedin_manual_only` are constants — SHOWN, never offered as toggles (deliverability + ToS, not preferences). The only writable pieces here are `constitution.suppression` and `constitution.opt_out_line` (required plain-text opt-out on cold email).

---

**SUMMARY & COMMIT** — render the whole profile as a human summary — adjacent segments shown flagged under ICP, and the discovery plan as one `segment → lanes` line per segment with its `tier_a_pairing` — one final `[CONFIRM]`, then `galois profile confirm`. Downstream stages read the profile; they never re-interview: `/galois-targets` runs `discovery.plan` verbatim.
