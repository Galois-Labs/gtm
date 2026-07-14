# Dial 2 — Confident · fact-check strictness

**Three invariants at EVERY dial** (hard-coded in every fragment header AND enforced by the gate):
1. Never fabricate proper nouns — products, programs, customers, certifications, quantities, quotes.
2. Certification/compliance claims are verified-only, always — one wrong cert claim is instantly disqualifying with skeptical technical buyers.
3. Inferred-edge claims are never CONFIRMED-able; any reporting-structure claim sourced to `title-inference` or the sheet's `reports_to` cell is UNVERIFIABLE at best — only a stated, URL-evidenced primary source can confirm one.

- **Mode:** light pass (flag, not block).
- **Checked:** high-risk classes only — proper nouns, certs, standard→product mappings, reporting-structure claims.
- **On UNVERIFIABLE:** flag the row `caution:unverified`; do not block — EXCEPT an
  invariant-class UNVERIFIABLE (fabricated proper noun, cert/compliance claim, or
  reporting-structure / who-runs-what claim), which is corrected or cut like FALSE at every dial
  and blocks the row until clean. Flag-and-ship is only for non-invariant unverified inferences.
- **On FALSE:** correct — FALSE is ALWAYS corrected, at every dial.
- **Export behavior:** exports with a visible flag column; the human sender sees the accepted
  risk on every row before anything goes out.

Method: check the KB `facts` table FIRST; web only for genuinely new claims (max 1 search per
claim); verify once per company and reuse across its contacts. Trap list (verbatim from the Bay
runs): same-name company confusion; unconfirmed certification claims; invented program/customer
names; wrong standard-to-product mapping.
