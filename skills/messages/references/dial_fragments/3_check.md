# Dial 3 — Adventurous · fact-check strictness

**Two invariants at EVERY dial** (hard-coded in every fragment header AND enforced by the gate):
1. Never fabricate proper nouns — products, programs, customers, certifications, quantities, quotes.
2. Certification/compliance claims are verified-only, always — one wrong cert claim is instantly disqualifying with skeptical technical buyers.

- **Mode:** spot pass (flag, not block).
- **Checked:** fabrication classes only — proper nouns + numbers.
- **On UNVERIFIABLE:** per-row flag count; no rewrite demanded.
- **On FALSE:** correct — FALSE is ALWAYS corrected, at every dial.
- **Export behavior:** exports; each affected row annotated "dial-3: N unverified inferences" so
  the human sender sees exactly how much risk this dial accepted.

Method: check the KB `facts` table FIRST; web only for genuinely new claims (max 1 search per
claim); verify once per company and reuse across its contacts. Trap list (verbatim from the Bay
runs): same-name company confusion; unconfirmed certification claims; invented program/customer
names; wrong standard-to-product mapping.
