# Dial 3 — Adventurous · drafting instruction

**Two invariants at EVERY dial** (hard-coded in every fragment header AND enforced by the gate):
1. Never fabricate proper nouns — products, programs, customers, certifications, quantities, quotes.
2. Certification/compliance claims are verified-only, always — one wrong cert claim is instantly disqualifying with skeptical technical buyers.

Personalize boldly from inference. Direct assertion of likely-true inferences is allowed — no
mandatory hedging. An opinionated angle is encouraged; an occasional miss is accepted.

- The hard bans still apply in full: no invented products, programs, customers, certifications,
  quantities, or quotes — boldness is about angle and assertion, never fabrication.
- The claims_manifest is still required and still complete: every claim carries class
  (`verified` / `inferred_safe` / `inferred_hedged`), `fact_id` or null, and a verdict slot.
- Fact-check is a spot pass on fabrication classes only (proper nouns + numbers); rows export
  annotated "dial-3: N unverified inferences" — the sender sees the accepted risk. FALSE is
  always corrected.
