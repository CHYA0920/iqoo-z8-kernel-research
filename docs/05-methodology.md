# Methodology — How the Claims Were Made

The most reusable artifact of this program is not any single result;
it is the discipline under which results were produced. This document
is that discipline, written down.

## Three evidence tiers

Every conclusion in the program carries one of exactly three grades.
There is no middle state.

| Tier | Definition | Authority |
|---|---|---|
| **A1** | Direct runtime observation, with a self-certifying log line present in the round | May support design and execution decisions on its own |
| **A2** | Indirect inference: elimination, cross-document reasoning, single-source static deduction | May never support a design or a launch alone; must be labeled when cited |
| **B** | Unprobed: not examined, or the evidence chain is not closed | Forbidden as a basis for any action |

Two hard rules:

- The only path from A2 to A1 is a dedicated criterion round designed
  to observe the thing directly.
- "Unprobed" is not "nonexistent." Blind spots are declared, never
  silently treated as absences.

## Criterion-first

Before any node is exercised, its observation channel must exist and
be closed. A round whose success criterion cannot be read out in-round
is not a legal round. When a channel breaks, the only legitimate
priority is repairing the channel — running rounds against an unread
criterion is noise generation, and noise laundered into "results" is
the specific failure mode this discipline exists to prevent.

Stage 0 is the physical expression of this rule: [0.2] and [0.3]
existed before any kernel claim, so that every later claim had a
witness that survives kernel death.

## Single-variable rounds

A legal test round changes exactly one thing, and that thing is the
round's adjudication target. Multi-variable rounds have no attribution
power — if two things changed and the outcome changed, nothing was
learned about either.

Paired with this: **pre-registered decision tables**. Before a round,
every branch outcome is written down as a binary verdict with its
follow-up action. A round whose outcome space was not pre-registered
cannot produce a conclusion, because the conclusion would be chosen
after the fact.

## Node stability

A research claim is tracked as a node with an explicit closure
condition. A node is STABLE only when:

1. the closure condition was written before the certifying rounds;
2. the runtime criterion passed in at least **three consecutive
   rounds**;
3. the reproduction path is documented precisely enough that a
   different session (different person, different day) can re-run it;
4. the claim is **mechanism-level**: it survives redesign of every
   higher stage, so downstream work never inherits a rotten floor.

Static analysis can inform design. It can never mark a node STABLE —
a disassembly shows what code looks like, not what it does at runtime
under a specific interleaving.

## The chain-legality rule

A test round is legal only if every premise it relies on is already
STABLE. If a premise is unsettled, the premise's adjudication round
comes first. This outlaws the most seductive failure in exploit
research: "we'll find out what the earlier stage did by looking at
what the later stage did" — which multiplies unknowns instead of
resolving them.

## Mandatory write-back

A round that is not written back to the tracking documents does not
exist — its results may not be cited by any later round. Five fields
per recorded result: what was attempted, on what evidence grade, where
the evidence lives, what the outcome was, and what it changed.

## Why this matters beyond this project

Exploit research fails in a characteristic way: enthusiasm converts
weak signals into confident narratives, the narratives acquire
dependencies, and the whole structure collapses when one load-bearing
"fact" turns out to be an untested assumption. The discipline above is
the countermeasure: claims are graded at birth, criteria precede
exercise, one variable moves at a time, verdicts are pre-registered,
and floors must be stable before anything stands on them.

Applied rigorously, it converts exploit research from storytelling
into an evidence system — which is the difference between "we believe
the walk consumed the geometry" and "ret2 = 0, eighteen consecutive
rounds, logs attached."
