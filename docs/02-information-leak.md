# Stage 1 — Information-Leak Domain

Four capabilities, all pure reads, all leaving the system healthy.
Together they answer: *where is the kernel, where are the objects we
care about, and where can we stage controlled data.*

## [1.1] KASLR text base via perf_event sampling

The kernel text base is disclosed from unprivileged
`perf_event_open`-based timing samples. The general idea of
perf-side-channel KASLR disclosure is known in the literature; the
engineering contribution here is reliability.

**The single-anchor trap.** A derivation anchored on one sample point
is probabilistically wrong on a measurable fraction of boots. Worse, a
wrong derivation is not obviously wrong: the derived base can be off
by megabytes while remaining a plausible-looking kernel address. A
wrong base silently poisons every downstream address computation.

**The voting fix.** The toolkit (`exploit/src/kernelsnitch/`) takes
multiple independent samples, derives candidate bases, and votes. The
round continues only on a decisive majority. Measured results:

- gate verdict PASS with score 0.905–1.000 on fourteen consecutive
  rounds;
- 100% pass across the validation batch after voting was adopted.

**Instrumentation notes.** The sampling side also performs batch-pacing:
waiter-thread batches are spawned in bounded groups with inter-batch
traversal-time probes (128x `FUTEX_WAKE_BITSET` with disjoint bitsets —
timing only, zero actual wakeups), exiting early when traversal time
exceeds a baseline multiple. This keeps the sampling pass itself from
destabilizing the system it measures.

## [1.2] Kernel symbol anchors by differential derivation

`kallsyms` is unreadable on this device (kptr_restrict). Symbol
addresses are instead derived from disclosed anchors plus fixed
deltas: the offsets *between* symbols are build constants, so knowing
one anchor address yields the others by addition.

- Input: the text base from [1.1] and measured anchor addresses.
- Output: addresses of selected .data-section symbols the later stages
  reason about.
- Criterion: the per-round geometry log prints derived values; they
  match the differential prediction on every round after the anchor
  arithmetic was locked down.

This is a general technique for restricted-kallsyms environments and
the reason the whole program never needed a symbol-table read.

## [1.3] Physical page disclosure and controlled staging

A leak primitive discloses the physical page backing a kernel
allocation. The disclosed page is then prepared as a staging area: the
user side controls the page's full contents. The harness prints the
page base each round; verification reads back staged content.

What this buys: a region of kernel-visible memory whose *contents* are
fully attacker-chosen and whose *address* is known. The published
boundary ends before any consumer of that staging area beyond the
walk-time geometry of stage 2/3.1.

## [1.4] Spray residency

Sixteen ordered sk_buff allocations of 65,536 bytes each, placed and
retained via the unix-socket order-3 spray family. Criterion: all
sixteen sends report 65536 bytes accepted — measured 16/16 on every
round.

Purpose: heap-shape control. Keeping the sprayed objects resident pins
the allocator layout the later stages reason about, so that the
geometry staged in [1.3] stays put through the timing window of
stage 2/3.1.

## Why this domain matters

Stages [1.1]–[1.4] are pure reads plus heap shaping: the system is
healthy and identical before and after. They are also the
mechanism-independent foundation — nothing here assumes anything about
how the later stages consume the disclosed information, so the whole
domain survives redesign of everything above it.
