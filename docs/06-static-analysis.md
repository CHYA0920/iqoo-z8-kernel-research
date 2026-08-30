# Kernel Static Analysis Notes

Selected static findings that underpin the published stages. These are
facts about the kernel binary (5.10.233, MTK vendor tree, this
device's build), each verified the way the methodology requires:
disassembly is the authority for values, decompilation is only ever
used to locate interesting code, and every load-bearing offset is
confirmed by direct file-level reads of the kernel image.

## rt_mutex_waiter layout (5.10.233, MTK vendor tree)

The waiter structure the futex PI machinery walks. Offsets as verified
in the requeue-path disassembly:

```text
struct rt_mutex_waiter {
  +0x00  rb_node  tree_entry        // waiter-tree membership
  +0x18  rb_node  pi_tree_entry     // PI-propagation tree membership
  +0x30  task_struct *task          // owning task
  +0x38  rt_mutex   *lock           // lock this waiter blocks on
  +0x40  int        prio            // effective priority
  +0x48  u64        deadline        // deadline scheduling key
  ...
};
```

Notes:

- `prio` at +0x40 as a plain int was specifically confirmed (some
  other 5.10 vendor trees differ here).
- Two rb_node memberships (plain tree and PI tree) is what makes the
  walk a *tree* traversal rather than a list walk — geometry stamped
  through [2.3] has to be valid for both consumers.

## The compat setsockopt copy (MCAST_JOIN_SOURCE_GROUP)

`setsockopt(fd, IPPROTO_IPV6, MCAST_JOIN_SOURCE_GROUP, buf, len)` with
len = 260 copies the caller's group-source filter request structure
into kernel memory for validation. Structural points that matter to
[2.3]:

- The structure is a fixed 260 bytes — the copy length is
  caller-declared and the kernel copies exactly `len` before
  validating semantic fields, so the 260-byte write is
  attacker-sized.
- On the compat path (32-bit caller, 64-bit kernel), the entry chain
  runs through compat syscall framing whose stack depth differs from
  the native path; the destination slot's depth is fixed per kernel
  build and entry path.
- The native (64-bit) pselect/sendmmsg deep-stack alternatives were
  examined and are not viable on this kernel for stack-geometry
  reasons; the 32-bit compat socket path is the one whose destination
  geometry is stable. This is why the published probes build as
  armeabi-v7a.

## Futex hash bucket placement

The kernel maps a futex key (address, size) to a hash bucket via a
multiplicative hash over the page and offset. Reasoning about *which*
waiter-tree a chain lands in requires reproducing this hash in user
space. `exploit/src/kernelsnitch/futex_hash.h` is that port: same
constants, same shift arithmetic, runnable offline. It is what lets
the harness predict bucket collisions (and thus waiter pile-ups)
before creating them.

## perf_event sampling surface

Unprivileged `perf_event_open` is permitted on this device
(`perf_event_paranoid ≤ 1`). The sampling side of [1.1] uses timing
histograms over kernel-side activity to derive the text base; the
`kernelsnitch.h` header documents the batch-pacing loop (bounded
waiter batches, inter-batch traversal-time probes with disjoint
wakeups, early-exit on traversal-time blowup) that keeps the sampler
from destabilizing the sampled system.

## CFI

This kernel build has Control-Flow Integrity enabled: indirect-call
targets go through jump-table stubs. For the published stages this
matters only as an environment fact (documented in
[01-observation.md](01-observation.md)); no published stage depends on
defeating it.

## Method rules for static analysis (the short version)

1. Disassembly is the authority for values. Decompilation output is
   for locating code, never for numbers.
2. Every offset that a runtime claim depends on is confirmed by
   reading the kernel image file directly (ELF program headers → file
   offset → bytes).
3. Cross-reference verification uses the disassembler's xref channel,
   not ad-hoc text scanning — hand-scanning for call sites has a
   documented history of missing the ones that matter.
4. A static finding never promotes a research node past tier A2 until
   a runtime criterion round observes it. (See
   [05-methodology.md](05-methodology.md).)
