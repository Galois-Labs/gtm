---
name: messages
description: >-
  Draft two-message DM packs (connect note + post-accept DM, email variant only
  on a verified email) for People rows that lack messages, write them to the
  workbook's Messages tab with claims and dial columns, fact-check every claim
  at the caution dial, and set STATUS per the dial gate. Use when the user asks
  to draft outreach, write messages, fact-check drafts, or re-check rows they
  edited in the sheet.
argument-hint: "[dial 0-3]"
model: sonnet
---

> If a structured question tool is available (`AskUserQuestion` on Claude Code — one question per call; `ask_user_question` on Codex — at most 3 per turn, 2–3 options plus free text, pause the auto-resolve countdown), use it. Otherwise restate each question **verbatim**, then lettered options with the recommended one marked, and accept a letter, "recommended", or the user's own answer. Never present options without restating the question (the spec-kit #1147 lesson). On Claude Desktop, at most 2 questions per message. All paths write the same profile fields.

# Messages — draft, fact-check, gate

**The workbook is the state.** This wave reads and writes one file: `~/GaloisGTM/<campaign>/gtm.xlsx` (tabs: Companies, People, Messages, Queue, Suppressed, Budget), with `galois.profile.yaml` beside it in the same folder. More than one campaign folder: ask which. No workbook, no profile, or the profile was never confirmed → stop and run `/galois-setup` first. Never re-interview here.

**File access.** Claude Code / Codex: inline `python3` + openpyxl heredocs in the main session; once per session run `python3 -c "import openpyxl" || pip install --user openpyxl`. If install is impossible, use the CSV fallback (`<campaign>/sheets/Messages.csv` etc., same headers, same rules). Claude Desktop: ask for the current workbook (or read it from the Project's files), edit it with the native xlsx tooling, and end the wave by returning the updated file for the user to save back over `gtm.xlsx`.

**Subagents NEVER open, read, or write the workbook. Ever.** They receive their inputs in their prompt and RETURN text; you — the main session — perform every workbook read and write yourself, single-threaded, checkpointing after each chunk.

> "If your platform supports parallel subagents, fan out at 8–12 (Claude Code) or ≤6 (Codex), chunks of 4–6. If not, process the same chunks sequentially, checkpointing to the workbook after each chunk."

## 1. Read the workbook + resolve the dial

- Load the profile yaml and the Companies, People, Messages, and Suppressed tabs.
- Dial: `$1` if given — applies to this run only and never rewrites the profile — else the profile's `claims.caution_dial`. Stamp the level into the `dial` column of every row you write. Mode is derived, never asked: dial 0–1 = block, dial 2–3 = flag.
- Read `references/dial_fragments/<N>_msg.md` NOW — it is part of your drafting instructions. Keep `references/dial_fragments/<N>_check.md` aside for the fact-check wave. (The check fragments predate this flow: read their "KB facts table" as the Companies row's `signals`/`sources`/`notes`, and "export blocked at `queue build`" as the §7 dial gate you apply yourself.)
- Build the work list, two parts:
  1. **New:** People rows with no Messages rows yet. Skip anyone matching the Suppressed tab (domain / person / email; a `domain`-scope key with no dot is a company name — match it against the normalized company name, since registry rows often have no domain yet), skip `SELECT=N`, skip contacts whose company's effective tier (`TIER` if non-blank else `tier`) is C. Order freshest signal first; tier A>B breaks ties only — a fresh-trigger B outranks a stale A; never stall on an empty A. Default scope ~30 contacts (about twice the daily queue) unless the user asks for more.
  2. **Edited:** existing Messages rows where the `claims` cell no longer matches the `body` — the tell that a human rewrote the message. Handle per §8.

## 2. Dossier per contact — from the sheet's own columns

- The dossier IS the sheet: the contact's People row (`title`, `notes`, `email`, `email_source`) plus their Companies row (`signals` with dates, `sources`, tier, `notes`). **This is your ONLY source of prospect facts.**
- Allowed extra: at most **1 web search per contact**, only to confirm the freshest signal is still current or to pin its date — never to discover new material.
- Thin dossier (no dated signal, empty notes)? Do not draft inventively — park the contact and name them in the outro as "needs a research pass (`/galois-targets`)". A thin dossier is fixed by research, not an inventive draft.

## 3. Draft — in THIS session

Rules: `references/message_gen.md` + the dial fragment from §1. Draft here on the pinned model (frontmatter `model: sonnet`); never delegate drafting to subagents; drafting does no research beyond §2's single verify search. Work in chunks of 6 contacts.

Per contact, the two-message DM pack — **one Messages row per message**:

- `li_note` — connect note, draft ≤280 chars (hard cap `voice.length_bands.dm_note_chars`, default 300).
- `li_dm` — post-accept DM, ≤ `voice.length_bands.dm_words`; opens on a DIFFERENT fact or angle than the note.
- `email` — ONLY when the People row carries both `email` and `email_source`. No verified email → no email row, ever. Subject ≤9 words, body inside `voice.length_bands.email_words`, `constitution.opt_out_line` verbatim in the body when set.

Every message genuinely different — no reused sentences, openers, or hook structures; two contacts at one company get different hooks and different pain narratives.

`message_gen.md` governs voice, structure, and manifest discipline unchanged; its CLI-era plumbing maps to the workbook: a manifest entry → one line of the `claims` cell; `fact_id` → the claim line's source field; `drafts.jsonl` + `messages ingest` → you lint (§4) and write the rows yourself (§5); `person_key` → the People row's `name` + `company`; channels `linkedin_note`/`linkedin_dm` → `li_note`/`li_dm`.

**The `claims` cell** — one line per factual assertion about the prospect (self-claims ride the `claims.assertable` whitelist and are not listed), `class | claim text | source`:

```
verified | cleared K261234 on Jun 12 | https://api.fda.gov/... (or sheet:Companies!signals)
inferred_safe | traceability to the DHF is built by hand today | -
```

Class ∈ `verified` (source required) / `inferred_safe` / `inferred_hedged`; the dial limits classes (0: verified only; 1: +inferred_safe; 2–3: +inferred_hedged). Once written, claim text is frozen — verdicts and the §8 edit tell match by exact text. The `claims` cell is the human-legible Claims-checked lane.

## 4. Lint — instructions, not code; a violating row is never written

Check every draft against the profile BEFORE it touches the workbook. Violations: fix the draft, then write — never write a violating row and fix later.

- Length bands per §3; note hard cap; email subject ≤9 words; opt-out line verbatim when set.
- `voice.banned_words` / `voice.banned_styles`: word-boundary, case-insensitive; em/en dashes and exclamation marks out when banned; apply `voice.say_instead` substitutions.
- Only claim classes the dial allows; every `verified` line carries a source; every prospect assertion in the body has a claims line, and every claims line appears in the body.
- Email rows only with a verified email (`email_source` ∈ `apollo_match` / `hunter≥80` / `user`). Never write a guessed email anywhere.

## 5. Write the Messages tab

Before the first write of the wave, say: *"If gtm.xlsx is open in Excel/Numbers, close it now — Excel's next save would silently overwrite what I'm about to write."* Write discipline every time: copy the current file to `gtm.prev.xlsx`, save to `gtm.tmp.xlsx`, atomic `os.replace`.

Append one row per message: `person | company | channel | subject | body | claims | verdict (blank) | dial | STATUS=draft | notes`. `subject` only on email rows. Checkpoint after each chunk of 6. `STATUS` is the one UPPERCASE column this skill advances, and only on rows it drafted, checked, or re-checked this wave (lifecycle: draft → checked → ready | blocked); any other non-blank human cell is truth — never overwrite it.

## 6. Fact-check wave

Spawn fact-checker subagents (Sonnet-class). Each prompt carries: `references/factcheck.md`, `references/dial_fragments/<N>_check.md`, and its batch — per company an evidence block (the Companies row's `signals`, `sources`, `notes`, plus each claim's cited source), and per message `{row_ref, body, claims lines}` where `row_ref` = Messages sheet row number + person + channel.

- Group by company — one checker owns ALL of a company's messages; verify once per company, reuse verdicts across its contacts.
- Evidence-only batches: chunks of 40–50 messages. Claims needing web: chunks of 4–6 companies, ≤1 search per claim, ≤2 WebSearch per subagent; verdict UNVERIFIABLE rather than loop.
- Concurrency 8–12 (Claude Code) / ≤6 (Codex) / serial (Desktop), per the degradation sentence above.
- Subagents RETURN `VERDICT |` and `FACT |` lines as text. You write each row's `verdict` cell (one line per claims line, same order: `CONFIRMED — note` / `UNVERIFIABLE — reason` / `FALSE — correct fact`), append returned FACT lines to the company's `signals`/`notes`, set `STATUS=checked`, and checkpoint per chunk.

## 7. The dial gate + repair

**The dial gate is an instruction you obey, not code that saves you.** After verdicts land, set `STATUS` yourself, row by row. At dial 0–1: any Messages row whose `verdict` cell contains FALSE or UNVERIFIABLE gets `STATUS=blocked` and the reason in `notes` — a blocked row is never copied to Queue, and no wording talks past this. At dial 2–3: a row with UNVERIFIABLE claims goes `ready` with the flag visible in `verdict` (first line `caution: N unverified`); FALSE is still always corrected first — there is no dial where a false claim ships. Rows whose claims are all CONFIRMED (or softened per the dial fragment and re-checked clean) go `ready`. The human sends every message; nothing in this skill sends anything, ever.

Repair before re-gating:

- FALSE — always corrected, every dial: rewrite with the verified substitute from the verdict note, update the claims line, preserve length band and voice.
- UNVERIFIABLE at dial 0: rewrite/generalize. Dial 1: soften to the whitelisted inference it is, or cut. Dial 2–3: leave it; the flag stays visible.
- Re-lint repaired drafts (§4), re-check only the changed claims (one small subagent batch), re-gate. At dial 0–1 a row stays blocked until this converges.

## 8. Human edits — the rerun loop

The human edits `body`, `verdict`, or anything else in Excel/Sheets, then reruns `/galois-messages`. Whatever is in a cell at wave start is the truth. Per row:

- **The claims-vs-body mismatch is the tell.** A body asserting something with no claims line, or a claims line no longer present in the body → the human rewrote the message. Rebuild the `claims` lines from the edited body, re-lint (§4), reset `STATUS=draft`, and send the row through the fact-check wave again — an edited body re-enters upstream of the gate, so a human edit can never smuggle an unverified claim past the check.
- **An edited `verdict` with a matching body** → the human overturned the check on a row they actually read. Keep their verdict and re-run only the gate (§7) with it.

## Outro

Append one Budget wave-log row (`ts | messages | <surface> | rows_added | rows_changed | 0 | note`). Then report in one line, always ending with the mandatory sentence, e.g.: *"Wave done: 14 packs drafted, 22 rows checked, 18 ready, 3 blocked (dial 1), 2 need a research pass — edit anything in gtm.xlsx; the next wave reads whatever is there."*

---

Optional: deterministic CLI — power users can run the old engine (`galois messages ingest` / `queue build`) per the repo appendix; no step above requires it.
