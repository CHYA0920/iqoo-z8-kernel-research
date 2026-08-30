# 01 — Observation, Provenance, and Lab Safety

[简体中文](zh-CN/01-observation.md) · [Project home](../README.md)

## Outcome first

Kernel results are credible only when the team can prove **what ran, where it ran, and what was observed before failure**. The observation system is therefore part of the security boundary. It must not require opaque preload objects or grant an unreviewed artifact access to a developer workstation, personal phone, signing key, or authenticated ADB session.

## Safe laboratory baseline

- Use a dedicated device with a recoverable factory image and no personal data, SIM, accounts, credentials, or production network access.
- Before the first run, obtain the exact factory/OTA recovery package where legally and technically available, confirm how to enter the model's normal reboot and recovery menus, charge the battery, and preserve any unlock/recovery credentials required by the owner.
- Place the host and device on an isolated network. Disable unnecessary outbound connectivity.
- Build from reviewed source in a disposable environment; record source commit, compiler/toolchain identity, configuration, and output digest.
- Do not run attachments from issues or chat. An `LD_PRELOAD` library executes before the nominal program's `main()` and can falsify logs as well as compromise the host process.
- Collect only the minimum logs needed. Redact serials, boot IDs, absolute runtime addresses, tokens, and user data before publication.

For a freeze or black screen, use the official vivo force-restart procedure referenced in the root README: on current full-screen models, hold Power + Volume Down together for more than 10 seconds. A forced restart is a first recovery step, not proof that no damage occurred. Do not choose data-wipe or flashing entries merely because a recovery menu appears.

## Evidence chain

Each test round should bind the following fields:

| Field | Why it matters |
| --- | --- |
| Device/build identity | Prevents results from being assigned to the wrong OTA or vendor branch |
| Source commit and dirty-tree state | Makes the tested logic reviewable |
| Toolchain and configuration | Explains ABI, CFI, sanitizer, and layout differences |
| Artifact SHA-256 | Detects substitution between build and execution |
| Boot identity and monotonic timestamps | Separates rounds without publishing stable device identifiers |
| Predeclared pass/fail criterion | Prevents post-hoc interpretation |
| Panic/oops evidence | Distinguishes timeout, userspace failure, and kernel failure |

A digest authenticates equality to a known artifact; it does not establish that the artifact is benign. Review and reproducibility come first.

## Supplied run: bounded interpretation

The supplied transcript reaches `route-summary ... route_done=1` and then prints `uid=2000(shell)` in `u:r:shell:s0`, matching the unprivileged startup identity. It is therefore a valid positive criterion for “the observation/route artifact completed and returned without privilege elevation” on that round. It is not evidence of root, of general binary safety, or of behavior on another build. Because startup records `enforce=0`, no SELinux transition can be attributed to this run.

Target gating must be tested independently. The observed device tuple is PD2314/V2314A, `k6895v1_64`, `mt6895`, `arm64-v8a`, display build `PD2314_A_15.2.18.0.W10`, and kernel release `5.10.233-android12-9-g44ec642832da-dirty`. The artifact's printed profile label names `15.2.17.1`, so exact-version refusal is not proven by the positive run. Add pre-probe fail-closed checks and negative criteria: mismatched model, board, platform, display build, fingerprint, kernel release, ABI, or unavailable property must terminate before active probing.

## Failure-resilient logging

Use two independent channels in engineering labs:

1. A low-latency host channel for milestones and timing.
2. A write-through local record for the last completed state before a crash.

Neither channel should contain secrets or be enabled in production. A successful syscall return is evidence only for that return condition; it does not prove internal memory safety. Conversely, a missing line is not automatically a kernel panic unless the host transport and userspace process state are independently known.

## Recommended diagnostic builds

When source control is available, validate the repair with separate engineering kernels enabling the relevant combinations of lockdep, debug rtmutex support, KASAN, KCSAN, UBSAN, and fault-injection facilities that the vendor build can tolerate. Keep these builds off production devices because instrumentation changes timing and performance.

Useful assertions focus on invariants rather than attacker-controlled geometry:

- A task returning from a PI wait path has `pi_blocked_on == NULL` unless it remains genuinely blocked.
- The task whose PI fields are changed is the task whose `pi_lock` is held.
- A waiter removed from both RB trees is no longer discoverable through `rb_leftmost`.
- Early-error cleanup accepts a waiter with only the fields guaranteed initialized at that point.

## Publication rule

Publish source excerpts, minimized traces, symbolic offsets, and validation matrices. Do not publish live device addresses, opaque payloads, prebuilt preload libraries, or a sequence that converts the lifetime bug into a privilege-changing primitive.
