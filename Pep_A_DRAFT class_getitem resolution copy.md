# PEP A Draft — TypeVar Resolution from `__class_getitem__` Signatures

> **Status:** Idea / Draft / Pre-proposal
> **Tested against:** Pyright (other checkers believed to share the same gap, unverified)
> **Related ideas:** PEP B (`__annotated_*__` protocol), PEP C (Val and value-level TypeVars), PEP D (Map and bounded TypeVarTuple)

---

## Abstract

Python type checkers already resolve TypeVars from `__init__` and `__new__` signatures — both from `T` (value) and `type[T]` (type) annotations. This proposal asks checkers to extend that same resolution logic to `__class_getitem__`, using the **actual declared signature** rather than an internally synthesized one.

This is a small, targeted change. All required machinery already exists in current checkers.

---

## Motivation

### The runtime already does this

Since [PEP 560](https://peps.python.org/pep-0560/) (Python 3.7), `__class_getitem__` is the official runtime mechanism for generic subscript expressions. When you write `Foo[int]`, Python calls `Foo.__class_getitem__(int)`. The method is a regular callable with a real signature.

Type checkers, however, do not validate subscript arguments against that signature. Instead, they synthesize their own understanding of what `Foo[...]` means based on the TypeVar declarations in `class Foo[T]`, ignoring `__class_getitem__` entirely.

This creates a gap: the runtime accepts whatever `__class_getitem__` accepts, but the checker enforces its own internal rules.

### Checkers already do this for `__init__` and `__new__`

The following already works in current type checkers (verified in Pyright):

```python
class Foo[T]:
    def __init__(self, t: type[T]) -> None: ...

class Bar[T]:
    def __init__(self, t: T) -> None: ...

x = Foo(int)   # T = int ✓
y = Bar(34)    # T = int ✓
```

Both `t: T` (resolve from value) and `t: type[T]` (resolve from type) are supported at construction time. This proposal asks for the exact same behavior at subscript time, via `__class_getitem__`.

### The current workaround is a hack

Library authors who want `Quantity[float, {L: 1, T: -1}]` to be a valid annotation are currently forced into patterns like:

```python
if TYPE_CHECKING:
    Quantity = Annotated
else:
    Quantity = QuantityClass
```

This is fragile, confusing, and misleading to users of the library. It exists solely because checkers do not honor the actual `__class_getitem__` signature.

---

## Proposal

### Core rule

When a class defines `__class_getitem__` explicitly, type checkers **must** validate subscript arguments against that method's signature, and **must** use it to resolve TypeVars — exactly as they currently do for `__init__` and `__new__`.

The checker reads and validates the declared signature statically. It does not execute __class_getitem__ — it analyzes its annotations the same way it analyzes __init__. No runtime execution is implied.

Resolution rules (consistent with current `__init__` behavior, verified in Pyright):

| Annotation | Argument | Resolves |
|---|---|---|
| `key: T` | a value `x` | `T = type(x)` |
| `key: type[T]` | a type `X` | `T = X` |

> **Note:** that the key: T path (resolve from value) is intentionally load-bearing for PEP C — it is the bridge to value-level inference, even though value-level TypeVars are out of scope here.

### Default behavior (no explicit `__class_getitem__`)

When a class does **not** define `__class_getitem__`, checkers continue to synthesize the signature from the TypeVar declaration, as today:

```python
class Foo[T]:       # implicit: __class_getitem__(cls, key: type[T])
class Bar[T, V]:    # implicit: __class_getitem__(cls, key: tuple[type[T], type[V]])
```

This is fully backward compatible — existing generic classes are unaffected.

### Explicit override

When `__class_getitem__` is declared, its signature takes precedence:

```python
class ValueRange[T: int | float]:
    def __class_getitem__(cls, key: tuple[type[T], T, T]) -> ValueRange[T]:
        t, lo, hi = key
        return cls(t, lo, hi)
```

The checker now validates:

```python
ValueRange[float, 0.0, 1.0]    # ✓ tuple[type[float], float, float]
ValueRange[float, "a", 1.0]    # ✗ "a" is not float
ValueRange[float, 0.0]         # ✗ tuple too short
```


## Examples

### Before (current state, Pyright)

```python
class Quantity[T]:
    def __class_getitem__(cls, key: tuple[type[T], dict]) -> Quantity[T]:
        ...

dx: Quantity[float, {L: 1, T: -1}] = 2.3
# Checker: error — Quantity takes 1 type argument, not 2
# Runtime: fine — __class_getitem__ handles this correctly
```

### After (proposed)

```python
class Quantity[T]:
    def __class_getitem__(cls, key: tuple[type[T], dict]) -> Quantity[T]:
        ...

dx: Quantity[float, {L: 1, T: -1}] = 2.3
# Checker: ✓ — tuple[type[float], dict] matches signature, T = float
```

### Existing behavior preserved

```python
class Foo[T]:
    pass  # no explicit __class_getitem__

x: Foo[int]  # unchanged — synthesized signature still used
```

---

## Backward Compatibility

This change is fully backward compatible:

- Classes without explicit `__class_getitem__` are unaffected.
- Classes with explicit `__class_getitem__` currently have their subscript arguments **ignored** by checkers. Checkers moving to validate them may surface new errors in previously unchecked code — but those are real errors, not regressions.
- The annotation-wins rule ensures construction-time inference does not break existing annotated code.

---

## Design Decisions

### 1. Unresolved TypeVars

If `__class_getitem__` is defined but does not mention `T`, the TypeVar remains unresolved at subscript time. `__init__` and `__new__` can still resolve it. This is consistent with how checkers treat `__init__` today — it is the library author's responsibility to ensure TypeVars are resolvable somewhere.

### 2. Return type of `__class_getitem__`

Checkers should not require `Foo[T]` as the return type. Library authors use varied runtime patterns, and the return type is relevant primarily for MRO entry validation (via `__mro_entries__`), which is currently unchecked in Pyright and is out of scope for this proposal.

> **Note:** Whether a `__class_getitem__` return type is valid as an MRO entry (i.e. implements `__mro_entries__`) is an open gap in checker support, independent of this proposal.

### 3. Inheritance and `__class_getitem__`

This is the most subtle design area. The relevant checker rule is simple, but it is worth explaining why.

**The checker rule (declaration-based):**

> If a class declares TypeVars — via `class Foo[T]` syntax or by inheriting from `Generic[T, ...]` — the checker assumes a fresh `__class_getitem__` exists on that class, shadowing any inherited concrete definition.

**Why this rule exists:**

When `Bar` inherits from a concrete `Foo` that defines `__class_getitem__`, the runtime does *not* automatically inject a fresh one onto `Bar`. `Generic` itself does not use `__init_subclass__` injection — it relies on metaclass machinery (`_GenericAlias`). This means `Foo.__class_getitem__` wins the MRO race and is called with `Bar`'s subscript arguments, producing incorrect behavior:

```python
class Foo(Generic[T]):
    def __class_getitem__(cls, t: type[T]) -> "Foo[T]":
        return Foo(t)

class Bar(Foo[T], Generic[T, V]): ...

Bar[int, int]
# Runtime: calls Foo.__class_getitem__ with (int, int) — wrong
# Returns: a Foo instance, not a Bar generic alias
```

This is a pre-existing CPython bug traceable to PEP 560, not a semantic contract to honor. Checkers should model *intended* semantics. The declaration-based rule does this cleanly without requiring runtime simulation.

**Inheritance behavior table:**

| Declaration | Checker behavior |
|---|---|
| `class Bar(Foo[T])` | Inherits `Foo.__class_getitem__` — no TypeVar declaration on `Bar` |
| `class Bar[T](Foo[T])` | TypeVar declared → fresh `__class_getitem__` assumed, `Foo`'s shadowed |
| `class Bar[T, V](Foo[T])` | Same |
| `class Bar(Foo[T], Generic[T])` | `Generic[T]` counts as TypeVar declaration → fresh injection assumed |
| `class Bar(Foo, Generic[T])` | Same |

**Explicit delegation:**

Library authors who genuinely want to delegate to the parent's `__class_getitem__` can do so explicitly via `super()`:

```python
class Bar[T, V](Foo[T]):
    def __class_getitem__(cls, key):
        return super().__class_getitem__(key)
```

This makes intent visible — explicit over implicit, consistent with Python's general philosophy.

---

## Interactions with Related Ideas

- **PEP B** (`__annotated_*__` protocol): Once checkers honor `__class_getitem__` signatures, `__annotated_type__` becomes a clean way to declare wrapper semantics without the `TYPE_CHECKING` hack. PEP A is a prerequisite for PEP B.
- **PEP C** (`Val` and value-level TypeVars): The `key: T` resolution path (value, not type) established here is the foundation for value-level generic inference.
- **PEP D** (`Map` and bounded `TypeVarTuple`): Extends the same resolution logic to variadic generics.

---

## Prior Art

- **PEP 560** — introduced `__class_getitem__` as the official runtime mechanism (Python 3.7)
- **PEP 695** — introduced `class Foo[T]` syntax (Python 3.12), which desugars to `Generic[T]` + `__class_getitem__`
- `dataclasses.InitVar.__class_getitem__` — standard library example of a class using `__class_getitem__` to return a non-generic instance; currently requires special-case knowledge in checkers

---

## Why Checkers Don't Do This Today

Type checkers predate PEP 560. They built their own internal representation of generic classes before `__class_getitem__` existed, and accumulated special cases for builtins (`list`, `dict`, `Optional`, `Final`, etc.) over time. The gap is a historical artifact, not a deliberate design choice.

As of the time of writing, Pyright parses `__class_getitem__` and recognizes `cls` as the first argument, but does not use the method's signature for subscript validation or TypeVar resolution. The delta between current behavior and this proposal is narrow — the method is already parsed; it just needs to be used.