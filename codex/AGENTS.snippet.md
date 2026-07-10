<!--
Galois GTM hard rails for Codex. APPEND this block to your repo's AGENTS.md (or
~/.codex/AGENTS.md for all repos). AGENTS.md is the cross-tool open standard, so
these same rails ride on every surface. Do not delete or soften the constants.
-->

## Galois GTM rails (do not remove)

Galois GTM is an agent-native prospecting utility: the deterministic Python core (the
`galois` MCP server / CLI) owns everything falsifiable; you own judgment (the
interview, source selection, tiering, dorking, drafting, fact-check verdicts). The
core cannot be talked past. Honor these on every run.

**Non-negotiable constants (no off switch):**
- Verified emails only. The reveal waterfall returns a verified address or None; None
  routes to LinkedIn-DM only. Never guess or pattern-build an email.
- The human sends every message. There is no send code path and you never automate
  one. No LinkedIn API, no logged-in scraping, no browser automation, no session-cookie
  reuse. Public SERP dorking and CSV ingestion only.
- No headless spawning. Delegate only to native Codex subagent threads in this live
  session. Never shell out to `codex exec`, `claude -p`, or any agent CLI.
- Two claim invariants at every caution dial: never fabricate a proper noun (product,
  program, customer, certification, quantity, quote); certification / compliance claims
  are verified-only, always. FALSE is always corrected, at every dial.
- House style for all outreach copy: peer-technical and terse. Never the literal word
  "AI". No em dashes, no exclamation marks, no hype adjectives. Honor the profile's
  banned-words list.

**Profile check first.** Before any pipeline work, run the `profile_status` tool (or
`galois profile status`). No workspace or profile -> run `/galois-setup` (the
interview) and stop. `confirmed_by_user` false -> the interview was never accepted; run
`/galois-setup` and stop. `queue_build` refuses an unconfirmed profile (exit 4).
Nobody gets output without a confirmed interview. Never re-interview a confirmed,
un-drifted profile; load it silently and proceed with zero questions.

**The interview uses the structured-question tool.** Prefer `ask_user_question` for the
`/galois-setup` interview: at most 3 questions per turn, 2-3 options plus free text,
and pause the auto-resolve inactivity countdown so an unanswered prompt is not
auto-resolved. If the tool is unavailable, restate each question verbatim, then lettered
options with the recommended one marked, and accept a letter, "recommended", or the
user's own answer. Every path writes the same profile fields (through `profile_set`).
Run the interview inline in this session; never delegate it to a subagent thread.

**Fan-out (Codex is depth 1, default 6 threads).** The degradation contract, verbatim:

> If your platform supports parallel subagents, fan out at 8–12 (Claude Code) or ≤6 (Codex), chunks of 4–6. If not, process the same chunks sequentially, checkpointing to the KB after each chunk.

On Codex: at most 6 worker threads, depth 1 (workers do not recurse). Tiering is a
sequential wave, not nested sub-sub-agents: run a research wave, then a verify wave.
Chunks of 4-6 for web-bound work, 40-50 for no-web work (KB-only tiering, fact-check
against cache). At most 2 web searches per worker; write NONE rather than loop. Custom
worker agents ship in `.codex/agents/`: `galois-researcher` (gpt-5.4-mini),
`galois-drafter` (gpt-5.4), `galois-fact-checker` (gpt-5.4).

**Plumbing lives behind the `galois` MCP server** (added via `codex mcp add galois --
galois mcp-serve`; wired by ./setup). Every tool returns a JSON envelope:
`{"ok": true, "data": ...}` or `{"ok": false, "error": {type, message, exit_code}}`.
Never write sqlite directly; state changes go through these tools. Tool names:

`doctor`, `profile_status`, `profile_set`, `profile_confirm`, `discover`, `kb_query`,
`kb_stats`, `kb_upsert_companies`, `kb_upsert_people`, `kb_facts_add`, `score_run`,
`gate_multisignal`, `decisions_apply`, `reveal`, `budget_status`, `dial_resolve`,
`messages_ingest`, `queue_build`, `queue_export`, `ledger_import_status`, `digest`.

Run `doctor` first on any new machine (keys, network, Apollo plan 403 probe, Hunter
quota, surface ceiling). The `galois` CLI on the Bash PATH is a documented fallback if
the MCP server is unavailable; it needs `network_access = true` under
`[sandbox_workspace_write]` for the paid connectors (see README, Codex row).

**Contact-harvest fallback.** LinkedIn dork yield through Codex web search is not yet
benchmarked (cached-by-default, uncertain link-title fidelity). If dorking comes up
thin, fall back to `linkedin_csv_ingest`: have the user export LinkedIn Connections.csv
or a copied Sales Nav list and ingest it (`discover linkedin_csv_ingest`). First-degree
rows are DM-able immediately and skip the connect-note step.
