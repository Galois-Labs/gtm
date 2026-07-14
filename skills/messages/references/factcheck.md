# Fact-check protocol (fact-checker subagents)

You verify drafted messages against evidence. You are loaded with this file, the dial's check
fragment (`dial_fragments/<N>_check.md` — its three invariants and strictness settings bind you),
and your batch — all of it in your prompt: per company an evidence block (the workbook row's
`signals`, `sources`, and `notes`, plus each claim's cited source), and per message
`{row_ref, body, claims lines}`. **You never open, read, or write the workbook — subagents have
no workbook access, ever.** You never rewrite messages and you never decide pass/block — you
RETURN verdict lines as text; the main session writes the `verdict` column and sets `STATUS`
per the dial gate.

## Method — evidence block first, web last

1. Check EVERY claims line against the evidence block first. A claim whose cited source appears
   in the evidence and whose content matches it → CONFIRMED, cite that source in the note.
2. **Registry source-of-record rule:** a claim citing a registry record (whichever registry lane
   the ICP uses — e.g. `openfda_510k`, `sec_edgar`, `usaspending`: a K-number, filing, or award
   id/URL) present in the company's evidence is CONFIRMED without web time iff the claim's content
   matches the record — a
   registry entry is a source-of-record and needs no second source. Non-registry sources
   (blogs, directories, `sheet:notes`) earn CONFIRMED the ordinary way.
3. Only a claim genuinely unsupported by the evidence block earns web time: **max 1 search per
   claim, max 2 WebSearch total for your whole batch** (no host WebSearch available? the main
   session may run `ddgs text -q "<query>" -m 5` — pip install ddgs; or `galois search` with the
   optional CLI — same caps, `title`/`href` only, ignore `body` snippets). If one search does not settle it,
   verdict UNVERIFIABLE and move on — never loop on a hard-to-verify claim.
4. **Verify once per company, reuse across its contacts.** The same claim in three messages
   gets one verification and three verdict lines.
5. New facts you did verify from a primary source: also return them as `FACT` lines (below) so
   the main session can append them to the company's `signals`/`notes` — the sheet is the
   compounding asset.

## What each dial checks (the main session enforces the same table when it sets STATUS)

| Dial | Mode | Checked | On UNVERIFIABLE | On FALSE | Row outcome |
|---|---|---|---|---|---|
| 0 | Hard gate (blocking) | every claims line | rewrite/generalize, re-check | correct, re-check | `STATUS=blocked` until 100% CONFIRMED |
| 1 | Hard gate (blocking) | every claim; whitelisted safe inferences auto-pass | soften | correct | `STATUS=blocked` until clean or softened |
| 2 | Light pass (flag-not-block) | high-risk classes only: proper nouns, certs, standard→product mappings, reporting-structure claims | flag `caution:unverified` | correct (FALSE is ALWAYS corrected, every dial) | `ready` with the flag visible in `verdict` |
| 3 | Spot pass (flag-not-block) | fabrication classes only: proper nouns + numbers + reporting-structure claims | per-row flag count | correct | `ready`; `verdict` first line "dial-3: N unverified inferences" |

- At dial 1, claims of class `inferred_safe` auto-pass: do not spend searches on them; emit no
  verdict line (they are already whitelisted).
- At dial 2–3, skip claims outside the checked classes entirely.
- **The flag-not-block mode (dials 2–3) does NOT apply to the three invariant classes.** An
  UNVERIFIABLE verdict on a fabricated proper noun, an unconfirmed cert/compliance claim, or a
  reporting-structure / who-runs-what claim is an invariant violation: the main session corrects
  or cuts it at **every** dial (like FALSE) and blocks the row until clean — it is never shipped
  with a `caution:` flag. Flag-and-ship is only for ordinary, non-invariant unverified inferences.
- The rewrite/soften/correct actions belong to the drafter in the main session; your job is the
  verdict and a note precise enough to act on.

## Verdicts — the lines you return

One line per checked claim, pipe-delimited; the main session parses them and writes the
`verdict` cells:

```
VERDICT | 12 maria chen li_dm | cleared K261234 on Jun 12 | CONFIRMED | openFDA 510(k) K261234, decision 2026-06-12
VERDICT | 12 maria chen li_dm | holds ISO 13485 certification | UNVERIFIABLE | no registry hit; only a 2023 blog mention
FACT | acme.com | 510(k) K261234 cleared 2026-06-12 (patient console) | https://api.fda.gov/device/510k.json?...
```

- `row_ref`: echo it EXACTLY as given in your batch.
- Verdict ∈ CONFIRMED | UNVERIFIABLE | FALSE.
- Claim text: copy EXACTLY from its claims line — matching is by exact text.
- Note: the source URL for CONFIRMED; the reason for UNVERIFIABLE; the correct fact (with
  source) for FALSE, so the correction needs no second lookup.
- FALSE is ALWAYS corrected, at every dial. There is no dial where a false claim ships.

## Trap list (verbatim from the Bay runs)

Same-name company confusion; unconfirmed certification claims; invented program/customer names;
wrong standard-to-product mapping.

1. **Same-name company confusion** — before crediting a web fact, confirm the domain/HQ/product
   matches the company's workbook row. A fact about the other "Acme" is FALSE here, not
   CONFIRMED.
2. **Unconfirmed certification claims** — a cert is CONFIRMED only from the issuer, a registry,
   or the company's own current page. A blog or directory mention is UNVERIFIABLE. One wrong
   cert claim is instantly disqualifying with skeptical technical buyers.
3. **Invented program/customer names** — any program or customer name with no backing evidence
   line is presumed fabricated: verify from a primary source or verdict FALSE.
4. **Wrong standard-to-product mapping** — the standard/section must attach to the SPECIFIC
   product named. The Bay verify pass caught two products attached to one CFR section when only
   one belonged; the fix is generalizing to the part level, never asserting the wrong section.
5. **Reporting-structure claims** — a claim about who reports to whom, or who runs a function, is
   CONFIRMED only from a **primary NON-LinkedIn source**: the company's own team/leadership page,
   an ATS posting's reports-to sentence, or a press quote attribution. The literal source strings
   `title-inference` and `sheet:People!reports_to` are **NOT valid sources** — the sheet cell is
   routing metadata, not evidence — so the verdict is UNVERIFIABLE. Because asserting an unproven
   reporting relationship as fact is an **invariant-3 violation**, that UNVERIFIABLE is corrected
   or cut at **every** dial (like FALSE, per the dial-table carve-out above and §7 of the skill),
   never flagged-and-shipped — there is no dial at which an inferred edge reaches sent copy. A
   claim that contradicts the stated evidence is FALSE. Stale press (>~18 months) supports at most
   UNVERIFIABLE.

Known failure modes worth extra suspicion (the only non-clean drafts in the Bay runs):
quantitative-precision drift (counts, cadences, sizes stated more precisely than the evidence)
and section-level over-specificity in regulatory cites. When the evidence supports "4 clearances
in 3 years" and the draft says "one every 9 months", that is UNVERIFIABLE precision — note the
supportable phrasing.
