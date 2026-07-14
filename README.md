# Galois GTM

This is the internal prospecting tooling we built at Galois Labs to find our own customers, cleaned up and shared with our Founders Inc cohort. It runs inside your own agent session (Claude Code, Codex, or Claude Desktop) — nothing hosted, nothing phoning home, your keys, your machine.

How it works: a clarifying interview (point it at a doc, answer in chat, or drop your URL) becomes a versioned profile — ICP rubric, personas, voice rules, a claims policy with a Caution Dial, channels, budgets. Everything else happens in one spreadsheet — `~/GaloisGTM/<campaign>/gtm.xlsx` — where your agent saves companies, people, drafts, and the send queue, then filters, ranks, fact-checks, and improves them wave by wave off free public sources matched to your ICP and public LinkedIn search. The workbook is the state, the work area, and your interface: open it in Excel or Sheets any time, edit anything, and the next wave reads whatever is there. Drafts come out in your voice with every claim traced. You send every message yourself. The workbook compounds on your machine, so re-runs get cheaper.

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

## Install — 30 seconds

Open Claude Code (or Codex) and paste this. The agent does the rest.

> Install galois-gtm: run **`git clone --single-branch --depth 1 https://github.com/Galois-Labs/gtm.git ~/.claude/skills/galois-gtm && cd ~/.claude/skills/galois-gtm && ./setup`** then add a "galois-gtm" section to CLAUDE.md that says: prospecting, lead lists, and outreach route through the galois-gtm skills — /galois-setup (interview -> profile + workbook), /galois-targets (discover + tier), /galois-enrich (contacts + a keyless buying map + verified-email reveal), /galois-messages (draft + fact-check, caution dial 0-3), /galois-queue (build the DM queue), /galois-daily (morning digest) — each wave works one spreadsheet at `~/GaloisGTM/<campaign>/gtm.xlsx`, and the human sends every message.

`./setup` auto-detects your hosts and wires each one with symlinks: Claude Code and Codex get the skills, Claude Desktop gets a ZIP of each skill to upload under Settings → Customize → Skills. No hosted service, no MCP server, no background process — just the skills and the workbook. Idempotent; `--host claude|codex|desktop` to target one. The only runtime dependency is `python3` + `openpyxl` for `.xlsx` waves (CSV mode needs nothing).

## Quickstart

Each command is a wave. It reads the workbook, works, writes it back, and tells you what changed in one line. No CLI, no flags — just the skill.

```
/galois-setup      # the interview -> ~/GaloisGTM/<campaign>/gtm.xlsx + galois.profile.yaml beside it
/galois-targets    # zero-key discovery from free public sources matched to your ICP -> Companies tab
/galois-enrich     # contacts + a keyless buying map + verified-only email -> People tab
/galois-messages   # draft + fact-check (caution dial 0-3) -> Messages tab
/galois-queue      # build today's DM-first queue -> Queue tab
/galois-daily      # a 3-line morning digest, then the same waves
```

Between any two waves, open `gtm.xlsx` in Excel or Sheets and edit anything — UPPERCASE columns (`TIER`, `SELECT`, `STATUS`) are yours and the agent never overwrites them. The next wave reads whatever is there. When the queue is built you open each profile, click Connect or Message, paste the drafted note, and mark it sent on the sheet. Nothing is sent for you. See [Working in waves](#working-in-waves).

## Working in waves

Every command works one workbook: `~/GaloisGTM/<campaign>/gtm.xlsx`, with `galois.profile.yaml` beside it. A wave reads it, works, writes it back, and tells you what changed in one line — *"Wave done: +12 companies, 8 auto-tiered, 3 need your TIER call."* Edit anything in Excel or Sheets whenever you like; the next wave reads whatever is there. The workbook **is** the state and the interface — there is no separate database, no export step, nothing hosted. One plain folder in your home directory, visible in Finder, never inside a git repo.

The workbook has six tabs, and their column headers are the contract:

| Tab | What it holds |
|---|---|
| **Companies** | discovered targets, their signals/sources, and tier |
| **People** | the LinkedIn-reachable contacts, their buying-map role and approach order, and a verified email only when there is one |
| **Messages** | drafts with a `claims` lane, fact-check `verdict`, and gate `STATUS` |
| **Queue** | today's rendered DM-first slice — the `STATUS` marks are the ledger |
| **Suppressed** | do-not-contact keys the agent checks before adding any row |
| **Budget** | your spend caps up top, one appended log row per wave below |

**UPPERCASE columns are yours.** `TIER`, `SELECT`, `STATUS`, `STATUS_NOTE` — the agent reads them and never overwrites a value you put there. `TIER` overrides the machine's `tier`; `SELECT` is `Y` to force-include, `N` to skip, blank to let rank decide; `STATUS` on the Queue tab is where you mark `sent` / `accepted` / `replied` / `skip`. lowercase columns are the agent's proposals — a wave fills blanks and refreshes cells with new evidence, and treats whatever it finds in a cell at wave start as current truth, so a hand edit is simply what the next wave believes.

One shared file has one risk worth naming: if you leave `gtm.xlsx` open in Excel while a wave writes, Excel's next ⌘S can silently overwrite the wave's work. So a wave warns you to close the file before its first write, saves atomically (`tmp` then replace), and keeps `gtm.prev.xlsx` as a one-generation undo. Working in daily waves, one person at a time, this stays a non-issue.

If `openpyxl` is not installable on your machine, the flow falls back to CSV tabs — `<campaign>/sheets/Companies.csv` and friends, same headers, same conventions — and needs nothing installed at all.

## The Caution Dial

One control governs how far a draft may go past what is proven. Set once in the interview (default 1), override per run, stamped on every exported row.

| Dial | Drafts from | Gate |
|---|---|---|
| **0** Locked | Verified facts only | Blocked until every claim confirms |
| **1** Careful *(default)* | Verified + whitelisted safe inferences | Blocked until clean or softened |
| **2** Confident | Hedged confident inferences | Flags, does not block |
| **3** Adventurous | Bold inference, misses accepted | Spot-flags fabrication classes |

At every dial: never fabricate proper nouns; certification claims are verified-only; FALSE is always corrected.

## What it costs to run

The list — and the LinkedIn route to every person on it — costs nothing beyond the agent subscription you already have. The only paid column is the optional verified email.

| What you get | Cost | How |
|---|---|---|
| Company lists from free public sources matched to your ICP — directories, "top N" lists, job boards, and registries (e.g. FDA 510(k) only if you sell to medtech), plus your CSVs | $0, no keys | Every discovery lane is keyless and `charges_when: never` |
| People + LinkedIn URLs | $0 in API credits | Public LinkedIn search by your own agent (your existing Claude/Codex subscription does the work); no host search tool — a bare terminal or cron — falls back to a free `ddgs` lane, same trust rules, still $0; registry filings, when your ICP uses them, include contact names; your Connections.csv ingests free |
| A buying map per company — who signs, who champions, who to approach first | $0, no keys | Built from keyless public sources (company team pages, the "reports to" line in job-post JSON, registry contact names, press quotes) — never a paid call, and an inferred reporting link only ever routes who you contact first, never a word of the message |
| Verified emails | ~1 Apollo credit or 1 Hunter search, **only on a hit** | Both paid connectors are `charges_when: on_verified_email` — a miss debits nothing, enforced in the budget ledger, not by policy. The free lanes above always run to completion first: no paid call fires while any contact still has an unsearched public-search lane |
| Phones | Not offered | Out of scope by design |
| Anything to us | Never | Nothing hosted, nothing metered |

Add keys only when you want the email column — set them as environment variables (never written to the workbook); the skill reads them, or asks once:

```
export APOLLO_API_KEY=<key>     # ~1 credit per verified email, on a hit only
export HUNTER_API_KEY=<key>     # fallback finder; free tier ~25/mo — resolved 6 of 44 in our runs
```

No key ever produces a guessed email: the waterfall returns a verified address or nothing, and a no-hit routes DM-only. Before any paid call the skill reads your caps from the Budget tab, asks you first, and logs every credit there — a miss debits nothing. No key at all is the default motion: every contact routes DM-only, at $0.

## Finding people, and who to approach first

Contact discovery is the free lane, and it runs to completion before anything paid. Your agent's own search harvests LinkedIn-reachable buyers and champions by public search — it reads result link titles only and never loads a LinkedIn page (loading pages is what gets accounts banned). If a host has no search tool at all, a free `ddgs` fallback keeps that lane at $0. Then, still keyless, `/galois-enrich` builds a **buying map** for each company: who signs, who champions, and who to approach first — assembled from company team pages, the "reports to" sentence in public job-post JSON, registry contact names, and press quotes. Every reporting link is flagged `stated` (a public source says it) or `inferred` (deduced from titles); an inferred link only ever decides who you reach out to first, and the drafting step is barred from putting it in a message at any dial. Only then, on the head of the queue, does the optional verified-email reveal spend a credit.

## Evidence

Every number below comes from our own GTM runs and the audit that followed, not a benchmark deck.

- **53% vs 11%.** The multi-signal gate (require 2+ independent free sources before auto-promoting a company to Tier A) tested at 53% precision against 11% for single-signal tiering. It is enforced deterministically, and multi-signal hits skip adversarial verification entirely.
- **8-12 concurrency.** Subagent fan-out is capped at 8-12 on Claude Code (<=6 on Codex). We watched a 33-agent run stall above that band; the cap is the fix, shipped as the default.
- **823 contacts, 0 credits.** Our Bay Area run harvested 823 contacts across 403 companies through public LinkedIn search without spending a single API credit. Revealing emails for 262 of them is what cost credits.
- **~32% reveal ceiling.** Verified-email coverage caps around 32% of contacts while public LinkedIn coverage was near 100%. Reveal, not discovery, is the throttle. That is why the motion is DM-first and email is the verified-only second channel.
- **Post-accept DM replies benchmark 25-35%** for technical personas against a ~8% planned cold-email reply rate, which is why every contact ships a two-message pack.
- **The pipeline scale:** 1,955 companies discovered, 523 tiered A/B, 439 send-ready contacts across two metros.

These numbers come from running this same tooling on our own outbound — the campaign data lives in our workspace, the machinery is what you are cloning.

## The rules

The same skills, profile, and workbook conventions on every host — only fan-out throughput differs. There is no send code path anywhere: no LinkedIn automation, no email sender, no scheduler; you send every message yourself. Everything runs inside your live session; subagents research and return rows but never write the workbook, and nothing ever spawns a headless agent (that pattern gets accounts banned). Full rails and the reasoning behind them: [docs/SAFETY.md](docs/SAFETY.md).


MIT. See [LICENSE](LICENSE).
