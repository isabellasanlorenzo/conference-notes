# GCD, Race Conditions & Privileged Daemons
### Talk Notes — Organized · March 2026

> 🎤 **Talk context:** *"GCD powers macOS concurrency. Misused queues and QoS can trigger race conditions, deadlocks, and sandbox escape in privileged services. Using a GSSCred sandbox-escape race, we will map failure modes to code reviews and telemetry for daemon behavioral detections."*

---

## Foundation: Processes vs. threads — the conceptual correction

A process is not a running entity — it's a container. It holds one or more threads and provides the virtual memory image, file descriptors, and ports shared by all threads. When someone says a process is "executing," the correct read is: *at least one of its threads is executing.*

> Race conditions live at the thread level, not the process level. This framing matters for everything that follows.

---

## Core concept: Why races happen — interleavings

Two threads (T1, T2) each execute: `read x` → `x = x + 1` → `write x`. Run sequentially, x increments by 2. But if T1 reads, then T2 reads before T1 writes, both see the same initial value and the increment happens only once. The number of possible interleavings grows combinatorially — only some are safe, and you can't test for all of them.

> This is the **TOCTOU problem** (Time of Check to Time of Use) applied to concurrency. The check is honest; the action operates on a different reality.

---

## GCD: Grand Central Dispatch — what it actually is

**Serial queues** — run one task at a time, in order. Safe for shared state — tasks can't interleave. Misuse: calling `dispatch_sync` back into the same serial queue freezes it.

**Concurrent queues** — run multiple tasks simultaneously. GCD manages the thread pool. Misuse: flooding with blocking tasks saturates the pool and starves other work.

### QoS levels (priority system)

| Level | Purpose |
|---|---|
| **User Interactive** | Animations, immediate UI |
| **User Initiated** | Seamless UX, e.g. table load |
| **Utility** | Long-running, user-aware (downloads) |
| **Background** | Invisible work, backups |
| **Default** | Between Initiated and Utility |
| **Unspecified** | Lowest priority, missing QoS info |

> ⚠️ GCD does not automatically prevent race conditions. It schedules execution. What happens to shared state is entirely your responsibility.

---

## Failure mode 1: Priority inversion

A high-priority thread ends up waiting on a low-priority thread that holds a lock. Because the low-priority thread can be preempted by a medium-priority thread, the high-priority work is frozen behind something it technically outranks.

### 3-thread swimlane: L holds lock, H blocks, M runs

*Assume priorities: H > M > L. Shared resource protected by mutex R.*

| Thread | Behavior |
|---|---|
| **L (low)** | Acquires lock R → runs critical section → eventually unlocks. But gets preempted by M before it can finish. |
| **H (high)** | Becomes runnable, needs R, blocks on L. Sits waiting while M (lower than H) runs freely. |
| **M (med)** | Preempts L while H is blocked. Runs despite being lower priority than H. This is the inversion. |

**Result:** H is indirectly delayed by M (lower priority than H) — that's the inversion.

> **Xcode runtime warning:** *"Thread running at QOS_CLASS_USER_INTERACTIVE waiting on a lower QoS thread running at QOS_CLASS_DEFAULT — investigate ways to avoid priority inversions."*

**Darwin's fix:** Priority inheritance — the kernel temporarily boosts the low-priority thread holding the lock to match whoever's waiting on it, until the lock is released.

### The right locking primitives

**`pthread_mutex_t`** — blocks/sleeps, scheduler can help:
```c
static pthread_mutex_t gMutex = PTHREAD_MUTEX_INITIALIZER;

pthread_mutex_lock(&gMutex);   // contended: thread can sleep
// critical section
pthread_mutex_unlock(&gMutex);
```

**`os_unfair_lock`** — Darwin replacement for spinlock:
```c
static os_unfair_lock gLock = OS_UNFAIR_LOCK_INIT;

os_unfair_lock_lock(&gLock);   // contended: waiter parks
// critical section
os_unfair_lock_unlock(&gLock);
```

> ⚠️ Avoid: reader-writer locks, semaphores, spinlocks. These cannot participate in QoS inheritance — they are the anti-patterns that allow inversions to persist.

---

## Failure mode 2: `dispatch_sync` deadlocks — the self-wait scenario

**Normal case (works fine):** Caller Queue (main thread) calls `dispatch_sync` on a *different* Target Queue (background thread). Caller blocks and waits. Task block runs and completes. Caller is unblocked. Clean.

**Deadlock scenario (app freezes):** Caller Queue and Target Queue are the *same* serial queue. Queue is blocked by the `dispatch_sync` call. Task cannot start because the queue is blocked. Nothing moves. App freezes.

### Circular wait — the full 4-step collapse

1. Main thread (serial queue) queues tasks to background threads (concurrent queue)
2. Background threads block, waiting for the main thread
3. Main thread makes a synchronous call, waiting for a worker thread
4. Circular wait: main thread waits for workers, workers wait for main thread → app freezes

> **Fix:** Use `dispatch_async` for cross-queue calls. Restructure code to avoid synchronous callbacks. Never call `dispatch_sync` from within a queue that could be waiting on itself.

---

## Failure mode 3: Thread pool saturation / resource starvation

`DISPATCH_ASYNC` with huge volume or blocking tasks floods the concurrent queue. GCD's worker thread pool saturates. All other tasks are starved of CPU and threads. High thread churn, CPU spikes, throughput collapses. Single-threaded daemons are especially exposed.

> GCD uses a fixed-size thread pool for concurrent tasks. Saturating it doesn't just slow your tasks — it starves the entire system of workers.

---

## Failure mode 4: Sandbox escape via XPC — the GSSCred exploit (CVE-2018-4331)

`com.apple.GSSCred` is a privileged macOS daemon managing Kerberos/GSS credentials. It accepted XPC connections from sandboxed, unprivileged clients while assuming a privileged context. A sandboxed attacker could send rapid-fire XPC messages that raced the daemon's state — exploiting the window between when trust was checked and when the privileged action was taken.

**The pattern:**
```
Unprivileged client
  → crosses XPC trust boundary
  → reaches privileged daemon
  → races TOCTOU window
  → code execution in privileged service context
```

**Telemetry signal:** Rapid-fire XPC calls from a sandboxed process. Anomalous credential activity. These are the behavioral indicators for daemon-level race exploitation.

> Queue misconfiguration + privileged XPC service + sandboxed client access = code execution. This is why GCD misuse is a security problem, not just a performance problem.

---

## Detection: Static code review signals

| Signal | What to flag |
|---|---|
| **XPC connection queueing** | Ensure `xpc_connection_set_target_queue()` is always called for accepted connections. Missing this is a flag. |
| **Serial vs concurrent assumptions** | Detect improper assumptions about serial execution in concurrent environments. A service that thinks it's serialized but isn't is a race waiting to trigger. |
| **Synchronous calls** | Flag all `dispatch_sync` usage and require explicit justification in code review. It is rarely the right choice. |
| **Shared mutable state without synchronization** | Identify globals accessed without locks or serial queue funnels. Any global touched by more than one queue without protection is a race condition. |

**Runtime tool:** Xcode Thread Performance Checker — detects priority inversions automatically at runtime. Enable in scheme settings. Also flags non-UI work running on the main thread.

---

## Conclusion — the four takeaways

- How GCD works and common concurrency hazards — queues, thread pools, QoS.
- Priority inversion and deadlock pitfalls with synchronous queue usage — and the right primitives to use instead.
- The GSSCred exploit: how queue misconfiguration in a privileged XPC service leads to sandbox escape and code execution.
- Detection: four static code review signals, Xcode Thread Performance Checker, and telemetry patterns for behavioral daemon detections.