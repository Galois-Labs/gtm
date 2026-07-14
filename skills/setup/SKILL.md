---
name: setup
description: The Galois GTM setup interview. Creates the campaign folder and workbook (~/GaloisGTM/<campaign>/gtm.xlsx), then builds and confirms galois.profile.yaml beside it (ICP plus confirmed adjacent profiles, per-segment discovery plan, personas, voice, caution dial, channels, budgets). Use when the user says "set up galois" / "set up galois-gtm", "new campaign", "interview me", "onboard me", drops a URL or doc to profile from, or when any galois-gtm skill finds no confirmed galois.profile.yaml next to the workbook. Also handles single-field tunes and drift check-ins.
argument-hint: "[doc path | URL]"
---

> If a structured question tool is available (`AskUserQuestion` on Claude Code — one question per call; `ask_user_question` on Codex — at most 3 per turn, 2–3 options plus free text, pause the auto-resolve countdown), use it. Otherwise restate each question **verbatim**, then lettered options with the recommended one marked, and accept a letter, "recommended", or the user's own answer. Never present options without restating the question (the spec-kit #1147 lesson). On Claude Desktop, at most 2 questions per message. All paths write the same profile fields.

# Galois GTM setup — the interview

You own two files, and you edit both directly with your file tools — there is no CLI in this flow:
- `~/GaloisGTM/<campaign>/gtm.xlsx` — the workbook. It is the state, the work area, and the human interface for every downstream wave.
- `~/GaloisGTM/<campaign>/galois.profile.yaml` — the profile, a flat file beside the workbook. Every downstream wave reads it and never re-interviews.

The interview runs INLINE in the main session, always. Never delegate it, never run it from a subagent, never use `context: fork`: the structured question tool does not exist outside the main session. Subagents NEVER open, read, or write the workbook or the yaml — if Mode C fans research out, researchers return findings as text and the main session performs every file write itself.

**Reference translation** — the references below predate this flow; their questions, wording, and rules are unchanged and still binding, but read their commands as file edits:
- `galois profile set <dotted.path> <value> --source <s>` → set `<dotted.path>` in `galois.profile.yaml` by editing the file; record `<s>` in `provenance.<dotted.path>`.
- `galois profile confirm` → set `meta.confirmed_by_user: true` in the yaml.
- `galois profile status` / `gaps` / `show` → read the yaml and compute or render it yourself.
- `galois init --slug <slug>` → Step 1 below.

Hard rules:
- One question at a time. Every multiple-choice question has exactly one `(Recommended)` option, listed first.
- Infer first, confirm second. Never ask a blank question you could have answered from context the user already gave you.
- Write every accepted answer to `galois.profile.yaml` IMMEDIATELY, before asking the next question. On each accepted edit: set the field, update `meta.updated` (ISO), bump `meta.profile_version`, write `provenance.<field>` (`{source, state}`), and append `{ts, field, old, new, source}` to `profile_changelog.jsonl` beside the yaml. The profile is valid at every instant; quitting mid-interview leaves a resumable partial.
- Budget: 12 or fewer questions typical, 18 hard max. Checkpoint after every round with a 3-line "what I have so far".
- Escape hatch: if the user signals impatience ("just use defaults", "skip ahead", short irritated answers), collapse to the 8-question core: Q1.1, Q1.2, Q2.1, Q2.3, Q3.1, Q4.1, Q5.1, Q5.3. Fill everything else with the template defaults and list the defaulted paths in `meta.defaulted_fields`. Drift check-ins re-ask these first.
- Four policy fields are ALWAYS asked, even when inferable, because they are policy, not fact: the Caution Dial (Q5.1), channel priority (Q4.1), banned words (Q4.4), budget caps (Q5.3).
- The `discovery` block is part of the accept gate: after the ICP segments settle, propose 1-2 inferable adjacent customer profiles (Q2.1a — one question, per-profile yes/no/edit, skipped if nothing is honestly inferable) and DERIVE the per-segment discovery plan (`discovery.plan`: segment → observable signals → lanes whose gates match, per `skills/targets/references/source_selection.md`). The derivation is your work, never a question — it renders in the Step 4 summary and is locked by the same final confirm. `/galois-targets` runs that plan verbatim; registry lanes (FDA/USAspending/SEC) enter it only on a matching ICP trait.

References (read on demand, not all up front):
- `references/questions.md` — the full scripted question set Q1.1–Q5.4, with field paths, options, evidence lines, skip rules.
- `references/intake_modes.md` — the three intake flows (file / chat / URL) and the confidence-band confirm rules.
- `references/profile_template.yaml` — every field path, type, enum, and default. It is also the scaffold you copy to create the yaml.
- `references/titles_library.md` — vertical-to-titles seed lists for pre-filling Q3.1.
- `references/drift.md` — re-run behavior: status check, drift triggers, the 3-question check-in.
- `../targets/references/source_selection.md` — the lane decision table, gates, and the canonical `discovery:` yaml shape (bottom section). Read it at the discovery-plan derivation step; never fork the shape.

## Step 0 — find the campaign, read the profile

You do the status check yourself. Locate the campaign: use any folder or workbook path in the opening message; else look for `~/GaloisGTM/*/galois.profile.yaml` (one hit → use it; several → ask which; none → fresh setup, go to Step 1). Read the yaml, then:

- `meta.confirmed_by_user: true`, `meta.updated` under 30 days old, no drift flags: do NOT interview. Print a 2-line banner ("Running as <company> · dial <N> · <channel>-first · Tier-A = 2+ signals · v<profile_version>") and stop, unless the user explicitly asked to re-interview or tune.
- Exists but unconfirmed: resume. Diff the yaml against `references/profile_template.yaml` to list unfilled and inferred fields, risk-ranked (targeting before style — a wrong segment costs more than a wrong sign-off); continue the interview from the top gap.
- Drift flags present (triggers in `references/drift.md`; recompute Mode-A doc hashes with `shasum -a 256`): offer — never force — the check-in: at most 3 questions, drifted fields only, `meta.defaulted_fields` first.
- User names one field to change ("switch the dial to 2"): edit that field in the yaml, bump the version, confirm the new value back. No interview.
- Profile fine but `gtm.xlsx` missing: run the Step 1 scaffold only (it never touches existing tabs), then stop.

## Step 1 — campaign folder + scaffold (fresh setup only)

First question, before the mode chooser: **"Where should this campaign live? A) `~/GaloisGTM/<kebab-company-name>/` (Recommended) B) another folder — type the path."** Take the company name from the opening message, doc, or URL; if unknown, ask for it in the same question. Never place the folder inside a git repo — prospect PII must not land in commits.

Then create the scaffold:

1. **The workbook.** Check once per session: `python3 -c "import openpyxl" || pip install --user openpyxl` (the only dependency). If `gtm.xlsx` already exists, first say: *"If gtm.xlsx is open in Excel/Numbers, close it now — Excel's next save would silently overwrite what I'm about to write."* Then run (substitute the confirmed folder):

```bash
python3 - <<'EOF'
import os, shutil
from openpyxl import Workbook, load_workbook
from openpyxl.comments import Comment

folder = os.path.expanduser("~/GaloisGTM/<campaign>")   # <- the confirmed folder
path, tmp = os.path.join(folder, "gtm.xlsx"), os.path.join(folder, "gtm.tmp.xlsx")
os.makedirs(folder, exist_ok=True)

TABS = {
  "Companies":  ["name","domain","city","state","sources","signals","tier","TIER","rank","SELECT","notes","org_map"],
  "People":     ["company","name","title","linkedin","email","email_source","status","SELECT","notes","org_role","seniority","function","reports_to","org_evidence","approach_order"],
  "Messages":   ["person","company","channel","subject","body","claims","verdict","dial","STATUS","notes"],
  "Queue":      ["date","person","company","channel","linkedin","subject","body","dial","STATUS","STATUS_NOTE"],
  "Suppressed": ["key","scope","reason","added","by"],
  "Budget":     ["monthly_cap_usd","spent_usd","apollo_credits","hunter_finds"],
}
NOTE = ("UPPERCASE columns are yours; the agent never overwrites them. "
        "Edit anything; the next wave reads what's here.")

wb = load_workbook(path) if os.path.exists(path) else Workbook()
if wb.sheetnames == ["Sheet"]: wb.remove(wb["Sheet"])
for tab, headers in TABS.items():
    if tab in wb.sheetnames: continue                    # never touch an existing tab
    ws = wb.create_sheet(tab)
    ws.append(headers)
    if tab == "Budget":
        ws.append([0, 0, 0, 0])                          # spend block; seeded for real at commit
        for c, h in enumerate(["ts","stage","surface","rows_added","rows_changed","credits_spent","note"], 1):
            ws.cell(row=4, column=c, value=h)            # wave log starts at row 5
    ws["A1"].comment = Comment(NOTE, "galois-gtm")
    ws.freeze_panes = "A2"
wb.save(tmp)
if os.path.exists(path): shutil.copy2(path, os.path.join(folder, "gtm.prev.xlsx"))
os.replace(tmp, path)
print("workbook ready:", path)
EOF
```

   If openpyxl cannot be installed, fall back to CSV tabs — `<campaign>/sheets/Companies.csv` etc., same headers, same conventions, spend block in Budget.csv rows 1–2 and log from row 4 — and put the A1-note sentence in `sheets/NOTES.txt`.

2. **The profile.** Copy `references/profile_template.yaml` to `<folder>/galois.profile.yaml`, comments intact (they document every field). Set `meta.slug` (folder basename), `meta.created`, `meta.updated`, `meta.agent` now. From here on, every accepted answer is an edit to this file.

## Step 2 — mode chooser

Skip this question entirely if the opening message already contains a URL (route to Mode C) or a file path (route to Mode A).

Otherwise ask, as the next question:

"How should I learn about you? **A)** Point me at a doc about you/your product **B)** Interview me in chat **C)** Drop your website URL and I'll draft it, you correct it (Recommended — fastest)."

Record the choice at `meta.intake_mode` (`file|chat|url`).

## Step 3 — run the mode

Follow `references/intake_modes.md` for the chosen mode (with the reference translation above):
- **Mode A (file):** read the doc, map statements onto profile fields tagging filled / inferred / missing, print the one-line-per-section gap report, ask only the missing questions plus one batched confirm of all inferred values, plus the four policy fields. Record `meta.source_doc` `{path, sha256}` (hash via `shasum -a 256`).
- **Mode B (chat):** the full 5-round set from `references/questions.md`, one question at a time, round checkpoints, escape hatch live at all times.
- **Mode C (URL):** bounded research pass (about 6 fetch/search calls total), auto-draft the ENTIRE profile with per-field provenance and confidence, then confirm by confidence band. Voice is always explicitly confirmed regardless of confidence. If your platform supports parallel subagents, fan out at 8–12 (Claude Code) or ≤6 (Codex), chunks of 4–6. If not, process the same chunks sequentially, checkpointing to the workbook after each chunk. Researchers return text only; ≤2 searches per agent.

All modes ask questions with the wording, options, and field paths in `references/questions.md`, and all converge on the same yaml and the same accept gate. All modes also run Q2.1a (adjacent-profile proposals, right after the segment read-back) and the discovery-plan derivation from `references/questions.md` §"R2 derivation" — Mode B derives after Round 2; Modes A/C infer both at draft time and confirm at the gate.

## Step 4 — summary and commit

1. Read the yaml and render it as a human summary, grouped by section: identity/offer, ICP + tiers (adjacent segments flagged `inferred_adjacent`), the discovery plan (one `segment → lanes` line per segment, `tier_a_pairing` shown, `gate_evidence` quoted on any registry lane), personas, channels, voice, claims policy + dial, CTA, budgets, constitution. Show the constitution constants (`verified_emails_only`, `linkedin_manual_only`) as constants, never as toggles.
2. One final confirm: "Lock it in? You can tune any single field later without redoing this."
3. On yes: set `meta.confirmed_by_user: true`, bump the version, update `meta.updated`. On edits: apply each to the yaml, re-render, confirm again.
4. Seed the workbook from the answers. Say the Excel-open warning first (the file now exists), then in one atomic write (tmp + `gtm.prev.xlsx` + `os.replace`, as in Step 1):
   - **Budget spend block (row 2):** `monthly_cap_usd` = any monthly dollar cap the user named in Q5.3, else 0 (free tiers only); `spent_usd` = 0; `apollo_credits` = `budget.apollo_credits_per_run`; `hunter_finds` = `budget.hunter_finds_per_month`. Waves decrement the credit cells and grow `spent_usd`.
   - **Suppressed tab:** one row per entry in `constitution.suppression` — `key | domain-or-person | "setup exclusion (Q2.4)" | <today> | setup`.
   - **Budget wave log (row 5):** `<ts> | setup | <surface> | 0 | 0 | 0 | campaign created; profile v<N> confirmed`.
5. Close with the mandatory outro line, then the next step: *"Setup done: <campaign> scaffolded (6 tabs), Budget and Suppressed seeded, profile v<N> confirmed — edit anything in gtm.xlsx; the next wave reads whatever is there."* Next: `/galois-targets` pulls and tiers the first target list. Downstream skills stop at an unconfirmed profile.

## Per-surface file access

- **Claude Code / Codex:** everything above as written — inline `python3` + openpyxl for the workbook, your file tools for the yaml.
- **Claude Desktop / claude.ai:** the sandbox filesystem is ephemeral, so build both files in-sandbox with the native xlsx tooling and, at the end of the session, return `gtm.xlsx` and `galois.profile.yaml` as downloads with: "save both into `~/GaloisGTM/<campaign>/`, overwriting." On re-runs, start by asking for the current files (or read them from the Project's files). Mode C fetches use the chat's web-fetch.

---

Optional: deterministic CLI — power users can drive the legacy `galois` engine kept in `src/` (see the repo appendix); no part of this flow requires it.
