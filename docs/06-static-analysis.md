# 06 — Instruction-Level Static Analysis

[简体中文](zh-CN/06-static-analysis.md) · [Project home](../README.md)

## Sample and address convention

Target: AArch64 ELF, `5.10.233-android12-9`; link base `0xffffffc008000000`. “rel” means virtual address minus link base. The image has no DWARF/BTF, so layouts are recovered from exact machine-code producers, consumers, and stack boundaries. Conclusions apply only to the ELF SHA-256 recorded in the root README.

## 1. `struct rt_mutex_waiter`

Layout A is correct for this image; layout B is incompatible with the instructions.

| Offset | Size | Field |
| --- | ---: | --- |
| `+0x00` | `0x18` | `rb_node tree_entry` |
| `+0x18` | `0x18` | `rb_node pi_tree_entry` |
| `+0x30` | `0x08` | `task_struct *task` |
| `+0x38` | `0x08` | `rt_mutex_base *lock` |
| `+0x40` | `0x04` | `int prio` |
| `+0x44` | `0x04` | alignment padding |
| `+0x48` | `0x08` | `u64 deadline` |
| `+0x50` | — | structure end / `sizeof` |

Direct initialization and assignment evidence:

```asm
rt_mutex_init_waiter (rel 0x227AA0)
+0x0004  add  x8, x0, #0x18
+0x0008  str  x0, [x0]
+0x000c  str  x8, [x0, #0x18]
+0x0010  str  xzr, [x0, #0x30]

task_blocks_on_rt_mutex (rel 0x224E6C)
+0x0088  stp  x20, x19, [x22, #0x30]  // task, lock
+0x008c  ldr  w8, [x20, #0x84]
+0x0094  str  w8, [x22, #0x40]        // prio
+0x0098  ldr  x8, [x20, #0x360]
+0x009c  str  x8, [x22, #0x48]        // deadline
```

`rt_mutex_slowlock` places the waiter at `sp+0x08`, initializes through `sp+0x57`, and stores the adjacent stack canary at `sp+0x58`, independently bounding the object at `0x50` bytes.

The requested leftmost back-conversion constant is **`0x18`**:

```asm
rt_mutex_adjust_prio_chain (rel 0x225C68)
+0x0110  ldr  x8, [x19, #0x888]
+0x0114  sub  x8, x8, #0x18
```

## 2. PI-chain task-field read order

One loop iteration directly consumes the following task offsets in this order, with later reads depending on tree/top-waiter changes:

| Order | Function offset | Instruction | Meaning |
| ---: | --- | --- | --- |
| 1 | `+0x00d8` | `ldr x28,[x19,#0x898]` | `pi_blocked_on` |
| 2 | `+0x0104/+0x0108` | `add ...,#0x880; ldar ...` | PI cached-tree root/empty check |
| 3 | `+0x0110/+0x0114` | `ldr ...,#0x888; sub ...,#0x18` | leftmost to waiter |
| 4 | `+0x0134` | `ldr w8,[x19,#0x84]` | effective priority |
| 5 | `+0x0150` | `ldr x8,[x19,#0x360]` | conditional deadline key |
| 6 | `+0x085c...` | access at `#0x86c` | owner's `pi_lock` |
| 7 | `+0x0a90/+0x1070` | `ldr ...,[x19,#0x888]` | old cached leftmost during removal |
| 8 | `+0x0afc/+0x10c0` | `ldr ...,[x19,#0x880]` | RB insertion root |
| 9 | `+0x115c/+0x1160` | root recheck | choose donor or NULL |
| 10 | `+0x1168/+0x116c` | leftmost then `[node+0x18]` | top waiter's task |
| 11 | `+0x1174` | `bl rt_mutex_setprio` | recalculate owner priority |
| 12 | `+0x11b8` | `ldr ...,[x19,#0x898]` | continue-chain decision |

`task+0x880` is the root pointer and `task+0x888` is `rb_leftmost`. A NULL root selects a NULL second argument but does not by itself skip `rt_mutex_setprio()`. The call is skipped when neither the old nor new top waiter is the waiter being reordered (`+0x0a68..+0x0a78`).

## 3. `rt_mutex_setprio()` decision tree

The candidate priority is `min(p->normal_prio, pi_task->prio)` when a donor exists. The fast return requires both `p->pi_top_task == pi_task` and a non-negative candidate equal to current `p->prio`.

```asm
rt_mutex_setprio (rel 0x1D4A70)
+0x0034  ldr  w25, [x20, #0x8c]  // normal_prio
+0x003c  ldr  w8, [x21, #0x84]   // donor prio, if non-NULL
+0x0048  ldr  x8, [x20, #0x890]  // pi_top_task
+0x030c  tbnz w25, #31, ...       // negative cannot use this fast return
+0x0310  ldr  w8, [x20, #0x84]
+0x0318  b.eq ...                  // return when unchanged
```

Class selection and task writes:

| Candidate | Class written to `task+0x98` | Additional direct data |
| --- | --- | --- |
| `< 0` | `dl_sched_class` | donor/task deadline state at `+0x84`, `+0x360`, `+0x36b`, `+0x400` as branch-dependent |
| `0..99` | `rt_sched_class` | restores own deadline entity when leaving DL |
| `>99` | `fair_sched_class` | clears RT timeout state when leaving RT |

```asm
rt_mutex_setprio
+0x02f0  tbnz w25, #0x1f, +0x0374
+0x03e4  str  x8, [x20, #0x98]   // sched_class
+0x03e8  str  w25, [x20, #0x84]  // effective prio
```

CFI checks indirect `sched_class` callbacks after loading them. It does not validate the lifetime/readability of the containing task or class pointer. `task->state` (`+0x30`) is not read by this function; setting state to zero does not make a partially initialized task safe.

## 4. `rb_erase()` two-child case

With both children present, successor `S` is the right subtree's minimum: either `R` itself when `R->left == NULL`, or the terminal node reached by following left links.

```asm
rb_erase (rel 0xB25CC4)
+0x0004  ldp  x8, x9, [x0, #8]    // R, L
+0x0010  ldr  x11, [x8, #0x10]   // R->left
+0x0014  cbz  x11, +0x00b4        // direct successor S=R
+0x001c  mov  x10, x9             // predecessor parent P
+0x0020  mov  x9, x11             // S
+0x0024  ldr  x11, [x11, #0x10]
+0x0028  cbnz x11, +0x001c        // descend left
```

For a deep successor, writes relink `P->left=S->right`, `S->right=R`, the parent metadata of `R`, `S->left=L`, the parent metadata of `L`, the original parent/root child slot, optional child `C` parent/color, and finally `S`'s inherited parent/color. These are ordinary RB replacement writes, not evidence of a separate bug.

Rebalancing occurs only when the replacement child is NULL and the successor's original color was black. This image enters an inlined fix-up loop at `rb_erase+0x168`; it does not call the separately symbolized `__rb_erase_color` from this branch.

## 5. `try_to_wake_up()` with `state == 0`

There is no exit before task dereference. For a non-self wakeup, the function first addresses/acquires `p->pi_lock` at `task+0x86c`, then reads `p->state` at `+0x30`. The earliest failure decision is `+0x00b4`; the eventual return is `+0x062c`.

```asm
try_to_wake_up (rel 0x1C8DCC)
+0x0058  add   x22, x21, #0x86c
+0x00a8  ldr   x8, [x21, #0x30]
+0x00b0  tst   x8, x9
+0x00b4  b.eq  +0x05bc
+0x05c0  stlrb w8, [x22]
+0x062c  ret
```

The self-wakeup branch avoids taking the task `pi_lock` but still loads `task+0x30` before deciding that state zero is not wakeable.

## Limits

The analysis proves machine-code layout and possible direct consumption. It does not by itself establish runtime exploitability, race reliability, or applicability to another vendor build. Missing source-level/inlined boundaries are marked conservatively rather than filled from a vanilla kernel.
