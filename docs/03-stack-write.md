# Stage 2 — The Deep-Stack Write Primitive

The star result of the published program: a **byte-exact, 260-byte,
attacker-authored write into a predictable slot of the calling task's
own kernel stack**, through a compat syscall path that production
Android kernels keep reachable for unprivileged apps.

## [2.3] The tree stamp primitive

### Entry point

A 32-bit (armeabi-v7a) task calls:

```c
setsockopt(fd, IPPROTO_IPV6 /* 0x29 */,
           MCAST_JOIN_SOURCE_GROUP /* 0x2e */, buf, 260);
```

The kernel copies the 260-byte group-source filter structure from user
memory into kernel memory on the way to validating it. Reachability of
the path is verified by the published `mcast_test` probe (one syscall,
delimited result block).

### Why 32-bit compat matters

The copy lands in a stack frame whose depth depends on the syscall
entry path. The compat path — a 32-bit caller entering a 64-bit kernel
— has a distinct, deeper frame chain than the native path. The stack
slot of the copy is fixed for a given kernel build and entry path; on
this kernel it is deep in the calling task's kernel stack, at an
offset the harness verifies every round.

### The proof: byte-exact readback

The harness stamps a fully attacker-authored 260-byte payload (32
qwords plus tail) through the primitive and then dumps the target
stack region:

- **Criterion (BUFFER DUMP)**: the dumped 32 qwords match the
  constructed payload byte for byte.
- **Evidence**: every round of the lineage.

This is the point where the write stops being "a kernel bug that
copies too much" and becomes an engineering primitive: fixed
destination, fixed size, arbitrary content, repeatable on demand.

### Where the name comes from

The destination slot sits in the region the futex priority-inheritance
machinery later walks through (see [2.1]/[2.2] below) — the primitive
"stamps" content onto the tree-consumption path. The combination of
[1.3] (a controlled staging page) and [2.3] (a byte-exact deep-stack
copier) is what allows the walk-time geometry to be authored on the
user side.

## [2.1] Futex PI chain-1: EDEADLK rollback

The futex side of the primitive domain builds a deterministic
pre-walk state. The base construct is a priority-inheritance chain
driven with `FUTEX_CMP_REQUEUE_PI`:

- **Closure condition**: the requeue returns `-EDEADLK` (errno 35) —
  the kernel detects a would-be cycle, rolls the operation back — and
  the rollback leaves the WAITERS bit set on the target word.
- **Criterion**: errno 35 plus the after-value of the target word
  carrying the WAITERS bit, both printed per round.
- **Evidence**: every round of the lineage.

The significance: EDEADLK rollback is a *kernel-executed* traversal
whose side effects (waiter-list state) are observable from user space
through subsequent futex returns. It is the first point where the
program gets the kernel to walk a structure the program authored.

## [2.2] Chain-2 / chain-3: ring closure

A second EDEADLK chain closes the waiter ring via WAITERS-bit
handshakes — the blocked waiter's wake from the requeue is itself the
criterion line. From the round that first closed it, every subsequent
round reproduced it.

The closed ring is what makes stage 3.1 deterministic: the final
re-prioritization has exactly one well-defined walk to perform, with
the pre-walk state fully staged (contents authored via [2.3], heap
shape pinned via [1.4]).

## Reusability

- [2.3] is entry-path-specific (compat setsockopt on this kernel
  build) but *method-general*: for any kernel, the recipe is "find a
  compat copy path whose destination frame depth is stable, verify
  byte-exactness with a dumped payload, then author content."
- [2.1]/[2.2] are semantic (futex PI), not gadget-specific: they use
  documented kernel semantics, not build-specific addresses.
