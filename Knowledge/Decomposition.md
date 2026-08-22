---
title: Decomposition
tags: [knowledge, cs, programming, design, thinking]
---

# 🔩 Decomposition

**Decomposition** is the process of **breaking a complex problem into smaller, manageable sub-problems** that are easier to understand, solve, and maintain. It's one of the four pillars of computational thinking alongside abstraction, pattern recognition, and algorithm design.

---

## 🧍 Decomposition for Humans

We decompose instinctively:

- **Planning a trip**: instead of "plan trip," you break it into → book flights, book hotel, pack bags, arrange transport.
- **Cooking a meal**: instead of "cook dinner," you break it into → chop vegetables, boil water, season meat, plate.
- **Writing an essay**: outline → sections → paragraphs → sentences.

The key insight: **a problem that feels overwhelming often becomes trivial once divided**.

---

## 💻 Decomposition in Programming

### Functional decomposition
Breaking a program's behavior into functions that each do one thing:

```python
# ❌ One giant function doing everything
def process_order(order):
    # validate
    # calculate total
    # apply discount
    # charge payment
    # send email
    # update inventory
    ...

# ✅ Decomposed
def process_order(order):
    validated = validate_order(order)
    total = calculate_total(validated)
    discounted = apply_discount(total)
    charge_payment(discounted)
    notify_customer(order)
    update_inventory(order)
```

Each function has a single responsibility — easier to test, debug, and reuse.

---

## 🏛️ Structural Decomposition

### Modular Architecture
At the file/module level, decomposition means separating concerns into distinct modules:

```
src/
├── auth/          # authentication concern
├── payments/      # billing concern
├── notifications/ # messaging concern
└── products/      # catalog concern
```

Each module hides its internals and exposes a clean interface to the rest of the system.

### Layered Architecture
Vertical decomposition by responsibility:

```
[Presentation Layer]  ← HTTP controllers, views
       ↓
[Application Layer]   ← use cases, business rules
       ↓
[Domain Layer]        ← entities, value objects
       ↓
[Infrastructure Layer] ← DB, external APIs, queues
```

Changes in one layer don't cascade into others.

### Microservices
The extreme form — decompose by **business capability** into independently deployable services:
- `auth-service` handles login/tokens
- `order-service` handles orders
- `notification-service` handles emails/SMS

Each service owns its data, scales independently, and can be deployed separately.

---

## 🔄 Decomposition vs Composition

These are two sides of the same coin:

| Decomposition | Composition |
|---------------|-------------|
| Break a big thing into small parts | Combine small parts into a big thing |
| Top-down thinking | Bottom-up thinking |
| "What are the pieces?" | "How do they connect?" |

In practice, you **decompose to understand**, then **compose to build**:

```javascript
// Decomposed parts
const validate = (data) => { ... }
const transform = (data) => { ... }
const save = (data) => { ... }

// Composed into a pipeline
const process = (data) => save(transform(validate(data)));
```

Function composition is so common it has dedicated utilities: `compose()` in Ramda, `pipe()` in functional libraries, method chaining in Laravel (`query->where()->orderBy()->get()`).

---

## 🧱 Principles That Guide Good Decomposition

### Single Responsibility Principle (SRP)
A function/class should have **one reason to change**. If you find yourself saying "and also," decompose further.

### Separation of Concerns (SoC)
Different concerns (logging, validation, business logic, persistence) should live in different places.

### Don't Repeat Yourself (DRY)
When you notice duplicate logic, that's a signal to extract and decompose it into a shared unit.

### High Cohesion, Low Coupling
- **High cohesion**: things in the same module are closely related
- **Low coupling**: modules depend on each other as little as possible

---

## 🎯 Decomposition in Problem Solving

When facing a hard problem, apply this process:

1. **State the full problem** clearly
2. **Identify natural boundaries** (data types, user actions, time phases)
3. **Split** into independent sub-problems
4. **Solve each part** in isolation
5. **Compose** the solutions

> "If I had an hour to solve a problem, I'd spend 55 minutes thinking about it and 5 minutes solving it." — often attributed to Einstein

Most of that thinking time is decomposition.

---

## 🔗 Relacionado
- [[Knowledge/Abstraction|Abstraction]] — decomposed parts are often abstracted behind interfaces
- [[Knowledge/Idempotence|Idempotence]] — decomposed units should have predictable, safe behavior
- [[DesignPatterns/DesignPatterns|Design Patterns]] — many patterns are formal decomposition strategies
- [[Frontend/Arquitectura de Modulos|Arquitectura de Módulos]] — module decomposition in frontend

## ↩️ Navegación
- [[Knowledge/Knowledge|🧠 Knowledge]] → [[HOME|🏠 HOME]]
