---
name: queue
description: Copy passed Messages rows into today's Queue tab (ranked, capped at daily-N, both DM messages + LinkedIn links), render the self-contained HTML cockpit from that tab, then run the STATUS paste-back loop. User-invoked and side-effecting — never auto-run.
disable-model-invocation: true
argument-hint: "[daily-n]"
---

> If a structured question tool is available (`AskUserQuestion` on Claude Code — one question per call; `ask_user_question` on Codex — at most 3 per turn, 2–3 options plus free text, pause the auto-resolve countdown), use it. Otherwise restate each question **verbatim**, then lettered options with the recommended one marked, and accept a letter, "recommended", or the user's own answer. Never present options without restating the question (the spec-kit #1147 lesson). On Claude Desktop, at most 2 questions per message.


# Queue — assemble today's Queue tab, then render the cockpit

The last stage, and pure plumbing: **no research, no drafting, no LLM calls, no subagents.** You read the `ready` rows the `/messages` wave already drafted, fact-checked, and gated; you copy the ones that clear selection into the **Queue tab**; you render the HTML cockpit *from that tab*; you run the STATUS paste-back loop. The pass/block decision is not yours — it already happened at `/messages` (a `blocked` Messages row never reaches you). You are single-threaded over the workbook; nothing here fans out.

The workbook is the state: **`~/GaloisGTM/<campaign>/gtm.xlsx`**. Locate it — if this skill got a path argument use it; else if exactly one folder exists under `~/GaloisGTM/` use that campaign; else ask which campaign (structured question per the preamble). On Claude Code / Codex read and write it with inline `python3` + openpyxl (`python3 -c "import openpyxl" || pip install --user openpyxl` once per session; CSV fallback = `<campaign>/sheets/*.csv`, same headers). On Desktop use the native xlsx tooling on the uploaded file and return the updated file at the end.

## Preconditions

- **Workbook exists.** No `gtm.xlsx` (and no `galois.profile.yaml` beside it)? The campaign was never set up — stop and send the user to `/galois-setup`.
- **`ready` Messages rows exist.** Read the Messages tab; keep rows with `STATUS == ready` (the passed / green state — `draft`, `checked`, and `blocked` are not eligible). If none are `ready`: the pipeline has not produced sendable drafts yet — point the user to `/galois-targets` → `/galois-enrich` → `/galois-messages`, then come back. Do not invent rows.

**Before your first write, say:** *"If gtm.xlsx is open in Excel/Numbers, close it now — Excel's next save would silently overwrite what I'm about to write."*

## 1. Read the inputs (one session, no subagents)

Load, in the main session only, the tabs you need:

- **Messages** — `STATUS == ready` rows: `person, company, channel, subject, body, claims, verdict, dial`.
- **People** — `SELECT`, `linkedin`, `email`, `email_source`, per person.
- **Companies** — `tier`/`TIER`, `rank`, `signals` (for trigger text + recency), per company.
- **Suppressed** — `key`, `scope` (`domain` / `person` / `email`).

```
python3 - <<'EOF'
import openpyxl, os
p = os.path.expanduser("~/GaloisGTM/<campaign>/gtm.xlsx")
wb = openpyxl.load_workbook(p, data_only=True)
def rows(name):
    ws = wb[name]; hdr = [c.value for c in ws[1]]
    return [dict(zip(hdr, [c.value for c in r])) for r in ws.iter_rows(min_row=2)]
msgs = [m for m in rows("Messages") if (m.get("STATUS") or "").strip() == "ready"]
people = {(r["company"], r["name"]): r for r in rows("People")}
comps  = {c["name"]: c for c in rows("Companies")}
supp   = rows("Suppressed")
# ... join + rank in Python below ...
EOF
```

## 2. Select + rank today's contacts

Work over the `ready` Messages rows joined to their People and Companies rows:

- **Suppression first.** Drop any row whose person, company `domain`, or `email` matches a Suppressed `key` at the matching `scope` (a `domain`-scope key with no dot is a company name — match it against the normalized company name). Suppressed keys never enter the Queue — no exceptions.
- **SELECT gate (People, human-owned).** `SELECT == N` → skip the person entirely. `SELECT == Y` → force-include (ahead of rank). Blank → the machine decides by rank.
- **Rank order = trigger-recency dominant, then effective tier A > B; tier C excluded.** A fresh-trigger Tier-B contact outranks a stale Tier-A one; forced (`SELECT=Y`) contacts float to the top. Effective tier = `TIER` if non-blank else `tier`. Trigger recency comes from the dated `signals` on the Companies row.
- **Daily-N cap.** Take the top **N contacts** (default 15; override = this skill's `[daily-n]` argument). N counts *contacts*, not rows — a person's two-message pack is one contact.
- **Channel plan per contact.** Copy every `ready` message the contact has: `li_note` + `li_dm` are the two-message DM pack (two rows); an `email` row rides along **only if** the People row carries a verified `email` **and** `email_source` — no verified email means DM-only, never a guessed address.
- **Dial-gate safety re-assert.** Even among `ready` rows, at dial 0–1 never copy a row whose `verdict` is not clean (any FALSE / UNVERIFIABLE claim). Finding one here means an upstream bug — leave it out and name it in your outro; do not talk past the gate.

## 3. Write the Queue tab

Append today's selection to the **Queue tab** (headers are the contract: `date | person | company | channel | linkedin | subject | body | dial | STATUS | STATUS_NOTE`), one row per message:

- `date` = today; `person`, `company`, `channel`, `subject`, `body`, `dial` copied through; `linkedin` = the People row's link (or `linkedin.com/in/<slug>`); `STATUS` and `STATUS_NOTE` **blank** (human-owned).
- **Never clobber a human STATUS.** If a matching `(date=today, person, company, channel)` row already exists from an earlier run today, preserve its `STATUS` / `STATUS_NOTE` and refresh only the lowercase cells.

Write with the standard discipline — one-generation undo, atomic replace:

```
python3 - <<'EOF'
import openpyxl, os, shutil
p = os.path.expanduser("~/GaloisGTM/<campaign>/gtm.xlsx")
shutil.copyfile(p, p.replace("gtm.xlsx", "gtm.prev.xlsx"))
wb = openpyxl.load_workbook(p)
q = wb["Queue"]
# append/refresh today's rows, preserving any non-blank STATUS/STATUS_NOTE
tmp = p.replace("gtm.xlsx", "gtm.tmp.xlsx")
wb.save(tmp); os.replace(tmp, p)
EOF
```

## 4. Render the cockpit — from the Queue tab

Then write the self-contained HTML cockpit **directly from today's Queue rows** to `~/GaloisGTM/<campaign>/queue-YYYY-MM-DD.html` (Write tool, or Desktop file output). It is a *rendering of the Queue tab* — it invents nothing. It is beloved; keep it. Hard requirements:

- **Zero external requests.** All CSS/JS inline; no CDN, no fonts, no remote images. Regenerated every run.
- **Header banner:** weekday · N remaining today · invites this week vs the 80 ceiling · acceptance % · dial level.
- **One card per contact** (group the contact's Queue rows): person — title, company (Tier X); the trigger with age in days; the dial level plus any `caution:` flags read from `verdict`.
- **Both messages of the pack:** the `li_note` connect note with a **live character badge** (`NNN/300`, warns over 300) and a **[Copy connect note]** button; the post-accept `li_dm` with **[Copy post-accept DM]**. `email` rows render the body with the **opt-out line appended** and a copy button.
- **[Open profile ↗]** deep-links `linkedin.com/in/<slug>` — the human clicks Connect/Message and pastes. Two clicks, zero automation; **no link ever prefills a DM.**
- **Status radios per contact:** `( ) sent  ( ) accepted  ( ) replied  ( ) skip: reason ▾`, persisted to `localStorage`, each group carrying a stable `data-key="person|company|channel"`.
- **Pacing rails are advisory UI** (the human is the enforcement): the weekly invite counter vs 80, a "done for today" banner at the daily cap, an acceptance meter that warns under 40% and suggests a pause under 25%.
- **[Export status]** button: serialize the radios to a JSON blob — a list of `{key, status, note}` — for paste-back (step 5).
- **[Copy row for Sheets]** per card + **[Copy all rows for Sheets]** at the bottom: build TSV lines matching the Queue tab headers exactly (`date | person | company | channel | linkedin | subject | body | dial | STATUS | STATUS_NOTE`), with `STATUS`/`STATUS_NOTE` read live from that card's radios/note field at click time; Excel-quote any field containing a tab, newline, or double quote (wrap in `"`, double inner quotes) so a paste lands one field per cell, multi-line bodies included.

The beloved layout, for reference:

```
┌ Today's DM queue — Wed · 15 remaining · invites this week: 34/80 · acceptance 44% · dial 1 ┐
│ Maria Chen — Dir. Quality, Acme Surgical (Tier A)                                         │
│   trigger: 510(k) K261234 cleared Jun 12 (21d)      flags: none                           │
│   [Open profile ↗]  [Copy connect note 278/300]  [Copy post-accept DM]                    │
│   ( ) sent  ( ) accepted  ( ) replied  ( ) skip: reason ▾                                  │
└────────────────────────────────────────────────────────────────────────────────────────────┘
```

Give the user the cockpit's absolute path when it lands.

## 5. The STATUS paste-back loop

The Queue tab's `STATUS` / `STATUS_NOTE` marks **are the ledger** (`STATUS` ∈ `sent` / `accepted` / `replied` / `skip` / `bounced`). Two equal paths write those same cells:

- **Edit the tab.** The human types `sent` / `accepted` / `replied` / `skip` / `bounced` straight into the Queue tab's `STATUS` column (reason in `STATUS_NOTE`). Editing the sheet *is* the command — the next wave and `/galois-daily` read whatever is there.
- **Paste TSV rows.** The cockpit's **[Copy row for Sheets]** buttons emit tab-separated lines with the live radio status already in the `STATUS`/`STATUS_NOTE` columns — paste them over the matching rows in the Queue tab in Excel/Sheets; same effect as typing the marks.
- **Paste the blob.** The human clicks **[Export status]** in the cockpit and pastes the JSON here. Match each `{key: "person|company|channel", status, note}` to today's Queue row on `(date=today, person, company, channel)` and write `STATUS` / `STATUS_NOTE` for it — atomic tmp+replace, and **never overwrite a non-blank human mark with a blank**.

Either path is what makes tomorrow's queue, the invite pacing, and the acceptance meter reflect what actually happened.

## The constitution (read this in the cockpit footer, every run)

**The human sends every message. There is no send code path anywhere in this tooling. Period.** The queue is deep-link + clipboard + a human click. Pacing numbers are advisory; the sender is the rail. Before anything goes out, the dial level and any flags are visible on every card — the human sees the accepted risk first. **Never paste queue output into an automation tool:** that burns the account the whole motion depends on.

---

**Wave outro (mandatory):** *"Wave done: 15 contacts queued (28 message rows) — cockpit at ~/GaloisGTM/<campaign>/queue-2026-07-05.html; mark STATUS in the Queue tab or the cockpit, and the next wave reads whatever is there."*

*Optional: deterministic CLI. Power users may instead run the old `galois queue build`/`export` engine from the repo's `src/` appendix; no skill requires it, and the workbook flow above is complete without it.*
