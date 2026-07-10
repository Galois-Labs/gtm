# Dial 0 — Locked · fact-check strictness

**Two invariants at EVERY dial** (hard-coded in every fragment header AND enforced by the gate):
1. Never fabricate proper nouns — products, programs, customers, certifications, quantities, quotes.
2. Certification/compliance claims are verified-only, always — one wrong cert claim is instantly disqualifying with skeptical technical buyers.

- **Mode:** hard gate (blocking).
- **Checked:** every manifest claim, no exceptions.
- **On UNVERIFIABLE:** rewrite/generalize the claim, then re-check.
- **On FALSE:** correct, then re-check — FALSE is ALWAYS corrected, at every dial.
- **Export behavior:** blocked until 100% of manifest claims are CONFIRMED. The block is enforced
  deterministically at `queue build`; you cannot talk past it.

Method: check the KB `facts` table FIRST; web only for genuinely new claims (max 1 search per
claim); verify once per company and reuse across its contacts. Trap list (verbatim from the Bay
runs): same-name company confusion; unconfirmed certification claims; invented program/customer
names; wrong standard-to-product mapping.
