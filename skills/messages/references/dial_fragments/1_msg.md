# Dial 1 — Careful (default) · drafting instruction

**Three invariants at EVERY dial** (hard-coded in every fragment header AND enforced by the gate):
1. Never fabricate proper nouns — products, programs, customers, certifications, quantities, quotes.
2. Certification/compliance claims are verified-only, always — one wrong cert claim is instantly disqualifying with skeptical technical buyers.
3. An INFERRED reporting edge is routing metadata and NEVER appears in message copy, at any dial; only a stated, URL-evidenced edge may be referenced, softly, with a verified claims line.

Draft from verified facts plus ONLY these whitelisted inference classes:
(a) hiring for X ⇒ X work exists at the company;
(b) product category ⇒ its domain's standard pain;
(c) location/size facts and what they plainly imply.

- Whitelisted inferences are stated plainly ("you're hiring test engineers, so..."), never dressed
  up as inside knowledge.
- Everything outside the whitelist is excluded — no speculation about roadmap, customers, tools,
  or org structure (a STATED, URL-evidenced reporting fact is not speculation — it rides the
  verified class like any other fact; an inferred edge stays out of copy per invariant 3).
- Manifest marks every claim `verified` (with its `fact_id`) or `inferred_safe`.
- Export is blocked until the manifest is clean or softened — draft so the hard gate can pass.
