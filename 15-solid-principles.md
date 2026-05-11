# SOLID principles

Five OO design guidelines (Robert C. Martin) for maintainable modules: easier to test, extend, and reason about. They complement—not replace—domain modeling, **observability**, and **distributed systems** concerns.

## SOLID

Acronym: **S**ingle responsibility · **O**pen/closed · **L**iskov substitution · **I**nterface segregation · **D**ependency inversion. They push you toward small modules, stable abstractions, and depending on contracts instead of concrete implementations.

## Single responsibility principle

A module (class, function, file) should have **one reason to change**—one axis of responsibility (e.g. “parse CSV” vs “send email”), not “do everything for invoices.”

### Smell

`InvoiceService` that formats PDFs, hits HTTP APIs, and writes audit rows—changes to PDF layout force redeploy risk for API code.

### Example (split responsibilities)

```python
# persistence vs reporting — each changes for different product reasons
class InvoiceRepository:
    def save(self, invoice: Invoice) -> None: ...

class InvoicePdfRenderer:
    def render(self, invoice: Invoice) -> bytes: ...
```

## Open closed principle

Open for **extension**, closed for **modification**: add new behavior by plugging new types/implementations, not by editing a growing `if/elif` chain.

### Smell

```python
def area(shape):
    if shape.kind == "circle": ...
    elif shape.kind == "rect": ...
    # every new shape edits this function — merge conflicts, regressions
```

### Example (polymorphism)

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

class Circle(Shape):
    def __init__(self, r: float):
        self.r = r
    def area(self) -> float:
        return 3.14159 * self.r * self.r
```

New shapes add files/classes; callers iterate `Shape` without branching on string tags.

## Liskov substitution principle

Subtypes must be substitutable for their base types **without breaking callers’ expectations** (pre/postconditions, invariants). Violations often hide behind inheritance “is-a” that isn’t true in behavior.

### Classic bad example

`Square(Rectangle)` where `Rectangle` has `set_width` / `set_height` independently—`Square` cannot honor both without surprising side effects on the other dimension.

### Guideline

Prefer composition; if you inherit, ensure the base class contract (methods that can be called, errors raised, idempotency) still holds.

### Better approaches for “square vs rectangle”

**1. No `Square` subtype — a square *is* a rectangle with equal sides.** Mutate through one type whose contract is always “width and height are independent numbers ≥ 0.”

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Rectangle:
    width: float
    height: float

    def __post_init__(self) -> None:
        if self.width < 0 or self.height < 0:
            raise ValueError("dimensions must be non-negative")

    def area(self) -> float:
        return self.width * self.height

def square(side: float) -> Rectangle:
    return Rectangle(side, side)
```

Callers that only need `area()` (or `width`/`height`) never see a subtype that violates “set width doesn’t change height.”

**2. Sibling types behind a small protocol — no `Square(Rectangle)`.** Both implement `area()` (and anything else you truly share); you do not reuse a mutable `Rectangle` API for `Square`.

```python
from typing import Protocol
from dataclasses import dataclass

class HasArea(Protocol):
    def area(self) -> float: ...

@dataclass(frozen=True)
class Rectangle:
    width: float
    height: float
    def area(self) -> float:
        return self.width * self.height

@dataclass(frozen=True)
class Square:
    side: float
    def area(self) -> float:
        return self.side * self.side

def print_area(shape: HasArea) -> None:
    print(shape.area())
```

**3. If you need shared “resizable box” behavior**, model operations explicitly (e.g. `scale(factor: float)`, `with_width(w: float) -> Rectangle`) instead of `set_width` / `set_height` that mean different things for a square. Prefer **immutable** returns (`with_*`) so invariants stay obvious.

The anti-pattern is `class Square(Rectangle)` plus mutators that enforce `width == height` by silently changing the other side—that surprises code written against `Rectangle`.

## Interface segregation principle

Clients should not depend on methods they do not use. Prefer several small protocols/interfaces over one “god” interface implemented with `NotImplementedError`.

### Smell

```python
class Worker(ABC):
    @abstractmethod
    def work(self): ...
    @abstractmethod
    def save_to_disk(self): ...  # forces HTTP-only job to fake disk
```

### Example (split protocols)

```python
from typing import Protocol

class Runnable(Protocol):
    def run(self) -> None: ...

class Persistable(Protocol):
    def save(self) -> None: ...
```

## Dependency inversion principle

High-level policy should depend on **abstractions**, not low-level details. In practice: inject `EmailSender` interface; concrete SMTP adapter lives at the edge.

### Smell

```python
class OrderService:
    def confirm(self, order_id: int):
        import smtplib  # hard-wired infra — hard to test, swap provider
```

### Example (constructor injection)

```python
class OrderService:
    def __init__(self, mailer: "Mailer"):
        self._mailer = mailer

    def confirm(self, order_id: int):
        self._mailer.send(...)

class SmtpMailer:
    def send(self, to: str, body: str) -> None: ...
```

Tests pass a fake `Mailer`; production wires `SmtpMailer`.

### See also

[Microservices](./13-architecture-and-scaling.md#microservices) (boundaries), [Python web stack](./12-python-web-stack.md#fastapi) (FastAPI `Depends()` for DI).
