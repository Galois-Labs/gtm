# Re-run and drift behavior

The interview is a living document, not a one-shot form. But re-runs are ZERO questions by default. Never re-interview from scratch unless the user explicitly asks for it.

## The status-check opener

Every skill invocation opens with a deterministic check — `galois profile status --json` (injected on Claude Code; the facade's `profile_status` elsewhere). If profile exists AND `confirmed_by_user` AND age < 30 days AND no drift triggers: load silently with a 2-line banner and zero questions. Banner shape:

```
Running as Galois Labs · dial 1 · DM-first · Tier-A = 2+ signals · v3
```

Single-field edits never require an interview: `galois profile set <path> <value> --source asked`, confirm the new value back, done. A full re-interview happens only on an explicit user request ("re-interview me", "redo the profile") — then run the full Mode B flow.

`queue build` hard-fails (exit 4) on `confirmed_by_user: false` — nobody gets output without the interview.

## Drift triggers

Any trigger → OFFER a check-in, never force one. The check-in is at most 3 questions, targeting only the drifted fields. Re-ask `meta.defaulted_fields` entries first.

1. **Staleness.** Profile older than 30 days, or `meta.defaulted_fields` non-empty, or any `provenance.<field>.state == stale`.
2. **Source drift.** Mode A: the source doc's sha256 no longer matches `meta.source_doc.sha256` (recompute with `shasum -a 256 <path>`). Mode C: a re-fetch of `meta.source_url` shows materially changed positioning.
3. **Outcome drift** (from the ledgers; MVP surfaces the raw numbers in `galois digest`, v1.1 computes it via regen report):
   - a segment's reply rate persistently ≤ half another's → offer a segment reweight
   - Tier-A precision below rubric expectation → offer tightening `tier_a.min_signals`
   - verified-email coverage far under plan → offer a channel rebalance
   - LinkedIn acceptance < 20% → surface it plainly: that is a targeting problem, and low acceptance SHRINKS LinkedIn's allowed volume
4. **Mid-session contradiction.** The user contradicts the profile in conversation ("actually we're focusing on Texas now") → propose the specific field edit inline: "Update `icp.geo.locations` to Texas metros? [yes/no]" → on yes, `galois profile set icp.geo.locations '["Austin, Texas, US", "Houston, Texas, US"]' --source asked`.

## The check-in shape

- Open with what drifted and why you noticed: "Your profile is 34 days old and 3 fields were defaulted during setup. Three quick questions to true it up? (skippable)"
- If declined: proceed with the existing profile, no nagging, no repeat offer this session.
- If accepted: ask ≤3 questions (drifted fields only, highest risk first, same wording as `questions.md`), write each with `galois profile set ... --source asked`.

## Changelog and traceability

- Every accepted change appends `{ts, field, old, new, source}` to `profile_changelog.jsonl` and bumps `profile_version` — the CLI does this on every `profile set`; never bypass it by editing YAML.
- Inspect history with `galois profile changelog --limit 20`.
- Every exported message row records the `profile_version` and `dial_level` that produced it — full traceability from any sent message back to the profile state that shaped it.
