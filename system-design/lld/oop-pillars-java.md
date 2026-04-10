# OOP Pillars — Encapsulation, Abstraction, Inheritance, Polymorphism

## Table of Contents

1. [Why OOP Matters in System Design Interviews](#1-why-oop-matters-in-system-design-interviews)
2. [The Four Pillars — Overview](#2-the-four-pillars--overview)
3. [Encapsulation](#3-encapsulation)
4. [Abstraction](#4-abstraction)
5. [Inheritance](#5-inheritance)
6. [Polymorphism](#6-polymorphism)
7. [How the Pillars Interact](#7-how-the-pillars-interact)
8. [SOLID Principles (OOP's practical extension)](#8-solid-principles-oops-practical-extension)
9. [OOP vs. Other Paradigms](#9-oop-vs-other-paradigms)
10. [What Interviewers Actually Evaluate](#10-what-interviewers-actually-evaluate)
11. [Anti-Patterns & Pitfalls](#11-anti-patterns--pitfalls)
12. [Quick-Reference Card](#12-quick-reference-card)
13. [References](#13-references)

---

## 1. Why OOP Matters in System Design Interviews

OOP questions appear in two interview contexts:

1. **LLD (Low-Level Design):** "Design a parking lot", "Design a chess game", "Design an elevator system." Here you must model the domain with classes, relationships, and interfaces.
2. **Conceptual round:** "Explain polymorphism with an example." "How would you use encapsulation in a payment system?"

The interviewer is evaluating:
- Whether you understand the *why* behind OOP, not just definitions
- Whether your design decisions reflect these principles naturally
- Whether you can critique a design ("this violates encapsulation because...")

---

## 2. The Four Pillars — Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                     THE FOUR OOP PILLARS                         │
├────────────────┬─────────────────────────────────────────────────┤
│ Encapsulation  │ Bundle state + behavior; hide internal details   │
│                │ → "What you CAN'T see"                          │
├────────────────┼─────────────────────────────────────────────────┤
│ Abstraction    │ Expose only what's needed; hide complexity       │
│                │ → "What you SHOULD see"                         │
├────────────────┼─────────────────────────────────────────────────┤
│ Inheritance    │ Derive new types from existing ones              │
│                │ → "What you ARE"                                 │
├────────────────┼─────────────────────────────────────────────────┤
│ Polymorphism   │ One interface, many implementations              │
│                │ → "What you DO"                                  │
└────────────────┴─────────────────────────────────────────────────┘
```

> **Memory anchor:** **E**very **A**wesome **I**dea **P**ays — Encapsulation, Abstraction, Inheritance, Polymorphism.

---

## 3. Encapsulation

### Concept

Encapsulation means **bundling data (fields) and the methods that operate on that data into a single unit (class)**, and **restricting direct access to internal state** from outside that unit.

It's the principle behind `private` fields and `public` methods.

### Why it exists

Without encapsulation, any part of the codebase can reach into an object and corrupt its state. Encapsulation enforces invariants.

```
WITHOUT encapsulation:              WITH encapsulation:
──────────────────────              ──────────────────
BankAccount account = ...;          BankAccount account = ...;
account.balance = -500;  ← WRONG!   account.withdraw(500);  ← Validated!
```

### Internals — how it works at the language level

- Access modifiers control visibility: `private`, `protected`, `package-private`, `public`
- Getters/setters (accessors/mutators) provide controlled read/write access
- The class maintains invariants: e.g., balance can never be negative

### Java example

```java
public class BankAccount {
    private double balance;         // hidden state
    private String accountId;       // hidden state

    public BankAccount(String id, double initialBalance) {
        if (initialBalance < 0) throw new IllegalArgumentException("Negative balance");
        this.accountId = id;
        this.balance = initialBalance;
    }

    // Controlled mutation — invariant enforced here
    public void withdraw(double amount) {
        if (amount > balance) throw new InsufficientFundsException();
        this.balance -= amount;
    }

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Invalid deposit");
        this.balance += amount;
    }

    // Read-only access — returns a copy, not the field itself
    public double getBalance() {
        return this.balance;
    }
}
```

### Trade-offs & pitfalls

| Benefit | Pitfall |
|---|---|
| Prevents invalid state | Excessive getters/setters = anemic domain model (encapsulation in name only) |
| Isolates change — internals can evolve without affecting callers | Over-hiding can make testing harder (use package-private or test helpers) |
| Enforces invariants at one place | Breaking encapsulation across a service boundary (returning mutable collections) |

**Anemic domain model anti-pattern:**
```java
// BAD — this is just a struct with extra steps:
public class Order {
    private List<Item> items;
    public void setItems(List<Item> items) { this.items = items; }  // ← raw setter
    public List<Item> getItems() { return items; }                  // ← mutable ref
}

// GOOD — rich domain model:
public class Order {
    private List<Item> items = new ArrayList<>();
    public void addItem(Item item) {
        validateItem(item);
        items.add(item);
    }
    public List<Item> getItems() {
        return Collections.unmodifiableList(items);  // ← immutable view
    }
}
```

### Interview signal

> "Encapsulation ensures the `BankAccount` class is the sole authority over the balance field. External code can't corrupt the account state — it can only call `withdraw()` and `deposit()`, which enforce business rules. This is why I'd never expose `balance` as public, even for convenience."

---

## 4. Abstraction

### Concept

Abstraction means **hiding complexity behind a simple, clean interface**. Users of an abstraction don't need to know how it works — only what it does.

Abstraction operates at the design level (interfaces, abstract classes, APIs), while encapsulation operates at the implementation level (access control).

```
                     ┌──────────────────────────┐
  Caller sees:  ──►  │  PaymentProcessor        │
                     │  + processPayment(amount)│  ← simple interface
                     └──────────┬───────────────┘
                                │ hides:
                     ┌──────────▼───────────────┐
                     │  Stripe SDK calls        │
                     │  Retry logic             │
                     │  Error normalization     │
                     │  PCI-DSS tokenization    │
                     │  Webhook handling        │
                     └──────────────────────────┘
```

### Mechanisms in Java/Python

- **Abstract classes:** Partial implementation, enforce contract on subclasses
- **Interfaces:** Pure contract (no state) — defines *what* without *how*
- **Access modifiers:** Hide implementation details (links to encapsulation)

### Java example

```java
// The abstraction — caller depends only on this
public interface PaymentProcessor {
    PaymentResult processPayment(PaymentRequest request);
    void refund(String transactionId, double amount);
}

// One concrete implementation — complexity hidden here
public class StripePaymentProcessor implements PaymentProcessor {
    private final StripeClient stripeClient;

    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        // ... 50 lines of Stripe-specific logic, retries, token handling
    }

    @Override
    public void refund(String transactionId, double amount) {
        // ... Stripe refund API calls
    }
}

// Another concrete implementation — caller doesn't care which
public class PayPalPaymentProcessor implements PaymentProcessor {
    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        // ... PayPal-specific logic
    }
    // ...
}

// Calling code — completely decoupled from implementation
public class OrderService {
    private final PaymentProcessor paymentProcessor;  // ← depends on abstraction

    public OrderService(PaymentProcessor processor) {
        this.paymentProcessor = processor;
    }

    public void checkout(Order order) {
        paymentProcessor.processPayment(new PaymentRequest(order.getTotal()));
    }
}
```

### Abstraction vs Encapsulation — the critical distinction

This is the most commonly confused pair. They are complementary, not synonymous.

```
Encapsulation:                      Abstraction:
────────────────────────────        ────────────────────────────
HOW something is hidden             WHAT is exposed to the caller
Access control (private/public)     Interface design (what methods exist)
Implementation-level concern        Design-level concern

Example: balance is private         Example: PaymentProcessor interface
  → encapsulation                     → abstraction
```

> Encapsulation = information hiding. Abstraction = complexity hiding.

### Trade-offs

| Benefit | Pitfall |
|---|---|
| Swap implementations without changing callers | Leaky abstraction — implementation details bleed through the interface |
| Simplifies the mental model for callers | Over-abstraction — abstracting before you have 2+ concrete cases |
| Enables testing with mocks/stubs | Wrong abstraction level — too fine-grained or too coarse |

**Leaky abstraction example:**
```java
// BAD — the method name reveals internal implementation:
interface DataStore {
    void flushToDisk();  // ← caller now knows it's disk-backed. What if we switch to S3?
}

// GOOD — implementation-agnostic:
interface DataStore {
    void persist();
}
```

---

## 5. Inheritance

### Concept

Inheritance lets a **child class (subclass) derive fields and methods from a parent class (superclass)**, enabling code reuse and expressing an "is-a" relationship.

```
             ┌───────────────────┐
             │     Animal        │
             │ ─────────────────-│
             │ + name: String    │
             │ + eat()           │
             │ + sleep()         │
             │ + makeSound()*    │  ← abstract — subclass must implement
             └─────────┬─────────┘
              ┌────────┴────────┐
    ┌─────────▼───────┐   ┌─────▼───────────┐
    │      Dog        │   │      Cat         │
    │ ────────────── -│   │ ─────────────────│
    │ + fetch()       │   │ + purr()         │
    │ + makeSound()   │   │ + makeSound()    │
    │   "Woof!"       │   │   "Meow!"        │
    └─────────────────┘   └─────────────────┘
```

### Types of inheritance

```
Single:      A → B               One parent
Multi-level: A → B → C           Chain of parents
Multiple:    A, B → C            Multiple parents (Java: via interfaces only)
Hierarchical: A → B, A → C       One parent, multiple children
Hybrid:      Combination         Java avoids this with interface default methods
```

### Java example

```java
public abstract class Vehicle {
    protected String make;
    protected int year;
    protected double fuelLevel;

    public Vehicle(String make, int year) {
        this.make = make;
        this.year = year;
        this.fuelLevel = 100.0;
    }

    // Concrete method — inherited as-is
    public void refuel(double amount) {
        fuelLevel = Math.min(100.0, fuelLevel + amount);
    }

    // Abstract method — subclass MUST override
    public abstract double calculateRange();

    // Template method pattern — skeleton defined here
    public void startJourney() {
        System.out.println("Starting " + make);
        performPreCheck();              // hook — subclass can override
        System.out.println("Range: " + calculateRange() + " km");
    }

    protected void performPreCheck() { /* default: no-op */ }
}

public class ElectricCar extends Vehicle {
    private double batteryCapacityKwh;

    public ElectricCar(String make, int year, double capacityKwh) {
        super(make, year);  // ← calls parent constructor
        this.batteryCapacityKwh = capacityKwh;
    }

    @Override
    public double calculateRange() {
        return batteryCapacityKwh * fuelLevel / 100.0 * 6;  // 6 km/kWh
    }

    @Override
    protected void performPreCheck() {
        System.out.println("Checking battery systems...");
    }
}
```

### The Liskov Substitution Principle (LSP)

This is the most important inheritance rule. LSP states:

> **Any code that uses a parent type must work correctly if given a subtype.**

```java
// If this works:
void processVehicle(Vehicle v) {
    v.startJourney();
}

// Then this must also work — without exceptions or behavioral surprises:
processVehicle(new ElectricCar("Tesla", 2024, 75));
processVehicle(new GasCar("Toyota", 2023, 50));
```

**Classic LSP violation — the Rectangle/Square problem:**
```java
class Rectangle {
    protected int width, height;
    public void setWidth(int w) { width = w; }
    public void setHeight(int h) { height = h; }
    public int area() { return width * height; }
}

class Square extends Rectangle {
    @Override
    public void setWidth(int w) { width = height = w; }  // ← violates LSP!
    @Override
    public void setHeight(int h) { width = height = h; } // ← surprising behavior
}

// This breaks with a Square:
void testArea(Rectangle r) {
    r.setWidth(5);
    r.setHeight(4);
    assert r.area() == 20;  // FAILS for Square — area is 16
}
```

**Fix:** Square should not extend Rectangle. Instead, model them as siblings sharing a Shape interface.

### Composition over Inheritance

The most important trade-off in OOP design:

```
Inheritance: "is-a" — a Dog IS AN Animal
Composition: "has-a" — a Car HAS AN Engine

Inheritance problems:
  - Tight coupling to parent class
  - Fragile base class problem (parent change breaks children)
  - Deep hierarchies become unmaintainable
  - Forces all-or-nothing: you get the whole parent

Composition benefits:
  - Loose coupling — swap components independently
  - Mix-and-match behaviors
  - Easier to test in isolation
  - Follows Dependency Inversion Principle

Rule of thumb: Favor composition. Use inheritance only when the
"is-a" relationship is stable and the LSP holds cleanly.
```

```java
// Inheritance approach — brittle:
class LoggingOrderService extends OrderService {
    @Override
    public void processOrder(Order o) {
        log("Processing order");
        super.processOrder(o);
    }
}

// Composition approach — flexible:
class OrderService {
    private final Logger logger;          // ← injected
    private final PaymentProcessor payment;

    public OrderService(Logger logger, PaymentProcessor payment) {
        this.logger = logger;
        this.payment = payment;
    }
}
```

---

## 6. Polymorphism

### Concept

Polymorphism means **one interface, many implementations**. The word comes from Greek: "many forms." It allows you to write code that works with objects of different types through a common interface.

There are two distinct types:

```
┌────────────────────────────────────────────────────────────────┐
│                      POLYMORPHISM                              │
├───────────────────────────────┬────────────────────────────────┤
│  Compile-time (Static)        │  Runtime (Dynamic)             │
│  Method Overloading           │  Method Overriding             │
│                               │                                │
│  Same name, different params  │  Same signature, different     │
│  Resolved by compiler         │  class; resolved by JVM        │
│  (ad-hoc polymorphism)        │  (subtype polymorphism)        │
│                               │                                │
│  add(int, int)                │  animal.makeSound()            │
│  add(double, double)          │  → Dog: "Woof"                 │
│  add(String, String)          │  → Cat: "Meow"                 │
└───────────────────────────────┴────────────────────────────────┘
```

### Runtime polymorphism — how it works internally

The JVM uses a **Virtual Method Table (vtable)** — a lookup table per class that maps method names to their actual implementations.

```
Object reference → vtable pointer → correct method implementation
                                     (resolved at runtime, not compile time)

Animal ref → Dog vtable → Dog.makeSound()
Animal ref → Cat vtable → Cat.makeSound()
```

This is why `Animal animal = new Dog(); animal.makeSound();` calls `Dog.makeSound()` — the vtable of the actual object (`Dog`) is used, not the declared type (`Animal`).

### Java example — compile-time (overloading)

```java
public class Calculator {
    // Same method name, different parameter types — resolved at compile time
    public int add(int a, int b) {
        return a + b;
    }

    public double add(double a, double b) {
        return a + b;
    }

    public String add(String a, String b) {
        return a + b;
    }

    public int add(int a, int b, int c) {
        return a + b + c;
    }
}
```

### Java example — runtime (overriding)

```java
public abstract class Shape {
    public abstract double area();
    public abstract double perimeter();

    // Common behavior — uses polymorphic method calls internally
    public void describe() {
        System.out.printf("Area: %.2f, Perimeter: %.2f%n", area(), perimeter());
    }
}

public class Circle extends Shape {
    private double radius;

    @Override
    public double area() { return Math.PI * radius * radius; }

    @Override
    public double perimeter() { return 2 * Math.PI * radius; }
}

public class Rectangle extends Shape {
    private double width, height;

    @Override
    public double area() { return width * height; }

    @Override
    public double perimeter() { return 2 * (width + height); }
}

// The power: process all shapes uniformly
public void renderAll(List<Shape> shapes) {
    for (Shape s : shapes) {
        s.describe();  // correct implementation called for each — no instanceof!
    }
}
```

### Interface-based polymorphism (most important for system design)

```java
// Multiple unrelated classes implement the same interface
public interface Serializable {
    byte[] serialize();
    static <T> T deserialize(byte[] data, Class<T> type) { ... }
}

// Objects with no inheritance relationship behave uniformly
public class UserProfile implements Serializable { ... }
public class OrderEvent    implements Serializable { ... }
public class MetricPoint   implements Serializable { ... }

// A single method handles all of them
public void publish(Serializable event) {
    byte[] bytes = event.serialize();
    kafka.send(bytes);
}
```

### Polymorphism vs switch statements

Polymorphism replaces `if/else` and `switch` chains that check type:

```java
// BAD — violates Open/Closed Principle, grows with every new type:
public double calculateArea(Shape s) {
    if (s instanceof Circle) {
        return Math.PI * ((Circle) s).getRadius() * ((Circle) s).getRadius();
    } else if (s instanceof Rectangle) {
        return ((Rectangle) s).getWidth() * ((Rectangle) s).getHeight();
    } else if (s instanceof Triangle) {
        // ...
    }
    // add another case for every new shape — fragile!
}

// GOOD — closed for modification, open for extension:
public double calculateArea(Shape s) {
    return s.area();  // each subclass handles itself
}
```

---

## 7. How the Pillars Interact

The four pillars are not independent — they work together:

```
                  ┌────────────────────────────────────────┐
                  │           A CLASS DEFINITION            │
                  │                                         │
                  │  class StripeProcessor                  │
                  │    implements PaymentProcessor {        │
                  │                       ▲                 │
   ABSTRACTION ───┼─── interface defines  │                 │
   defines the    │    the contract;      │                 │
   contract       │    caller sees only   │                 │
                  │    processPayment()   │                 │
                  │                                         │
                  │    private apiKey;    ◄── ENCAPSULATION  │
                  │    private retryCount;    hides internal │
                  │    private httpClient;    state from     │
                  │                          callers        │
                  │    @Override                            │
                  │    processPayment(req) {                │
                  │      // Stripe-specific code            │
                  │    }    ▲                               │
   INHERITANCE ───┼─────────┘                               │
   StripeProcessor          Stripe-specific override        │
   inherits common          is POLYMORPHISM at              │
   retry logic from         runtime ──────────────── ──────┘
   BaseProcessor
                  └────────────────────────────────────────┘
```

---

## 8. SOLID Principles (OOP's practical extension)

These are the natural evolution of the four pillars into actionable design guidelines.

```
S — Single Responsibility Principle
      A class should have only one reason to change.
      "A class that handles DB + email + formatting is doing too much."

O — Open/Closed Principle
      Open for extension, closed for modification.
      "Add a new payment type by adding a class, not editing existing ones."
      → Achieved via polymorphism + abstraction

L — Liskov Substitution Principle
      Subtypes must be substitutable for their base types.
      "A Square extending Rectangle breaks this if it changes setWidth behavior."
      → The correctness test for inheritance

I — Interface Segregation Principle
      No client should depend on methods it doesn't use.
      "Don't put print(), scan(), fax() in one Printer interface — split them."

D — Dependency Inversion Principle
      Depend on abstractions, not concretions.
      "OrderService should depend on PaymentProcessor interface, not StripeClient."
      → Achieved via abstraction + encapsulation
```

---

## 9. OOP vs. Other Paradigms

| Concern | OOP | Functional | Procedural |
|---|---|---|---|
| State management | Encapsulated in objects | Immutable values, no shared state | Global/local variables |
| Code reuse | Inheritance, composition | Higher-order functions, currying | Functions, include files |
| Extensibility | Polymorphism, open/closed | Typeclasses (Haskell), protocols | Function pointers |
| Concurrency | Challenging (shared mutable state) | Natural (immutable = thread-safe) | Difficult |
| Modeling real-world domains | Excellent (entities as objects) | Less natural | Moderate |
| Testability | Good with DI + interfaces | Excellent (pure functions) | Moderate |

**Modern practice:** Most production systems mix paradigms. Java/Kotlin use OOP structure with functional streams/lambdas. Scala/Kotlin blend both. The key is using the right tool for the right problem.

---

## 10. What Interviewers Actually Evaluate

### In LLD rounds ("Design a parking lot"):

```
Dimension                   What they look for
────────────────────────────────────────────────────────────────
Encapsulation               Private fields with meaningful methods.
                            No public setters for things that shouldn't change.

Abstraction                 Interfaces for major behaviors (ParkingStrategy,
                            PaymentHandler). Caller code depends on interfaces.

Inheritance                 Sensible "is-a" hierarchies (Vehicle → Car, Bike).
                            Not overusing — flat or shallow hierarchies.

Polymorphism                Processing different types uniformly.
                            No instanceof chains. Strategy pattern used.

SOLID                       SRP: each class has one job.
                            OCP: add vehicle types without editing ParkingLot.
                            DIP: ParkingLot depends on abstractions.

Design patterns             Strategy, Factory, Observer used where natural.
                            Not forced in where they don't fit.
```

### Red flags:

- Putting all logic in one `Main` or `ParkingLot` class
- Using `instanceof` chains instead of polymorphism
- Public fields on domain objects
- Inheriting just to reuse a utility method (use composition instead)
- Naming interfaces `IPaymentProcessor` (Java convention is not to use "I" prefix)

---

## 11. Anti-Patterns & Pitfalls

```
Anti-pattern              Description                     Fix
────────────────────────────────────────────────────────────────────
God Object               One class knows/does everything   SRP: split by responsibility

Anemic Domain Model      Objects are just data bags;       Move business logic INTO
                         all logic is in service classes   the domain objects

Yo-yo Problem            Deep inheritance hierarchy;       Flatten hierarchy; use
                         must trace many levels to         composition
                         understand a method

Fragile Base Class       Changing a parent breaks         Seal/final classes where
                         subclasses in subtle ways        appropriate; prefer composition

instanceof Abuse         Long if-instanceof chains         Polymorphism / visitor pattern

Leaky Abstraction        Interface exposes implementation  Redesign interface; hide
                         details (SQL-specific method      all tech details
                         on a generic Repository)

Refused Bequest          Subclass inherits methods it     LSP violation — remove from
(LSP violation)          doesn't want / can't implement   hierarchy or use interface
```

---

## 12. Quick-Reference Card

```
╔══════════════════════════════════════════════════════════════════╗
║              OOP PILLARS — QUICK REFERENCE                       ║
╠══════════════════════════════════════════════════════════════════╣
║  ENCAPSULATION                                                   ║
║    - Private fields, public methods                              ║
║    - Class enforces its own invariants                           ║
║    - Return copies / immutable views, not raw references         ║
║    - ≠ just adding getters/setters (that's an anemic model)      ║
║                                                                  ║
║  ABSTRACTION                                                     ║
║    - Interfaces > abstract classes (unless sharing state)        ║
║    - Program to the interface, not the implementation            ║
║    - Interface should be stable; implementation can change       ║
║    - Avoid leaky abstractions — hide all tech details            ║
║                                                                  ║
║  INHERITANCE                                                     ║
║    - Use only for genuine "is-a" relationships                   ║
║    - Always verify LSP: can subtype replace supertype safely?    ║
║    - Prefer composition for code reuse                           ║
║    - Keep hierarchies shallow (max 2–3 levels)                   ║
║                                                                  ║
║  POLYMORPHISM                                                    ║
║    - Prefer runtime (override) > compile-time (overload)         ║
║    - Eliminate instanceof chains — use polymorphism              ║
║    - Strategy pattern = polymorphism at its best                 ║
║    - Understand vtable: runtime dispatch on actual type          ║
║                                                                  ║
║  SOLID REMINDER                                                  ║
║    S: One reason to change per class                             ║
║    O: Extend via new classes, not edits                          ║
║    L: Subtypes safely replace supertypes                         ║
║    I: Small, focused interfaces                                  ║
║    D: Depend on abstractions                                     ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 13. References

- [Head First Design Patterns — Freeman & Robson](https://www.oreilly.com/library/view/head-first-design/0596007124/) — best intro to OOP patterns
- [Clean Code — Robert C. Martin](https://www.oreilly.com/library/view/clean-code-a/9780136083238/) — practical application of OOP principles
- [Effective Java — Joshua Bloch (3rd ed.)](https://www.oreilly.com/library/view/effective-java-3rd/9780134686097/) — items on composition, inheritance, interfaces
- [Refactoring — Martin Fowler](https://refactoring.com/) — recognizing and fixing OOP anti-patterns
- [SOLID Principles — Robert C. Martin's original papers](https://web.archive.org/web/20150906155800/http://www.objectmentor.com/resources/articles/Principles_and_Patterns.pdf)
- [The Liskov Substitution Principle — Barbara Liskov, 1987](https://www.cs.cmu.edu/~wing/publications/LiskovWing94.pdf)
- [Java OOP Tutorial — Oracle Docs](https://docs.oracle.com/javase/tutorial/java/concepts/index.html)