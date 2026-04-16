# PEP B Draft — The `__annotated_*__` Protocol

> **Status:** Idea / Draft / Pre-proposal
> **Depends on:** PEP A (TypeVar resolution from `__class_getitem__` signatures)
> **Tested against:** Pyright (other checkers believed to share the same gap, unverified)
> **Related ideas:** PEP C (Val and value-level TypeVars), PEP D (Map and bounded TypeVarTuple)

---

## Abstract

Type checkers currently hardcode special behavior for a small set of typing constructs: `Final`, `Optional`, `ReadOnly`, `Required`, `InitVar`, and others. There is no mechanism for third-party library authors to declare equivalent semantics without writing a checker plugin.

This proposal introduces `__annotated_*__` — a namespace of class-level annotation dunders that checkers read to understand wrapper and modifier semantics. It is an open protocol: anyone can define `__annotated_*__` tags for their own use, and checkers read the ones they recognize. No central registry is required.

---

## Motivation

### The hardcoding problem

Consider what `Final`, `Optional`, and `ReadOnly` have in common:

```python
x: Final[int] = 42          # int, but immutable
y: Optional[int] = None     # int | None
z: ReadOnly[int]             # int, but no reassignment in TypedDict
```

Each of these is a *wrapper* — the annotation carries a type `T` plus some extra semantic. Checkers handle them by recognizing the symbol name and applying hardcoded rules. There is no way for a third-party class to declare the same intent.

This means library authors who want `Quantity[float, ...]` or `ValueRange[float, 0.0, 1.0]` to behave like a typed wrapper must either write a checker plugin, or resort to hacks:

```python
if TYPE_CHECKING:
    Quantity = Annotated
else:
    Quantity = QuantityClass
```

`Annotated[int, my_metadata]` requires the caller to attach metadata at every use site, and is limited to a fixed two-argument shape. `__annotated_*__` lets the class declare its semantics once, at definition time — and because it composes with `__class_getitem__`, the class author is free to choose any argument shape they want.

### The `__annotated_*__` solution

Instead of hardcoding symbol names, checkers should read a well-known namespace of class-level annotations. A class that declares `__annotated_type__` is saying: *"treat instances of me as wrapping this type."* A class that declares `__annotated_mutable__` is saying: *"tell checkers whether reassignment is allowed."*

Crucially, this is an **open namespace**. A checker can recognize `__annotated_type__` and `__annotated_mutable__` without any central registry of what tags exist. Third-party checkers or linters can define their own `__annotated_dim__`, `__annotated_repr__`, or `__annotated_range__` tags without coordinating with the Python typing community. Each checker reads the tags it cares about and ignores the rest.

This is the same philosophy as dunder methods: `__len__` does not require registration, it is simply a protocol the runtime reads.

---

## Proposal

### The namespace

`__annotated_*__` is a reserved annotation namespace on class bodies. Any class-level annotation whose name starts with `__annotated_` and ends with `__` participates in this protocol.

Tags are declared as **annotations**, not as instance attributes. A custom `__class_getitem__` may optionally set them as instance attributes at runtime for introspection, following the pattern already used by `dataclasses.InitVar`:

```python
class InitVar:
    def __class_getitem__(cls, type):
        return InitVar(type)  # sets self.type at runtime
```

Checkers read the annotation. Runtime reads the instance attribute. No conflict.

However this PEP formally specifies __annotated_type__ and __annotated_mutable__. Other tags are left to implementors.

### `__annotated_type__`

The primary tag. Declares that instances of this class wrap a type `T`, and should be treated as `T`-compatible by the checker.

```python
class Wrapper[T]:
    __annotated_type__: type[T]
```

With this declaration, a checker knows that `Wrapper[int]` is `int`-compatible — assignment, narrowing, and inference all behave as if the value is `T`.

**Standard library classes expressible with this tag:**

```python
class Final[T]:
    __annotated_type__: type[T]

class Optional[T]:
    __annotated_type__: type[T] | type[None]

class ReadOnly[T]:
    __annotated_type__: type[T]

class Required[T]:
    __annotated_type__: type[T]
```

These classes do not need to be redefined — this is illustrative. The point is that their checker behavior *could* be derived from `__annotated_type__` rather than hardcoded symbol recognition.

### `__annotated_mutable__`

Declares whether reassignment to a variable of this type is permitted after binding.

```python
class Final[T]:
    __annotated_type__: type[T]
    __annotated_mutable__: Literal[False]
```

`Literal[False]` means: once bound, the variable may not be reassigned. This is exactly `Final`'s mutation semantic, now expressible without hardcoding.

Default (tag absent) is `Literal[True]` — mutable, current behavior preserved.

(using Literal from typing, available since Python 3.8)

### Fallback rule

If a class does not declare `__annotated_type__`, checkers fall back to current behavior: a bare type in an annotation is that type. No existing code is affected.

### Open namespace

The `__annotated_*__` namespace is intentionally open. Checkers recognize the tags they implement; unknown tags are silently ignored. This means:

- A dimensional analysis checker can define `__annotated_dim__`
- A units library can define `__annotated_unit__`
- A value-range library can define `__annotated_range__`

None of these require a PEP. The library author declares the tag; the checker reads it. This is composable in a way the current world — where every special form requires a PEP and checker hardcoding — is not.

---

## Examples

### Third-party wrapper without the `TYPE_CHECKING` hack

```python
# Before
if TYPE_CHECKING:
    Quantity = Annotated
else:
    class Quantity[T]:
        def __class_getitem__(cls, key): ...

# After
class Quantity[T]:
    __annotated_type__: type[T]

    def __class_getitem__(cls, key: tuple[type[T], dict]) -> Quantity[T]:
        ...

dx: Quantity[float, {L: 1, T: -1}] = 2.3
# Checker: T = float, dx is float-compatible ✓
```

### Immutable wrapper

```python
class Frozen[T]:
    __annotated_type__: type[T]
    __annotated_mutable__: Literal[False]

x: Frozen[int] = 42
x = 99  # ✗ reassignment not allowed
```

### Custom checker tag (no PEP required)

```python
# Defined by a dimensional analysis library
class Quantity[T]:
    __annotated_type__: type[T]
    __annotated_dim__: ...  # read by the library's own checker/LSP

# Standard checkers ignore __annotated_dim__ — no conflict
# The library's checker reads it for dimensional validation
```

---

## Out of Scope

### Context-sensitive semantics (`InitVar`)

`InitVar[T]` is only valid in `__init__` parameter position within a dataclass. This context-sensitivity — "this annotation means something different depending on where it appears" — is not expressible with `__annotated_*__` tags, which are class-level declarations.

`InitVar` and similar constructs remain special-cased in checkers. This proposal does not attempt to replace all hardcoding, only the most common pattern.

### `Callable` meaning

PEP A allows `Callable.__class_getitem__` to validate its signature shape. The *semantic* of "this object is callable with these argument types" still requires special-case handling in checkers. `__annotated_*__` tags do not address callable semantics.

---

## Interactions with Related Ideas

- **PEP A** (`__class_getitem__` signatures): Required prerequisite. PEP B's `__annotated_type__` is most useful once subscript arguments are validated — otherwise the type `T` in `__annotated_type__: type[T]` has nothing to bind to.
- **PEP C** (`Val` and value-level TypeVars): `__annotated_type__: type[T]` where `T: Val[...]` would allow wrapper semantics over value-level types.
- **PEP D** (`Map` and bounded `TypeVarTuple`): Same extension applies to variadic wrappers.

---

## Prior Art

- `dataclasses.InitVar` — uses `__class_getitem__` to set a runtime attribute, the pattern this proposal formalizes for the annotation side
- `typing.Annotated` — the current workaround for attaching metadata to types; `__annotated_*__` is a more structured alternative for classes that want to declare their own semantics
- Dunder method protocol — `__len__`, `__iter__`, etc. are read by the runtime without registration; `__annotated_*__` applies the same philosophy to static analysis

---

## Why Checkers Don't Do This Today

Checker support for typing constructs grew incrementally, each special form added as a one-off. There was no protocol mechanism to generalize from. This proposal provides that mechanism retroactively — existing special forms are unaffected, new ones no longer need to be hardcoded.