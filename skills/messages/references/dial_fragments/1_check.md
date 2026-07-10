# Dial 1 — Careful (default) · fact-check strictness

**Two invariants at EVERY dial** (hard-coded in every fragment header AND enforced by the gate):
1. Never fabricate proper nouns — products, programs, customers, certifications, quantities, quotes.
2. Certification/compliance claims are verified-only, always — one wrong cert claim is instantly disqualifying with skeptical technical buyers.

- **Mode:** hard gate (blocking).
- **Checked:** every claim; whitelisted safe inferences (class `inferred_safe`) auto-pass.
- **On UNVERIFIABLE:** soften the claim (state it as the whitelisted inference it is, or cut it).
- **On FALSE:** correct — FALSE is ALWAYS corrected, at every dial.
- **Export behavior:** blocked until the manifest is clean or softened. Enforced deterministically
  at `queue build`.

Method: check the KB `facts` table FIRST; web only for genuinely new claims (max 1 search per
claim); verify once per company and reuse across its contacts. Trap list (verbatim from the Bay
runs): same-name company confusion; unconfirmed certification claims; invented program/customer
names; wrong standard-to-product mapping.
