# 02 — Information Exposure and Attack-Surface Reduction

[简体中文](zh-CN/02-information-leak.md) · [Project home](../README.md)

## Outcome first

Address disclosure is an **amplifier**, not the rtmutex root cause. Fixing CVE-2026-43499 remains mandatory. Restricting perf, debug filesystems, vendor diagnostics, and address-bearing logs reduces reliability and observability available to an attacker and limits damage from future memory-safety bugs.

## Exposure classes

| Surface | Intended purpose | Production risk | Defensive action |
| --- | --- | --- | --- |
| `perf_event_open` and PMU sampling | Profiling and performance diagnosis | Timing or instruction-pointer correlated data may weaken KASLR | Set a restrictive policy; grant access only to trusted profiling domains |
| debugfs/tracefs/kprobes | Kernel development and tracing | Kernel addresses, object identities, and privileged control paths | Do not mount for untrusted domains; enforce SELinux and capabilities |
| Vendor debug nodes | Device/SoC diagnosis | Often expose implementation-specific logs or memory windows | Remove from production builds or require a narrowly scoped privileged domain |
| `/proc`, sysctls, panic logs | Operations and postmortem analysis | Stable pointers and build details improve address inference | Apply pointer restrictions and redact exported diagnostics |
| Build artifacts and symbol maps | Crash triage | Exact offsets enable binary matching | Control distribution; publish only what disclosure policy permits |

## KASLR is not a memory-safety boundary

KASLR raises uncertainty. It does not make a stale pointer safe and must not be credited as remediation. A secure review therefore records two separate statuses:

- **Semantic status:** is the wrong-task cleanup fixed?
- **Exposure status:** can an untrusted process obtain address-correlated information?

Closing the second while leaving the first open is defense in depth only. Conversely, fixing the lifetime bug does not justify leaving broad debug access enabled.

## Device-specific review points

The supplied laboratory logs indicate that performance sampling and vendor/debug facilities were reachable in the tested configuration. That observation is build- and policy-specific. A production audit should verify the effective state on the shipping image rather than copying lab assumptions:

- actual `perf_event_paranoid` behavior and SELinux domain permissions;
- whether debugfs and tracefs are mounted and accessible to shell/app domains;
- whether vendor nodes under debug paths disclose kernel addresses or permit arbitrary-position reads/writes;
- whether crash, trace, or telemetry output prints unmasked pointers;
- whether engineering properties accidentally persist in release builds.

## Hardening acceptance criteria

1. An ordinary application and ADB shell cannot open privileged perf events or kernel-address-bearing trace sources unless product requirements explicitly allow it.
2. Production SELinux policy denies vendor debug nodes to untrusted domains.
3. Release builds do not expose generic kernel-memory windows through debug handlers.
4. Pointer-bearing telemetry is masked or removed outside authenticated engineering modes.
5. Any exception is documented with an owner, threat model, and automated regression test.

## Safe validation

Validate permissions and data classification, not exploitability. Use a minimal program that attempts to open the intended diagnostic interface and records only success/failure and sanitized metadata. Do not combine the check with heap shaping, PI manipulation, or privilege-changing behavior.
