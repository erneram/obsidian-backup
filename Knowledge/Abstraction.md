---
title: Abstraction
tags: [knowledge, cs, programming, thinking]
---

# 🧠 Abstraction

Abstraction is the process of **hiding complexity behind a simpler interface** — focusing on *what* something does rather than *how* it does it. It exists both as a cognitive tool humans use naturally and as a foundational principle in programming.

---

## 🧍 Abstraction for Humans

Every time you interact with something without understanding its internals, you're using abstraction:

- You **drive a car** — you don't know how the combustion engine works, you just know: pedal → go, wheel → turn.
- You **flip a light switch** — you don't understand electrical circuits, you just know: switch up → light on.
- You **read a map** — the map is an abstraction of the real terrain. It hides irrelevant detail (every tree, every rock) and keeps only what matters for navigation.

### Why our brains need abstraction
The human brain can hold roughly **7 ± 2 chunks** of information in working memory (Miller's Law). Without abstraction, every task would require thinking about thousands of low-level details simultaneously — which is cognitively impossible at scale.

Abstraction lets you **compress complexity into a single mental token** so you can reason about it as a unit.

---

## 💻 Abstraction in Programming

In software, abstraction is the mechanism that lets you **build systems of increasing complexity without exploding in cognitive load**.

### Levels of abstraction (bottom → top)

```
Hardware (transistors, logic gates)
    ↓
Machine code (binary instructions)
    ↓
Assembly language (MOV, ADD, JMP)
    ↓
Low-level languages (C, Rust)
    ↓
High-level languages (Python, JavaScript)
    ↓
Frameworks & libraries (React, Laravel)
    ↓
Your application (the feature the user clicks)
```

At each level, the layer below is **hidden**. When you write `array.sort()` in Python, you're not thinking about quicksort or memory addresses — those are abstracted away.

---

## 🔑 Forms of Abstraction in Code

### 1. Functions / Methods
The most basic abstraction unit. Instead of repeating 10 lines of logic, you name them:
```python
# Without abstraction
tax = price * 0.15
total = price + tax

# With abstraction
def calculate_total(price):
    return price * 1.15
```

### 2. Classes & Objects
Bundles state + behavior under a name. You interact with a `User` object without knowing how its data is stored.

### 3. Interfaces & Contracts
Define *what* a component can do, not *how*. A `PaymentGateway` interface could have `charge(amount)` — the implementation (Stripe, PayPal) is hidden.
```typescript
interface PaymentGateway {
    charge(amount: number): Promise<Receipt>;
}
```

### 4. APIs
Every API is abstraction. You call `GET /users/1` and get a user object — the database query, joins, and serialization are invisible.

### 5. Data Structures
A `Queue` abstracts the concept of FIFO ordering. You don't care if it's implemented as a linked list or circular buffer.

---

## ⚖️ The Abstraction Trade-off

| Benefit | Cost |
|---------|------|
| Reduces cognitive load | Hides details that sometimes matter |
| Enables reuse | Can be over-engineered (over-abstraction) |
| Decouples components | Can introduce performance overhead |
| Makes code self-documenting | Leaky abstractions can be confusing |

> **Leaky abstraction**: when the hidden complexity "leaks" through the interface. Example: a database ORM that forces you to understand SQL to debug performance — the abstraction wasn't perfect.

---

## 🔗 Relacionado
- [[Knowledge/Decomposition|Decomposition]] — breaking problems into parts is a form of abstraction
- [[Knowledge/Idempotence|Idempotence]] — designing operations with clear contracts
- [[DesignPatterns/DesignPatterns|Design Patterns]] — most patterns are abstractions for recurring problems

## ↩️ Navegación
- [[Knowledge/Knowledge|🧠 Knowledge]] → [[HOME|🏠 HOME]]
