# PEP D Draft — `Map` and Bounded `TypeVarTuple`

> **Status:** Idea / Draft / Pre-proposal
> **Depends on:** PEP A (TypeVar resolution from `__class_getitem__` signatures), PEP C (Val and value-level TypeVars)
> **Tested against:** Pyright (other checkers believed to share the same gap, unverified)
> **Related ideas:** PEP B (`__annotated_*__` protocol)

---

## Abstract

Variadic generics (`TypeVarTuple`, PEP 646) have a fundamental expressibility problem: there is no way to write a `__class_getitem__` signature that tracks per-element type identity across a `TypeVarTuple`. The closest approximation loses all element-level information.

This proposal introduces two additions:

- `Map[F, V]` — a type-level functor that applies a type constructor `F` element-wise over a `TypeVarTuple` `V`, making variadic `__class_getitem__` signatures fully expressible
- `*V: Val` — a `TypeVarTuple` whose elements are values rather than types, the natural variadic extension of PEP C's `Val[T]`, made possible by `Map`

---

## Motivation

### The classic variadic wall

Consider a plain variadic generic class:

```python
class Foo[*V]: ...
```

from PEP A we can establish that the default synthesized `__class_getitem__` for `class Foo[*V]` is:

```python
def __class_getitem__(cls, key: tuple[type, ...]): ...
```

This is the closest currently expressible signature — but it loses all per-element type identity. The checker sees "some tuple of types" rather than "these specific types in these specific positions." `V` as a tracked entity disappears from the signature.

There is no way to write:

```python
def __class_getitem__(cls, key: tuple[*V]): ...  # not currently valid in type position
```

...because `*V` in a `tuple[...]` annotation has no meaning the checker can act on when `V` holds types.

### `Map` fixes this

`Map[F, V]` applies a type constructor `F` element-wise over `V`:

```python
# V = (int, float, str)
# Map[type, V] = (type[int], type[float], type[str])
```

This makes a fully expressive `__class_getitem__` signature possible:

```python
class Foo[*V]:
    def __class_getitem__(cls, key: tuple[*Map[type, V]]) -> Foo[*V]: ...
```

Now `Foo[int, float, str]` is validated as `tuple[type[int], type[float], type[str]]` — each element tracked individually. `V` retains its identity throughout.

This is the core motivation for `Map`. Everything else follows from it.

### `*V: Val` — the value-level extension

With `Map` established, the value-level case is natural. PEP C introduced `Val[T]` for scalar TypeVars:

- `V: T` — V is a type inheriting from T
- `V: Val[T]` — V is a value of type T

The same distinction applies to `TypeVarTuple`:

- `*V` (implicit `*V: type`) — elements of V are types
- `*V: Val` — elements of V are values

And the `__class_getitem__` signature for `*V: Val` is simply:

```python
def __class_getitem__(cls, key: tuple[*V]): ...
```

Where `*V` now unpacks as values. To lift them back to type position, `Map[type, V]` applies element-wise — exactly as in the classic case.

---

## Proposal

### `Map[F, V]`

`Map[F, V]` is valid in any type position where a `TypeVarTuple` unpacking is expected. It applies type constructor `F` element-wise over `TypeVarTuple` `V`:

| Expression | Input `V` | Result |
|---|---|---|
| `Map[type, V]` | `(int, float, str)` | `(type[int], type[float], type[str])` |
| `Map[list, V]` | `(int, float, str)` | `(list[int], list[float], list[str])` |
| `Map[Literal, V]` | `(1, 2.0, "x")` | `(Literal[1], Literal[2.0], Literal["x"])` |

`Map` is not limited to `*V: Val` — it is equally useful for classic type-level `TypeVarTuple`.

> **Note:** Cases outside of `Map[type, V]` are not the focus of this PEP, but are a nice addition 

### `*V: Val`

`Val` without a type argument is the variadic bound. It declares that the elements of `*V` are values, not types:

```python
class Foo[*V: Val]: ...          # *V holds values of any types
class Bar[T, *V: Val[T]]: ...    # *V holds values all of type T
```

> `*V: Val[T]` constrains all elements to be instances of `T`. `*V: Val` places no type constraint on elements — any value of any type is accepted.

The default synthesized `__class_getitem__` for `*V: Val` is:

```python
def __class_getitem__(cls, key: tuple[*V]): ...
```

Where `*V` unpacks as a tuple of values. Contrast with plain `*V`:

```python
def __class_getitem__(cls, key: tuple[*Map[type, V]]): ...
```

Where `*V` unpacks as a tuple of types, and `Map` is needed to make the signature expressible.

`*V: Val` outside generic argument position is a checker error, consistent with scalar `Val[T]`.

> **Note:** For Val-bound TypeVars, arg: V is not a valid annotation — it would mean "an instance of a value", which is nonsensical. The valid forms are type[V], Literal[V] (scalar), *Map[type, V], or *Map[Literal, V] (variadic).

---

## Examples

### Classic variadic — per-element identity restored

```python
class Foo[*V]:
    def __class_getitem__(cls, key: tuple[*Map[type, V]]) -> Foo[*V]: ...

f: Foo[int, float, str]   # ✓ — V = (int, float, str), each tracked
g: Foo[int, 42]           # ✗ — 42 is not a type
```

### Value-level variadic

```python
class Row[*V: Val]:
    def __class_getitem__(cls, key: tuple[*V]) -> Row[*V]: ...

r: Row[1, 2.0, "x"]   # ✓ — V = (1, 2.0, "x"), types inferred as (int, float, str)
```

### Mixed type and value variadics

```python
class Foo[T, *V: Val[T]]:
    def __class_getitem__(cls, key: tuple[type[T], *V]) -> Foo[T, *V]: ...

f: Foo[int, 1, 2, 3]    # ✓ — T = int, *V = (1, 2, 3)
g: Foo[int, 1, 2.0, 3]  # ✗ — 2.0 is not int
```

### Lifting values to types with `Map`

```python
class Schema[*V: Val]:
    def __init__(self, *args: *Map[type, V]) -> None: ...
    def column_types(self) -> tuple[*Map[type, V]]: ...

s = Schema(2, 4.2, "x")   # V = (2, 4.2, "x"), Map[type, V] = (int, float, str)
types = s.column_types()  # tuple[int, float, str]
```

### TypeAlias with value variadics

```python
type Scalar = type[None] | bool | int | float | str | Enum
type TypedRow[*V: Val[Scalar]] = tuple[*Map[Literal, V]]

row: TypedRow[1, 2, 3]   # tuple[Literal[1], Literal[2], Literal[3]]
```

> This pattern requires PEPs A, C and D working together: PEP A (TypeVar resolution from `__class_getitem__`), PEP C (`Val` and value-level TypeVars), and PEP D (`Map` and bounded `TypeVarTuple`). None of the three alone is sufficient.

---

## Out of Scope

### Full `Literal` preservation for `*V: Val`

`Map[Literal, V]` is specified here as a valid expression. However the full interaction between value-level variadics and literal inference — including how checkers narrow `*V` when specific literal values are provided — is not fully specified. We believe the natural extension is `tuple[*Map[Literal, V]]`, consistent with scalar `Literal[T]` in PEP C. Full specification is left for a follow-up.

### Edge cases in unpacking positions

How `*V: Val` interacts with existing `TypeVarTuple` features (`Unpack`, `*args: *V`) is not fully explored here. The core declarations and `Map` functor are specified; edge cases in unpacking positions are left to implementors.

---

## Interactions with Related Ideas

- **PEP A** (`__class_getitem__` signatures): The resolution machinery that allows `key: tuple[*Map[type, V]]` to bind `*V`. PEP A is a prerequisite.
- **PEP B** (`__annotated_*__` protocol): `__annotated_type__` can compose with `*V: Val` for wrapper semantics over value-level variadic types.
- **PEP C** (`Val` and value-level TypeVars): `*V: Val` is the direct variadic generalization of `V: Val[T]`. `Map` mirrors `Literal[T]` as the lifting mechanism.

---

## On the Name `Map`

`Map` is the natural mathematical name for a functor applied element-wise. However it may clash with existing mental models (`map()` builtin, `typing.Mapping`). Alternatives considered:

- `Each[F, V]` — reads naturally: "each element of V wrapped in F"
- `Over[F, V]` — "F applied over V"

The name is intentionally left open for community discussion. The semantics are the same regardless of spelling.

---

## Why Checkers Don't Do This Today

`TypeVarTuple` (PEP 646, Python 3.11) is relatively recent and its checker support is still maturing. The per-element identity problem with `tuple[type, ...]` has not been addressed because there was no mechanism to express it. `Map` provides that mechanism with minimal surface area — one new type-level functor that composes with existing `TypeVarTuple` machinery. `*V: Val` then extends the same pattern to values, requiring no runtime changes.