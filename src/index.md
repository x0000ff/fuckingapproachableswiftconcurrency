---
layout: base.njk
title: Home
---

<section class="hero">
  <div class="container">
    <h1>Fucking Approachable<br><span class="accent">Swift Concurrency</span></h1>
    <p class="subtitle">A no-bullshit guide to understanding async/await, actors, and Sendable. No jargon. Just clear mental models.</p>
    <p class="tribute">In the tradition of <a href="https://fuckingblocksyntax.com/">fuckingblocksyntax.com</a> and <a href="https://fuckingifcaseletsyntax.com/">fuckingifcaseletsyntax.com</a></p>
  </div>
</section>

<section id="basics">
  <div class="container">

## The One Thing You Need to Understand

<div class="mental-model">
<h4>The Core Mental Model</h4>

**[Isolation](https://www.massicotte.org/intro-to-isolation/)** is the key to everything. It's Swift's answer to the question: "Who is allowed to touch this data right now?"

Think of isolation like rooms in a house. Each room (isolation domain) can only have one person working in it at a time. If you want to share something between rooms, you either need to:
1. Make a copy (Sendable values)
2. Pass it through a secure handoff (actors)
</div>

Swift concurrency is **not** about threads. Stop thinking about threads. Start thinking about **where your code runs** and **who owns the data**.

### The Two Types of Data

Swift divides ALL data into two groups:

| Type | What it means | Examples |
|------|--------------|----------|
| **Sendable** | Safe to share across isolation boundaries | Integers, Strings, Structs with Sendable properties |
| **[Non-Sendable](https://www.massicotte.org/non-sendable/)** | Must stay in one isolation domain | UIView, most classes with mutable state |

That's it. Everything else in Swift concurrency flows from this simple division.

  </div>
</section>

<section id="async-await">
  <div class="container">

## Async/Await: It's Not What You Think

<div class="warning">
<h4>Common Misconception</h4>

"Adding `async` makes my code run in the background."

**Wrong.** The `async` keyword just means the function *can pause*. It says nothing about *where* it runs.
</div>

### The Cafe Analogy

Imagine a cafe with one barista:

**Without async/await:**
1. Customer orders
2. Barista makes drink (everyone waits)
3. Barista serves drink
4. Next customer orders

**With async/await:**
1. Customer orders
2. Barista starts espresso machine
3. While machine runs, barista takes next order
4. Machine beeps, barista finishes first drink
5. Continues with next order

The barista isn't cloned (no new threads). They're just **not standing idle** while waiting.

### Suspension vs Blocking

This is the key insight:

```swift
// BLOCKING - Thread sits idle, doing nothing
Thread.sleep(forTimeInterval: 5)  // Bad!

// SUSPENSION - Thread is freed to do other work
try await Task.sleep(for: .seconds(5))  // Good!
```

<div class="mental-model">
<h4>Think of it this way</h4>

**Blocking** = Sitting in the doctor's waiting room staring at the wall.

**Suspension** = Leaving your phone number and running errands. They'll call when ready.
</div>

### The Pizza Ordering Pattern

```swift
func makeDinner() async {
    let pizza = await orderPizza()   // Pause here, don't block
    let salad = await makeSalad()    // Pause here too
    serve(pizza, salad)
}
```

The code reads top-to-bottom, but executes with pauses. No callback hell. No nested closures.

  </div>
</section>

<section id="actors">
  <div class="container">

## Actors: The Security Guards

An actor is like a security guard standing in front of your data. Only one visitor allowed at a time.

```swift
actor BankAccount {
    var balance: Double = 0

    func deposit(_ amount: Double) {
        balance += amount  // Safe! Only one caller at a time
    }
}
```

**Without actors:**
Two threads read balance = 100, both add 50, both write 150. You lost $50.

**With actors:**
Swift automatically queues access. Both deposits complete correctly.

### When Should You Use an Actor?

<div class="warning">
<h4>Don't overuse actors!</h4>

According to [Matt Massicotte](https://www.massicotte.org/actors/), you need an actor **only when ALL FOUR** conditions are met:

1. You have non-Sendable state
2. Multiple places need to access that state
3. Operations must be atomic
4. It can't just live on MainActor

If ANY condition is false, you probably don't need an actor.
</div>

### MainActor: The Special One

`@MainActor` is a special actor that runs on the main thread. Use it for UI code.

```swift
@MainActor
class ViewModel: ObservableObject {
    @Published var data: [Item] = []

    func loadData() async {
        let items = await fetchItems()  // May suspend
        self.data = items  // Guaranteed back on main thread
    }
}
```

<div class="tip">
<h4>Practical advice</h4>

For most apps, `@MainActor` is your best friend. Matt Massicotte [recommends](https://www.massicotte.org/singletons/) putting it on:
- ViewModels
- Any class that touches UI
- Singletons that need thread-safe access

Performance concerns are usually overblown. Start with `@MainActor`, optimize only if you measure actual problems.
</div>

  </div>
</section>

<section id="sendable">
  <div class="container">

## Sendable: The Thread-Safety Certificate

`Sendable` is Swift's way of saying "this type is safe to share between isolation domains."

### Automatically Sendable

These are Sendable without any work:
- Value types (structs, enums) with Sendable properties
- Actors (they protect themselves)
- Immutable classes (`final class` with only `let` properties)

### Not Sendable

These require thought:
- Mutable classes
- Closures that capture mutable state

```swift
// Automatically Sendable
struct Point: Sendable {
    let x: Int
    let y: Int
}

// NOT Sendable - has mutable state
class Counter {
    var count = 0  // Danger zone!
}
```

<div class="mental-model">
<h4>The Non-Sendable First Design</h4>

[Matt Massicotte advocates](https://www.massicotte.org/non-sendable/) starting with regular, non-isolated, non-Sendable types. Add isolation only when you need it.

This isn't laziness - it's strategic simplicity. A non-Sendable type:
- Stays simple
- Works synchronously from any actor that owns it
- Avoids protocol conformance headaches
</div>

### When the Compiler Complains

If Swift says something isn't Sendable, you have options:

1. **Make it a value type** (struct instead of class)
2. **Isolate it** (`@MainActor`)
3. **Keep it non-Sendable** and don't cross boundaries
4. **Last resort:** `@unchecked Sendable` (you're promising it's safe)

  </div>
</section>

<section id="patterns">
  <div class="container">

## Patterns That Work

### The Network Request Pattern

```swift
@MainActor
class ViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var isLoading = false

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

<section>
  <div class="container">

## Common Mistakes to Avoid

These are [common mistakes](https://www.massicotte.org/mistakes-with-concurrency/) that even experienced developers make:

### Mistake 1: Thinking async = background

```swift
// This STILL blocks the main thread!
@MainActor
func slowFunction() async {
    let result = expensiveCalculation()  // Synchronous = blocking
    data = result
}

// Fix: Move work off MainActor
func slowFunction() async {
    let result = await Task.detached {
        expensiveCalculation()
    }.value
    await MainActor.run { data = result }
}
```

### Mistake 2: Actors everywhere

Don't create an actor for everything. Too many actors = too many isolation boundaries = slow code.

```swift
// Over-engineered
actor NetworkManager { }
actor CacheManager { }
actor DataManager { }
// Every call hops between actors!

// Better: One actor or MainActor for most things
@MainActor
class AppState { }
```

### Mistake 3: MainActor.run everywhere

This is a [problematic pattern](https://www.massicotte.org/problematic-patterns/):

```swift
// Don't do this
await MainActor.run { doMainActorStuff() }

// Do this instead
@MainActor func doMainActorStuff() { }
```

Let isolation be part of the function signature, not scattered through your code.

### Mistake 4: Making everything Sendable

Not everything needs to be Sendable. If you're adding `@unchecked Sendable` everywhere, you're probably creating too many isolation boundaries.

### Mistake 5: Ignoring compiler warnings

Every Sendable warning is pointing to a potential data race. Don't suppress them - understand them. [Enable complete concurrency checking](https://www.massicotte.org/complete-checking/) to learn how Swift concurrency actually works.

  </div>
</section>

<section>
  <div class="container">

## The Mental Model Cheat Sheet

<div class="summary-grid">

<div class="summary-card">
<h4>Async/Await</h4>

**What it is:** Pause and resume without blocking.

**Mental model:** "I'll leave my number, call me when ready."

**Key insight:** `async` doesn't mean background.
</div>

<div class="summary-card">
<h4>Actors</h4>

**What it is:** A security guard for your data.

**Mental model:** "One visitor at a time."

**Key insight:** Don't overuse them.
</div>

<div class="summary-card">
<h4>MainActor</h4>

**What it is:** The main thread, but type-safe.

**Mental model:** "UI code lives here."

**Key insight:** Use it more than you think.
</div>

<div class="summary-card">
<h4>Sendable</h4>

**What it is:** A certificate saying "safe to share."

**Mental model:** "Can I hand this to another room?"

**Key insight:** Start non-Sendable, add only when needed.
</div>

</div>

  </div>
</section>

<section>
  <div class="container">

## Three Levels of Swift Concurrency

You don't need to learn everything at once. Progress through these levels:

### Level 1: Basic Async (Start Here)

- Use `async/await` for network calls
- Mark UI classes with `@MainActor`
- Use SwiftUI's `.task` modifier

This handles 80% of apps.

### Level 2: Structured Concurrency

- Use `async let` for parallel work
- Use `TaskGroup` for dynamic parallelism
- Understand task cancellation

For when you need performance.

### Level 3: Advanced Safety

- Create custom actors for shared state
- Deep understanding of Sendable
- Custom executors

For library authors and complex systems.

<div class="tip">
<h4>Start simple</h4>

Most apps never need Level 3. Don't over-engineer.
</div>

  </div>
</section>

<section>
  <div class="container">

## Quick Reference

### Making Things Work on Main Thread

```swift
// Entire type on main thread
@MainActor
class MyViewModel { }

// Single function on main thread
@MainActor
func updateUI() { }

// One-off main thread work (rarely needed)
await MainActor.run {
    // UI updates here
}
```

### Running Work in Parallel

```swift
// Fixed number of parallel tasks
async let a = fetchA()
async let b = fetchB()
let results = await (a, b)

// Dynamic number of parallel tasks
await withTaskGroup(of: Item.self) { group in
    for id in ids {
        group.addTask { await fetch(id) }
    }
    for await item in group {
        process(item)
    }
}
```

### Making Types Sendable

```swift
// Value types are usually Sendable automatically
struct MyData: Sendable {
    let id: Int
    let name: String
}

// Actors are Sendable (they protect themselves)
actor MyActor { }

// Classes need work - consider if you really need this
final class MyClass: Sendable {
    let immutableValue: Int  // Must be let, not var
}
```

  </div>
</section>

<section>
  <div class="container">

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
