# Stage 3.1 — Fire: the PI Walk Executes and Returns

The published boundary of this repository. Everything before this line
is preparation; this stage is the kernel *consuming* the prepared
geometry, and the proof that it does so and comes back.

## The trigger

`sched_setattr` on the ring-head task re-prioritizes it. Under futex
priority-inheritance semantics, a re-prioritized task that is blocked
on a PI chain forces the kernel to walk the chain: adjust priorities
upward through the waiter structure, rebalance the waiter tree, and
propagate the change through the ring closed in [2.2].

Reachability of the trigger syscall from the shell domain is verified
by the published `sched_test` probe (one syscall, delimited result
block). The probe exists precisely so this claim is checkable without
any of the unpublished harness.

## What "fire" means mechanically

At the moment of the trigger:

- [1.1] has fixed the kernel text base (KASLR defeated by voting);
- [1.2] has derived the symbol addresses the harness reasons about;
- [1.3] has staged a fully user-authored page at a known kernel
  address;
- [1.4] has pinned the heap shape around it;
- [2.1]/[2.2] have closed a deterministic waiter ring with observable
  intermediate states;
- [2.3] has stamped byte-exact content onto the deep-stack slot the
  walk will traverse.

The trigger then asks the kernel to walk. The walk reads the staged
geometry from the stamped stack region and the staged page, performs
its priority propagation, completes, and control returns to userspace.

## The criterion

The harness prints the trigger's return code. **ret2 = 0 is the pass
line**: the walk ran to completion and returned.

- **Evidence**: eighteen consecutive rounds of the automation lineage,
  plus every round of the later validation batches.

## What this stage does and does not claim

"The walk ran to completion" is a precise, bounded claim. It is the
published milestone: kernel-side consumption of user-authored
geometry, with a clean return, reproducibly.

What it deliberately does not cover:

- the consequences of the walk for objects adjacent to the staged
  geometry;
- any post-walk stage, in any direction.

This is where the repository stops. The boundary is not rhetorical —
the code is physically cut at it, the build system builds nothing
beyond the two reachability probes, and the symbol/offset tables the
unpublished stages would need are absent from the tree.

## Why stop here as a publication

Stages 0–3.1 form a closed, self-contained, and individually
verifiable corpus: measurement infrastructure, information disclosure,
a byte-exact write primitive, and a demonstrated kernel-side walk over
authored state. Each piece is independently checkable by a third party
with the same device and the published probes. That corpus has
defensive value on its own — every stage it demonstrates is a surface
a hardened kernel should close, and the methodology generalizes beyond
this specific chain.
