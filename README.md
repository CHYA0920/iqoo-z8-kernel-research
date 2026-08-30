# iQOO Z8 Kernel Research

[中文文档 / Chinese documentation](README.zh-CN.md) — the Chinese mirror includes a complete set of Chinese research documents (docs/zh-CN/).

## Research notice

This repository publishes **defensive security research** performed on a
privately owned device (vivo iQOO Z8, PD2314, kernel 5.10.233, Android 15).
All experiments were conducted on the researcher's own hardware, in a
controlled lab environment, for the purpose of understanding kernel attack
surface and improving detection and hardening posture.

The published material covers the **information-disclosure and
kernel-interaction stages only** of the overall research program. Every
component beyond the published boundary — including anything downstream of
the priority-inheritance walk — was deliberately excluded from this
repository. Nothing here is a complete exploit, and nothing here should be
expected to produce one.

Reproducing the observations requires: physical access to the same device
model, the same kernel build, and a controlled test harness. Misuse against
devices you do not own is illegal in most jurisdictions.

## Scope boundary

What is published:

- The full research map for stages 0 → 3.1 (see below), with per-node
  closure criteria, observation channels, and round counts.
- Two self-contained syscall-path probes (`mcast_test`, `sched_test`) that
  verify the reachability of the two kernel entry points this research
  relies on.
- A header-only KASLR side-channel sampling toolkit (`kernelsnitch/`)
  implementing the perf-based text-base disclosure used throughout the
  program.
- Research documents describing the methodology, the observation
  infrastructure, and the kernel static analysis behind each stage.

What is intentionally not published: every stage after 3.1. The repository
is cut at the "PI walk executes and returns" milestone. No post-walk
artifacts, no privilege-escalation material, no persistence components.

## Research map

The program is tracked as a tree of minimal-closure semantic nodes. A node
is marked **STABLE** only when: (a) its closure condition is written down
before the experiment, (b) the runtime criterion passes in at least three
consecutive rounds, and (c) the reproduction path is documented well enough
that a different session can re-run it. Static analysis alone never marks
a node STABLE.

```text
[0] Infrastructure & observation channels
 ├─[0.1] device / adb / CI build & deploy pipeline ......... STABLE
 ├─[0.2] TCP microsecond probe channel ..................... STABLE
 └─[0.3] O_DSYNC diagnostic persistence channel ............ STABLE

[1] Information-leak domain (pure reads, system healthy)
 ├─[1.1] KASLR text base via perf_event sampling ........... STABLE
 ├─[1.2] kernel symbol anchors (differential derivation) ... STABLE
 ├─[1.3] physical page disclosure & controlled staging ..... STABLE
 └─[1.4] spray residency (sk_buff 16/16 reclaim) ........... STABLE

[2] Kernel-interaction primitive domain
 ├─[2.1] futex PI chain-1 EDEADLK .......................... STABLE
 ├─[2.2] chain-2 / chain-3 ring closure .................... STABLE
 └─[2.3] tree stamp primitive (compat setsockopt
          deep-stack write, 260 bytes, byte-exact) ......... STABLE ★

[3.1] fire → PI walk executes and returns to userspace
        (sched_setattr trigger, walk consumes the staged
         geometry, ret2 = 0) ............................... STABLE
```

A rendered version of this map with per-node detail is in
[docs/research-map.md](docs/research-map.md).

## Research value

**1. A working microsecond-scale observation harness.** Kernel exploitation
research on production Android devices dies quietly: when the kernel panics,
shell-level diagnostics are typically unavailable (pstore, console, and
kmsg are all permission-gated on this device). This program solved that
structurally. A TCP probe channel reports per-stage timing marks
(connection, payload armed, trigger) with microsecond resolution and
survives kernel death long enough to be captured, and an `O_DSYNC`
write-through log channel guarantees that the last diagnostic line before
a panic is already on disk when the device reboots. Eighteen-plus
consecutive rounds ran with both channels alive. This is the infrastructure
that makes every later claim falsifiable.

**2. A three-tier evidence discipline applied to kernel research.** Every
conclusion in this program carries an explicit evidence grade: A1 (direct
runtime observation with a self-certifying log line), A2 (indirect
inference, never allowed to support a design alone), or B (unprobed). A
conclusion is only promoted from B to A1 by a dedicated criterion round.
This is documented in [docs/05-methodology.md](docs/05-methodology.md) and
is, in our view, the most reusable artifact of the whole program: it is a
working answer to "how do you know your exploit research claims are real".

**3. KASLR disclosure via perf_event timing side channels, hardened by
voting.** The text base is recovered from unprivileged
`perf_event_open` sampling. A key engineering finding: a single-anchor
derivation is probabilistically wrong on a measurable fraction of boots
(a wrong base can be off by megabytes and still look plausible). The fix
is a multi-sample voting scheme — and with it the disclosure pass rate
reached 100% across the validation batch. The toolkit that implements this
is published under `exploit/src/kernelsnitch/`.

**4. A byte-exact 260-byte deep-stack write through the 32-bit compat
setsockopt path.** The core kernel-interaction result. On this kernel,
`setsockopt(IPPROTO_IPV6, MCAST_JOIN_SOURCE_GROUP)` invoked from a 32-bit
task copies a 260-byte caller-supplied buffer into a fixed slot deep in
the kernel stack of the calling task. The research demonstrates, with
byte-exact readback verification (32-qword dumps), that this copy lands
at a predictable stack offset and can carry arbitrary content. This is
[2.3] in the map — the star asset of the primitive domain.

**5. Futex PI chain geometry as a deterministic pre-walk state machine.**
The research constructed a three-lock futex PI chain (EDEADLK rollback
path) whose every intermediate state is observable: chain-1 returns
`EDEADLK` (errno 35) on every round, chain-2/3 close the ring with
WAITERS-bit handshakes. The final walk — triggered by `sched_setattr`
re-prioritization — consumes the staged geometry and returns cleanly
(`ret2 = 0`), which is the 3.1 boundary this repository is cut at.

## Kernel static analysis background

Selected static findings that underpin the published stages are collected
in [docs/06-static-analysis.md](docs/06-static-analysis.md). Highlights:

- **rt_waiter geometry (5.10.233, MTK vendor tree).** The
  `rt_mutex_waiter` layout used by this kernel: tree_entry at +0x00,
  pi_tree_entry at +0x18, task pointer at +0x30, lock pointer at +0x38,
  prio at +0x40, deadline at +0x48. Verified against disassembly of the
  futex requeue path, not against decompilation.
- **Compat setsockopt copy path.** The 260-byte group-source filter
  structure copied on `MCAST_JOIN_SOURCE_GROUP`, and why the compat
  (32-bit caller) path is the one that matters for stack placement on
  this 64-bit kernel.
- **Futex hash bucket placement.** Address hashing of futex keys
  (`futex_hash.h` ports the in-kernel hash to userspace) — required for
  reasoning about which hb bucket a chain lands in.
- **Method rule.** Disassembly is the authority for values; decompilation
  is only ever used to locate interesting code. Every offset that matters
  was confirmed in the ELF file directly.

## Repository layout

```text
exploit/
  Makefile                 builds the two probes (32-bit ARM, NDK r29)
  src/
    mcast_test.c           setsockopt(MCAST_JOIN_SOURCE_GROUP) probe
    sched_test.c           sched_setattr reachability probe
    kernelsnitch/          KASLR side-channel sampling toolkit (headers)
docs/
  research-map.md          full node map with closure criteria
  01-observation.md        stage 0: observation infrastructure
  02-information-leak.md   stage 1: KASLR / anchors / page disclosure
  03-stack-write.md        stage 2: compat deep-stack write primitive
  04-fire-walk.md          stage 3.1: PI walk execution
  05-methodology.md        evidence tiers, criterion-first, single-variable rounds
  06-static-analysis.md    kernel static analysis notes
docs/zh-CN/               complete Chinese mirror of the seven documents
.github/workflows/build.yml   CI: builds and uploads both probes
```

## Building

Requires the Android NDK (r29) with the 32-bit ARM toolchain, or a host
clang with an Android sysroot:

```bash
cd exploit
make mcast-test sched-test
```

CI builds both probes on every push to `main` and uploads them as
artifacts (`mcast-test-z8`, `sched-test-z8`).

Both probes are static PIE armeabi-v7a binaries with zero dependencies.
They each make exactly one syscall and print a delimited result block —
they are reachability probes, not exploits.

**Device verification**: the CI-built artifacts of this repository were
executed on the target device (PD2314) side by side with the artifacts
of the internal research build (same source commit), in the same boot.
The outputs matched line for line on both probes.

## License

Published for research and defensive education. No warranty. You are
responsible for complying with the laws of your jurisdiction.
