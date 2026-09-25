# Swift Notes

*A working reference for Swift as used in macOS agent / system-extension development, written from a C++ background.*

---

## Table of Contents

1. [Data Types](#1-data-types)
2. [Optionals](#2-optionals)
3. [Arrays](#3-arrays)
4. [Dictionaries](#4-dictionaries)
5. [Sets](#5-sets)
6. [Closures](#6-closures)
7. [Struct vs Class vs Actor vs Enum](#7-struct-vs-class-vs-actor-vs-enum)
8. [Value vs Reference Semantics](#8-value-vs-reference-semantics)
9. [Switch](#9-switch)
10. [Error Handling](#10-error-handling)
11. [Protocols](#11-protocols)
12. [Extensions](#12-extensions)
13. [Memory Management (ARC)](#13-memory-management-arc)
14. [Reference Types: Strong, Weak, Unowned](#14-reference-types-strong-weak-unowned)
15. [Swift vs C++](#15-swift-vs-c)
16. [Swift in macOS Agent Development](#16-swift-in-macos-agent-development)
17. [Functions](#17-functions)
18. [OpaquePointer](#18-opaquepointer)
19. [Tasks](#19-tasks)
20. [Equatable](#20-equatable)
21. [Hashable vs Codable](#21-hashable-vs-codable)
22. [Observer Pattern](#22-observer-pattern)
23. [NotificationCenter](#23-notificationcenter)
24. [Grand Central Dispatch (GCD)](#24-grand-central-dispatch-gcd)
25. [Further Reading](#further-reading)

---

## 1. Data Types

Swift has automatic type inference. Common types:

- `Int`
- `Double`
- `Float`
- `Bool`
- `String`

```swift
var age: Int = 30
let price = 19.5          // inferred as Double
var flag: Bool = true
```

---

## 2. Optionals

Swift avoids null-pointer crashes with **Optionals**: a type that explicitly says the value may be absent.

```swift
var name: String? = "Alice"
var x: Int? = nil
```

### Unwrapping Optionals

**Safe unwrapping (`if let`)**

```swift
if let value = name {
    print(value)
}
```

**Guard** — used often in production code; exits early if the value is missing.

```swift
guard let value = name else {
    return
}
```

**Force unwrap (dangerous)** — crashes if `nil`.

```swift
print(name!)
```

---

## 3. Arrays

```swift
var numbers = [1, 2, 3]
```

| Operation | Code | Notes |
|---|---|---|
| Add | `numbers.append(4)` | |
| Access | `numbers[0]` | |
| Remove | `numbers.removeFirst()` / `numbers.remove(at: 1)` | Both are O(n) — expensive |
| Iterate | `for num in numbers { print(num) }` | |

### Higher-order operations

In the examples below, `events` is an array of structs.

**Filter**

```swift
let suspicious = events.filter {
    $0.path.contains("secret")
}
```

**Map** — transform items.

```swift
let pids = events.map { $0.pid }
```

**CompactMap** — transform and drop `nil` results.

```swift
let names = events.compactMap { $0.processName }
```

**Contains**

```swift
events.contains { $0.pid == 123 }
```

**Sort**

```swift
events.sort { $0.timestamp < $1.timestamp }
```

**Prefix / Suffix** — useful for batching and windowing.

```swift
let latest = events.suffix(10)
```

**RemoveAll(where:)**

```swift
events.removeAll { $0.timestamp < cutoff }
```

---

## 4. Dictionaries

Equivalent to C++ `unordered_map`.

```swift
var dict: [String: Int] = [:]
dict["apple"] = 3
dict["banana", default: 0] += 1
```

**Lookup**

```swift
if let value = dict["apple"] {
    print(value)
}
```

**Remove**

```swift
dict.removeValue(forKey: "apple")
```

**Filter**

```swift
let active = dict.filter { $0.value > 100 }
```

---

## 5. Sets

Equivalent to C++ `unordered_set`.

```swift
var s: Set<Int> = [1, 2, 3]

s.insert(4)
s.contains(2)
s.remove(1)
```

---

## 6. Closures

Swift closures are the equivalent of C++ lambdas.

```swift
let square = { (x: Int) -> Int in
    return x * x
}
```

They are used constantly with collections, usually in shorthand form:

```swift
numbers.map { $0 * 2 }
```

---

## 7. Struct vs Class vs Actor vs Enum

Swift strongly prefers structs over classes.

### Struct

The default choice for modeling data. Structs are **copied** when passed around, making them efficient and safe from unintended side effects.

```swift
struct Point {
    var x: Int
    var y: Int
}
```

### Class

Use when you need **shared mutable state** or **inheritance**. Variables point to a single instance in memory, so multiple variables can update the same object.

```swift
class Person {
    var name: String
}
```

### Actor

A reference type introduced in Swift 5.5 to eliminate data races. Only one task can access an actor's mutable state at a time.

**The problem: a data race with a class**

```swift
class ClassCounter {
    var value = 0

    func increment() {
        value += 1   // Danger: multiple threads can hit this at once
    }
}

let classCounter = ClassCounter()

// Simulating concurrent access from background tasks
Task { classCounter.increment() }
Task { classCounter.increment() }
```

**The solution: an actor**

Actors look like classes but automatically force tasks to wait their turn.

```swift
actor ActorCounter {
    var value = 0

    func increment() {
        value += 1   // Safe: Swift ensures only one task modifies this at a time
    }
}
```

**Usage** — because actors isolate their data, you must `await` when calling into them from outside. This signals the code may need to pause and wait for its turn.

```swift
func testActor() async {
    let counter = ActorCounter()

    // Modify data safely across multiple tasks
    await counter.increment()

    // Reading also requires await
    print("Count is: \(await counter.value)")
}
```

### Enum

Defines a common type for a group of related values. Unlike many languages, Swift enums can have **methods** and **associated values**, letting each case carry additional data.

```swift
enum Decision {
    case allow
    case block
}

var d: Decision = .allow
```

### Comparison

| Feature | Struct | Class | Actor | Enum |
|---|---|---|---|---|
| Memory | Stack | Heap | Heap | Stack |
| Inheritance | No | Yes | No | No |
| Copy semantics | Value | Reference | Reference | Value |
| Concurrency | Thread-safe if local | Not safe by default | Built-in isolation | Thread-safe if local |
| Identity | No (compared by value) | Yes (compared by instance) | Yes (compared by instance) | No (compared by value) |

---

## 8. Value vs Reference Semantics

Structs use **value semantics**.

```swift
struct A {
    var x: Int
}

var a = A(x: 5)
var b = a
b.x = 10
```

After this:

- `a.x == 5`
- `b.x == 10`

A class behaves differently: `a` and `b` would share the same instance, so `a.x` would also be `10`.

---

## 9. Switch

A Swift `switch` must be **exhaustive**.

```swift
switch d {
case .allow:
    print("allowed")
case .block:
    print("blocked")
}
```

---

## 10. Error Handling

Swift uses `throws` / `try` / `catch`.

```swift
func readFile() throws {
    // ...
}
```

**Calling a throwing function:**

```swift
do {
    try readFile()
} catch {
    print(error)
}
```

---

## 11. Protocols

Protocols define behavior, similar to interfaces.

```swift
protocol Logger {
    func log(message: String)
}
```

**Conforming to a protocol:**

```swift
class FileLogger: Logger {
    func log(message: String) {
        print(message)
    }
}
```

---

## 12. Extensions

Add functionality to existing types, including ones you don't own.

```swift
extension String {
    func shout() -> String {
        return self.uppercased()
    }
}
```

---

## 13. Memory Management (ARC)

Swift uses **Automatic Reference Counting**. Every class instance has a reference count.

- **Increment (+1):** a new reference to the instance is created.
- **Decrement (−1):** a reference is set to `nil` or goes out of scope.
- **Deallocation:** when the count hits zero, ARC immediately frees the memory.

```swift
class Person {}

var p1: Person? = Person()
var p2 = p1
// Reference count = 2
```

When all references drop, the count hits zero and the memory is freed.

---

## 14. Reference Types: Strong, Weak, Unowned

| Kind | Affects ref count? | Optional? | Use when |
|---|---|---|---|
| **Strong** | Yes (+1) | No | The default. Most parent → child relationships. |
| **Weak** | No | Must be optional | Breaking cycles where one object can disappear before the other. |
| **Unowned** | No | Non-optional | Two objects share the same lifetime; assumes the instance is never `nil` while referenced. |

```swift
weak var parent: Node?
```

Weak references are the standard tool for avoiding **retain cycles**.

---

## 15. Swift vs C++

| Concept | Swift | C++ |
|---|---|---|
| Null safety | Optionals | Raw pointers |
| Memory management | ARC | Manual / RAII |
| Collections | `Array` / `Dictionary` / `Set` | `vector` / `map` / `set` |
| Error handling | `try` / `catch` (typed `throws`) | Exceptions |

---

## 16. Swift in macOS Agent Development

In macOS system extensions you commonly see:

- `ESClient` (Endpoint Security)
- `DispatchQueue`
- XPC communication

Swift is used mainly for:

- Glue logic
- Event handling
- System extension code

Core performance-critical parts may still be written in C++.

---

## 17. Functions

```swift
func add(a: Int, b: Int) -> Int {
    return a + b
}

add(a: 5, b: 3)
```

Note: Swift uses **named parameters** at the call site by default.

---

## 18. OpaquePointer

`OpaquePointer` represents a C pointer to a type that cannot be directly represented in Swift.

**Key characteristics:**

- **Encapsulation:** points to data whose internal structure (fields, size) is unknown to the Swift compiler.
- **C interoperability:** when a C header declares a pointer to an incomplete struct (e.g. `struct Database;`), Swift imports it as `OpaquePointer` rather than `UnsafePointer<T>`.
- **Type safety:** because the type is opaque, you cannot access properties or do pointer arithmetic without first casting to a typed pointer.

**Example (Endpoint Security client):**

```swift
var client: OpaquePointer?

let res = es_new_client(&client) { (client, message) in
    // Process the received message
}
```

---

## 19. Tasks

A `Task` is the fundamental unit of asynchronous work. It provides the execution context for `async`/`await` code and lets operations run concurrently.

### The three kinds of Task

**Unstructured — `Task { }`**

- **Inheritance:** inherits priority, actor context, and task-local values from where it was created.
- **Lifespan:** outlives the scope that created it unless manually managed.

**Detached — `Task.detached { }`**

- **Inheritance:** inherits nothing; runs fully independent of the spawning context.
- **Use case:** heavy CPU work (e.g. image processing) that must be isolated from the Main Actor.

**Child — `async let` or `TaskGroup`**

- **Context:** created automatically inside structured concurrency.
- **Lifespan:** strictly scoped; the parent must wait for children to finish before it can exit.

### Quirks you must know

**1. Actor context inheritance (the MainActor surprise)**

Creating a plain `Task { }` inside a `@MainActor` class (SwiftUI View, UIKit ViewController) runs the task on the Main Actor by default.

- *Quirk:* a heavy CPU loop inside that Task freezes the UI.
- *Fix:* use `Task.detached`, or move the heavy work to a non-isolated function or background actor.

**2. Cooperative cancellation is NOT automatic**

`task.cancel()` does not stop execution mid-flight. It only flips a hidden `isCancelled` flag.

- *Quirk:* a long `while` loop keeps running to completion after cancellation, wasting resources.
- *Fix:* check `Task.isCancelled` inside long loops, or call `try Task.checkCancellation()` to throw on cancellation.

**3. Priorities are hints, not guarantees**

You can assign `.high`, `.medium`, `.low`, or `.background`.

- *Quirk:* Swift uses **priority inversion avoidance**. If a `.background` task holds a lock or data a `.high` task needs, Swift temporarily boosts the background task to clear the bottleneck.

**4. Implicit strong references (retain cycles) inside `Task { }`**

Referencing `self` inside a Task creates a strong reference that lasts until the task finishes.

- *Quirk:* if a user dismisses a screen while a long network call runs in a Task, the screen stays alive in memory until the call completes or times out.
- *Fix:* capture `[weak self]` if the object should deallocate immediately on dismissal.

### Example

```swift
func performWork() {
    // Inherits calling context (e.g. Main Actor if called from UI)
    let myTask = Task(priority: .high) {
        for i in 1...1000 {
            // Cooperatively check for cancellation
            if Task.isCancelled { break }

            print("Processing item \(i)")
        }
    }

    // Stop the task later if needed
    myTask.cancel()
}
```

---

## 20. Equatable

[`Equatable`](https://developer.apple.com/documentation/Swift/Equatable) allows instances of a type to be compared with `==` and `!=`.

### How it works

- **Synthesized conformance:** for most structs and enums, Swift auto-generates equality if all stored properties are `Equatable`.
- **Manual implementation:** for custom logic (e.g. comparing only some fields) or for classes, implement `static func ==` yourself.
- **Standard library:** `String`, `Int`, `Double`, etc. already conform.

### Basic example

```swift
struct User: Equatable {
    let id: Int
    let name: String
}

let user1 = User(id: 1, name: "Alice")
let user2 = User(id: 1, name: "Alice")

print(user1 == user2)   // true
```

### Custom logic

```swift
struct Product: Equatable {
    let sku: String
    let price: Double

    // Only consider products equal if their SKUs match
    static func == (lhs: Product, rhs: Product) -> Bool {
        return lhs.sku == rhs.sku
    }
}
```

### Why use it?

- **Collection operations:** `.contains(_:)` and friends require `Equatable` elements.
- **Foundation for other protocols:** prerequisite for [`Hashable`](https://developer.apple.com/documentation/Swift/Hashable) (needed for `Set` and `Dictionary` keys) and `Comparable` (needed for sorting).
- **SwiftUI performance:** the [`equatable()`](https://developer.apple.com/documentation/swiftui/view/equatable()) view modifier prevents unnecessary redraws when data hasn't changed.

---

## 21. Hashable vs Codable

Both are protocols that govern how data is compared, stored, and converted.

### Hashable

[`Hashable`](https://developer.apple.com/documentation/Swift/Hashable) lets a type be reduced to an integer hash value.

- **Purpose:** use custom types as `Dictionary` keys or `Set` elements.
- **How it works:** the hash gives O(1) lookup instead of a linear scan.
- **Relationship to Equatable:** `Hashable` inherits from `Equatable`. If two values are `==`, they must hash identically.
- **Automatic conformance:** synthesized for structs/enums when all stored properties are `Hashable`.

```swift
// Synthesized: String and Int are already Hashable
struct Player: Hashable {
    let name: String
    let jerseyNumber: Int
}

// Set removes duplicates automatically
var teamRoster: Set<Player> = [
    Player(name: "Alice", jerseyNumber: 10),
    Player(name: "Bob",   jerseyNumber: 7),
    Player(name: "Alice", jerseyNumber: 10)   // duplicate
]
print(teamRoster.count)   // 2

// Dictionary key
var playerScores: [Player: Int] = [:]
let playerAlice = Player(name: "Alice", jerseyNumber: 10)

playerScores[playerAlice] = 25
print(playerScores[playerAlice] ?? 0)   // 25
```

### Codable

`Codable` is a type alias for `Encodable & Decodable`.

- **Purpose:** convert data to/from external formats like JSON or Property Lists.
- **Encodable:** sending data to a server or saving to a file.
- **Decodable:** receiving data from an API or reading a saved file.
- **Ease of use:** add `: Codable` and `JSONEncoder` / `JSONDecoder` handle the rest.

```swift
import Foundation

struct Product: Codable {
    let id: Int
    let name: String
    let price: Double
}

// --- Encoding (Swift object -> JSON) ---
let laptop = Product(id: 101, name: "MacBook Air", price: 999.99)
let encoder = JSONEncoder()
encoder.outputFormatting = .prettyPrinted

if let jsonData = try? encoder.encode(laptop),
   let jsonString = String(data: jsonData, encoding: .utf8) {
    print("Encoded JSON:\n\(jsonString)")
}
/*
Encoded JSON:
{
  "id" : 101,
  "name" : "MacBook Air",
  "price" : 999.99
}
*/

// --- Decoding (JSON -> Swift object) ---
let incomingJSON = """
{
    "id": 102,
    "name": "iPad Pro",
    "price": 799.99
}
""".data(using: .utf8)!

let decoder = JSONDecoder()
if let decodedProduct = try? decoder.decode(Product.self, from: incomingJSON) {
    print("Decoded Product Name: \(decodedProduct.name)")   // iPad Pro
}
```

### Comparison

| Feature | Hashable | Codable |
|---|---|---|
| Main goal | Uniquely identifying and comparing values | Converting data to/from external formats |
| Common use | `Dictionary` keys, unique `Set` items | JSON APIs, persisting app data |
| Inherits from | `Equatable` | `Encodable` & `Decodable` |
| Key requirement | `hash(into:)` | `init(from:)` and `encode(to:)` |

---

## 22. Observer Pattern

The classic, decoupled approach using a custom protocol for type safety, with no dependency on system frameworks.

> **Memory note:** always store observers **weakly** to avoid retain cycles.

```swift
// 1. Define the observer contract
protocol StoreObserver: AnyObject {
    func newProductDidArrive(name: String)
}

// 2. Define the subject
class Store {
    // Wrapper keeps observers weak to prevent leaks
    private class WeakObserver {
        weak var value: StoreObserver?
        init(_ value: StoreObserver) { self.value = value }
    }
    private var observers = [WeakObserver]()

    func subscribe(_ observer: StoreObserver) {
        observers.append(WeakObserver(observer))
    }

    func broadcastNewProduct(name: String) {
        // Drop deallocated observers, then notify the rest
        observers = observers.filter { $0.value != nil }
        observers.forEach { $0.value?.newProductDidArrive(name: name) }
    }
}

// 3. Implement an observer
class Customer: StoreObserver {
    let name: String
    init(name: String) { self.name = name }

    func newProductDidArrive(name: String) {
        print("\(self.name) received notification: \(name) is now in stock!")
    }
}

// --- Usage ---
let store = Store()
let alice = Customer(name: "Alice")

store.subscribe(alice)
store.broadcastNewProduct(name: "iPhone 18")
// Prints: Alice received notification: iPhone 18 is now in stock!
```

---

## 23. NotificationCenter

A centralized broadcast system. Best when the subject and observers are fully independent components that should not know about each other.

```swift
import Foundation

// 1. Define a unique notification name
extension Notification.Name {
    static let didReleaseNewProduct = Notification.Name("didReleaseNewProduct")
}

// 2. The subject posts to the center
class BroadcastStore {
    func releaseProduct(name: String) {
        NotificationCenter.default.post(
            name: .didReleaseNewProduct,
            object: nil,
            userInfo: ["productName": name]
        )
    }
}

// 3. The observer listens on the center
class BroadcastCustomer {
    private var observerToken: NSObjectProtocol?

    init() {
        observerToken = NotificationCenter.default.addObserver(
            forName: .didReleaseNewProduct,
            object: nil,
            queue: .main
        ) { notification in
            if let name = notification.userInfo?["productName"] as? String {
                print("Broadcast received: \(name) is ready!")
            }
        }
    }

    deinit {
        // Unregister to prevent leaks
        if let token = observerToken {
            NotificationCenter.default.removeObserver(token)
        }
    }
}
```

---

## 24. Grand Central Dispatch (GCD)

Think of GCD as **a task scheduling system**. You submit work:

```swift
queue.async {
    doWork()
}
```

| Apple decides | You manage |
|---|---|
| Thread management | Ordering |
| Scheduling | Synchronization |
| Execution | Concurrency structure |

### Serial Queue

Only **one** task executes at a time.

```swift
let queue = DispatchQueue(label: "serial.queue")

queue.async { print("Task 1") }
queue.async { print("Task 2") }
```

Guaranteed output, never parallel:

```
Task 1
Task 2
```

**Use case: protect shared state without locks**

```swift
final class EventStore {
    private let queue = DispatchQueue(label: "event.store")
    private var events: [String] = []

    func add(_ event: String) {
        queue.async {
            self.events.append(event)
        }
    }
}
```

Because only one task runs at once there is no race condition and no mutex needed.

> **Trap:** a serial queue does **not** mean a dedicated thread. It only guarantees *serialized execution*; GCD may reuse threads internally.

**Deadlock**

```swift
queue.sync {
    // ...somewhere down the call chain, another sync on the same queue:
    queue.sync {
        print("Deadlock")
    }
}
```

The outer block already occupies the queue, so the inner `sync` waits forever.

### Concurrent Queue

Multiple tasks **may** run simultaneously.

```swift
let queue = DispatchQueue(
    label: "concurrent",
    attributes: .concurrent
)

queue.async { heavyTask1() }
queue.async { heavyTask2() }
```

High-throughput use cases: event processing, async workers, telemetry upload, scanning tasks.

> **Trap:** shared mutable state becomes dangerous.
>
> ```swift
> var counter = 0
> queue.async { counter += 1 }   // race condition
> ```

### sync vs async

**Async** — submit and continue immediately. The caller does **not** wait; output order is not guaranteed.

```swift
queue.async { doWork() }
print("continues")
```

Common usage: background work, telemetry upload, async processing, endpoint analytics.

**Sync** — submit and **block** until completion.

```swift
queue.sync { doWork() }
print("after work")
```

Common usage: reading synchronized state, atomic operations, coordination.

> **Huge sync trap:** used badly it causes UI freezes, thread starvation, and deadlocks.

### Barriers

Very important for high-performance shared state. Concurrent reads are fine, but writes need exclusivity — barriers solve this.

```swift
let queue = DispatchQueue(
    label: "cache.queue",
    attributes: .concurrent
)

queue.async {
    readCache()
}

queue.async(flags: .barrier) {
    updateCache()
}
```

A barrier task:

1. Waits until all previously submitted tasks finish,
2. Executes exclusively,
3. Then lets subsequent tasks continue.

Classic pattern: **many readers, occasional writers**. Very common for caches, policy stores, and event maps.

> **Gotcha:** barriers only work on **custom concurrent queues**, not on global queues.

### QoS (Quality of Service)

Priority level for work.

| QoS | Meaning | Example |
|---|---|---|
| `.userInteractive` | Critical, immediate work | UI responsiveness |
| `.userInitiated` | The user is waiting for the result | File open |
| `.utility` | Long-running background work (very common in endpoint agents) | Telemetry upload, analytics |
| `.background` | Lowest priority | Cleanup, maintenance |

> **Huge QoS trap: priority inversion.** A low-priority task holds a resource a high-priority task needs. Causes **latency spikes**, not deadlocks. Good interview talking point.

### DispatchGroup

Wait for multiple async tasks.

```swift
let group = DispatchGroup()

group.enter()
queue.async {
    work1()
    group.leave()
}

group.enter()
queue.async {
    work2()
    group.leave()
}

group.notify(queue: .main) {
    print("done")
}
```

Use for parallel processing, batching, coordination.

### DispatchSemaphore

Limits concurrency.

```swift
let semaphore = DispatchSemaphore(value: 3)   // at most 3 tasks run simultaneously
```

> **Trap:** improper semaphore usage leads to deadlocks, starvation, and blocking. Modern Swift prefers structured concurrency where possible.

### Global Queues

```swift
DispatchQueue.global()
```

Shared, system-managed queues. Convenient, but:

- Less control
- Potential contention
- Barrier limitations (see above)

Endpoint systems often prefer dedicated queues.

### Main Queue

```swift
DispatchQueue.main
```

The UI thread. macOS agents usually avoid heavy work here.

---

## Further Reading

- [Swift Interview Questions and Answers — Kodeco](https://www.kodeco.com/762435-swift-interview-questions-and-answers)
