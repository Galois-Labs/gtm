# Safety

Galois GTM is built so that the risky decisions follow explicit, readable rules — not an agent's improvisation — and the one irreversible action (sending) is always yours. This document is the plain-language version of those rails. They are carried by three things working together: the skill instructions the agent runs each wave, the workbook's conventions (what each tab and column means, which columns are yours), and you as the human who sends every message. The same rules ride on every launch surface — Claude Code, Codex, and Claude Desktop. None of them depends on the optional CLI in `src/`; the workbook flow enforces every rail on its own.

## The eight rails

**1. LinkedIn stays manual.** You send every DM, InMail, and connect by hand. The tool prepares deep links and clipboard-ready copy and tracks status in the workbook; it does not send. There is no send code path anywhere in this tooling, on any surface — this is a constant with no off switch, because there is nothing to switch.

**2. Verified emails only.** The email waterfall returns a verified address or nothing. No pattern guessing, ever. The enrich skill only writes a value into the `email` column when it carries a verified `email_source` — Apollo match, Hunter (confidence ≥ 80), or an address you supplied. No verified source means the cell stays blank and the contact routes to DM-only. A blank email cell is normal and safe; a guessed one never exists.

**3. Claims policy by dial.** The Caution Dial governs how a draft's claims are handled, and it is a gate the skill applies row by row. At dial 0–1, a Messages row whose `claims` include anything marked FALSE or UNVERIFIABLE gets `STATUS=blocked` and is never copied to the Queue tab. At dial 2–3 the same check runs but the row goes ready with the verdict shown as a visible flag rather than blocked. Two invariants hold at every dial — no fabricated proper nouns; certification and compliance claims are verified-only — and a claim marked FALSE is always corrected. The block is a rule stated in the skill, applied to the fact-check verdicts, so it is decided by the recorded evidence, not by a draft that argues it is fine.

**4. Message rules at draft time.** Before a draft is marked ready it must satisfy the message rules the skill applies: the length bands, the banned words and styles from your profile, the 300-character connect-note cap, and a complete `claims` manifest (every falsifiable statement listed so the fact-check gate can act on it). A draft that violates them is fixed or held, one step before the dial gate.

**5. Suppression is honored.** Competitors, current customers, excluded domains, and anyone on your do-not-contact list live on the Suppressed tab, and the skill checks it before adding any Companies, People, or Queue row. Suppressed keys never re-enter, and the check survives every re-run because the tab is part of the workbook.

**6. Budget rails.** Before any paid call the skill reads your caps from the Budget tab, and cache/dedupe run before any spend. It asks you before spending, and logs a wave row to the Budget tab afterward with the credits used. At the cap the paid lane stops and the run degrades to the keyless (DM-only) lane so it always completes rather than failing. A miss debits nothing.

**7. No headless spawning.** No code path spawns an agent or shells to any agent CLI. Fan-out uses the live session's own subagents, and they return rows, verdicts, and edits as text to the main session — **subagents never open, read, or write the workbook.** The main session is the single writer, so there is no agent-versus-agent race and no background process driving `claude -p`. That whole class of trap cannot happen here by construction.

**8. Honest degradation, never a crash.** When a key is missing or a plan blocks a call, the skill says so in plain language and falls through — no Apollo key means every contact routes DM-only, a 403 plan means the waterfall falls to Hunter then None — rather than erroring out. The verified-or-nothing rule means a degraded run simply carries fewer emails and more DM-only rows; it never emits a guess and never stops the pipeline.

## LinkedIn: the ToS boundary

The line is drawn so your account and your reputation are never put at risk by the tool. There is no LinkedIn-API code and no browser-automation code anywhere in this tooling, so the boundary is structural, not a preference you could toggle.

| Allowed | Forbidden, regardless of what you ask for |
|---|---|
| Public search-engine dorking restricted to `linkedin.com` | Automated DM, InMail, or connect in any form |
| Ingesting CSVs you legitimately export (Connections.csv, copied Sales Nav list views, extractions made with your own tooling) | Logged-in scraping of LinkedIn pages |
| Deep links out to a profile so you can open it | Reusing session cookies or auth tokens |
| Clipboard preparation of a drafted message | Calling the private Voyager API |
| Status tracking in the workbook of what you sent | Bundling, invoking, or configuring a browser-automation extension |

**Why no automation exists.** The human sends every message because the founder's account is the credibility asset and the downside of a ban is unbounded. LinkedIn now enforces quality over volume, and enforcement is real. Rather than give you a switch that could burn your account, the tooling simply has no send path. The rail cannot be toggled because there is nothing to toggle.

Volume guidance in the queue is advisory, and the enforcement is you: about 15–20 personalized invites per day, roughly 80 per week, with acceptance meters that warn below 40% and suggest a pause below 25% (low acceptance shrinks the volume LinkedIn will allow you). New or low-activity accounts should cap near 20 per day and warm up over 30 days.

## Deliverability defaults

Cold email is a reputation instrument. The defaults keep your complaint rate near zero.

- **Verified-only, no guessing.** Covered by rail 2.
- **Never send cold from the domain your deals live on.** The interview warns you and records your sending domain and its warmed status. Use a dedicated, warmed domain with SPF, DKIM, and DMARC configured.
- **Warm-up ramp.** Start at 5–10 sends per day and ramp to 20–25 per day per inbox over 2–4 weeks. Rows that need a warmed domain you do not have yet queue with a hold flag and warm-up guidance instead of going out cold.
- **Opt-out on every cold email.** A plain-text opt-out line rides on every cold email row.
- **Cadence.** 3–4 touches (day 0, +3, +7, +14), each follow-up adding a new concrete hook. The email export is compatible with Instantly and Smartlead import; the sending infrastructure is downstream and complementary.

## Session-native by design

Galois GTM runs entirely inside your own live agent session, with no hosted backend and no autonomous background process. It never spawns a headless agent and never shells to an agent CLI; the orchestration text lives in the skill files the session executes, and the durable state is one workbook on your own disk. Tools that drive logged-in sessions with headless automation are the pattern behind the recent round of platform crackdowns and account bans. Galois GTM structurally cannot behave that way, which is both a safety property and a trust one: the tool researches and drafts, and you remain the only thing that ever sends. The optional `galois` CLI in `src/` is a faster deterministic engine for power users over the same connectors and the same rules — it is not part of the wave flow, and no rail on this page depends on it.
