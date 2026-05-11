# Python syntax (cheatsheet)

Modern Python (3.10+) patterns seniors use daily: not a full tutorial, but quick recall for syntax and typing.

## Imports and modules

```python
from pathlib import Path
import asyncio
from collections.abc import Sequence, Mapping, Callable
```

Prefer `collections.abc` for abstract types in annotations (PEP 585 style: `list[int]` not `List[int]` in 3.9+).

## Type hints

```python
from typing import TypeVar

T = TypeVar("T")

def total(xs: list[float], *, scale: float = 1.0) -> float:
    return scale * sum(xs)

def first_or_none(xs: list[T] | tuple[T, ...]) -> T | None:
    return xs[0] if xs else None
```

`X | Y` unions (3.10+); older code may use `Optional[X]` / `Union[X, Y]`.

## match and case (structural pattern matching)

```python
def http_status_line(code: int) -> str:
    match code:
        case 200:
            return "ok"
        case 404 | 410:
            return "missing"
        case n if 500 <= n < 600:
            return "server error"
        case _:
            return "other"
```

```python
match point:
    case (x, y) if x == y:
        print("on diagonal")
    case {"x": x, "y": y}:
        print(x, y)
```

## Walrus operator

Assignment as an expression: `:=` — useful in `while` and comprehensions when you need the value twice.

```python
while (line := fp.readline()) != "":
    process(line)

if (m := re.search(r"\d+", s)):
    print(m.group())
```

## f-strings and formatting

```python
name = "Ada"
f"{name=!s}"        # name='Ada' (debug syntax)
f"{value:.2f}"      # float format
```

## Comprehensions and generators

```python
squares = [n * n for n in range(10) if n % 2 == 0]
gen = (n * n for n in range(10))  # lazy

nested = [[i * j for j in range(3)] for i in range(3)]
```

## Unpacking

```python
first, *rest, last = [1, 2, 3, 4, 5]
merged = {**defaults, **overrides}
def f(a, b, *, kwonly=1): ...
f(*pos_args, **kw_dict)
```

## Decorators

```python
from functools import wraps

def trace(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        print(fn.__name__)
        return fn(*args, **kwargs)
    return wrapper

@trace
def add(x: int, y: int) -> int:
    return x + y
```

Class decorators and `@dataclass` / `@property` / `@staticmethod` / `@classmethod` follow the same `def deco(f): ... return inner` idea.

## Context managers

```python
with open("x.txt") as f:
    data = f.read()

from contextlib import contextmanager

@contextmanager
def resource():
    acquire()
    try:
        yield value
    finally:
        release()
```

## dataclasses

`@dataclass` is a **class decorator** (stdlib `dataclasses`) that **generates boilerplate** from **annotated fields** on the class body. You write the fields once; Python fills in the repetitive parts.

### Compared to a plain `class`

For the same fields, a hand-written class usually needs an explicit `__init__`, `__repr__`, and often `__eq__` (and maybe `__hash__`, ordering, copy semantics). `@dataclass` can synthesize those so you focus on behavior and invariants.

**Rough equivalence (default `@dataclass`):**

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float = 0.0
```

…is in the same spirit as writing `__init__(self, x: float, y: float = 0.0)`, a readable `__repr__`, and `__eq__` that compare instances field-wise—without doing it by hand.

### What it can add (flags you opt into)

| Feature | Meaning |
|--------|---------|
| `frozen=True` | Instances immutable; `__hash__` can be generated if safe → usable as `dict` keys / `set` members |
| `slots=True` | `__slots__` on the class (smaller objects, faster attribute access; 3.10+ on `@dataclass`) |
| `order=True` | Rich comparison methods (`__lt__`, …) from field order |
| `kw_only=True` | All fields after the first must be passed as keywords in `__init__` (3.10+) |
| `unsafe_hash=True` | Force `__hash__` even when not frozen (rare; know why you need it) |

### `field()` and mutable defaults

Never use `meta: dict = {}` on the class—shared mutable default. Use `field(default_factory=dict)` so each instance gets a **new** dict.

```python
from dataclasses import dataclass, field

@dataclass(frozen=True, slots=True)
class Point:
    x: float
    y: float = 0.0
    meta: dict[str, str] = field(default_factory=dict)
```

`slots=True` (3.10+) reduces memory; `frozen=True` for hashable/immutable value objects.

### When you might skip `@dataclass`

Heavy custom `__init__` logic, non-standard descriptors on every field, or types where generated `__eq__`/`__repr__` are wrong—use a plain class (or a dataclass plus `__post_init__` for validation only).

## `__repr__` and `__eq__` (dunder methods)

“Dunder” = **double-underscore** names Python calls for built-in operations.

### `__repr__(self) -> str`

Used by **`repr(obj)`** and by the interactive interpreter when you echo a value. Goal: an **unambiguous, developer-oriented** string—ideally something you could paste back to recreate the object (“evaluable `__repr__`” is a guideline, not always possible).

```python
class Point:
    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y
    def __repr__(self) -> str:
        return f"Point(x={self.x!r}, y={self.y!r})"

repr(Point(1, 2))  # Point(x=1, y=2)
```

### `__str__(self) -> str` (related)

Used by **`print(obj)`** and **`str(obj)`**. If `__str__` is missing, Python falls back to **`__repr__`**. Often `__repr__` is technical; `__str__` can be shorter for end users.

### `__eq__(self, other)`

Defines behavior for **`==`**. If you cannot compare to `other`, return **`NotImplemented`** (not `False`) so Python can try the other type’s reflected `__eq__` or fall back to identity.

```python
class Point:
    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y
    def __eq__(self, other: object):
        if not isinstance(other, Point):
            return NotImplemented
        return self.x == other.x and self.y == other.y
```

`@dataclass` generates **`__eq__`** (field-wise) and **`__repr__`** by default so you do not hand-write these for simple data holders.

## Protocol (structural subtyping)

```python
from typing import Protocol

class Readable(Protocol):
    def read(self, n: int = -1) -> bytes: ...

def dump(src: Readable) -> None:
    ...
```

Implementations need not inherit—duck typing checked by type checkers.

## Enums

```python
from enum import Enum, auto

class Color(Enum):
    RED = auto()
    GREEN = auto()
```

## Exceptions

```python
try:
    ...
except ValueError as e:
    raise RuntimeError("bad") from e  # preserve chain
finally:
    cleanup()
```

## async def and await

```python
async def fetch(url: str) -> str:
    async with aiohttp.ClientSession() as s:
        async with s.get(url) as r:
            return await r.text()
```

See [Async](./04-async-concurrency-and-io.md#async) for event-loop semantics.

## __slots__

```python
class Row:
    __slots__ = ("id", "name")
    def __init__(self, id: int, name: str):
        self.id = id
        self.name = name
```

Restricts attributes → less `__dict__` overhead; incompatible with multiple inheritance quirks—use deliberately.

## pathlib

```python
from pathlib import Path
root = Path("/data")
for p in root.glob("**/*.csv"):
    text = p.read_text(encoding="utf-8")
```

## TypedDict and Literal

```python
from typing import TypedDict, Literal

class UserJson(TypedDict):
    id: int
    name: str

Mode = Literal["r", "w", "a"]
```

## ParamSpec and Concatenate (advanced generics)

For typing decorators that preserve wrapped function signatures—use when building generic middleware; see typing docs for `ParamSpec`, `Concatenate`, `TypeVarTuple`.
