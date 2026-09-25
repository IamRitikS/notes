# Deep Dive: Building a Good Endpoint Security Extension on macOS

*From "log every exec" to "block writes to protected directories with full process ancestry" — what it takes to build an Endpoint Security extension that the kernel will not kill.*

---

## Why this is hard

Endpoint Security (ES) is Apple's replacement for kernel authorization hooks. It lets a user-space System Extension subscribe to process, file, and system events, and for a subset of them, *authorize* the operation before the kernel lets it proceed. It is the foundation of every EDR, DLP, and anti-tamper product on the platform.

It is also unforgiving in a very specific way. Your code runs while a syscall in some other process is suspended. If you are slow, the machine is slow. If you miss a deadline, the kernel does not wait; it drops your client and every decision you were supposed to make becomes "allow". The entitlement to do any of this is restricted and has to be requested from Apple, the extension needs Full Disk Access granted by the user or MDM, and the whole thing runs as root in a process you do not launch.

This article documents how I built one, in two deliberate iterations:

1. **A skeleton that proves the plumbing**: entitlement, packaging, activation, a client that subscribes to two harmless notify events and logs them, and a clean shutdown path driven over IPC.
2. **A real extension on top of that skeleton**: a process map with ancestry and signing information, a health checker for pid reuse, and an authorization handler that denies file operations inside protected directories without ever making the kernel wait on a log line.

Doing it in that order was the single best decision in the project. The plumbing has more ways to fail than the logic, and you want to discover them before there is any logic to blame.

---

## 1. Packaging and activation: getting a process to exist

An ES extension is a System Extension bundle embedded inside a host application at `Contents/Library/SystemExtensions/`. Three pieces of configuration make it one:

**Entitlements.** The extension binary needs `com.apple.developer.endpoint-security.client`. This is a *restricted* entitlement: you apply for it, Apple grants it to a specific Team ID, and your provisioning profile has to carry it. Without it `es_new_client` returns `ES_NEW_CLIENT_RESULT_ERR_NOT_ENTITLED` and nothing else you do matters.

**Info.plist.** The extension declares itself with `NSExtensionPointIdentifier = com.apple.security.endpoint-security.extension`, and provides an `NSSystemExtensionUsageDescription` that the OS shows to the user during approval.

**Activation.** The host app (which must live in `/Applications`) submits an `OSSystemExtensionRequest.activationRequest` for the extension's bundle identifier. The first time, the user has to approve it in System Settings, or an MDM profile has to pre-approve the Team ID. Deactivation is the mirror request. I wrapped activation, deactivation and a status check behind one small manager per extension, and wired the host app's existing install/uninstall command path to activate or deactivate based on current status, waiting on a dispatch group so the installer does not exit before the request completes.

Then there is the fourth requirement nobody's checklist mentions: **Full Disk Access**. An ES client must have the TCC Full Disk Access grant or `es_new_client` returns `ERR_NOT_PERMITTED`. On a managed fleet this comes from a PPPC profile keyed to the extension's signing requirement; on a developer machine you grant it by hand to the *extension*, not the host app, and forget you did it, and then spend an hour on a fresh machine wondering why the client will not create.

The result codes from `es_new_client` are your first diagnostic surface. Log them by name:

| Result | Meaning |
|---|---|
| `ERR_NOT_ENTITLED` | Missing or unprovisioned entitlement |
| `ERR_NOT_PERMITTED` | No Full Disk Access |
| `ERR_NOT_PRIVILEGED` | Not running as root |
| `ERR_TOO_MANY_CLIENTS` | You have leaked clients across restarts |
| `ERR_INVALID_ARGUMENT` | Usually a nil handler |

---

## 2. The extension is just a process

The extension has a `main`. It is not a framework callback; it is a program:

```swift
let controller = EndpointProtectionController()
controller.startMonitoring()

signal(SIGTERM) { _ in controller.stopMonitoringAndExit() }

// ... establish IPC (below) ...

dispatchMain()
```

`dispatchMain()` parks the main thread forever; all real work happens on ES's own threads and on Swift concurrency. Two design decisions here:

**Shutdown is a first-class path.** `stopMonitoring` unsubscribes everything, deletes the client, and only then stops any background tasks. If you `exit()` with a live client you will eventually hit `ERR_TOO_MANY_CLIENTS` on relaunch. `SIGTERM` is handled because that is how the OS asks nicely before it stops asking.

**Stop is also reachable over IPC.** The extension exports a one-method protocol (`stopMonitoring(withReply:)`) to the privileged daemon. The daemon can tell the extension to stand down without deactivating it, which matters for policy toggles and for uninstall ordering. Note the order inside the handler:

```swift
func stopMonitoring(withReply reply: @escaping () -> Void) {
    reply()                    // acknowledge first
    controller.stopMonitoring() // then tear down
}
```

Reply first, then act. If teardown hangs, the caller has already been told "acknowledged" and can proceed to its own timeout logic; it is not stuck waiting on us.

### IPC direction

The extension *dials out* to the daemon's existing privileged Mach service and exports its object on that connection. The daemon accepts, validates the peer's code signature against a Team ID requirement and an explicit allowlist of bundle identifiers, and stores the connection keyed by the peer's bundle ID. When the daemon later wants to call the extension, it looks up that stored connection and uses its remote proxy.

Why dial out instead of having the extension host its own Mach service? Because the daemon already had a hub that every other component connects to, with signature validation and lifecycle handling built in. Adding one more allowed bundle identifier to that hub was a two-line change; standing up a second listener would have been a second thing to secure. The general principle: **one validated IPC hub, every component connects to it, and the hub owns the "who is allowed to talk to whom" table.**

---

## 3. Iteration one: subscribe and log

The first version of the controller did exactly this:

```swift
es_new_client(&client) { client, message in
    switch message.pointee.event_type {
    case ES_EVENT_TYPE_NOTIFY_EXEC:
        // log the target executable path
    case ES_EVENT_TYPE_NOTIFY_OPEN:
        // log the file path
    default: break
    }
}
es_subscribe(client, [ES_EVENT_TYPE_NOTIFY_EXEC, ES_EVENT_TYPE_NOTIFY_OPEN], 2)
```

Two notify events. No state. No authorization. It looks trivial, and it validated: the entitlement is provisioned, FDA is granted, activation works, the daemon can see the extension, stop over IPC works, `SIGTERM` is handled, the client is created and subscribed, and events arrive. Every one of those had at least one thing wrong the first time.

It also taught the two rules that everything else is built on:

**The handler runs on an ES-owned thread, concurrently.** Do not assume serialisation. Do not touch shared mutable state without a lock or an actor.

**The message is only valid for the duration of the callback.** Every string, every path, every audit token you want to keep must be copied *synchronously* inside the handler. If you hand the raw pointer to an async task, you are reading freed memory. (`es_retain_message` exists for the cases where you genuinely must defer, but you should treat it as the exception.)

---

## 4. Iteration two, part one: the process map

The moment you want to say anything useful about an event — "*who* is doing this, and who launched *them*?" — you discover that ES tells you about the acting process (pid, audit token, executable, signing information) but not about its history. If the process was started before your extension, or if you want the ancestry chain up to the login session, you have to have been watching.

So the extension maintains a **process map**: pid → record, populated from `NOTIFY_EXEC`, `NOTIFY_FORK` and `NOTIFY_EXIT`.

### What to capture, and when

At exec time you have access to things that will not be available later: the binary may be deleted, the signing information may change on update, the arguments exist only in the process's memory. Capture them now:

```swift
struct ProcessRecord {
    let pid: pid_t, uid: uid_t, ppid: pid_t
    let path: String
    let arguments: [String]          // from es_exec_arg_count / es_exec_arg
    let cdHash: String               // hex of es_process_t.cdhash
    let signingID: String            // es_process_t.signing_id
    let csFlags: UInt32              // codesigning_flags
    let isPlatformBinary: Bool
    let timestamp: Date
}
```

Use `audit_token_to_pid` and `audit_token_to_euid` on the audit token rather than reaching into its raw fields. The token layout is documented but the accessors are the contract.

Fork and exec are different events and want different handling. **Fork** creates a child that is still running the parent's image: same path, no new arguments; record it so the ppid chain is unbroken. **Exec** replaces the image: new path, new arguments, new signing information. A shell that forks then execs produces both events for the same pid, in that order, and the exec record should overwrite the fork record.

### Extraction is synchronous; mutation is asynchronous

The handler copies everything it needs out of the message, then hands a value type to an actor:

```swift
case ES_EVENT_TYPE_NOTIFY_EXEC:
    let record = ProcessRecord(fromExec: message)  // sync, copies everything
    Task { await processMap.insert(record) }        // async, actor-isolated
```

The actor is the only thing that touches the dictionary. The ES thread returns immediately. There are no locks in the notify path at all.

### Seed the map at startup

Everything running before the extension started is invisible to `NOTIFY_EXEC`. So at startup, on a detached task (never on the ES thread), enumerate live pids with `proc_listallpids`, and for each one fetch parent pid and uid via `proc_pidinfo(PROC_PIDTBSDINFO)` and the path via `proc_pidpath`. Insert the whole batch into the actor in one hop.

Deliberately, the snapshot does *not* fetch arguments or signing information. Doing so per-process at startup is expensive, and it is a startup cost paid on every machine. Those fields stay empty for snapshot entries; the ancestry walk still works because `pid` and `ppid` are always populated. Log how many of the enumerated pids you successfully captured, because processes you cannot query (permissions, or they exited during enumeration) are silently skipped and you want to know the ratio.

### Ancestry with a cycle guard

The reason for all of this is one query: given a pid, walk `ppid` links through the map until you fall off it (a process older than the map) or hit pid 1.

```swift
func ancestorChain(from pid: pid_t) -> [pid_t] {
    var chain: [pid_t] = [], current = pid, seen = Set<pid_t>()
    while let rec = processes[current], !seen.contains(current) {
        chain.append(current); seen.insert(current); current = rec.ppid
    }
    return chain
}
```

The `seen` set is not paranoia. Pid reuse plus a stale entry can produce a cycle, and an infinite loop inside an actor is a hang you will not enjoy diagnosing.

### Pid reuse and the health checker

`NOTIFY_EXIT` is best-effort from your point of view: if the extension restarts, or an event is dropped under load, the map accumulates entries for processes that are gone. Worse, the pid gets reused, and now the map confidently reports the *wrong* process for that pid.

A periodic health checker fixes this in three steps that are carefully split across the actor boundary:

1. **On the actor:** take a lightweight snapshot of `(pid, path)` pairs. No syscalls.
2. **Off the actor:** for each pid, call `proc_pidpath`. If it fails, the process is gone. If it succeeds and the path differs from the recorded one, the pid has been reused. (Snapshot entries with an empty recorded path are treated as unverifiable rather than stale.)
3. **On the actor:** evict the confirmed-stale set.

The point of the split is that the syscalls run with no actor hop held. A few thousand `proc_pidpath` calls every few seconds is cheap; a few thousand `proc_pidpath` calls *while every ES callback is queued behind you* is not.

---

## 5. Iteration two, part two: authorization without stalling the machine

This is the part that makes an ES extension different from a fancy log source. `AUTH_*` events suspend the calling syscall until you respond with `es_respond_auth_result`. Every rule about the notify path applies twice as hard.

### Choose your AUTH events for cost, not coverage

The goal was a **directory guard**: deny any file operation whose path falls under a configured set of protected directories. The instinct is to subscribe to `AUTH_OPEN`. Do not. `AUTH_OPEN` fires for *every* file open on the system, thousands per second during normal use, each one a blocked syscall waiting on your process. Even a fast handler makes the machine feel wrong; a slow one makes it unusable.

Instead, subscribe to the events that *change* the filesystem:

```
AUTH_CREATE  AUTH_UNLINK  AUTH_RENAME  AUTH_TRUNCATE
AUTH_CLONE   AUTH_COPYFILE  AUTH_LINK  AUTH_EXCHANGEDATA
```

These are orders of magnitude rarer than opens, and together they cover every way a file inside a directory can be created, destroyed, replaced, or moved. Keep `NOTIFY_OPEN` if you want visibility into reads; it costs nothing because nobody waits on it.

### Extract every path an event touches

Most of the bugs in a directory guard live in path extraction, because the ES message layout differs per event and several events name *two* files:

- `TRUNCATE`, `UNLINK`, `CLONE`: one target path.
- `CREATE`: the destination is either an existing file (a path) or a new path (a directory *plus* a filename token you must join yourself). Check `destination_type`.
- `RENAME`: source path, and a destination that has the same existing-vs-new split as `CREATE`.
- `LINK`, `COPYFILE`: source path, plus a target directory and a target filename to join.
- `EXCHANGEDATA`: two full paths.

The guard evaluates *all* paths from an event against the blocklist and denies if any of them match. If you check only the destination of a rename, an attacker moves the protected file *out* of the directory and edits it there. If you check only the source, they overwrite it by renaming a file *in*.

### The decision is synchronous and lock-light

The blocklist can change at runtime (policy pushed from the daemon), so it lives behind a lock. But the lock is held only long enough to copy the array:

```swift
lock.lock(); let snapshot = blocklist; lock.unlock()
for path in paths { for dir in snapshot where path.hasPrefix(dir) { return path } }
return nil
```

No lock is held during comparison. The comparison itself is a prefix match on a handful of strings. There is no allocation beyond the path strings you already had to copy out of the message. This whole function is what runs while the kernel waits.

### Respond first. Then, and only then, do anything else.

```swift
if let blocked = fileGuard.blockedPath(in: message) {
    es_respond_auth_result(client, message, ES_AUTH_RESULT_DENY, false)
    let pid = audit_token_to_pid(message.pointee.process.pointee.audit_token)
    let eventName = message.pointee.event_type.shortName
    Task {
        let description = await processMap.blockEventDescription(pid: pid, path: blocked, event: eventName)
        logger.info("\(description, privacy: .public)")
    }
} else {
    es_respond_auth_result(client, message, ES_AUTH_RESULT_ALLOW, false)
}
```

The deny goes back to the kernel before the extension has done anything except a prefix match. The rich log line — who did it, their arguments, their signing ID, their full ancestor chain — is assembled afterwards, on the actor, from data that was captured at exec time and from scalars copied out of the message *before* the handler returned. The message pointer itself is never touched inside the task.

This ordering is the whole article in miniature. Every ES message carries a deadline. If you have not responded by then, ES kills your client, and a dead client means every AUTH event you subscribed to is now silently allowed. A logging call that blocks on a slow disk is enough to get you there. **Nothing that can block goes before `es_respond_auth_result`.**

### About the cache flag

The last parameter to `es_respond_auth_result` asks ES to cache the verdict so future events for the same file skip your handler. It is tempting for performance. The guard passes `false`, because the blocklist is dynamic: a cached `ALLOW` for a file that is later added to a protected directory is a hole you cannot see. If you do use caching, you must call `es_clear_cache` on every policy change, and you must reason about what "the same file" means to ES (it is per-vnode, not per-path).

---

## 6. Things I would add before shipping

The two iterations produce a working, well-behaved extension. These are the gaps between "working" and "product":

**Mute yourself.** Use `es_mute_process` (or the newer `es_mute_process_events`) on your own daemon, host app, and updater. Otherwise your own updater writing into your own install directory gets denied by your own guard, and your own log writes generate `NOTIFY_OPEN` events that you then log. Exempt by audit token or signing ID, never by path.

**Mute the noise.** `es_mute_path` for paths you will never care about (your own log directory, system caches) cuts event volume before it reaches you. Fewer events is the cheapest performance optimisation available.

**Respect the deadline explicitly.** Read `message.pointee.deadline` and treat it as a budget. If your decision requires anything you cannot do inside it, the answer is a fast default plus an async investigation, not a slower decision.

**Handle `NOTIFY_EXEC` for yourself.** When the daemon restarts, the extension should re-establish IPC. When the extension restarts, the map is empty until the snapshot completes; decide what your guard does in that window and make it explicit.

**Persist nothing you cannot rebuild.** The process map is rebuilt from the snapshot plus live events. That is a feature. A persisted map survives restarts with stale pids in it.

**Version the protocol.** The one-method IPC protocol will grow (push blocklist, query map, fetch a dump). Put a version handshake in before the second method, not after the fifth.

---

## 7. The checklist

1. **Prove the plumbing before writing logic.** Entitlement, FDA, activation, IPC, shutdown, `SIGTERM`. Two notify events and a log line is enough to validate all of it.
2. **The message dies when the callback returns.** Copy synchronously, process asynchronously.
3. **Handlers run concurrently on ES threads.** No unprotected shared state; an actor for anything stateful.
4. **Respond before you log.** `es_respond_auth_result` first; everything else after. A dead client fails open.
5. **Subscribe to AUTH events by cost.** Mutation events, not `AUTH_OPEN`. Keep observation on the notify side.
6. **Extract every path an event names.** Renames, links and copies have two ends; guard both.
7. **Capture signing info and arguments at exec time.** They may not exist later.
8. **Seed the map from libproc at startup, off the ES thread, without the expensive fields.**
9. **Assume pid reuse.** Validate periodically off-actor; evict on-actor; guard ancestry walks against cycles.
10. **Fork and exec are different.** Record both; let exec overwrite fork.
11. **Reply, then act, on IPC stop.** Never make the caller wait on your teardown.
12. **Mute your own processes** before you enforce on your own directories.
13. **Delete the client on every exit path.** `ERR_TOO_MANY_CLIENTS` is a self-inflicted wound.
14. **Never cache a verdict you cannot invalidate.**

Endpoint Security gives you a seat inside every syscall you ask about. The price is that you must always answer, and answer quickly. Build the extension so that the fast answer is the default and everything interesting happens after it, and the kernel will leave you alone.
