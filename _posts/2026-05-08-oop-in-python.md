---
title: "OOP in Python"
date: 2026-04-30 00:00:00 +0000
categories: [oop]
tags: [oop]
---

# Goals, Principles, and Patterns

As the name suggests (Object-Oriented Programming), the main actors are **objects**. Each object is an instance of a **class**.

---

## Object-Oriented Design Goals

Software systems aim to achieve three main goals:
- Robustness
- Adaptability
- Reusability

---

### Robustness

We want software that works correctly and handles unexpected situations gracefully, without crashing or breaking other parts of the system.

---

### Adaptability

Software should be flexible enough to adapt when requirements change, without requiring major rewrites of existing code.

---

### Reusability

We aim to build software or modules that can be reused in other systems, reducing development time and cost.

---

# Object-Oriented Design Principles

The main principles that help achieve these goals are:
- Modularity
- Abstraction
- Encapsulation

---

## Modularity

Modularity is the practice of breaking a large system into smaller, independent components called modules. Each module handles a specific responsibility.

In :contentReference[oaicite:1]{index=1}, a module is simply a file containing related functions, classes, or variables. For example, the `math` module provides mathematical functions, while the `os` module provides operating system utilities.

Modules allow code reuse, better organization, and easier maintenance by separating concerns such as authentication, payments, or notifications into independent components.

---

## Abstraction

Abstraction simplifies complex systems by exposing only essential features while hiding implementation details. It focuses on *what* a system does rather than *how* it does it.

This leads to **Abstract Data Types (ADTs)**, which define:
- the type of data stored
- the operations supported
- expected behavior of those operations

For example, a List ADT defines operations like add, remove, and search, without specifying how they are implemented internally.

In Python, abstraction is supported through **duck typing**, where an object is considered valid if it provides the required behavior. It is also formally supported using **Abstract Base Classes (ABCs)** via the `abc` module.

---

## Encapsulation

Encapsulation hides internal implementation details and exposes only a controlled interface. It ensures that objects are used through their defined methods rather than direct access to internal data.

This improves safety, maintainability, and flexibility, because internal implementation can change without affecting external code.

In Python, encapsulation is implemented through conventions. Attributes prefixed with `_` (e.g., `_balance`) are considered internal and should not be accessed directly.

---

## Too Much Theory, Let’s Now Implement

We will start with a class called `CreditCard`.

In OOP, a class is a primary tool for abstraction because it combines data and behavior into a single structure.

Before implementation, it is important to understand `self`. The `self` keyword refers to the current object instance, allowing access to its attributes and methods. Without `self`, Python would not know which object is being referenced.

---

## Implementation

```python
class CreditCard:
    """A consumer credit card"""

    def __init__(self, customer, bank, acnt, limit):
        """Create a new credit card instance.
        The initial balance is zero.
        """

        self._customer = customer
        self._bank = bank
        self._account = acnt
        self._limit = limit
        self._balance = 0

    def get_customer(self):
        """Return name of the customer"""
        return self._customer

    def get_bank(self):
        """Return name of the bank"""
        return self._bank
```