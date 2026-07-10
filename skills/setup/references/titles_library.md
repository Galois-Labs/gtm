# Vertical → titles seed library (G11)

Seeds for Q3.1. Pick the vertical(s) matching the segments chosen in Q2.1, present the lists as a pre-filled draft, let the user prune/add, read back, then write `personas.buyer`, `personas.champion`, `personas.exclude_tokens`.

Rules that ride with every list:
- **Buyer** = who signs (economic buyer). At sub-100-person companies this is often the founder/CTO directly.
- **Champion** = who does the grind your product kills and fights for it internally. Managers and senior ICs, not HR.
- A hiring req is a COMPANY signal (pain + budget + clock); the CONTACT is the engineering chain below, found by title — never "whoever posted the job."
- Exclude tokens and seniorities are shared across verticals unless the user edits them.

**Shared exclude tokens** (`personas.exclude_tokens`):
```
recruit, talent acquisition, talent partner, sourcer, human resources, "hr ",
people operations, people ops, staffing, office manager, executive assistant, administrative
```

**Shared seniorities** (`personas.seniorities`, Apollo tokens):
```
owner, founder, c_suite, vp, head, director, manager, senior
```

---

## aero-defense

Buyer:
```
Founder, Co-Founder, CEO, CTO, Chief Technology Officer, Chief Engineer,
VP Engineering, VP of Engineering, VP Hardware, VP Product,
Director of Engineering, Director of Hardware, Director of Electrical,
Head of Engineering, Head of Hardware
```
Champion:
```
Principal Engineer, Staff Engineer, Principal Hardware Engineer, Senior Hardware Engineer,
Lead Hardware Engineer, Principal Electrical Engineer, Senior Electrical Engineer,
Senior Systems Engineer, Lead Systems Engineer,
Test Engineering Manager, Director of Test Engineering,
V&V Lead, Verification and Validation Manager, Verification Lead,
Quality Manager, Director of Quality, Design Assurance Manager, Design Assurance Lead,
Head of Compliance
```

## medtech

Buyer:
```
Founder, Co-Founder, CEO, CTO, Chief Technology Officer, Chief Engineer,
VP Engineering, VP of Engineering, VP Hardware, VP Product,
Director of Engineering, Director of Hardware,
Head of Engineering, Head of Hardware
```
Champion:
```
Director of Regulatory Affairs, VP Regulatory Affairs, Regulatory Affairs Manager,
Quality Manager, Director of Quality, VP Quality, Quality Engineering Manager,
Design Assurance Manager, Design Assurance Lead,
V&V Lead, Verification and Validation Manager, Verification Lead,
Test Engineering Manager, Principal Engineer, Senior Systems Engineer,
Head of Compliance
```

## robotics

Buyer:
```
Founder, Co-Founder, CEO, CTO, Chief Technology Officer,
VP Engineering, VP of Engineering, VP Hardware,
Director of Engineering, Director of Hardware, Head of Engineering, Head of Robotics
```
Champion:
```
Principal Robotics Engineer, Senior Robotics Engineer, Staff Engineer,
Lead Systems Engineer, Senior Systems Engineer,
Controls Engineering Lead, Senior Controls Engineer, Autonomy Lead,
Embedded Software Manager, Firmware Lead,
Functional Safety Engineer, Functional Safety Manager,
Test Engineering Manager, Quality Manager
```

## dev-tools

Buyer:
```
Founder, Co-Founder, CEO, CTO, Chief Technology Officer,
VP Engineering, VP of Engineering, VP Platform,
Director of Engineering, Head of Engineering, Head of Developer Experience
```
Champion:
```
Staff Engineer, Principal Engineer, Senior Software Engineer, Tech Lead,
Engineering Manager, Platform Engineering Lead, Infrastructure Lead,
DevOps Lead, Senior DevOps Engineer, SRE Lead, Site Reliability Engineer,
Developer Experience Engineer, Developer Advocate
```
Note: champion-heavy vertical — ICs live in their inbox. Offer flipping `personas.channel_sequence.champion` to `[email, linkedin_dm]` at Q4.1.

## fintech

Buyer:
```
Founder, Co-Founder, CEO, CTO, Chief Technology Officer, Chief Product Officer,
VP Engineering, VP of Engineering, Director of Engineering, Head of Engineering,
Head of Payments, Head of Risk
```
Champion:
```
Staff Engineer, Principal Engineer, Engineering Manager, Tech Lead,
Payments Engineering Lead, Platform Lead, Data Engineering Lead,
Compliance Manager, Compliance Officer, Head of Compliance,
Risk Manager, Risk Operations Lead, Fraud Operations Lead
```
