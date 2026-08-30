# 03 — Root Cause: Futex PI Proxy-Waiter Lifetime

[简体中文](zh-CN/03-stack-write.md) · [Project home](../README.md)

> The historical filename is retained for link compatibility. This document describes the lifetime defect and repair, not a write primitive.

## Conclusion

The repair is necessary because `remove_waiter()` serves two semantic contexts:

1. ordinary rtmutex slow-lock cleanup, where the executing task normally owns the waiter; and
2. futex requeue-PI proxy-lock rollback, where one task may operate on a waiter belonging to another.

The old implementation used `current`. That works accidentally in the first context but violates task identity in the second. The authoritative identity is `waiter->task`.

## Why proxy locking exists

A PI futex combines a userspace futex word with an in-kernel rtmutex. Requeue-PI can move a waiting task from a non-PI wait queue to a PI futex without waking it into an uncontrolled race. Kernel code therefore performs a **proxy** operation: the current requeuer manipulates locking state on behalf of the actual waiting task.

That distinction is fundamental:

| Role | Meaning |
| --- | --- |
| `current` | Task executing the requeue/rollback code |
| `waiter->task` | Task represented by the rtmutex waiter |
| lock owner | Task currently receiving priority donation |

These can be three different tasks. A helper reused across ordinary and proxy paths must not infer waiter ownership from execution context.

## Normal PI state

For each task:

- `pi_blocked_on` points to the live waiter on which the task is blocked;
- `pi_waiters` is a cached RB tree of waiters donating priority to this task as a lock owner;
- `pi_lock` serializes mutations of that task's PI state.

The key lifetime rule is:

```text
task->pi_blocked_on != NULL
    implies waiter is live, belongs to task, and task is genuinely blocked.
```

When a waiter is removed because of rollback, timeout, signal, or acquisition, the pointer must be cleared before the waiter storage can disappear.

## The failing cleanup

In the affected proxy rollback, `waiter->task != current`. Cleanup against `current` produces three related errors documented by the upstream fix:

1. waiter-tree removal occurs without holding the real waiter's `pi_lock`;
2. the real waiter task's `pi_blocked_on` is not cleared, leaving a dangling pointer after stack lifetime ends;
3. `rt_mutex_adjust_prio_chain()` receives the wrong top-task identity.

The bug is therefore not “the scheduler writes too much.” It is an **ownership and lifetime violation** created during cleanup. Later scheduler code merely trusts state that should have been made impossible.

## Semantic repair

The vendor implementation must be adapted to the exact branch/API, but all three task-sensitive operations must use the same `waiter_task`:

```c
/* Schematic only; use the exact upstream/vendor backport. */
struct task_struct *waiter_task = waiter->task;

raw_spin_lock(&waiter_task->pi_lock);
rt_mutex_dequeue(lock, waiter);
waiter_task->pi_blocked_on = NULL;
raw_spin_unlock(&waiter_task->pi_lock);

rt_mutex_adjust_prio_chain(/* branch-specific arguments */,
                           waiter_task);
```

Review the actual upstream commit rather than copying this schematic signature: stable and Android vendor branches differ in `rt_mutex_base` types, argument order, locking helpers, and guard style.

## Follow-up regression: partial initialization

Changing cleanup to dereference `waiter->task` exposes another required invariant: every path reaching `remove_waiter()` must have initialized that field. A self-deadlock error can return early before assignment for a non-top requeued waiter. The follow-up tracked as CVE-2026-53166 prevents that path from calling cleanup with a NULL waiter task.

This is why a correct backport set contains both concepts:

- CVE-2026-43499: choose the correct task;
- CVE-2026-53166: do not reach correct-task cleanup before that task exists.

## Source-review decision table

| Observation in vendor tree | Decision |
| --- | --- |
| `current->pi_lock` / `current->pi_blocked_on` in shared `remove_waiter()` | Affected semantic pattern |
| Local `waiter_task = waiter->task`, used for lock, clear, and chain adjustment | Primary repair present |
| Requeue self-deadlock checked before proxy cleanup, or equivalent proven initialization | Follow-up guard present |
| Kernel version alone | Insufficient evidence |

## Validation matrix

Test an instrumented engineering kernel across ordinary lock/unlock, requeue success, rollback, timeout, signal interruption, owner exit, top and non-top waiter cases, and self-deadlock. For every exit, assert correct `pi_lock` ownership, `pi_blocked_on` cleanup, balanced task references, consistent RB-tree caches, and absence of lockdep/KASAN/KCSAN reports.

## References

- [Upstream commit 3bfdc63936dd](https://git.kernel.org/linus/3bfdc63936dd4773109b7b8c280c0f3b5ae7d349)
- [CVE-2026-43499 description and fixed-package tracking](https://security-tracker.debian.org/tracker/CVE-2026-43499)
- [CVE-2026-53166 follow-up](https://access.redhat.com/security/cve/cve-2026-53166)
