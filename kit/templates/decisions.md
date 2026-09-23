> hint: append-only. Records rulings, not narrative — `progress.md` is the
> narrative log. Newest entries appended at the end. Fill every
> `<placeholder>`; delete this hint line after the first fill.

# `<CHAIN-ID>` — decisions

Owner-level decision points that shaped the chain-brief live in the brief
itself (§0/§1). This file records rulings taken inside the chain's
conservative-default mandate (`kit/RULES.md` §15): lead rulings before each
resume, orchestrator-level executor decisions, and owner rulings verbatim.
Every stamp is the event time (UTC).

## DEC-`<NN>` — `<UTC-stamp>` — `<who: owner | lead | orchestrator>` ruling: `<short title>`

`<Verbatim quote if this is an owner ruling. Otherwise: the ruling itself,
plus the reasoning in 1-3 sentences.>`

**Evidence (if a code fact is asserted):** `<path:line>`

**Scope:** `<what this decision covers and does not cover>`

---

## Retracting a decision

Never edit a decision silently. Strike the retracted text
(`~~like this~~`) and append a new dated row explaining why:

## DEC-`<NN>` — `<UTC-stamp>` — retraction of DEC-`<MM>`

`<reason the earlier ruling no longer holds>`
