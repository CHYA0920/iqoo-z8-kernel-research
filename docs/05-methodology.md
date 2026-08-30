# 05 — Evidence Methodology and Safe Validation

[简体中文](zh-CN/05-methodology.md) · [Project home](../README.md)

## Evidence grades

| Grade | Meaning | Permitted conclusion |
| --- | --- | --- |
| A1 | Direct observation with artifact/build identity and a predeclared criterion | Supports a claim for that exact image and configuration |
| A2 | Static analysis, cross-reference, or indirect runtime inference | Supports design and review, not a standalone runtime claim |
| B | Unverified, ambiguous, or outside the collected evidence | Must be stated as unknown |

Static disassembly can prove that an instruction reads `task+0x898`; it cannot prove that a particular interleaving reached that instruction. A successful lab trigger can prove reachability on that build; it cannot prove all kernels carrying the same version string are affected.

## Criterion-first workflow

1. State the invariant or branch to be tested.
2. Define a minimally sufficient, non-privilege-changing observation.
3. Record exact source and build identity.
4. Change one independent variable.
5. Run repeated rounds on an isolated engineering device.
6. Preserve both positive and negative outcomes.
7. Promote the claim only to the evidence grade actually earned.

## Binary-analysis discipline

- Anchor function starts with ELF symbols/kallsyms, then decode the exact image.
- Express both absolute and function-relative addresses, while avoiding live runtime addresses in public logs.
- Treat decompiler output as navigation; use machine instructions for constants, offsets, widths, signs, and branch conditions.
- Cross-check a field with independent producers and consumers where possible.
- Infer structure size from object boundaries only when stack slots, call convention, and adjacent data are unambiguous.
- Mark missing DWARF/BTF explicitly; do not silently import a vanilla layout into a vendor image.

## Patch-review discipline

Review the repair as an invariant transformation:

| Before | After |
| --- | --- |
| Execution context chooses the cleanup task | Waiter ownership chooses the cleanup task |
| Wrong task's `pi_lock` may be held | Real waiter's `pi_lock` is held |
| Real task can retain stale `pi_blocked_on` | Real task is cleared before waiter lifetime ends |
| Chain adjustment receives `current` | Chain adjustment receives `waiter_task` |
| Partial initialization not separately guarded | Self-deadlock/early-error precondition is checked |

Do not accept a backport only because it applies cleanly. Android vendor trees frequently change rtmutex types, function signatures, trace hooks, and locking wrappers.

## Regression coverage

The minimum matrix includes:

- ordinary PI lock acquisition and release;
- proxy requeue success and rollback;
- timeout and signal interruption;
- owner exit/recovery paths;
- top and non-top waiters;
- nested donation chains;
- self-deadlock before and after waiter initialization;
- scheduler policy/priority changes while PI state is active;
- CPU migration and cancellation stress where supported.

For every case, verify return semantics, no stale `pi_blocked_on`, correct task lock ownership, consistent cached RB roots/leftmost pointers, balanced references, and clean lockdep/KASAN/KCSAN output.

## Publication discipline

Security documentation should make repairs reproducible without making privilege escalation turnkey. Include root cause, invariants, source-level remediation, affected-code detection, and safe regression tests. Exclude operational memory-reclamation schedules, forged kernel objects, live targets, privilege modification, and prebuilt executable payloads.

## Interpreting uncertainty

“Not confirmed” is a useful result. Typical reasons include missing vendor source, absence of DWARF/BTF, an inlined helper with no stable symbol boundary, or insufficient runtime instrumentation. Record which evidence would close the gap; do not fill it with an upstream assumption.
