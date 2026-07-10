# Message generation rules

You draft in the main session. The dial fragment (`dial_fragments/<N>_msg.md` from `galois dial
resolve`) is loaded alongside this file and governs what you may claim; this file governs how you
write. The profile governs voice — nothing about tone, bans, lengths, or CTAs is hardcoded here.

## Drafting model

Drafts happen in THIS session. On Claude Code the skill frontmatter pins `model: sonnet` (ADR-7,
declarative — it cannot drift); on Codex the drafter agent pins `gpt-5.4`. Researchers and
fact-checkers are separate subagents; drafting is never delegated to them, and drafting agents
never do their own web research. Everything you say about the prospect comes from the dossier
pack. If the pack is thin, stop and request a research pass — never compensate with invention.

## Every message genuinely different — no templates

- No reused sentences, openers, or hook structures across contacts. If you notice you have
  written the same sentence twice in a chunk, rewrite the second from a different angle.
- Two contacts at the same company get different hooks and different pain narratives, not the
  same message with a swapped noun.
- Buyer vs champion differ by PAIN NARRATIVE: buyers get the economic/speed framing (what the
  grind costs, what the review cycle delays); champions get the practitioner framing (the
  specific artifact they build by hand, the clause they answer to). Persona and role_class come
  from the dossier pack's person row.

## Voice — read from the profile, enforced by lint

- Register: `voice.register` (default peer_technical), sender first person, sign off with
  `voice.signoff`.
- `voice.banned_words` and `voice.banned_styles` are lint-enforced at ingest (word-boundary,
  case-insensitive; em/en dashes and exclamation marks rejected when banned). Apply
  `voice.say_instead` substitutions where the profile provides them. Do not spend a draft-fix
  cycle on a ban you could have respected the first time.
- Length bands from `voice.length_bands`:
  - `linkedin_note`: draft ≤280 chars; lint hard-caps at `dm_note_chars` (default 300).
  - `linkedin_dm`: ≤ `dm_words` (default 60).
  - `email`: body inside `email_words` (default 75–120); subject ≤9 words; the
    `constitution.opt_out_line` must appear verbatim in the body when set (lint rejects
    otherwise).
- Short is a feature. Cut the sentence you cannot tie to a fact or a whitelisted inference;
  do not hedge it into staying.

## Structure

- **Hook first, trigger first.** Lead with the freshest trigger from the pack, with its date or
  age (timeline hooks reply ~10% vs ~4% for problem hooks). The connect note and the DM open on
  DIFFERENT facts or angles — the note earns the accept, the DM earns the reply.
- **Wedge as lead.** One sentence of what the user's product does for THIS pain, phrased from
  `identity.one_liner` / `offer.artifact` in the profile's own terms.
- **Confirm-or-generalize.** Anchor on 1–3 confirmed facts. Downgrade every inferred specific to
  safe generic phrasing per the dial fragment. Never stack 4–6 specific standards a skeptical
  engineer can reject one of.
- **Self-claims.** Only items in `claims.assertable` may be asserted about the user's own
  product. Sovereignty/deployment lines (in-environment, own model endpoint, nothing leaves the
  network) appear ONLY if `claims.assertable` includes them — and once, sharply, never as
  repeated boilerplate.
- **CTA.** Take `cta.default.text`. If a `cta.segmented` entry matches the company (segment
  and/or condition, e.g. `itar_likely` against the pack's itar flag), it wins. Include
  `cta.meeting_link` only if set. Never invent an offer the profile does not make.
- Email variant only when the person row carries `email` + `email_source` — the waterfall is
  verified-or-None and `queue build` refuses anything else. The email is not the DM pasted into
  an inbox: same hook discipline, subject ≤9 words, body in band, opt-out line present.

## Manifest

Every message carries a `claims_manifest`: one entry per factual assertion ABOUT THE PROSPECT
(self-claims ride the `claims.assertable` whitelist and are not manifested). Entry shape
(CONTRACTS.md, lint-validated at ingest — all four keys required):

```json
{"claim": "cleared K261234 on Jun 12", "class": "verified", "fact_id": "f_8812", "verdict": "pending"}
```

- `class`: `verified` (requires `fact_id`) | `inferred_safe` | `inferred_hedged`. Lint rejects
  classes the current dial does not allow (dial 0: verified only; dial 1: + inferred_safe;
  dial 2–3: + inferred_hedged).
- `fact_id`: `f_<id>` of the facts row in the dossier pack that backs the claim; `null` for
  inferences.
- `verdict`: always `"pending"` at draft time; the fact-check wave fills it via decisions.

## Ingest row shape (`galois messages ingest drafts.jsonl`)

One JSON object per line:

```json
{"person_key": "maria-chen-quality", "channel": "linkedin_note", "subject": null, "body": "...", "dial_level": 1, "claims_manifest": [...]}
```

- `person_key`: the person's `linkedin_slug` (preferred), else email, else exact full name, else
  KB person id — from the dossier pack. Unknown keys fail lint.
- `channel`: `linkedin_note` | `linkedin_dm` | `email` (exact strings).
- `dial_level`: the resolved level for this run, on every row.
- `subject`: email only; `null` elsewhere.

## Worked example (dial 1, sample profile; your voice fields come from the profile)

Connect note (`linkedin_note`, 200 chars):

> Maria, saw the K261234 clearance on Jun 12. We generate the PCB-to-DHF traceability docs your
> team is likely building by hand for the next submission. Worth a look at your console boards?
> Alex, Galois

Post-accept DM (`linkedin_dm`, 51 words, opens on a different angle than the note):

> Thanks for connecting, Maria. Between clearances the grind is the traceability matrix from
> your Altium files to the design history file. Galois generates it, deterministic and
> reviewable, running inside your network on your own model endpoint. Could I run it live on one
> of your actual boards next week?
> Alex, Galois

The sovereignty sentence appears once, and only because this sample profile's
`claims.assertable` lists the in-environment deployment claim. DM manifest:

```json
[{"claim": "cleared K261234 on Jun 12", "class": "verified", "fact_id": "f_8812", "verdict": "pending"},
 {"claim": "traceability from Altium design files to the DHF is built by hand today", "class": "inferred_safe", "fact_id": null, "verdict": "pending"}]
```

The second entry is dial-1 whitelisted inference class (b): product category implies its
domain's standard pain — stated plainly, never dressed up as inside knowledge.
