---
name: enrich
description: Harvest LinkedIn-reachable buyer and champion contacts for the head-of-queue companies in the workbook, then reveal verified emails ONLY for the head of queue via the Apollo to Hunter to None waterfall. Contact harvest runs through LinkedIn-dork subagents that trust link titles and ignore SERP prose and return rows as text; the main session writes the People tab. Reveal is budget-gated and ask-first, logs every credit to the Budget tab, and never guesses an email. Run after targets has populated and tiered Companies, once per daily slice, before drafting messages.
when_to_use: Companies is tiered and ranked (targets skill done) and you need contacts plus verified emails on the top companies before messages. Run per 10-20 company daily slice.
argument-hint: "[max-credits]"
allowed-tools: Bash, Agent, Read
---

> If a structured question tool is available (`AskUserQuestion` on Claude Code — one question per call; `ask_user_question` on Codex — at most 3 per turn, 2–3 options plus free text, pause the auto-resolve countdown), use it. Otherwise restate each question **verbatim**, then lettered options with the recommended one marked, and accept a letter, "recommended", or the user's own answer. Never present options without restating the question (the spec-kit #1147 lesson). On Claude Desktop, at most 2 questions per message.

# enrich — contacts, then verified-only reveal

Where this sits: `setup -> targets -> **enrich (contacts + reveal)** -> messages -> queue`.

**The workbook is the state.** One file per campaign: `~/GaloisGTM/<campaign>/gtm.xlsx`, with `galois.profile.yaml` beside it. Resolve the campaign folder from this session (setup created it); if more than one exists and it is ambiguous, ask which folder. You do two things this wave: (1) harvest LinkedIn-reachable buyer/champion contacts for the head-of-queue companies and write them to the **People** tab, and (2) reveal verified emails for the head of queue only, writing `email` + `email_source` and logging spend to the **Budget** tab. No contact you harvest gets messaged by anything but a human, later.

**THE SINGLE-WRITER RULE.** Only this main session ever opens, reads, or writes `gtm.xlsx`. Dork subagents receive their inputs in their prompt and **return rows as text**; you write every People row yourself, single-threaded, one chunk at a time. A subagent never opens the workbook and never runs any command that writes state.

**Column conventions.** UPPERCASE columns (`SELECT` on Companies and People) are human-owned — read them, never overwrite a non-blank one. lowercase columns are machine-proposed — fill blanks, refresh only with new evidence. Effective tier = `TIER` if non-blank else `tier`. On Companies, `SELECT`: blank = decide by `rank`; `Y` = force include; `N` = skip.

## 0. Surface, dependency, and the open-file warning

- **Claude Code / Codex:** read and write the workbook with inline `python3` + openpyxl (heredocs in this session). Check once: `python3 -c "import openpyxl" || pip install --user openpyxl` — the only dependency. If install is impossible, fall back to CSV tabs at `<campaign>/sheets/People.csv` etc. (same headers, same conventions). Keyed reveal calls run via `curl`.
- **Claude Desktop:** ask for the current `gtm.xlsx` (or read it from the Project files), edit with Desktop's native xlsx tooling, and return the updated file at the end. **Keyed calls (Apollo/Hunter) are not available on Desktop** — the sandbox blocks network egress, so every head-of-queue contact stays email-less and routes **DM-only**. That is already the zero-key motion; harvest still runs.

**Before your first write of the wave, say:** *"If gtm.xlsx is open in Excel/Numbers, close it now — Excel's next save would silently overwrite what I'm about to write."* Write discipline for every workbook write: copy the current file to `gtm.prev.xlsx` first (one-generation undo), write to `gtm.tmp.xlsx`, then atomic `os.replace` onto `gtm.xlsx`.

## 1. Pick the head-of-queue companies

Read the **Companies** tab and build the worklist:

- Drop rows where `SELECT` = `N`, and rows whose effective tier is `C`.
- Keep every row where `SELECT` = `Y`.
- For the rest (`SELECT` blank), take in `rank` order (lower `rank` = higher priority; a fresh-trigger Tier-B row can outrank a stale Tier-A one — that is the ranking doing its job, not a stall).
- Take the top **10–20**. If only 1–2 companies clear the bar, that is the multi-signal gate being strict — say so in one line and proceed with what is there.

Read the personas from the profile beside the workbook: `personas.buyer`, `personas.champion`, `personas.exclude_tokens` in `galois.profile.yaml` (plain YAML — read it directly). You inject these into every dork worker prompt.

## 2. Harvest contacts (dork subagents return text; you write People)

Load `references/dork_rules.md` and follow it exactly — the stall-fixed harvest contract: **restrict WebSearch to `linkedin.com`; trust the result LINK TITLES only; IGNORE the SERP prose summary because it confabulates; record a contact only when its name and company are both visible in a real `/in/` URL and title; normalize slugs; ≤2 searches per company; write `NONE` rather than loop.**

**Degradation (G7), verbatim:**

> If your platform supports parallel subagents, fan out at 8–12 (Claude Code) or ≤6 (Codex), chunks of 4–6. If not, process the same chunks sequentially, checkpointing to the workbook after each chunk.

Chunk the worklist into groups of **4–6 companies** and spawn one dork worker per chunk:

- **Claude Code:** spawn workers 8–12 concurrent. Paste into each worker prompt the **full body of `references/dork_rules.md`** plus the profile personas and each company's `name` + `domain`. The worker's only job is dork-and-return; it runs **no** state-writing command and never touches the workbook.
- **Codex:** native subagent threads, ≤6, depth 1, sequential waves, same pasted prompt.
- **Claude Desktop:** run the chunks serially yourself in-session.

Every worker: 2–3 rows per company (one buyer + one or two champions), **≤2 WebSearch calls per company**, `NONE` rather than a third search or a guessed person. Workers **return PersonRows as text** — one JSON object per line: `{"company_name", "full_name", "title?", "linkedin_slug?"}`, no email.

**After each chunk returns, you write People (checkpoint):**

1. **Suppression check.** Read the **Suppressed** tab. Skip any returned row whose company `domain` (scope `domain`) or person slug/name (scope `person`) is a suppressed key. Suppressed keys never re-enter.
2. **Dedupe** on `linkedin_slug` (fall back to `company`+`name`); re-ingest is idempotent, so a double run is a no-op.
3. **Map to People columns** (`company | name | title | linkedin | email | email_source | status | SELECT | notes`): `company` ← company_name, `name` ← full_name, `title` ← title, `linkedin` ← `https://www.linkedin.com/in/<slug>/`, `email` and `email_source` **left blank** (harvest never sets them), `status` ← `harvested`, leave `SELECT`/`notes` blank (human's).
4. **Write the workbook now** (tmp + `os.replace`, `gtm.prev.xlsx` copy). A stall then loses at most one chunk.

Example checkpoint write (append new People rows, skipping suppressed and existing slugs):

```
python3 - <<'EOF'
import openpyxl, os, shutil
wb_path = os.path.expanduser("~/GaloisGTM/<campaign>/gtm.xlsx")
shutil.copyfile(wb_path, wb_path.replace("gtm.xlsx","gtm.prev.xlsx"))
wb = openpyxl.load_workbook(wb_path)
ppl, sup = wb["People"], wb["Suppressed"]
supp = {(r[1].value, str(r[0].value)) for r in sup.iter_rows(min_row=2)}  # (scope, key)
have = {r[3].value for r in ppl.iter_rows(min_row=2) if r[3].value}       # existing linkedin urls
rows = [ ... ]   # the PersonRows the workers returned this chunk
for r in rows:
    url = f"https://www.linkedin.com/in/{r['linkedin_slug']}/" if r.get('linkedin_slug') else ""
    if ("domain", r.get("domain","")) in supp or ("person", r.get("linkedin_slug","")) in supp: continue
    if url and url in have: continue
    ppl.append([r["company_name"], r["full_name"], r.get("title",""), url, "", "", "harvested", "", ""])
tmp = wb_path.replace("gtm.xlsx","gtm.tmp.xlsx"); wb.save(tmp); os.replace(tmp, wb_path)
EOF
```

After the last chunk lands, append one **Budget** wave-log row for the harvest (`ts, stage=enrich-harvest, surface, rows_added, rows_changed=0, credits_spent=0, note`).

## 3. Reveal — head of queue only, budget-gated, ask first

Reveal is the throttle, not discovery: **verified-or-nothing**, only the head of queue, only rows with (full name OR linkedin) and no email yet, and only rows whose company is not `SELECT=N` and not suppressed. Pace 5–8 reveals/day — spend where the send happens.

**Keys — env or ask once.** Read `$APOLLO_API_KEY` (and `$HUNTER_API_KEY`). If a key is unset, ask the human once to paste it (used this session only, **never written to the workbook**) or to say "skip". No key → that lane cannot run; see Degradation.

**The credit ceiling N (was `--max-credits`, now the Budget cap).** Read the **Budget** spend block (row 2: `monthly_cap_usd | spent_usd | apollo_credits | hunter_finds`). N = the credits left before `spent_usd` would hit `monthly_cap_usd` (~1 credit per Apollo hit), or the `[max-credits]` argument if the human passed a smaller one. **Stop the wave at N credits.** Before making any paid call, show the human the spend block and the planned reveal count/cost and get a yes.

**The waterfall, per contact** (stop the wave the instant cumulative credits reach N):

1. **Apollo `/people/match`** (recipe 4): `curl -s -X POST https://api.apollo.io/api/v1/people/match -H "X-Api-Key: $APOLLO_API_KEY" -H "Content-Type: application/json" -d '{"first_name":"…","last_name":"…","domain":"<company domain>","reveal_personal_emails":true}'` (or `linkedin_url` instead of name+domain). Take `person.email` **only if present and its status is verified** → write `email` + `email_source=apollo_match`, `status=revealed`. Count ~1 credit. **Never call `mixed_people/search` — it is 403 on this plan.**
2. **Hunter fallback** (recipe 5), only if Apollo returned nothing: `curl -s "https://api.hunter.io/v2/email-finder?domain=<d>&first_name=<f>&last_name=<l>&api_key=$HUNTER_API_KEY"`. Accept **only `score ≥ 80`** → `email` + `email_source=hunter`, `status=revealed`. Below 80 = discard (verified-or-nothing). Log each Hunter find.
3. **None:** miss on both → leave `email` and `email_source` **blank**, set `status=dm_only`. That row routes DM-only downstream. This is expected: verified coverage caps well under 100% (~32% in our run data) and the DM channel carries the motion.

**`email` is only ever a verified reveal or a human-entered address.** Never write a guessed or pattern email. `email_source` ∈ {`apollo_match`, `hunter`, `user`} and must accompany any value; a value the human typed into `email` themselves is theirs — leave it, do not re-reveal it, and set `email_source=user` if it is blank.

**After the reveal wave, update the Budget tab in the same write:** bump the spend block (`spent_usd += cost`, `apollo_credits += credits used`, `hunter_finds += finds`) and append one wave-log row (`ts, stage=enrich-reveal, surface, rows_added=0, rows_changed=<#emails written>, credits_spent, note`). Then the tmp + `os.replace` write.

## Degradation — say it plainly, never crash, never guess

The waterfall is verified-or-None by construction, so no path emits a guessed email. Surface these honestly, in one line each, then move on — do not retry a keyed call in a loop:

- **No key** (env var unset and the human skipped the paste): the reveal lane cannot run; every head-of-queue contact stays email-less and routes **DM-only**. This is the zero-key motion, not a bug — the full DM-first pipeline still ships at $0. On **Claude Desktop** this is the only mode (keyed egress is blocked).
- **Apollo 403 / PlanBlocked:** the key exists but the plan refused the call. Fall through to Hunter, then None; affected contacts route DM-only. Do not fabricate an address.

State the outcome plainly, e.g. *"no Apollo key, so all 12 head-of-queue contacts route DM-only."*

## Fallback lane — the user already has a LinkedIn CSV

When dorking is unavailable or low-yield, or the user has a LinkedIn export (official `Connections.csv`, an extraction-tool export, or any file with a `linkedin_url` column), skip the harvest subagents: read the CSV yourself, map its columns to People (key on `linkedin_slug`), suppression-check, and append. Flag 1st-degree connections in `notes` as DM-able-now (they skip the connect-note step). We never bundle or invoke an extractor — the user produces the CSV with their own tooling. Then resume at step 3 (reveal).

## Wave outro (mandatory closing line)

End with one line, e.g.: *"Wave done: +14 contacts harvested, 6 emails revealed (4 Apollo, 2 Hunter, 2 credits + a Hunter find logged to Budget), 8 route DM-only — edit anything in gtm.xlsx; the next wave reads whatever is there."*

---

*Optional: deterministic CLI.* The `galois` CLI + sqlite KB remain in the repo as a power-user appendix (`galois reveal`, `galois kb upsert-people`) — a faster engine for heavy runs. No part of this skill requires it; the workbook flow above is complete without it.
