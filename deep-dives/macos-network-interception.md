# Deep Dive: Building Robust Network Interception and Packet Filtering Layers on macOS

*How a user-space agent ends up in the middle of every socket on the machine, why that goes wrong, and the design rules that stop it going wrong.*

---

## The business pain

If you build a VPN, a zero-trust access client, a DLP agent, or an endpoint firewall for macOS, you are building on top of Apple's Network Extension framework whether you like it or not. Kernel extensions are gone. Your code runs in a System Extension process that the OS starts, stops, suspends, and kills on its own schedule, inside a sandbox, under a completion-handler contract enforced with a timer.

The symptoms customers report are always the same three:

1. **"The Mac's network hung."** Sometimes for seconds, sometimes until reboot.
2. **"Your agent is eating CPU."**
3. **"Packets are dropping under load."**

None of these are exotic. They are the predictable consequences of getting a handful of design decisions wrong in a process that sits between every application and the network. This article is about those decisions: what a sound architecture looks like, where latency bottlenecks and deadlocks actually come from in this kind of software, and the rules that fall out of debugging them.

The examples are drawn from experience building this class of product, but they are deliberately generic. Every failure shape described here is one that any team shipping a Network Extension will meet.

---

## 1. Three interception primitives, three different contracts

macOS offers three ways to get between an application and the network. A serious agent usually ends up using all of them, and the first lesson is that they carry different performance and failure contracts.

| Primitive | What you receive | What you return | Cost of being slow |
|---|---|---|---|
| Content filter (`NEFilterDataProvider`) | Metadata about a new socket flow, optionally the first N bytes | A verdict: allow / drop / inspect more | Every new socket on the machine waits on you |
| App proxy / transparent proxy (`NEAppProxyProvider`, `NETransparentProxyProvider`) | A flow object you own; you read and write the bytes | "I'll handle it" or "let it through" | Every new flow matching your rules waits on `handleNewFlow`; per-flow throughput is your problem |
| Packet tunnel (`NEPacketTunnelProvider`, or a utun interface) | Raw IP packets from a virtual interface | Packets written back | You are the NIC. Every byte crosses your read loop |

The reason to run more than one is that each covers a hole in the others. A content filter can block a flow before the SYN leaves the machine. A transparent proxy sees the *application identity* (signing identifier, audit token) and the destination *hostname* of a flow, which a packet tunnel never sees. A packet tunnel can carry protocols that are not TCP or UDP. They are not interchangeable, and treating them as if they were is the root of a lot of "why is this slow" investigations.

---

## 2. Who talks to whom, and what must never cross that boundary

A typical enterprise agent decomposes into four kinds of process:

```
       user session                                root / system
+---------------------------+      IPC      +-------------------------------+
|  UI / login-session       |<------------->|  Privileged daemon (launchd)  |
|  process (auth, policy)   |               |  - lifecycle, kernel tunables |
+---------------------------+               |  - credential store, IPC hub  |
                                            +-------+---------------+-------+
                                                    | IPC           | IPC
                                                    |               |
                            +-----------------------+-----+   +-----+-----------------------+
                            |  Tunnel / proxy engine      |   |  Filtering System Extension |
                            |  - packet path or flow path |   |  - content filter           |
                            |  - encrypted transport      |   |  - transparent proxy        |
                            |    to the cloud             |   |  - packet tunnel provider   |
                            +-----------------------------+   +-----------------------------+
```

Every arrow is XPC over a Mach service. Every connection should be pinned to your Team ID with a code-signing requirement, the peer's audit token verified before the connection is trusted, and the first message on every connection should be a version handshake so that a half-upgraded machine (old extension, new daemon) fails loudly instead of subtly.

The single most important architectural rule, and the one that prevents most of the latency problems in the rest of this article:

> **IPC is control plane only. Application data never crosses it.**

Packets go through the tunnel interface. Flows go through Network Extension flow objects, or through a local socket pair if you need to bridge them into existing socket-based code. What crosses IPC is policy, credentials, state changes, and telemetry. If you find yourself serialising payload and sending it to another process for a decision, you have already lost the throughput argument.

---

## 3. The symptom: hangs, CPU, drops

### "Kernel panic" is now "the extension got killed"

In the kext era, a bug in your filter could take the machine down. In the System Extension era, the same bug gets you terminated by the OS's extension manager, and while you are dead, traffic either stalls (content filter) or bypasses you (proxy, depending on configuration). Users describe both as "the network hung".

The OS enforces two contracts you must never violate:

- **Call the completion handler.** Start, stop, sleep: each has a handler and a deadline. Miss the deadline and you are terminated, and you do not get a crash log you can act on.
- **Return from `handleNewFlow` promptly.** For the filter and the proxy alike, new-flow callbacks are effectively serialised. A slow one is not slow for one app; it is slow for every app.

Because the OS kills you without a dump, arm your own deadline on the stop path: schedule a timer inside the OS's patience window that, if teardown has not finished, requests a process dump from a *different* process (a wedged process cannot reliably write its own), waits briefly for it to land, then exits hard. That dump is the difference between a fixable bug and a mystery.

### CPU: one syscall per packet

A packet-path engine is a dedicated thread in a loop: `read()` one packet, hand it to a classifier, then either queue it for the encrypted transport, write it back into the interface, or bypass it out a raw socket. Every write back into the interface takes a lock, because several threads (the read loop, the transport's response path, health checks) can all want to write.

Two things dominate CPU here:

1. **Syscall count.** One `read()` and one `write()` per packet. At high packet rates the syscall overhead exceeds the crypto. The fix is batching: wait for readability, drain many packets per wake-up, and hand them downstream as a batch.
2. **Per-packet allocation.** Wrapping each packet in a heap-allocated, reference-counted object is convenient and shows up immediately in profiles. Pre-allocated buffers and batching fix it; neither is glamorous.

The write lock is also a latency source: a slow writer stalls the read loop's write-back path. Keep the critical section to the `write()` itself. Never log under it.

### Drops: file descriptors, not the network

A transparent proxy terminates every TCP connection on the machine. That is at least two file descriptors per connection, plus the transport's own sockets. On a developer laptop with a browser and a few Electron apps, thousands of concurrent flows is normal. The default per-process file limit is not.

So the privileged daemon should raise the kernel and launchd file-descriptor limits and the listen backlog at start. And it should *only raise, never lower*: writing your preferred values unconditionally will silently lower limits that an administrator or MDM profile already set higher. When a descriptor ceiling is hit, the symptom is not an error dialog; it is `accept()` failing quietly, and the user sees "packets dropping".

---

## 4. IPC without latency bottlenecks

The IPC layer is where most "randomly slow" reports come from. Rules worth treating as non-negotiable:

**Every call is asynchronous with a reply block, and every wait on a reply is bounded.** Avoid synchronous proxy objects entirely. Where a synchronous answer is genuinely required (a C++ engine asking the daemon a question mid-computation), wrap the async call in a semaphore with a hard timeout *and a defined fallback value*.

**Connection setup is guarded, versioned, and bounded — and understand what it serialises.** A sane client wrapper creates the connection, installs the code-signing requirement, resumes it, sends a version request, and waits a bounded time for the reply before declaring the connection usable, all under a lock so that ten callers racing at startup produce one connection rather than ten. The consequence is that while one caller is in the handshake, everyone else who needs that connection queues behind them. That is fine for the control plane. It is catastrophic if any data-path code ever touches it. The line between the two must be a hard architectural boundary, not a convention.

**Interruption and invalidation are different events.** *Interruption* means the peer died and launchd will bring it back: drop the connection and reconnect lazily on next use. *Invalidation* means the connection is permanently gone (entitlement, configuration, or you invalidated it yourself): do not retry blindly. In both handlers, reset state and return. Never reconnect *inside* the handler: it runs on the connection's queue, and a synchronous reconnect there serialises against the very teardown that triggered it.

**Never nest synchronous hops.** A common shape: the engine asks the daemon for credentials; the daemon, if a user is logged in, forwards to the UI process, which answers. If the daemon waits synchronously on the UI, a hung UI takes the data plane down with it. Bound the wait, and have a fallback that does not involve the UI at all (e.g. credentials cached in the keychain). Chains of synchronous IPC are how a hang in one process becomes a hang in all of them.

**Have a non-IPC fallback for every IPC dependency on the startup path.** Startup must complete even if every peer is missing, because on boot every peer *is* missing.

---

## 5. Flow redirection: the transparent proxy, two ways

`NETransparentProxyProvider` can be used in two very different modes, and each teaches something.

### Mode 1: the observer that must not observe slowly

One useful pattern is a transparent proxy that includes all outbound TCP and UDP and returns "not handled" from every `handleNewFlow`. It never takes a flow. It exists because a proxy gets the flow's signing identifier and audit token, and the packet tunnel does not. For applications that policy says should bypass the tunnel, the observer learns the destination and tells the tunnel engine to install a bypass route *before* the flow proceeds.

"Before" is the problem. The route must exist before the SYN goes out, so the observer waits on an IPC round trip inside `handleNewFlow`. That is a blocking wait on the most sensitive callback in the system. It is survivable only because of everything around it:

- The wait is bounded, and the timeout behaviour is defined (the flow proceeds unmodified; log it).
- A per-destination cache means the wait happens once per destination, not once per flow.
- DNS flows are excluded, because a resolver flow that waits even a few seconds is a system-wide stall (see below).
- Flows to any synthetic address ranges the agent itself hands out are short-circuited, so the packet path handles them without a round trip.
- The cache is invalidated when the tunnel restarts, so stale routes do not outlive it.

The general rule: anything blocking inside `handleNewFlow` is a *system-wide latency multiplier*. If you cannot remove it, make it once-per-key, bound it tightly, and define what happens on timeout.

### Mode 2: the whole engine inside a provider

The other mode takes flows for real. `handleNewFlow` consults a decision layer (include / drop / bypass) and hands included flows to a flow controller. The interesting engineering problem: most teams already have a mature, socket-based, event-driven proxy engine. Network Extension flows are not sockets. They are objects with `readDataWithCompletionHandler:` and `writeData:withCompletionHandler:`.

Rather than rewrite the engine, bridge: each accepted flow gets a **local socket pair**. The flow handler pumps bytes from the NE flow into one end; the engine sees the other end as an ordinary accepted client socket and does what it has always done (protocol detection, policy, tunnelling). A FIFO buffer absorbs the impedance mismatch between NE's completion-driven reads and the engine's readiness-driven writes.

This works, and it preserves years of hardened proxy logic. It also produces a list of edge cases that should be a checklist for anyone implementing an app proxy:

**Half-close is not close.** Read-side and write-side closes on a flow are independent, and the engine's shutdown signal must carry a mode (read / write / both) mapped onto `shutdown(SHUT_RD)` / `shutdown(SHUT_WR)` on the bridge. Treating either as a full close breaks HTTP long-polling and any protocol where the client sends FIN and then waits for the response.

**A zero-length read is EOF, not an error.** And EOF *before any bytes were sent* is a distinct event: the application opened a connection and gave up. Surface it as a health metric, because a spike in it is the earliest signal that you are accepting flows you cannot serve.

**Write completion is not backpressure.** `writeData:` returns immediately and the framework buffers on your behalf. A fast upstream and a slow application means unbounded memory growth inside the extension, and extensions have a memory ceiling. The correct design is a byte-credit window: stop reading from upstream when outstanding, uncompleted writes to the flow exceed a threshold; resume on completion. Fire-and-forget writes are the version everyone ships first and regrets.

**Weak self in every completion block, and idempotent close.** The OS holds references to flow objects longer than you expect. Completion handlers fire after your handler has decided it is done. Capture weakly, promote to strong inside the block, and make every close path tolerate being called twice.

**Re-arm reads from the completion handler, not from a loop.** The read path is recursive: read → on data, forward to the bridge → read again. There is no outer loop and no thread parked on a flow. Thousands of idle flows cost nothing.

### The DNS stall

This deserves its own heading because it is the most disproportionate failure in this domain: a single unanswered UDP datagram can freeze every application on the Mac.

On macOS, `mDNSResponder` is *the* resolver. Every hostname lookup in every process funnels through it. If your proxy includes UDP port 53 (and a full tunnel must, to carry DNS), then you are holding the system resolver's flows. If upstream never answers and you never answer, the resolver waits, and with it every app whose lookup is queued behind that query. The user sees the browser hang, then chat, then everything.

The rule: **a resolver flow always gets an answer.** If nothing comes back from upstream within a short deadline, synthesise a response (an empty answer or NXDOMAIN carrying the original transaction ID) and write it back so the resolver fails fast and moves on. Slow DNS is a bug; hung DNS is an outage. Also assume the resolver has cached state from the pre-tunnel network and plan to invalidate it when the tunnel comes up.

---

## 6. Deadlocks, by shape

Every one of these shapes produces tickets titled some variant of "network stopped working". None of them looks like a networking bug when you find it.

### Shape 1: main-queue re-entrancy across a framework boundary

A policy update arrives on the main queue. The handler synchronously calls a framework API whose result is delivered *on the main queue*. Nothing in either piece of code is wrong in isolation; together they never return.

Fix: dispatch the dependent work to a background queue. Rule: **which queue a callback is delivered on is part of that API's contract.** Write it down at the call site, and never block the main queue of a long-lived process waiting for anything.

### Shape 2: a lock held across a wait

A worker thread holds a mutex while waiting on an event with a timeout, in a loop. The stop path needs that mutex to set the stop flag. Stop cannot acquire the lock; the loop cannot see the stop flag. On shutdown the process hangs, the OS deadline expires, and you are killed without a dump.

Fix: wait *outside* the lock; take the lock only for the state mutation; on the stop path, acquire with a timeout so that if the invariant is violated again, stop degrades to "slow" rather than "never". Rule: **a wait and a lock in the same scope is a bug until proven otherwise.**

### Shape 3: the watchdog that could not die

A deadlock detector fires, captures a dump, and calls `exit()`. `exit()` runs `atexit` handlers and static destructors. Those destructors take locks. The locks are the ones that are deadlocked. The recovery mechanism hangs on the thing it is recovering from, and now the process is *provably* wedged with no watchdog left.

Fix: `kill(getpid(), SIGKILL)`. Not `exit(SIGKILL)`, which is just `exit(9)` with a misleading name. Rule: **once you have decided the process is wedged, execute no more of its code.** Let launchd or Network Extension restart you; that is what they are for.

### Shape 4: the watchdog with a false positive

macOS suspends background processes aggressively. During sleep, and during dark wake, a state machine simply stops progressing. A detector that counts consecutive iterations in which a *transient* state (connecting, authenticating) has not changed will see a laptop with its lid closed as a deadlock, and kill a healthy process every night.

Fix: the watchdog must know about sleep. Subscribe to sleep/wake notifications *and* ask another process for the current sleep state at startup, because a process launched during dark wake has missed the sleep notification that already happened. Rule: **sleep and wake are first-class states in every state machine that has a timer.**

### Shape 5: stop racing with a network change

A network-change notification fires while the engine is tearing down. Both paths touch the same route and interface objects. The result is usually a crash rather than a hang, but the cause is the same family: two lifecycles with no ordering between them. Fix: the stop flag is checked under the same lock that protects the objects, and change handlers early-out when a stop is in progress.

### Shape 6: join-with-timeout on a thread blocked in I/O

Stop paths that `join` worker threads with a timeout and, on failure, escalate to dump-and-die, discover that a thread blocked in a TLS read does not join on their schedule. Where the same stop path closes the thread's socket, the join is unnecessary: closing the socket wakes the thread, and process exit reaps it. Rule: **do not join threads you cannot wake.**

---

## 7. A watchdog design that survives contact

After all of the above, deadlock handling that actually holds up looks like this:

1. A monitoring task runs periodically and records whether the engine's state has changed. States that are legitimately long-lived are excluded: idle, forwarding, "connecting" with a recent retry timestamp, and any error state where policy says to hold rather than retry. Sleep suppresses counting entirely.
2. If the count of "unchanged while transient" iterations exceeds a deliberately high threshold, a deadlock is declared and a telemetry event is emitted.
3. Dumps are rate-limited by a minimum interval persisted across restarts, so a crash loop does not fill the disk.
4. If a dump is due: a detached thread pauses briefly (so the deadlock re-manifests after any transient state), requests a dump from a healthy peer process over IPC, waits for it to land, then `SIGKILL`s the process.
5. If a dump is not due: `SIGKILL` after a short pause.
6. Separately, a "startup has been running far longer than plausible while the machine is awake" check catches hangs in initialisation that the state-machine check would miss.

The daemon restarts the engine. Network Extension restarts the extension. Users see a few seconds of reconnect instead of a reboot.

---

## 8. The checklist

Compressed into what I would tell an engineer starting a Network Extension product tomorrow:

1. **Provider callbacks are the hot path.** No IPC, no unbounded waits, no locks shared with slow paths, no synchronous logging. `handleNewFlow` is a system-wide latency multiplier.
2. **IPC is control plane only.** Data goes through the tunnel interface, flow objects, or a local socket pair. Never through a serialised blob over XPC.
3. **Push state into the extension; read the cache on the verdict path.** Define the default for every state during the window before the daemon is up.
4. **Every wait is bounded, and every bound has a defined fallback.** "It timed out" must map to a specific behaviour, not to whatever the code happens to do.
5. **Always allow your own traffic**, identified by signing identifier and Team ID. A filter that blocks its own tunnel is a filter that never recovers.
6. **A resolver flow always gets an answer.** Synthesise failure rather than let the system resolver wait.
7. **Backpressure is your job.** The framework buffers writes; it does not stop you from overrunning memory.
8. **Half-close is real.** Map read and write shutdown independently.
9. **Sleep and wake are states.** Every timer-driven state machine must know about them, including watchdogs.
10. **A wait inside a lock is a bug.** Callback queue affinity is part of the API contract.
11. **Once wedged, run no more code.** `kill(getpid(), SIGKILL)`, not `exit()`.
12. **Arm your own deadline on the OS's stop path**, and capture a dump before the OS's deadline kills you without one.
13. **Only raise kernel limits, never lower them.** File descriptors, not bandwidth, are the first ceiling a transparent proxy hits.
14. **Test shutdown as hard as startup.** Every deadlock in this article lives in a stop path, a state transition, or a race with sleep. None lives in steady-state forwarding.

None of this is specific to any one product. It is specific to putting a user-space process in the middle of every connection on a machine whose operating system will not wait for you. Build for that, and the three symptoms at the top of this article become rare enough to be interesting.
