# iQOO Z8 Kernel Research — Defensive Documentation

[简体中文](README.zh-CN.md)

> [!CAUTION]
> **Do not execute shared libraries, preload objects, binaries, scripts, or archives obtained from issues, comments, chat groups, forks, or unofficial mirrors.** A `.so` loaded through `LD_PRELOAD` runs inside the launching process and can read credentials, alter files, contact external systems, or damage a connected device before any advertised kernel test begins. Treat every executable artifact as hostile until its source, build recipe, review history, and cryptographic digest have been independently verified. Use an isolated, non-personal test device with no accounts, tokens, SIM, or valuable data.

## Purpose and boundary

This repository is a documentation-only, defensive study of scheduler priority inheritance (PI), futex requeue-PI cleanup, and related kernel invariants on a privately owned vivo iQOO Z8 (PD2314) sample. It explains design intent, binary evidence, failure semantics, remediation, and safe validation.

The project does **not** endorse unauthorized access, privilege escalation, persistence, or deployment of vulnerability payloads. These documents intentionally omit a runnable exploit chain, heap-reclamation recipes, forged-object layouts intended for exploitation, privilege-modification procedures, and weaponized parameters. Kernel addresses and instruction offsets are included only where needed to audit the supplied image and review a repair.

The target is an AArch64 vendor kernel identified as `5.10.233-android12-9`. Android userspace/OTA labels can differ; the kernel version string alone never proves vulnerability or remediation because vendors backport fixes.

## Main conclusion

The relevant failure is a task-identity mismatch in the rtmutex proxy-lock rollback path associated with **CVE-2026-43499**. In the affected pattern, `remove_waiter()` performs cleanup against `current`, although the waiter belongs to `waiter->task`. During futex requeue-PI these tasks may differ. The real waiter can therefore retain a stale `pi_blocked_on` pointer to a waiter object whose lifetime has ended. A later, legitimate PI priority-propagation walk trusts that internal pointer and consumes stale state.

The semantic repair is to use `waiter->task` consistently for the task `pi_lock`, clearing `pi_blocked_on`, and the top-task argument passed into priority-chain adjustment. Backport the exact upstream/vendor-approved change and also include the follow-up self-deadlock handling associated with **CVE-2026-53166**; applying an isolated hand-edited hunk without regression coverage is not sufficient.

## Documentation map

| English | 简体中文 | Subject |
| --- | --- | --- |
| [Research map](docs/research-map.md) | [研究地图](docs/zh-CN/research-map.md) | Trust boundaries, invariant failure, remediation gates |
| [01 — Observation](docs/01-observation.md) | [01 — 观测](docs/zh-CN/01-observation.md) | Safe lab operation and evidence collection |
| [02 — Information exposure](docs/02-information-leak.md) | [02 — 信息暴露](docs/zh-CN/02-information-leak.md) | KASLR/perf/debug surfaces and hardening |
| [03 — Root cause](docs/03-stack-write.md) | [03 — 根因](docs/zh-CN/03-stack-write.md) | Futex PI proxy-lock lifetime and the repair |
| [04 — PI propagation](docs/04-fire-walk.md) | [04 — PI 传播](docs/zh-CN/04-fire-walk.md) | How later scheduler activity re-consumes stale state |
| [05 — Methodology](docs/05-methodology.md) | [05 — 方法论](docs/zh-CN/05-methodology.md) | Evidence grading and safe validation |
| [06 — Static analysis](docs/06-static-analysis.md) | [06 — 静态分析](docs/zh-CN/06-static-analysis.md) | AArch64 instruction-level findings |

English and Chinese files are complete mirrors with the same section structure. Historical filenames `03-stack-write.md` and `04-fire-walk.md` are retained only to preserve repository links; their content has been reframed around defensive root-cause and propagation analysis.

## Repair priorities

1. **Correct the lifetime invariant:** update the vendor `remove_waiter()` path to operate on `waiter->task`, under that task's `pi_lock`, and propagate the same task identity into chain adjustment.
2. **Include the follow-up guard:** verify the requeue-PI self-deadlock path cannot call cleanup with an uninitialized/NULL `waiter->task` (CVE-2026-53166).
3. **Reduce exposure:** restrict production perf events, debugfs/tracefs, vendor debug nodes, and kernel-address-bearing logs according to product requirements.
4. **Validate by invariant:** after every rollback/timeout/signal path, a task returning to userspace must not retain `pi_blocked_on`; all RB-tree membership and task references must be protected by the correct locks.

## Artifact policy for contributors

- Documentation pull requests are welcome. Do not attach prebuilt `.so`, APK, ELF, kernel modules, encrypted archives, or device images.
- Do not ask reviewers to run an opaque artifact. Provide source, exact build instructions, compiler identity, and reproducible digests when a test artifact is genuinely necessary outside this documentation branch.
- Redact device identifiers, account data, tokens, boot identifiers, and absolute runtime kernel addresses from logs before publication.
- Report a suspected security issue privately to the device/kernel vendor before publishing material that increases operational exploitability.

## Evidence identity

The static findings in this documentation were derived from the supplied AArch64 ELF and kallsyms set. Recorded SHA-256 values:

- ELF: `77cfbe299179360f54b5cb41f119766d3642a07208ce09d87b238072fbf19a52`
- ELF container archive: `149ba95260bf86806f98771bc86403ce2317d3b359b60b29d7865ebda3756dd0`
- kallsyms: `539dddd5bc02d86460b1fa9d6bc6808b0610395462fa271a8be261dd2ec518a2`

## References

- [Linux upstream fix 3bfdc63936dd — `rtmutex: Use waiter::task instead of current in remove_waiter()`](https://git.kernel.org/linus/3bfdc63936dd4773109b7b8c280c0f3b5ae7d349)
- [Debian tracker: CVE-2026-43499](https://security-tracker.debian.org/tracker/CVE-2026-43499)
- [Red Hat: CVE-2026-53166 follow-up regression](https://access.redhat.com/security/cve/cve-2026-53166)

## Disclaimer

No warranty is provided. Authorization, safety, disclosure coordination, export rules, and compliance with local law remain the user's responsibility.
