# GCD, Race Conditions & Privileged Daemons
### Plain English Edition — no experience required

> 💡 This is a layman's breakdown of a technical security talk. Read this first, then the technical notes will make a lot more sense.
>
> **Talk context:** *"GCD powers macOS concurrency. Misused queues and QoS can trigger race conditions, deadlocks, and sandbox escape in privileged services. Using a GSSCred sandbox-escape race, we will map failure modes to code reviews and telemetry for daemon behavioral detections."*

---

## What was this talk actually about?

Your Mac is constantly doing dozens of things at the same time — playing music, syncing files, running security checks, responding to your clicks. Apple built a system called **GCD** to manage all of that juggling. The talk was about what happens when developers use that system incorrectly inside sensitive, high-trust parts of macOS — and how an attacker can turn those mistakes into a way to break out of security sandboxes or take control of privileged processes.

The real-world example: a bug in a macOS credential storage system (GSSCred) that let a sandboxed, untrusted app race its way into privileged territory. This became **CVE-2018-4331**.

---

## Vocab — words you need before anything else makes sense

| Term | Plain English |
|---|---|
| **Process** | A running app or system service. Think of it as a workspace — it holds memory, open files, and network connections. It's the container, not the worker. |
| **Thread** | The actual worker inside that container. One process can have many threads all doing things simultaneously. Races happen between threads, not processes. |
| **GCD** | Grand Central Dispatch. Apple's built-in system for managing threads. Instead of creating threads yourself, you hand tasks to GCD and it figures out when and how to run them. |
| **Dispatch queue** | A line of tasks waiting to run. You submit work to a queue; GCD pulls from the queue and runs it. Like a to-do list that GCD manages for you. |
| **Race condition** | When two things happen at the same time and the outcome depends on which one finishes first — and the programmer didn't account for that. Sometimes the wrong one wins. |
| **XPC** | A macOS system for apps to send messages to each other securely — especially between low-privilege apps and high-privilege system services. Like a secure intercom between rooms with different clearance levels. |
| **Daemon** | A background system service that runs quietly without a window or user interaction. macOS has hundreds of them managing things like Bluetooth, credentials, networking, etc. |
| **Sandbox** | A security cage that limits what an app can do. A sandboxed app can't read your files, access the network freely, or talk to system services without explicit permission. |
| **Privileged** | Having elevated trust or power on the system. A privileged process can do things a normal app can't — like modifying system files, accessing credentials, or talking to the kernel. |
| **QoS** | Quality of Service. A priority label you attach to work, telling the system how urgently it needs to run. Higher QoS = gets scheduled sooner. |
| **Mutex / lock** | A "one at a time" gate. Only one thread can hold the lock at a time. Others have to wait. It's how you prevent two threads from trampling the same data simultaneously. |
| **TOCTOU** | Time of Check to Time of Use. You check if something is safe, then go to use it — but in the gap between those two moments, something changed. You acted on stale information. |

---

## Foundation: processes vs. threads

A process is just a container — a workspace with memory and resources. What actually *runs* is a thread inside that process. One process can have many threads all executing at once.

> 🏢 **Analogy:** A process is like an office building. It has desks, files, phones, a kitchen. But the building doesn't do work — the *employees* (threads) do. You can have one employee or fifty, all working inside the same building at the same time. Race conditions are when two employees grab the same file simultaneously and make conflicting edits.

When someone says a process is "executing," what's actually true is: *at least one of its threads is executing.* Race conditions live at the thread level, not the process level.

---

## Why race conditions happen at all

Imagine a shared counter that starts at 0. Thread 1 reads it (sees 0), plans to make it 1, and writes 1. Thread 2 does the same thing. If they run one after the other, the counter becomes 2. But if they both *read* before either *writes*, they both see 0, both write 1, and the counter only becomes 1. One increment got lost — and neither thread did anything wrong.

> 🎟️ **Analogy:** Two people are buying the last concert ticket online at the same time. Both see "1 ticket remaining." Both click buy. The system checks availability for both before either purchase goes through — so both succeed. Now you've sold the same ticket twice. Neither buyer cheated. The system just didn't handle the timing.

### TOCTOU — the gap attackers exploit

The system checks something ("is this user authorized?") and then acts on that check. Between the check and the action, an attacker changes the thing that was checked. The check was honest — but the action operated on a different reality than the one that was verified.

> ⚠️ This is why race conditions are so hard to catch: they only happen under specific timing conditions. You can test an app a thousand times and never see the race — then it triggers in production under load.

---

## What Grand Central Dispatch actually does

Managing threads manually is a nightmare. GCD is Apple's solution: instead of managing threads, you describe *work* and hand it to a queue. GCD decides when to run it, which thread to use, and how to prioritize it.

**Serial queue** — tasks run one at a time, in order. Like a single checkout lane. Safe for shared data, but can be a bottleneck.

**Concurrent queue** — multiple tasks run simultaneously. Like several checkout lanes open at once. Faster, but requires care when tasks touch shared data.

### QoS levels (the priority system)

| Level | What it's for |
|---|---|
| **User Interactive** | Must happen right now — animations, button responses |
| **User Initiated** | User is waiting — loading something they just clicked |
| **Utility** | Running in background, user is aware — download progress bar |
| **Background** | Invisible work — syncing, backups, indexing |
| **Default** | No explicit label set |
| **Unspecified** | Lowest possible priority, missing QoS info |

> ⚠️ **Critical point:** GCD does not prevent race conditions. It schedules when your work runs. What happens to shared data while that work runs is entirely your problem.

---

## Failure mode 1: Priority inversion

When a high-priority task ends up stuck waiting on a low-priority task that holds a lock — and a medium-priority task keeps getting the CPU in between, preventing the low-priority task from ever finishing.

> 🚗 **Analogy:** A CEO (high priority) is waiting for an intern (low priority) to finish using the only conference room. While the CEO waits in the hallway, a manager (medium priority) sees the CEO waiting and schedules back-to-back meetings in other rooms, blocking the intern from finishing. The CEO ends up waiting longer than a random employee would have. That's inversion.

**How macOS fixes this:** Darwin uses *priority inheritance* — when a high-priority thread is blocked on a low-priority thread, the kernel temporarily boosts the low-priority thread's priority to match the waiter. The intern gets treated like a CEO until they release the room.

The Xcode runtime warning: *"Thread running at QOS_CLASS_USER_INTERACTIVE waiting on a lower QoS thread running at QOS_CLASS_DEFAULT — investigate ways to avoid priority inversions."*

---

## Failure mode 2: Deadlock

Two things waiting on each other forever. In GCD, the classic case is calling `dispatch_sync` on a serial queue *from within that same queue*. The queue is blocked waiting for the sync call. The sync call needs the queue to be free. Nobody moves. App freezes.

> 🚪 **Analogy:** You're inside a room and you call out "I'll leave as soon as someone else enters." Someone outside says "I'll enter as soon as you leave." Neither of you moves. In the GCD version, the "room" is a serial queue and both parties are the same queue — it's trying to wait on itself.

### The circular wait — the full collapse

1. Main thread queues tasks to background threads
2. Background threads block, waiting for something from the main thread
3. Main thread makes a synchronous call waiting for the background threads
4. Main waits for workers, workers wait for main → app freezes

**Fix:** Don't use `dispatch_sync` to call back into a queue from within itself. Use async calls instead, or restructure so queues don't depend on each other in a circle.

---

## Failure mode 3: Thread pool starvation

GCD has a fixed pool of worker threads. If you flood it with too many tasks at once — especially ones that block waiting for something — all the workers get tied up. New tasks can't start. The whole system slows or freezes.

> 🍕 **Analogy:** A pizza shop has 10 delivery drivers (the thread pool). Someone orders 50 pizzas at once, sending all 10 drivers out on long deliveries. A regular order comes in — but there are no drivers left. That order sits waiting even though it has nothing to do with the massive order. The shop is starved of capacity.

This is both a reliability problem and a security problem — a saturated thread pool can cause denial-of-service and make other attacks easier.

---

## Failure mode 4: Sandbox escape via GSSCred (CVE-2018-4331)

**What GSSCred is:** A privileged macOS background daemon that stores and manages authentication credentials (like Kerberos tickets used in enterprise environments). Because it handles credentials, it runs with elevated privileges and accepts messages from other apps via XPC.

**What the attacker did:** A sandboxed app — locked in a security cage with very limited permissions — could send XPC messages to GSSCred. GSSCred had a race condition in how it handled those messages. By sending requests very rapidly, the attacker could win the race: slip something through in the gap between when GSSCred checked whether to trust the request and when it actually acted on it.

> 🏦 **Analogy:** Imagine a bank vault (GSSCred) with a security guard (the trust check). A visitor (the sandboxed app) walks up. The guard looks at their ID, turns around to buzz open the door — and in that split second, the visitor swaps their visitor badge for an employee badge. The door opens based on what the guard saw a moment ago. The visitor is now inside with employee access. That gap between "check" and "act" is the TOCTOU window.

**Why this was a GCD problem specifically:** GSSCred used GCD queues to handle incoming XPC requests, but the queue configuration was wrong — it didn't properly serialize or protect the state being checked. With the right timing, an attacker could flood requests and land one in the exact window where state had been read but not yet protected.

> GCD misuse isn't just a performance bug. In a privileged service, it can be a full sandbox escape — letting an untrusted app reach system-level credentials.

---

## Detection: how you spot these problems

### In code review — things to flag
- Any use of `dispatch_sync` — requires justification
- Shared variables or data that multiple queues can touch without a lock
- XPC services that don't explicitly set which queue handles their connections
- Code that assumes serial behavior but runs on a concurrent queue

### In the wild — behavioral signals
- Rapid-fire XPC messages from a sandboxed process to a privileged daemon — a classic race attempt
- Anomalous credential access patterns — a low-privilege app suddenly touching things it shouldn't reach
- Priority inversion warnings in Xcode at runtime — enable **Thread Performance Checker** in scheme settings

---

## Why should a security student care about any of this?

Race conditions are one of the oldest and most underappreciated vulnerability classes. They show up everywhere — operating systems, web apps, smart contracts (the Ethereum DAO hack was a race condition), file systems, authentication flows. They're especially dangerous in privileged code because the blast radius is so much larger.

**For OSINT / threat intel work:** When you see a CVE that mentions "race condition" or "TOCTOU" in a privileged daemon or system service, you now know exactly what that means: a timing window between check and action that an attacker exploited. That framing helps you evaluate severity, understand the attack surface, and read technical writeups without getting lost.

- Race condition ≠ broken code. It's code that works correctly most of the time, but fails under specific timing that an attacker can engineer.
- GCD misuse = a reliability bug in normal apps, but a security vulnerability in privileged services.
- The TOCTOU pattern is everywhere — file systems, auth flows, APIs, smart contracts. Recognizing it is a core skill.
- Behavioral telemetry (rapid XPC calls, anomalous credential access) is how defenders detect race exploitation in the wild — not just in the binary.