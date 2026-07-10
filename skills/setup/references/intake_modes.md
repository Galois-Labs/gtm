# The three intake modes

All three modes converge on the identical artifact (`galois.profile.yaml`) and the same accept gate: full human summary → one final confirm → `galois profile confirm`. The interview always runs in the main session; only Mode C's web research may fan out.

**Mode chooser** is the very first interaction — skipped if the opening message already contains a URL (→ Mode C) or a file path (→ Mode A):

"How should I learn about you? **A)** Point me at a doc about you/your product **B)** Interview me in chat **C)** Drop your website URL and I'll draft it, you correct it (Recommended — fastest)."

**Confidence → interaction rule (all modes):**
- confidence ≥ 0.8 → one-tap `[CONFIRM]` batch ("look right? yes / edit")
- 0.4–0.8 → `[MC]` with the inference as the `(Recommended)` first option
- < 0.4 → real question from `questions.md`

Plus the four always-ask policy fields regardless of confidence: Caution Dial (Q5.1), channel priority (Q4.1), banned words (Q4.4), budget caps (Q5.3).

**Adjacent profiles + discovery plan (all modes, schema_version 2):** once the ICP segments are structured, propose 1-2 inferable adjacent customer profiles (Q2.1a — one question total, per-profile yes/no/edit, never batch-confirmed whatever the confidence, skipped if nothing is honestly inferable), then DERIVE the per-segment discovery plan (`discovery.plan`) against the lane gates in `skills/targets/references/source_selection.md` — agent work, zero questions; it renders in the accept-gate summary. Timing per mode below; full wording and write rules in `questions.md` (Q2.1a + §"R2 derivation").

---

## Mode A — doc file (load → map → ask only the gaps)

1. Read the doc (positioning doc, YC app, ICP notes, CLAUDE.md-style memory). Map every extractable statement onto the profile schema, tagging each field `filled` (explicit in the doc), `inferred` (derivable, needs confirm), or `missing`.
2. Write `filled` and `inferred` values immediately with provenance:
   `galois profile set <path> <value> --source file:<path-to-doc>`
3. Print a one-line-per-section gap report, exactly this shape:
   `ICP: filled (2 segments) · Personas: inferred — confirm · Voice: partial · Caution dial: missing · Budgets: missing`
4. Ask ONLY the `missing` questions (risk-ranked: targeting fields before style fields — a wrong segment costs more than a wrong sign-off), plus ONE batched confirm of all `inferred` values, plus the four always-ask policy fields, plus Q2.1a — asked even when the doc filled the segments completely, because a positioning doc states the core ICP, not its neighbors; propose adjacents from the doc's product mechanics and who-else-has-this-pain reasoning. Once segments (core + confirmed adjacent) are settled, derive `discovery.plan` per `questions.md` §"R2 derivation" — zero questions; it rides the accept-gate summary.
5. Record the source doc for drift detection:
   - `shasum -a 256 <path-to-doc>` for the hash
   - `galois profile set meta.source_doc '{path: "<path>", sha256: "<hash>"}' --source file:<path>`
   - `galois profile set meta.intake_mode file --source asked`
6. Typical cost: 4–9 questions (Q2.1a included when it fires). Accept gate as always.

## Mode B — chat Q&A (the full gstack-style interview)

1. No inputs. Run the full 5-round set from `questions.md`, one structured question at a time (one `AskUserQuestion` call per question on Claude Code; verbatim-restatement prose elsewhere per the preamble).
2. Write each answer back immediately (`--source asked`) — quitting mid-interview leaves a partial, resumable profile (`galois profile gaps` shows where to pick up).
3. Checkpoint after each round: a 3-line "what I have so far". Q2.1a (adjacent profiles) is asked right after the Q2.1 read-back; the discovery plan is derived right after the Round-2 checkpoint — by then segments, tier signals, and exclusions are known, which is everything the lane gates read — and the checkpoint shows one `segment → lanes` line per segment.
4. Escape hatch at any point collapses remaining rounds to the 8-question core (Q1.1, Q1.2, Q2.1, Q2.3, Q3.1, Q4.1, Q5.1, Q5.3); default the rest with `--source default` and list the defaulted paths in `meta.defaulted_fields`. The hatch drops Q2.1a (no adjacent proposals); the discovery plan is still derived from the core segments — it costs zero questions.
5. `galois profile set meta.intake_mode chat --source asked`. Typical cost ≤12 questions, hard max 18.

## Mode C — URL (infer first, confirm second — the recommended default)

1. **Bounded research pass, ~6 fetch/search calls total:** fetch the homepage + 2–4 key pages (product, about, pricing/docs, customers) + 1–2 name searches (company name ± "LinkedIn", ± "customers"). Do not exceed the budget; write what you found and move on. If the pass is delegated to a researcher subagent, the 1–2-WebSearch-per-agent hard cap applies.
2. **Auto-draft the ENTIRE profile:** identity/offer from site copy; likely ICP + personas from who the copy addresses, logos, case studies; geo/size from customer evidence; voice from their own copy; CTA from their existing funnel (demo? pilot? waitlist?). Also draft the Q2.1a adjacent-profile proposals — site evidence is this mode's edge: case-study logos outside the stated ICP, integrations, who the docs address — and a provisional `discovery.plan` for every drafted segment, gates checked, `gate_evidence` quoted on any registry lane.
3. Every auto-drafted field carries provenance and confidence. Write drafts as you go:
   `galois profile set <path> <value> --source url:<page-url>`
4. **Confirm/correct by confidence band** (rule above): ≥0.8 fields go in one batched `[CONFIRM]`; 0.4–0.8 become MCs with the inference as `(Recommended)`; <0.4 become real questions — plus the four always-ask policy fields.
5. **Site voice ≠ DM voice (the Jasper caveat).** A website's marketing voice is not the sender's DM voice. `voice.*` is ALWAYS explicitly confirmed with Q4.4, whatever its confidence. Adjacent-profile proposals get the same treatment: each is confirmed one by one with its rationale (Q2.1a), never batch-confirmed — a wrong inferred segment poisons every downstream wave. The discovery plan needs no question of its own; it renders in the accept-gate summary, one `segment → lanes` line per segment.
6. `galois profile set meta.intake_mode url --source asked` and `galois profile set meta.source_url "<url>" --source asked`. Typical cost: 5–10 questions. Accept gate as always.
