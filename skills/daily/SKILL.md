---
name: daily
description: The 5-minute morning GTM ritual — read the workbook's Budget/Log, Queue, and Companies tabs, inline a short digest (new triggers, queue state, invites vs cap, acceptance %, budget), append the wave-log line, then offer exactly three actions.
when_to_use: Start of a prospecting day, a morning check-in, or when the user asks "what's on deck / what should I do today" for outreach. Keep it to five minutes.
argument-hint: ""
---

> If a structured question tool is available (`AskUserQuestion` on Claude Code — one question per call; `ask_user_question` on Codex — at most 3 per turn, 2–3 options plus free text, pause the auto-resolve countdown), use it. Otherwise restate each question **verbatim**, then lettered options with the recommended one marked, and accept a letter, "recommended", or the user's own answer. Never present options without restating the question (the spec-kit #1147 lesson). On Claude Desktop, at most 2 questions per message.


# Daily — the morning ritual (G2)

A five-minute retention loop, not a work session. Read the workbook, open with the digest, then offer exactly three actions and stop. Do not run the research pipeline from here (that is `/galois-targets`), do not re-interview, do not chase rabbit holes.

The workbook is the state: **`~/GaloisGTM/<campaign>/gtm.xlsx`**. Locate it — if exactly one folder exists under `~/GaloisGTM/` use that campaign; else ask which campaign (structured question per the preamble). Read it in the main session with inline `python3` + openpyxl (CSV fallback = `<campaign>/sheets/*.csv`, same headers); on Desktop, the native xlsx tooling on the uploaded file. No `galois` CLI, no subagents — one deterministic read.

## 1. Open with the digest

Read three tabs and compute a few numbers — do not dump raw rows:

- **Budget** — the spend block (`monthly_cap_usd`, `spent_usd`, `apollo_credits`, `hunter_finds`) → budget remaining; and the wave log (row 4+) → yesterday's last wave line.
- **Queue** — today's and this week's rows by `date` and `STATUS`: how many are unsent (blank), `sent`, `accepted`, `replied`; **invites this week** = `li_note` rows marked `sent`/`accepted`/`replied` in the last 7 days, against the **80** ceiling; **acceptance %** = accepted+replied over sent-or-further among `li_note` rows.
- **Companies** — **new triggers since yesterday** = rows whose dated `signals` landed since the last wave's timestamp.

```
python3 - <<'EOF'
import openpyxl, os
p = os.path.expanduser("~/GaloisGTM/<campaign>/gtm.xlsx")
wb = openpyxl.load_workbook(p, data_only=True)
def rows(name):
    ws = wb[name]; hdr = [c.value for c in ws[1]]
    return [dict(zip(hdr, [c.value for c in r])) for r in ws.iter_rows(min_row=2)]
# summarize Budget spend block + wave log, Queue STATUS this week, Companies new signals
EOF
```

Summarize it back in **two or three lines** — new triggers, queue state + invites/cap + acceptance %, budget remaining. Then **append one wave-log line** to the Budget tab's wave log (the gstack-style manifest): `ts | stage=daily | surface | rows_added=0 | rows_changed=<statuses touched> | credits_spent=0 | note="morning digest"`. Write it with the standard discipline (copy to `gtm.prev.xlsx`, save to `gtm.tmp.xlsx`, atomic `os.replace`), and say the close-the-file warning before that write.

## 2. Offer exactly three actions

Ask one question (structured tool or the verbatim-restatement fallback per the preamble). Exactly these three, no more:

- **A) Open today's queue** → invoke `/galois-queue` (copy `ready` Messages rows into the Queue tab, render the HTML cockpit from it, open it).
- **B) Log yesterday's statuses** → if the user has a cockpit **[Export status]** blob, paste it and write the Queue tab's `STATUS` / `STATUS_NOTE` cells from it (match on `date`, `person`, `company`, `channel`); otherwise they mark `sent`/`accepted`/`replied`/`skip`/`bounced` straight in the Queue tab. This is what makes tomorrow's queue and the acceptance meter reflect reality — nudge it if the STATUS column looks stale.
- **C) Delta discovery pull** → invoke `/galois-targets --since 1d` (only triggers new since yesterday, appended to the Companies tab; feeds tomorrow's queue).

Do the one action the user picks, then stop. One action per morning is the whole ritual.

## 3. If acceptance is low or triggers have drifted

Acceptance under 40% → warn; under 25% → suggest pausing invites (low acceptance shrinks LinkedIn's allowed volume). Acceptance and reply rates come straight from the Queue `STATUS` marks, so the usual fix is **B above (log statuses)**, not a profile edit. If the shape of what's landing has genuinely drifted from the ICP, offer `/galois-setup --tune <field>` (a check-in of at most three questions, drifted fields only — never a re-interview). It is an offer, not a detour: if the user declines, move on.

---

**Wave outro (mandatory):** *"Wave done: digest read, 1 wave-log line appended — edit anything in gtm.xlsx (STATUS marks, TIER calls); the next wave reads whatever is there."*

*Optional: deterministic CLI. Power users may instead run the old `galois digest` engine from the repo's `src/` appendix; no skill requires it, and the workbook flow above is complete without it.*
