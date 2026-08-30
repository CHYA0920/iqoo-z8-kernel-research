# Research Map

The full node map of the published research stages (0 → 3.1). Each node
carries: its closure condition (written before the experiments that
certify it), the runtime criterion (the observable signal that certifies
it), and the measured evidence (how many consecutive rounds passed).

Node status vocabulary:

- **STABLE** — closure condition defined in advance; runtime criterion
  passed in at least three consecutive rounds; reproduction path
  documented; mechanism-level (survives redesign of higher stages).
- Static analysis results can inform design but never mark a node STABLE.

## Tree

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
        (sched_setattr trigger, ret2 = 0) .................. STABLE
```

## Node detail

### [0.1] Build & deploy pipeline — STABLE
- **Closure condition**: code change → commit → CI build → artifact
  download → device deployment → test-launch runs with no manual
  intervention end to end.
- **Criterion**: preflight prints the SHA256 of the downloaded artifact
  and it matches the CI artifact identity, followed by a round-start
  log line.
- **Evidence**: fourteen consecutive fully automated rounds
  (the automated rounds lineage) plus the later validation batches.
- **Reusability**: mechanism-independent; nothing in the pipeline
  assumes anything about later stages.

### [0.2] TCP microsecond probe channel — STABLE
- **Closure condition**: the device-side harness connects back to the
  host over an adb-reversed TCP socket and every stage marker leaves a
  timestamped line in the host-side probe log.
- **Criterion**: per-round marker chain — connection, early milestones,
  payload-armed ("HELLO"), and trigger-fired ("FIRE") — all present.
- **Evidence**: every round of the automation lineage and all later
  batches, including rounds in which the kernel died after the fire
  stage (the channel outlives the kernel long enough to report it).

### [0.3] O_DSYNC diagnostic persistence channel — STABLE
- **Closure condition**: harness diagnostics are written with `O_DSYNC`
  so each write reaches storage before the next statement executes.
- **Criterion**: after a round-ending kernel panic and device reboot,
  the diagnostic file is still pullable and contains every line written
  before the panic.
- **Evidence**: dual pull at T+120s and T+180s succeeded on every round
  of the automation lineage.

### [1.1] KASLR text base — STABLE
- **Closure condition**: unprivileged `perf_event_open` sampling feeds a
  timing histogram from which the kernel text base is derived; a
  dedicated gate marker is computed to accept or reject the derivation
  (score threshold 0.5).
- **Criterion**: the gate prints an explicit verdict line; only PASS
  verdicts allow the round to continue.
- **Evidence**: fourteen consecutive rounds with scores 0.905–1.000,
  all PASS; with the multi-sample voting design (see
  [02-information-leak.md](02-information-leak.md)) the validation
  batch reached 100%.
- **Reusability**: orthogonal to all higher-stage design decisions.

### [1.2] Kernel symbol anchors — STABLE
- **Closure condition**: starting from addresses disclosed by [1.1],
  the addresses of selected .data symbols are derived by fixed
  differential offsets (symbol-relative deltas), not by reading
  kallsyms (which is restricted on this device).
- **Criterion**: the geometry log line of each round prints the
  derived task/text values and they match the differential prediction.
- **Evidence**: verified on every round after an early anchor fix
  (from the early rounds onward), across the whole the automated rounds lineage and all
  later batches.

### [1.3] Physical page disclosure & controlled staging — STABLE
- **Closure condition**: a leak primitive discloses a physical page
  backing a kernel allocation; the page is then prepared as a staging
  area whose full contents the user side controls.
- **Criterion**: the harness prints the leaked base address of the
  prepared page; subsequent verification reads back staged content.
- **Evidence**: successful on every round of the lineage (address
  varies with KASLR; semantics invariant).
- **Reusability**: mechanism-independent of higher stages.

### [1.4] Spray residency — STABLE
- **Closure condition**: a batch of 16 ordered sk_buff allocations
  (65,536 bytes each) is placed and retained via the unix-socket
  order-3 spray family.
- **Criterion**: all 16 sends report 65536 bytes accepted.
- **Evidence**: 16/16 on every round.

### [2.1] Futex PI chain-1 EDEADLK — STABLE
- **Closure condition**: `FUTEX_CMP_REQUEUE_PI` on the constructed
  chain returns `-EDEADLK` (errno 35) and the rollback leaves the
  WAITERS bit set on the target word.
- **Criterion**: errno 35 plus the after-value of the target word
  carrying the WAITERS bit, both printed by the harness.
- **Evidence**: every round.

### [2.2] Chain-2 / chain-3 ring closure — STABLE
- **Closure condition**: a second EDEADLK chain closes the waiter ring
  via WAITERS-bit handshakes, so that the final re-prioritization walk
  has a deterministic pre-walk state.
- **Criterion**: chain-2 errno 35 every round; chain-3 errno 35 plus
  the blocked waiter waking from the requeue.
- **Evidence**: from the round that first closed it, every round after.

### [2.3] Tree stamp primitive — STABLE ★
- **Closure condition**: a 32-bit task calling
  `setsockopt(IPPROTO_IPV6, MCAST_JOIN_SOURCE_GROUP)` with a 260-byte
  buffer writes that buffer, byte for byte, into a fixed deep-stack slot
  of the calling task's kernel stack.
- **Criterion**: the harness dumps 32 qwords of the target stack region
  and they match the constructed payload exactly.
- **Evidence**: every round of the lineage.
- **Reusability**: the star asset of the primitive domain — the
  mechanism is what allows the staged geometry of [1.3] to be placed
  where the later PI walk reads it.

### [3.1] Fire → PI walk executes and returns — STABLE
- **Closure condition**: `sched_setattr` re-prioritization on the ring
  head triggers the priority-inheritance walk; the walk runs through
  the staged geometry, completes, and control returns to userspace.
- **Criterion**: the harness prints the trigger return code; ret2 = 0
  is the pass line.
- **Evidence**: every round of the automation lineage (eighteen rounds)
  and every later validation round.
- **Note**: "the walk ran to completion" and "the walk's consequences
  are benign" are two different claims; the published boundary is the
  former. Consequence analysis is outside the scope of this repository.
