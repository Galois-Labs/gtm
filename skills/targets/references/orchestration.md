# orchestration.md — subagent recipes: discovery lanes + tiering the residue (workbook flow)

The residue is every **Companies** row left at single-source with a blank `tier`
**and** a blank `TIER` (one signal, or fit-only). The multi-signal Tier-A accounts
(2+ distinct source ids) are already proposed and skip verify — never re-tier them.
This file is the fan-out mechanics; the rubric each subagent applies is in
`tiering.md`.

**The one rule that governs everything here: subagents NEVER open, read, or write
the workbook.** The main session already holds the residue rows in memory (from the
discover + dedupe wave). It passes each chunk's rows into the subagent prompt as
text, the subagent **returns verdict lines as text**, and the **main session** parses
them and writes the `tier` column of the Companies tab itself, single-threaded. There
is exactly one writer, so the concurrent-clobber class simply has no writers to race.
Checkpoint = the main session saves the workbook after each chunk returns; a stall
loses at most the one open chunk.

## 0. Discovery-lane fan-out (§2 of the skill) — same contract, different cargo

The universal discovery lanes (`web_search`, `hiring_signal`) fan out under the
exact same rules as tiering: one subagent per segment-lane, the §3 degradation
contract and the concurrency ceiling apply unchanged, and **subagents return
candidate rows as text** — `name | domain | city | state | signals` — while the
main session stamps the lane id into `sources` and does every workbook write.
The per-subagent budget is the lane's own (**≤6 searches per segment per wave**,
listing pages only, link titles only); the ≤2-searches-per-company cap governs
per-company verification, which stays in the tiering lanes below. A discovery
subagent that finds nothing within budget returns an empty list — never loops.
Registry lanes do not fan out: they are `curl`/urllib recipes run in the main
session per `registry_recipes.md`.

## 1. Assemble the residue (in memory — no file, no query)

From the rows the main session already read for dedupe, keep those with a single
`sources` id and blank `tier`/`TIER`. Carry each row's `domain` (or `name`),
`segment`, `description`, and `signals` into the subagent prompt. Nothing is pulled
from a database — the Companies tab you loaded this wave is the whole population.

## 2. Chunk by whether the chunk touches the web

- **No-web tiering: chunks of 40–50.** Description + segment + cached signals decide
  most rows with zero searches. Near-deterministic classification — big, cheap chunks.
- **Web verification: chunks of 4–6.** Only for a genuinely ambiguous name or a
  suspected buried designer. One WebSearch per company, **≤2 per company hard cap,
  write the tier rather than loop** (chunk-of-4 was the operator's own stall fix).

Route a company to the web lane only if the no-web pass could not tier it
confidently. Most residue never leaves the no-web lane.

## 3. The degradation contract (verbatim)

> If your platform supports parallel subagents, fan out at 8–12 (Claude Code) or ≤6 (Codex), chunks of 4–6. If not, process the same chunks sequentially, checkpointing to the workbook after each chunk.

Per surface:

| Surface | Fan-out | Wave shape |
|---|---|---|
| **Claude Code** | 8–12 concurrent (`Agent` tool) | one pool; launch the next chunk as slots free; main session writes the workbook between chunks |
| **Codex** | ≤6, depth 1 | sequential waves; main session writes the workbook after each wave |
| **Claude Desktop** | serial | one chunk at a time, in-session; write the workbook after each chunk |

The ceiling is the profile's `budget.subagent_concurrency` (default 8, band 8–12);
`budget.websearch_calls_per_agent` (default 2) is the per-company search cap. Do not
exceed the band — 33 agents stalled above it in the audit.

## 4. Agent defs

- **`lane-scout`** (model: haiku) — the discovery-lane runner (§0). Gets one
  segment's plan entry for one lane (queries or templates, geo, the ≤6-search
  budget), tools limited to WebSearch/WebFetch. **Returns candidate rows as text
  in its final message** — it writes nothing to disk.
- **`tier-verifier`** (model: sonnet) — the residue tierer. Gets the interpolated
  rubric block from `tiering.md`, its chunk of residue rows (as text in the prompt),
  and the ≤2-search discipline. **Returns verdict lines as text in its final message**
  — it writes nothing to disk.
- **`company-researcher`** (model: haiku, optional) — a cheap scout for a residue row
  missing the one fact needed to tier it. Tools limited to WebSearch/WebFetch, ≤2
  searches. It **returns its findings as text** (`fact, source_url, verified?`) in its
  reply; the main session pastes those into the `tier-verifier`'s prompt for that row.
  There is no shared cache to hand off through, so if the split is not worth the extra
  hop, fold the research into the `tier-verifier`'s own web lane and skip this agent.

Give each subagent an explicit `maxTurns` cap and the WebSearch allowlist so a single
stubborn company cannot burn the wave.

## 5. Checkpoint after every chunk (the main session writes)

The instant a chunk's subagents return, the **main session**:

1. Parses the returned verdict lines (the JSONL shape in `tiering.md`). Resolve
   `company_key` by domain first, else normalized name, against the rows in memory.
2. Sets `tier` on each matching Companies row (never `TIER`, never a human column).
3. **Writes the workbook** (atomic `gtm.tmp.xlsx` → `os.replace`, per the SKILL's
   write discipline). This save IS the checkpoint.

- Idempotent: re-running the wave re-derives the residue as whatever rows are *still*
  blank, so an already-written tier is simply not in the next residue. A crash or
  context compaction loses at most the one open chunk.
- **Write NONE rather than loop.** A subagent that cannot classify a company within
  its search budget returns **no line** for it (the row stays blank) and moves on; it
  surfaces again next wave. Never spin on a hard company.

## 6. Wave loop (pseudocode, any surface)

```
residue = [ rows in memory where sources has 1 id and tier == "" and TIER == "" ]
for chunk in batches(residue, size = 40–50 no-web / 4–6 web):
    lines = fan_out(tier-verifier, chunk)     # ≤ concurrency ceiling; RETURNS text
    for v in parse(lines):                    # main session, single-threaded
        set row[v.company_key].tier = v.new_tier
    save_workbook()                           # atomic; this is the checkpoint
```

Serial degrades to the same loop with a fan-out of one — identical result, just
slower. Parallel, wave-limited, and serial runs all converge on the same tiered
Companies tab, because there is exactly one writer and the tab is the only state.
