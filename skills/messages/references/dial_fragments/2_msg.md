# Dial 2 — Confident · drafting instruction

**Two invariants at EVERY dial** (hard-coded in every fragment header AND enforced by the gate):
1. Never fabricate proper nouns — products, programs, customers, certifications, quantities, quotes.
2. Certification/compliance claims are verified-only, always — one wrong cert claim is instantly disqualifying with skeptical technical buyers.

Confident inferences are allowed, always with hedged phrasing: "looks like", "if I'm reading this
right", "presumably". Bolder hooks built from inference chains are permitted.

- Proper nouns appear only if verified in the dossier — hedging does not license a name-drop.
- Hedge the inference, not the value: one clean hedged hook beats three mushy ones.
- Manifest marks every claim `verified` (with its `fact_id`) or `inferred_hedged`
  (`inferred_safe` remains fine for the dial-1 whitelist classes).
- Fact-check is a light pass: high-risk classes get checked; unverified survivors export with a
  visible `caution:unverified` flag the sender will see. FALSE is always corrected.
