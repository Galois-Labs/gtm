# Dial 3 — Adventurous · drafting instruction

**Three invariants at EVERY dial** (hard-coded in every fragment header AND enforced by the gate):
1. Never fabricate proper nouns — products, programs, customers, certifications, quantities, quotes.
2. Certification/compliance claims are verified-only, always — one wrong cert claim is instantly disqualifying with skeptical technical buyers.
3. An INFERRED reporting edge is routing metadata and NEVER appears in message copy, at any dial; only a stated, URL-evidenced edge may be referenced, softly, with a verified claims line.

Personalize boldly from inference. Direct assertion of likely-true inferences is allowed — no
mandatory hedging. An opinionated angle is encouraged; an occasional miss is accepted.

- The hard bans still apply in full: no invented products, programs, customers, certifications,
  quantities, or quotes — boldness is about angle and assertion, never fabrication.
- The claims_manifest is still required and still complete: every claim carries class
  (`verified` / `inferred_safe` / `inferred_hedged`), `fact_id` or null, and a verdict slot.
- Fact-check is a spot pass on fabrication classes only (proper nouns + numbers +
  reporting-structure claims); rows export annotated "dial-3: N unverified inferences" — the
  sender sees the accepted risk. FALSE is always corrected, and an invariant-class UNVERIFIABLE
  (fabricated proper noun, cert, or inferred reporting edge in copy) is corrected or cut at this
  dial too, never shipped with a flag.
