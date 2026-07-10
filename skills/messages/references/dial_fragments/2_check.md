# Dial 2 — Confident · fact-check strictness

**Two invariants at EVERY dial** (hard-coded in every fragment header AND enforced by the gate):
1. Never fabricate proper nouns — products, programs, customers, certifications, quantities, quotes.
2. Certification/compliance claims are verified-only, always — one wrong cert claim is instantly disqualifying with skeptical technical buyers.

- **Mode:** light pass (flag, not block).
- **Checked:** high-risk classes only — proper nouns, certs, standard→product mappings.
- **On UNVERIFIABLE:** flag the row `caution:unverified`; do not block.
- **On FALSE:** correct — FALSE is ALWAYS corrected, at every dial.
- **Export behavior:** exports with a visible flag column; the human sender sees the accepted
  risk on every row before anything goes out.

Method: check the KB `facts` table FIRST; web only for genuinely new claims (max 1 search per
claim); verify once per company and reuse across its contacts. Trap list (verbatim from the Bay
runs): same-name company confusion; unconfirmed certification claims; invented program/customer
names; wrong standard-to-product mapping.
