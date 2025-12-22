---
layout: base.njk
title: Fucking Approachable Swift Concurrency
description: A no-bullshit guide to Swift concurrency. Learn async/await, actors, Sendable, and MainActor with simple mental models. No jargon, just clear explanations.
---

<section class="hero">
  <div class="container">
    <h1>Fucking Approachable<br><span class="accent">Swift Concurrency</span></h1>
    <p class="subtitle">Finally understand async/await, actors, and Sendable. Clear mental models, no jargon.</p>
    <p class="credit">Huge thanks to <a href="https://www.massicotte.org/">Matt Massicotte</a> for making Swift concurrency understandable.</p>
    <p class="credit">Put together by <a href="https://pepicrft.me">Pedro Piñera</a>. Found an issue? <a href="mailto:pedro@tuist.dev">pedro@tuist.dev</a></p>
    <p class="tribute">In the tradition of <a href="https://fuckingblocksyntax.com/">fuckingblocksyntax.com</a> and <a href="https://fuckingifcaseletsyntax.com/">fuckingifcaseletsyntax.com</a></p>
  </div>
</section>

<section class="tldr">
  <div class="container">

## The Honest Truth

There's no cheat sheet for Swift concurrency. Every "just do X" answer is wrong in some context.

**But here's the good news:** Once you understand [isolation](#basics) (5 min read), everything clicks. The compiler errors start making sense. You stop fighting the system and start working with it.

*This guide targets Swift 6+. Most concepts apply to Swift 5.5+, but Swift 6 enforces stricter concurrency checking.*

<a href="#basics" class="read-more">Start with the mental model &darr;</a>

  </div>
</section>

<section id="basics">
  <div class="container">

## The One Thing You Need to Understand

**[Isolation](https://www.massicotte.org/intro-to-isolation/)** is the key to everything. It's Swift's answer to the question: *Who is allowed to touch this data right now?*

<div class="analogy">
<h4>The Office Building</h4>

Think of your app as an **office building**. Each office is an **isolation domain** - a private space where only one person can work at a time. You can't just barge into someone else's office and start rearranging their desk.

We'll build on this analogy throughout the guide.
</div>

### Why Not Just Threads?

For decades, we wrote concurrent code by thinking about threads. The problem? **Threads don't prevent you from shooting yourself in the foot.** Two threads can access the same data simultaneously, causing data races - bugs that crash randomly and are nearly impossible to reproduce.

On a phone, you might get away with it. On a server handling thousands of concurrent requests, data races become a certainty - usually surfacing in production, on a Friday. As Swift expands to servers and other highly concurrent environments, "hope for the best" doesn't cut it.

The old approach was defensive: use locks, dispatch queues, hope you didn't miss a spot.

Swift's approach is different: **make data races impossible at compile time.** Instead of asking "which thread is this on?", Swift asks "who is allowed to touch this data right now?" That's isolation.

### How Other Languages Handle This

| Language | Approach | When you find out about bugs |
|----------|----------|------------------------------|
| **Swift** | Isolation + Sendable | Compile time |
| **Rust** | Ownership + borrow checker | Compile time |
| **Go** | Channels + race detector | Runtime (with tooling) |
| **Java/Kotlin** | `synchronized`, locks | Runtime (crashes) |
| **JavaScript** | Single-threaded event loop | Avoided entirely |
| **C/C++** | Manual locks | Runtime (undefined behavior) |

Swift and Rust are the only mainstream languages that catch data races at compile time. The tradeoff? A steeper learning curve upfront. But once you understand the model, the compiler has your back.

Those annoying errors about `Sendable` and actor isolation? They're catching bugs that would have been silent crashes before.

  </div>
</section>

<section id="domains">
  <div class="container">

## The Isolation Domains

Now that you understand isolation (private offices), let's look at the different types of offices in Swift's building.

<div class="analogy">
<h4>The Office Building</h4>

- **The front desk** (`MainActor`) - where all customer interactions happen. There's only one, and it handles everything the user sees.
- **Department offices** (`actor`) - accounting, legal, HR. Each department has its own office protecting its own sensitive data.
- **Hallways and common areas** (`nonisolated`) - shared spaces anyone can walk through. No private data here.
</div>

### MainActor: The Front Desk

The `MainActor` is a special isolation domain that runs on the main thread. It's where all UI work happens.

```swift
@MainActor
@Observable
class ViewModel {
    var items: [Item] = []  // UI state lives here

    func refresh() async {
        let newItems = await fetchItems()
        self.items = newItems  // Safe - we're on MainActor
    }
}
```

<div class="tip">
<h4>When in doubt, use MainActor</h4>

For most apps, marking your ViewModels and UI-related classes with `@MainActor` is the right choice. Performance concerns are usually overblown - start here, optimize only if you measure actual problems.
</div>

### Actors: Department Offices

An `actor` is like a department office - it protects its own data and only allows one visitor at a time.

```swift
actor BankAccount {
    var balance: Double = 0

    func deposit(_ amount: Double) {
        balance += amount  // Safe! Only one caller at a time
    }
}
```

Without actors, two threads read balance = 100, both add 50, both write 150 - you lost $50. With actors, Swift automatically queues access and both deposits complete correctly.

<div class="warning">
<h4>Don't overuse actors</h4>

You need a custom actor only when **all four** of these are true:
1. You have non-Sendable (thread-unsafe) mutable state
2. Multiple places need to access it
3. Operations on that state must be atomic
4. It can't just live on MainActor

If any condition is false, you probably don't need an actor. Most UI state can live on `@MainActor`. [Read more about when to use actors](https://www.massicotte.org/actors/).
</div>

### Nonisolated: The Hallways

Code marked `nonisolated` is like the hallways - it doesn't belong to any office and can be accessed from anywhere.

```swift
actor UserSession {
    let userId: String          // Immutable - safe to read from anywhere
    var lastActivity: Date      // Mutable - needs actor protection

    nonisolated var displayId: String {
        "User: \(userId)"       // Only reads immutable data
    }
}

// Usage - no await needed for nonisolated
let session = UserSession(userId: "123")
print(session.displayId)  // Works synchronously!
```

Use `nonisolated` for computed properties that only read immutable data.

  </div>
</section>

<section id="propagation">
  <div class="container">

## How Isolation Propagates

When you mark a type with an actor isolation, what happens to its methods? What about closures? Understanding how isolation spreads is key to avoiding surprises.

<div class="analogy">
<h4>The Office Building</h4>

When you're hired into a department, you work in that department's office by default. If the Marketing department hires you, you don't randomly show up in Accounting.

Similarly, when a function is defined inside a `@MainActor` class, it inherits that isolation. It "works in the same office" as its parent.
</div>

### Classes Inherit Their Isolation

```swift
@MainActor
class ViewModel {
    var count = 0           // MainActor-isolated

    func increment() {      // Also MainActor-isolated
        count += 1
    }
}
```

Everything inside the class inherits `@MainActor`. You don't need to mark each method.

### Tasks Inherit Context (Usually)

```swift
@MainActor
class ViewModel {
    func doWork() {
        Task {
            // This inherits MainActor!
            self.updateUI()  // Safe, no await needed
        }
    }
}
```

A `Task { }` created from a `@MainActor` context stays on `MainActor`. This is usually what you want.

### Task.detached Breaks Inheritance

```swift
@MainActor
class ViewModel {
    func doWork() {
        Task.detached {
            // NOT on MainActor anymore!
            await self.updateUI()  // Need await now
        }
    }
}
```

<div class="analogy">
<h4>The Office Building</h4>

`Task.detached` is like hiring an outside contractor. They don't have a badge to your office - they work from their own space and must go through proper channels to access your stuff.
</div>

<div class="warning">
<h4>Task.detached is usually wrong</h4>

Most of the time, you want a regular `Task`. Detached tasks don't inherit priority, task-local values, or actor context. Use them only when you explicitly need that separation.
</div>

  </div>
</section>

<section id="sendable">
  <div class="container">

## What Can Cross Boundaries

Now that you know about isolation domains (offices) and how they propagate, the next question is: **what can you pass between them?**

<div class="analogy">
<h4>The Office Building</h4>

Not everything can leave an office:

- **Photocopies** are safe to share - if Legal makes a copy of a document and sends it to Accounting, both have their own copy. No conflict.
- **Original signed contracts** must stay put - if two departments could both modify the original, chaos ensues.

In Swift terms: **Sendable** types are photocopies (safe to share), **non-Sendable** types are originals (must stay in one office).
</div>

### Sendable: Safe to Share

These types can cross isolation boundaries safely:

```swift
// Structs with immutable data - like photocopies
struct User: Sendable {
    let id: Int
    let name: String
}

// Actors protect themselves - they handle their own visitors
actor BankAccount { }  // Automatically Sendable
```

**Automatically Sendable:**
- Value types (structs, enums) with Sendable properties
- Actors (they protect themselves)
- Immutable classes (`final class` with only `let` properties)

### Non-Sendable: Must Stay Put

These types can't safely cross boundaries:

```swift
// Classes with mutable state - like original documents
class Counter {
    var count = 0  // Two offices modifying this = disaster
}
```

**Why is this the key distinction?** Because every compiler error you'll encounter boils down to: *"You're trying to send a non-Sendable type across an isolation boundary."*

### When the Compiler Complains

If Swift says something isn't Sendable, you have options:

1. **Make it a value type** - use `struct` instead of `class`
2. **Isolate it** - keep it on `@MainActor` so it doesn't need to cross
3. **Keep it non-Sendable** - just don't pass it between offices
4. **Last resort:** `@unchecked Sendable` - you're promising it's safe (be careful)

<div class="tip">
<h4>Start non-Sendable</h4>

[Matt Massicotte advocates](https://www.massicotte.org/non-sendable/) starting with regular, non-Sendable types. Add `Sendable` only when you need to cross boundaries. A non-Sendable type stays simple and avoids conformance headaches.
</div>

  </div>
</section>

<section id="async-await">
  <div class="container">

## How to Cross Boundaries

You understand isolation domains, you know what can cross them. Now: **how do you actually communicate between offices?**

<div class="analogy">
<h4>The Office Building</h4>

You can't just barge into another office. You send a request and wait for a response. You might work on other things while waiting, but you need that response before you can continue.

That's `async/await` - sending a request to another isolation domain and pausing until you get an answer.
</div>

### The await Keyword

When you call a function on another actor, you need `await`:

```swift
actor DataStore {
    var items: [Item] = []

    func add(_ item: Item) {
        items.append(item)
    }
}

@MainActor
class ViewModel {
    let store = DataStore()

    func addItem(_ item: Item) async {
        await store.add(item)  // Request to another office
        updateUI()             // Back in our office
    }
}
```

The `await` means: "Send this request and pause until it's done. I might do other work while waiting."

### Suspension, Not Blocking

<div class="warning">
<h4>Common Misconception</h4>

Many developers assume that adding `async` makes code run in the background. It doesn't. The `async` keyword just means the function *can pause*. It says nothing about *where* it runs.
</div>

The key insight is the difference between **blocking** and **suspension**:

- **Blocking**: You sit in the waiting room staring at the wall. Nothing else happens.
- **Suspension**: You leave your phone number and run errands. They'll call when ready.

<div class="code-tabs">
<div class="code-tabs-nav">
<button class="active">Blocking</button>
<button>Suspension</button>
</div>
<div class="code-tab-content active">

```swift
// Thread sits idle, doing nothing for 5 seconds
Thread.sleep(forTimeInterval: 5)
```

</div>
<div class="code-tab-content">

```swift
// Thread is freed to do other work while waiting
try await Task.sleep(for: .seconds(5))
```

</div>
</div>

### Starting Async Work from Sync Code

Sometimes you're in synchronous code and need to call something async. Use `Task`:

```swift
@MainActor
class ViewModel {
    func buttonTapped() {  // Sync function
        Task {
            await loadData()  // Now we can use await
        }
    }
}
```

<div class="analogy">
<h4>The Office Building</h4>

`Task` is like assigning work to an employee. The employee handles the request (including waiting for other offices) while you continue with your immediate work.
</div>

  </div>
</section>

<section id="patterns">
  <div class="container">

## Patterns That Work

### The Network Request Pattern

<div class="isolation-legend">
  <span class="isolation-legend-item main">MainActor</span>
  <span class="isolation-legend-item nonisolated">Nonisolated (network call)</span>
</div>
<div class="code-isolation">
<div class="isolation-sidebar">
  <div class="segment main" style="flex-grow: 8"></div>
  <div class="segment nonisolated" style="flex-grow: 2"></div>
  <div class="segment main" style="flex-grow: 6"></div>
</div>
<div class="isolation-overlay">
  <div class="segment" style="flex-grow: 8"></div>
  <div class="segment nonisolated-highlight" style="flex-grow: 2"></div>
  <div class="segment" style="flex-grow: 6"></div>
</div>

```swift
@MainActor
@Observable
class ViewModel {
    var users: [User] = []
    var isLoading = false

    func fetchUsers() async {
        isLoading = true

        // This suspends - thread is free to do other work
        let users = await networkService.getUsers()

        // Back on MainActor automatically
        self.users = users
        isLoading = false
    }
}
```

</div>

No `DispatchQueue.main.async`. The `@MainActor` attribute handles it.

### Parallel Work with async let

```swift
func loadProfile() async -> Profile {
    async let avatar = loadImage("avatar.jpg")
    async let banner = loadImage("banner.jpg")
    async let details = loadUserDetails()

    // All three run in parallel!
    return Profile(
        avatar: await avatar,
        banner: await banner,
        details: await details
    )
}
```

### Preventing Double-Taps

This pattern comes from Matt Massicotte's guide on [stateful systems](https://www.massicotte.org/step-by-step-stateful-systems):

```swift
@MainActor
class ButtonViewModel {
    private var isLoading = false

    func buttonTapped() {
        // Guard SYNCHRONOUSLY before any async work
        guard !isLoading else { return }
        isLoading = true

        Task {
            await doExpensiveWork()
            isLoading = false
        }
    }
}
```

<div class="warning">
<h4>Critical: The guard must be synchronous</h4>

If you put the guard inside the Task after an await, there's a window where two button taps can both start work. [Learn more about ordering and concurrency](https://www.massicotte.org/ordering-and-concurrency).
</div>

  </div>
</section>

<section id="mistakes">
  <div class="container">

## Common Mistakes to Avoid

These are [common mistakes](https://www.massicotte.org/mistakes-with-concurrency/) that even experienced developers make:

### Thinking async = background

<div class="analogy">
<h4>The Office Building</h4>

Adding `async` doesn't move you to a different office. You're still at the front desk - you can just wait for deliveries now without freezing in place.
</div>

```swift
// This STILL blocks the main thread!
@MainActor
func slowFunction() async {
    let result = expensiveCalculation()  // Synchronous = blocking
    data = result
}
```

If you need work done in another office, explicitly send it there:

```swift
func slowFunction() async {
    let result = await Task.detached {
        expensiveCalculation()  // Now in a different office
    }.value
    await MainActor.run { data = result }
}
```

### Creating too many actors

<div class="analogy">
<h4>The Office Building</h4>

Creating a new office for every piece of data means endless paperwork to communicate between them. Most of your work can happen at the front desk.
</div>

```swift
// Over-engineered - every call requires walking between offices
actor NetworkManager { }
actor CacheManager { }
actor DataManager { }

// Better - most things can live at the front desk
@MainActor
class AppState { }
```

### Using MainActor.run everywhere

<div class="analogy">
<h4>The Office Building</h4>

If you keep walking to the front desk for every little thing, just work there. Make it part of your job description, not a constant errand.
</div>

```swift
// Don't do this - constantly walking to front desk
await MainActor.run { doMainActorStuff() }

// Do this - just work at the front desk
@MainActor func doMainActorStuff() { }
```

### Making everything Sendable

Not everything needs to be `Sendable`. If you're adding `@unchecked Sendable` everywhere, you're making photocopies of things that don't need to leave the office.

### Ignoring compiler warnings

Every compiler warning about `Sendable` is the security guard telling you something isn't safe to carry between offices. Don't ignore them - [understand them](https://www.massicotte.org/complete-checking/).

  </div>
</section>

<section id="errors">
  <div class="container">

## Common Compiler Errors

These are the actual error messages you'll see. Each one is the compiler protecting you from a data race.

### "Sending 'self.foo' risks causing data races"

<div class="compiler-error">
Sending 'self.foo' risks causing data races
</div>

<div class="analogy">
<h4>The Office Building</h4>

You're trying to carry an original document to another office. Either make a photocopy (Sendable) or keep it in one place.
</div>

**Fix 1:** Use a `struct` instead of a `class`

**Fix 2:** Keep it on one actor:

```swift
@MainActor
class MyClass {
    var foo: SomeType  // Stays at the front desk
}
```

### "Non-sendable type cannot cross actor boundary"

<div class="compiler-error">
Non-sendable type 'MyClass' cannot cross actor boundary
</div>

<div class="analogy">
<h4>The Office Building</h4>

You're trying to carry an original between offices. The security guard stopped you.
</div>

**Fix 1:** Make it a struct:

```swift
// Before: class (non-Sendable)
class User { var name: String }

// After: struct (Sendable)
struct User: Sendable { let name: String }
```

**Fix 2:** Isolate it to one actor:

```swift
@MainActor
class User { var name: String }
```

### "Actor-isolated property cannot be referenced"

<div class="compiler-error">
Actor-isolated property 'balance' cannot be referenced from the main actor
</div>

<div class="analogy">
<h4>The Office Building</h4>

You're reaching into another office's filing cabinet without going through proper channels.
</div>

**Fix:** Use `await`:

```swift
// Wrong - reaching in directly
let value = myActor.balance

// Right - proper request
let value = await myActor.balance
```

### "Call to main actor-isolated method in synchronous context"

<div class="compiler-error">
Call to main actor-isolated instance method 'updateUI()' in a synchronous nonisolated context
</div>

<div class="analogy">
<h4>The Office Building</h4>

You're trying to use the front desk without waiting in line.
</div>

**Fix 1:** Make the caller `@MainActor`:

```swift
@MainActor
func doSomething() {
    updateUI()  // Same isolation, no await needed
}
```

**Fix 2:** Use `await`:

```swift
func doSomething() async {
    await updateUI()
}
```

  </div>
</section>

<section>
  <div class="container">

## Three Levels of Swift Concurrency

You don't need to learn everything at once. Progress through these levels:

<div class="analogy">
<h4>The Office Building</h4>

Think of it like growing a company. You don't start with a 50-floor headquarters - you start with a desk.
</div>

These levels aren't strict boundaries - different parts of your app might need different levels. A mostly-Level-1 app might have one feature that needs Level 2 patterns. That's fine. Use the simplest approach that works for each piece.

### Level 1: The Startup

Everyone works at the front desk. Simple, direct, no bureaucracy.

- Use `async/await` for network calls
- Mark UI classes with `@MainActor`
- Use SwiftUI's `.task` modifier

This handles 80% of apps. Apps like [Things](https://culturedcode.com/things/), [Bear](https://bear.app/), [Flighty](https://flighty.com/), or [Day One](https://dayoneapp.com/) likely fall into this category - apps that primarily fetch data and display it.

### Level 2: The Growing Company

You need to handle multiple things at once. Time for parallel projects and coordinating teams.

- Use `async let` for parallel work
- Use `TaskGroup` for dynamic parallelism
- Understand task cancellation

Apps like [Ivory](https://tapbots.com/ivory/)/[Ice Cubes](https://github.com/Dimillian/IceCubesApp) (Mastodon clients managing multiple timelines and streaming updates), [Overcast](https://overcast.fm/) (coordinating downloads, playback, and background sync), or [Slack](https://slack.com/) (real-time messaging across multiple channels) might use these patterns for certain features.

### Level 3: The Enterprise

Dedicated departments with their own policies. Complex inter-office communication.

- Create custom actors for shared state
- Deep understanding of Sendable
- Custom executors

Apps like [Xcode](https://developer.apple.com/xcode/), [Final Cut Pro](https://www.apple.com/final-cut-pro/), or server-side Swift frameworks like [Vapor](https://vapor.codes/) and [Hummingbird](https://hummingbird.codes/) likely need these patterns - complex shared state, thousands of concurrent connections, or framework-level code that others build on.

<div class="tip">
<h4>Start simple</h4>

Most apps never need Level 3. Don't build an enterprise when a startup will do.
</div>

  </div>
</section>

<section id="glossary">
  <div class="container">

## Glossary: More Keywords You'll Encounter

Beyond the core concepts, here are other Swift concurrency keywords you'll see in the wild:

| Keyword | What it means |
|---------|---------------|
| `nonisolated` | Opts out of an actor's isolation - runs without protection |
| `isolated` | Explicitly declares a parameter runs in an actor's context |
| `@Sendable` | Marks a closure as safe to pass across isolation boundaries |
| `Task.detached` | Creates a task completely separate from current context |
| `AsyncSequence` | A sequence you can iterate with `for await` |
| `AsyncStream` | A way to bridge callback-based code to async sequences |
| `withCheckedContinuation` | Bridges completion handlers to async/await |
| `Task.isCancelled` | Check if current task was cancelled |
| `@preconcurrency` | Suppresses concurrency warnings for legacy code |
| `GlobalActor` | Protocol for creating your own custom actors like MainActor |

### When to Use Each

#### nonisolated - Reading computed properties

<div class="analogy">
Like a nameplate on your office door - anyone walking by can read it without needing to come inside and wait for you.
</div>

By default, everything inside an actor is isolated - you need `await` to access it. But sometimes you have properties that are inherently safe to read: immutable `let` constants, or computed properties that only derive values from other safe data. Marking these `nonisolated` lets callers access them synchronously, avoiding unnecessary async overhead.

<div class="isolation-legend">
  <span class="isolation-legend-item actor">Actor-isolated</span>
  <span class="isolation-legend-item nonisolated">Nonisolated</span>
</div>
<div class="code-isolation">
<div class="isolation-sidebar">
  <div class="segment actor" style="flex-grow: 4"></div>
  <div class="segment nonisolated" style="flex-grow: 4"></div>
  <div class="segment actor" style="flex-grow: 1"></div>
</div>
<div class="isolation-overlay">
  <div class="segment" style="flex-grow: 4"></div>
  <div class="segment nonisolated-highlight" style="flex-grow: 4"></div>
  <div class="segment" style="flex-grow: 1"></div>
</div>

```swift
actor UserSession {
    let userId: String  // Immutable, safe to read
    var lastActivity: Date  // Mutable, needs protection

    // This can be called without await
    nonisolated var displayId: String {
        "User: \(userId)"  // Only reads immutable data
    }
}
```

</div>

```swift
// Usage
let session = UserSession(userId: "123")
print(session.displayId)  // No await needed!
```

#### @Sendable - Closures that cross boundaries

<div class="analogy">
Like a sealed envelope with instructions inside - the envelope can travel between offices, and whoever opens it can follow the instructions safely.
</div>

When a closure escapes to run later or on a different isolation domain, Swift needs to guarantee it won't cause data races. The `@Sendable` attribute marks closures that are safe to pass across boundaries - they can't capture mutable state unsafely. Swift often infers this automatically (like with `Task.detached`), but sometimes you need to declare it explicitly when designing APIs that accept closures.

```swift
@MainActor
class ViewModel {
    var items: [Item] = []

    func processInBackground() {
        Task.detached {
            // This closure crosses from detached task to MainActor
            // It must be @Sendable (Swift infers this)
            let processed = await self.heavyProcessing()
            await MainActor.run {
                self.items = processed
            }
        }
    }
}

// Explicit @Sendable when needed
func runLater(_ work: @Sendable @escaping () -> Void) {
    DispatchQueue.global().asyncAfter(deadline: .now() + 1) {
        work()
    }
}
```

#### withCheckedContinuation - Bridging old APIs

<div class="analogy">
Like a translator between the old paper memo system and modern email. You wait by the mailroom until the old system delivers a response, then forward it through the new system.
</div>

Many older APIs use completion handlers instead of async/await. Rather than rewriting them entirely, you can wrap them using `withCheckedContinuation`. This function suspends the current task, gives you a continuation object, and resumes when you call `continuation.resume()`. The "checked" variant catches programming errors like resuming twice or never resuming at all.

<div class="isolation-legend">
  <span class="isolation-legend-item main">Async context</span>
  <span class="isolation-legend-item nonisolated">Callback context</span>
</div>
<div class="code-isolation">
<div class="isolation-sidebar">
  <div class="segment nonisolated" style="flex-grow: 5"></div>
  <div class="segment main" style="flex-grow: 3"></div>
  <div class="segment nonisolated" style="flex-grow: 3"></div>
  <div class="segment main" style="flex-grow: 2"></div>
</div>
<div class="isolation-overlay">
  <div class="segment" style="flex-grow: 5"></div>
  <div class="segment main-highlight" style="flex-grow: 3"></div>
  <div class="segment nonisolated-highlight" style="flex-grow: 3"></div>
  <div class="segment main-highlight" style="flex-grow: 2"></div>
</div>

```swift
// Old callback-based API
func fetchUser(id: String, completion: @escaping (User?) -> Void) {
    // ... network call with callback
}

// Wrapped as async
func fetchUser(id: String) async -> User? {
    await withCheckedContinuation { continuation in
        fetchUser(id: id) { user in
            continuation.resume(returning: user)  // Bridges back!
        }
    }
}
```

</div>

For throwing functions, use `withCheckedThrowingContinuation`:

```swift
func fetchUserThrowing(id: String) async throws -> User {
    try await withCheckedThrowingContinuation { continuation in
        fetchUser(id: id) { result in
            switch result {
            case .success(let user):
                continuation.resume(returning: user)
            case .failure(let error):
                continuation.resume(throwing: error)
            }
        }
    }
}
```

#### AsyncStream - Bridging event sources

<div class="analogy">
Like setting up mail forwarding - every time a letter arrives at the old address, it automatically gets routed to your new inbox. The stream keeps flowing as long as mail keeps coming.
</div>

While `withCheckedContinuation` handles one-shot callbacks, many APIs deliver multiple values over time - delegate methods, NotificationCenter, or custom event systems. `AsyncStream` bridges these to Swift's `AsyncSequence`, letting you use `for await` loops. You create a stream, store its continuation, and call `yield()` each time a new value arrives.

```swift
class LocationTracker: NSObject, CLLocationManagerDelegate {
    private var continuation: AsyncStream<CLLocation>.Continuation?

    var locations: AsyncStream<CLLocation> {
        AsyncStream { continuation in
            self.continuation = continuation
        }
    }

    func locationManager(_ manager: CLLocationManager,
                        didUpdateLocations locations: [CLLocation]) {
        for location in locations {
            continuation?.yield(location)
        }
    }
}

// Usage
let tracker = LocationTracker()
for await location in tracker.locations {
    print("New location: \(location)")
}
```

#### Task.isCancelled - Cooperative cancellation

<div class="analogy">
Like checking your inbox for a "stop working on this" memo before starting each step of a big project. You're not forced to stop - you choose to check and respond politely.
</div>

Swift uses cooperative cancellation - when a task is cancelled, it doesn't stop immediately. Instead, a flag is set, and it's your responsibility to check it periodically. This gives you control over cleanup and partial results. Use `Task.checkCancellation()` to throw immediately, or check `Task.isCancelled` when you want to handle cancellation gracefully (like returning partial results).

```swift
func processLargeDataset(_ items: [Item]) async throws -> [Result] {
    var results: [Result] = []

    for item in items {
        // Check before each expensive operation
        try Task.checkCancellation()  // Throws if cancelled

        // Or check without throwing
        if Task.isCancelled {
            return results  // Return partial results
        }

        let result = await process(item)
        results.append(result)
    }

    return results
}
```

#### Task.detached - Escaping the current context

<div class="analogy">
Like hiring an outside contractor who doesn't report to your department. They work independently, don't follow your office's rules, and you have to explicitly coordinate when you need results back.
</div>

A regular `Task { }` inherits the current actor context - if you're on `@MainActor`, the task runs on `@MainActor`. Sometimes that's not what you want, especially for CPU-intensive work that would block the UI. `Task.detached` creates a task with no inherited context, running on a background executor. Use it sparingly though - most of the time, regular `Task` with proper `await` points is sufficient and easier to reason about.

<div class="isolation-legend">
  <span class="isolation-legend-item main">MainActor</span>
  <span class="isolation-legend-item detached">Detached</span>
</div>
<div class="code-isolation">
<div class="isolation-sidebar">
  <div class="segment main" style="flex-grow: 10"></div>
  <div class="segment detached" style="flex-grow: 2"></div>
  <div class="segment main" style="flex-grow: 1"></div>
  <div class="segment detached" style="flex-grow: 1"></div>
  <div class="segment main" style="flex-grow: 3"></div>
</div>
<div class="isolation-overlay">
  <div class="segment" style="flex-grow: 10"></div>
  <div class="segment detached-highlight" style="flex-grow: 2"></div>
  <div class="segment" style="flex-grow: 1"></div>
  <div class="segment detached-highlight" style="flex-grow: 1"></div>
  <div class="segment" style="flex-grow: 3"></div>
</div>

```swift
@MainActor
class ImageProcessor {
    func processImage(_ image: UIImage) {
        // DON'T: This still inherits MainActor context
        Task {
            let filtered = applyFilters(image)  // Blocks main!
        }

        // DO: Detached task runs independently
        Task.detached(priority: .userInitiated) {
            let filtered = await self.applyFilters(image)
            await MainActor.run {
                self.displayImage(filtered)
            }
        }
    }
}
```

</div>

<div class="warning">
<h4>Task.detached is usually wrong</h4>

Most of the time, you want a regular `Task`. Detached tasks don't inherit priority, task-local values, or actor context. Use them only when you explicitly need that separation.
</div>

#### @preconcurrency - Living with legacy code

Silence warnings when importing modules not yet updated for concurrency:

```swift
// Suppress warnings from this import
@preconcurrency import OldFramework

// Or on a protocol conformance
class MyDelegate: @preconcurrency SomeOldDelegate {
    // Won't warn about non-Sendable requirements
}
```

<div class="tip">
<h4>@preconcurrency is temporary</h4>

Use it as a bridge while updating code. The goal is to eventually remove it and have proper Sendable conformance.
</div>

## Further Reading

This guide distills the best resources on Swift concurrency.

<div class="resources">
<h4>Matt Massicotte's Blog (Highly Recommended)</h4>

- [A Swift Concurrency Glossary](https://www.massicotte.org/concurrency-glossary) - Essential terminology
- [An Introduction to Isolation](https://www.massicotte.org/intro-to-isolation/) - The core concept
- [When should you use an actor?](https://www.massicotte.org/actors/) - Practical guidance
- [Non-Sendable types are cool too](https://www.massicotte.org/non-sendable/) - Why simpler is better
- [Crossing the Boundary](https://www.massicotte.org/crossing-the-boundary/) - Working with non-Sendable types
- [Problematic Swift Concurrency Patterns](https://www.massicotte.org/problematic-patterns/) - What to avoid
- [Making Mistakes with Swift Concurrency](https://www.massicotte.org/mistakes-with-concurrency/) - Learning from errors
</div>

<div class="resources">
<h4>Official Apple Resources</h4>

- [Swift Concurrency Documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [WWDC21: Meet async/await](https://developer.apple.com/videos/play/wwdc2021/10132/)
- [WWDC21: Protect mutable state with actors](https://developer.apple.com/videos/play/wwdc2021/10133/)
- [WWDC22: Eliminate data races](https://developer.apple.com/videos/play/wwdc2022/110351/)
</div>

<div class="resources">
<h4>Tutorials</h4>

- [Swift Concurrency by Example - Hacking with Swift](https://www.hackingwithswift.com/quick-start/concurrency)
- [Async await in Swift - SwiftLee](https://www.avanderlee.com/swift/async-await/)
</div>

  </div>
</section>
