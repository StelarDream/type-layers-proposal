# PEP C Draft — `Val` and Value-Level TypeVars

> **Status:** Idea / Draft / Pre-proposal
> **Depends on:** PEP A (TypeVar resolution from `__class_getitem__` signatures)
> **Tested against:** Pyright (other checkers believed to share the same gap, unverified)
> **Related ideas:** PEP B (`__annotated_*__` protocol), PEP D (Map and bounded TypeVarTuple)

---

## Abstract

In generic argument position, `V: T` means "a type that inherits from T." There is currently no way to say "a value that is an instance of T." This proposal introduces `Val[T]` as a bound modifier for TypeVars, flipping the semantics from subtype to instance — strictly in generic argument position.

This unlocks value-level generic classes and TypeAliases that are currently inexpressible in the type system.

---

## Motivation

### The subtype/instance gap

TypeVar bounds have consistent meaning across Python's type system:

| Position | Annotation | Meaning |
|---|---|---|
| Variable | `v: T` | `v` is an instance of `T` |
| Generic argument | `V: T` | `V` is a type that inherits from `T` |

This asymmetry is intentional and correct for most uses. But it creates a blind spot: there is no way to declare a generic parameter that holds a *value*, not a type.

Consider a range type:

```python
class ValueRange[T: int | float]:
    def __class_getitem__(cls, key: tuple[type[T], T, T]) -> ValueRange[T]:
        t, lo, hi = key
        return cls(t, lo, hi)
```

PEP A allows the checker to validate `ValueRange[float, 0.0, 1.0]` against the `__class_getitem__` signature. But there is no way to declare `Lo` and `Hi` as TypeVars that hold values — so the bounds cannot be tracked across the class declaration.

### TypeAlias hits the same wall

The same gap appears in TypeAliases:

```python
type Clamped[T: int | float, Lo: T, Hi: T] = ValueRange[T, Lo, Hi]
```

`Lo: T` in generic argument position means "Lo is a subtype of T" — which is not what is intended. The checker reads it as a type constraint, not a value. There is no syntax to express "Lo is an instance of T."

### `Val[T]` fills the gap

`Val[T]` is a bound modifier. In generic argument position, it switches the meaning of a TypeVar bound from "subtype of T" to "instance of T":

```python
class ValueRange[T: int | float, Lo: Val[T], Hi: Val[T]]: ...

type Clamped[T: int | float, Lo: Val[T], Hi: Val[T]] = ValueRange[T, Lo, Hi]
```

Now `Lo` and `Hi` are understood by the checker as values of type `T`, not as types that inherit from `T`.

---

## Proposal

### `Val[T]`

`Val[T]` is valid only as a TypeVar bound in generic argument position:

```python
class Foo[V: Val[int]]: ...         # V holds an int value
class Bar[T, V: Val[T]]: ...        # V holds a value of type T
type Alias[V: Val[int]] = Foo[V]    # same in TypeAlias
```

`Val[T]` outside of generic argument position is nonsensical and should be a checker error:

```python
v: Val[int] = 42    # ✗ — Val is not a variable annotation tool
```

Variables are already instance-typed. `Val` is strictly a generic parameter modifier.

### Inference rules

PEP A established two resolution paths for `__class_getitem__`:

| Annotation | Argument | Resolves |
|---|---|---|
| `key: type[T]` | a type `X` | `T = X` |
| `key: T` | a value `x` | `T = type(x)` — i.e. `int`, not `Literal[23]` |

To preserve the specific value rather than widening to its type, see the Literal preservation section below.

`Val` TypeVars follow the `key: T` path — the checker infers the type of the value, not the specific value:

```python
ValueRange[float, 0.0, 1.0]   # Lo = float, Hi = float ✓
ValueRange[float, 0, 1.0]     # Lo = int, Hi = float — type mismatch if Lo: Val[T] and T = float ✗
```

### Opting into value preservation with `Literal[T]`

If the specific value matters — not just its type — the author can opt in using `Literal[T]`:

```python
class Baz[T: Val[int]]:
    def __class_getitem__(cls, key: Literal[T]) -> Baz[T]: ...
    def __init__(self, val: Literal[T]) -> None: ...

x: Baz[23]   # T = Literal[23], not int
```

`Literal` already carries the semantic of "this specific value matters." Using it as an annotation for a `Val`-bound TypeVar tells the checker to preserve the value rather than widen to its type.

The `Val` bound is required here — `Literal` only accepts scalar types and enum members, and `T: Val[int]` is what constrains `T` to a scalar domain.

### Interaction with `__class_getitem__` (PEP A)

`Val` TypeVars compose naturally with explicit `__class_getitem__` signatures:

```python
class ValueRange[T: int | float, Lo: Val[T], Hi: Val[T]]:
    def __class_getitem__(cls, key: tuple[type[T], T, T]) -> ValueRange[T]:
        t, lo, hi = key
        return cls(t, lo, hi)
```

The checker reads:
- `key[0]: type[T]` → `T = float` (from `float`)
- `key[1]: T` → `Lo = float` (value of type T)
- `key[2]: T` → `Hi = float` (value of type T)

The TypeVar declaration (`Lo: Val[T]`) and the `__class_getitem__` signature (`key: tuple[type[T], T, T]`) agree — no conflict.

### Interaction with `__annotated_type__` (PEP B)

When a class is parameterized with a `Val`-bound TypeVar, the declaration of `__annotated_type__` determines how the checker treats the annotated variable. The wrapping level matters:

```python
class Foo[T: Val[int]]:
    __annotated_type__: T  # ✗ — T is a value, not a type. Invalid as annotation.

x: Foo[22]  # this becomes nonsensical ("x is an instance of an instance of an instance of int")
```

```python
class Bar[T: Val[int]]:
    __annotated_type__: type[T]  # T is a value, type[T] is value-oriented

x: Bar[34] # this is also nonsensical — equivalent to 'x: 34' ("x is an instance of an instance of int")
x: Literal[Bar[34]] # Bar[34] is only valid only where a Val is expected — not as a standard variable annotation
```

```python
class Baz[T: Val[int]]:
    __annotated_type__: type[type[T]]  # lifts T (value) → type(T) (its type)

x: Baz[34] = 223  # ✓ — equivalent to 'x: int' ("x is an instance of int")
```

The canonical form for a `Val`-parameterized wrapper that should be usable as a standard variable annotation is `type[type[T]]`. Each additional `type[...]` wrapping lifts one level: value → its type → a valid annotation type.

### TypeAlias revisited

With `Val` defined, the original motivation now works:

```python
type Clamped[T: int | float, Lo: Val[T], Hi: Val[T]] = ValueRange[T, Lo, Hi]

x: Clamped[float, 0.0, 1.0]   # ✓ — T = float, Lo = float, Hi = float
```

---

## Examples

### Value-level generic class

```python
class Percentage[V: Val[float]]:
    # V is a float value. __annotated_type__ (PEP B) tells the checker
    # to treat Percentage[V] as float-compatible.
    __annotated_type__: type[V]

    def __class_getitem__(cls, key: V) -> Percentage[V]:
        assert 0.0 <= key <= 1.0
        return cls(key)

x: Percentage[0.75]   # ✓
y: Percentage[1.5]    # ✓ statically (runtime asserts), or ✗ with Literal bounds
```

### Literal preservation

```python
class Port[N: Val[int]]:
    def __class_getitem__(cls, key: Literal[N]) -> Port[N]: ...

x: Port[8080]          # T = Literal[8080]
y: Port[8080 + 1]      # T = int (expression, not literal) — checker may warn
```

### TypeAlias with value constraints

```python
type RGB[R: Val[int], G: Val[int], B: Val[int]] = tuple[Literal[R], Literal[G], Literal[B]]

red: RGB[255, 0, 0] = (255, 0, 0)   # ✓
```

> **Note: ** This pattern — `Val` TypeVars wrapped in `Literal` inside a TypeAlias — cleanly solves a longstanding pain point. Typed sequences of specific values currently require verbose `Literal[1] | Literal[2] | ...` unions or lose static value tracking entirely. A `Val`-based alias is reusable, readable, and composes naturally with the rest of the type system.

---

## Out of Scope

### `TypeVarTuple` interaction

How `Val` interacts with variadic generics (`*V: Val`) is deferred to PEP D. The machinery for value-level variadics requires `Map` and bounded `TypeVarTuple`, which are sufficiently complex to warrant a separate proposal.

### Runtime enforcement

`Val` is a static analysis tool. It does not add runtime validation. Library authors who want runtime enforcement should use `__class_getitem__` or `__init__` assertions as they do today.

---

## Interactions with Related Ideas

- **PEP A** (`__class_getitem__` signatures): The `key: T` resolution path established in PEP A is the runtime foundation that `Val` builds on. PEP A is a prerequisite.
- **PEP B** (`__annotated_*__` protocol): `__annotated_type__: type[V]` where `V: Val[T]` allows wrapper semantics over value-level types.
- **PEP D** (`Map` and bounded `TypeVarTuple`): Extends `Val` to variadic generic parameters.

---

## Why Checkers Don't Do This Today

TypeVars have always represented types, not values. The distinction between "subtype of T" and "instance of T" in generic argument position was never needed before value-level generics were a design goal. `Val` introduces the concept with minimal surface area — one new modifier, one new inference rule, composing cleanly with existing machinery from PEP A.