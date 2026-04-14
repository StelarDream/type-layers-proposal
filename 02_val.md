# 02 — `Val`: value-kinded generic bounds

## Motivation

Python's generic system has two distinct binding contexts that use identical syntax:

```python
x: T      # annotation: x is an instance of T
T: int    # bound:      T is a subtype of int
```

In annotations, `:` means *instancing* — "x lives at the instance level of T."
In bounds, `:` means *inheritance* — "T lives at the class level, constrained by subtyping."

This works well for type-level generics. But there is no way to express a **value-level bound** — a generic parameter that is constrained to be an *instance* of something, not a subtype of it:

```python
class ValueRange[T, Lo, Hi]:
    ...

ValueRange[float, 0.0, 1.0]  # Lo = 0.0, Hi = 1.0 — instances, not types
```

Today there is no way to say "Lo must be an instance of T" in a bound. The closest available tools are:

- `Lo: T` — wrong: means Lo is a *subtype* of T, not an instance
- `Lo: Literal[T]` — wrong: `Literal` requires scalar primitives and cannot be parametrised by a TypeVar
- leaving Lo unbounded — loses all static information about what Lo actually is

The missing primitive is an operator that **shifts a bound from the class level to the instance level**.

---

## The `Val` operator

`Val[X]` is a bound modifier. It shifts the semantics of `:` from subtyping to instancing:

```
T: X        — T is a subtype of X      (class level, inheritance)
T: Val[X]   — T is an instance of X    (instance level, value)
```

This maps directly onto Python's object model:

```
type[X]   = ↑   walk up a layer   (instance → class)
Val[X]    = ↓   walk down a layer (class → instance)
```

`Val` is the operator that makes a bound descend one layer — from "must inherit from X" to "must be a value of type X."

### Why `Val` cannot appear in variable annotations

Variable annotations are already at the instance level:

```python
x: T        # x is an instance of T — correct
x: Val[T]   # x is an instance of an instance of T — not a valid concept
```

You cannot instantiate an instance. `class Foo(23): ...` fails at runtime for the same reason. `Val` only has meaning in **bound position**, where you are currently at the class level and want to descend. In annotation position, there is nothing to descend from.

### Val hits the floor

`Val` shifts a bound down one layer — from class to instance. Applying it twice would mean
"an instance of an instance", which is not a valid concept in Python's object model:

```
Val[int]       — an instance of int                 ✓  (e.g. 42)
Val[Val[int]]  — an instance of an instance of int  ✗  (you cannot instantiate an instance)
```

`Val` is a one-way descent. The floor is the instance level — nothing exists below it.

This contrasts with `type`, which hits a fixed point at the top:

```
type[type[int]] = type   ← ceiling, loops back to itself
Val[Val[int]]   = ✗      ← floor, invalid
```

---

## `Val` inference from call arguments

When a generic class has `Val` bounds, the checker can infer the exact values from call arguments without requiring explicit subscripts.

Given:

```python
class ValueRange[T: int | float, Lo: Val[T] = -inf, Hi: Val[T] = +inf]:
    __type_checking__: type[T]
    _lo: T
    _hi: T

    def __class_getitem__(cls, key: tuple[type[T], type[Lo], type[Hi]]) -> "ValueRange[T, Lo, Hi]":
        t, lo, hi = key
        out = ValueRange()
        out._lo = lo
        out._hi = hi
        return out

    def __init__(self, val: T, /, lo: type[Lo] | None = None, hi: type[Hi] | None = None):
        self._lo = lo if lo is not None else self._lo
        self._hi = hi if hi is not None else self._hi
        if not (self._lo <= val <= self._hi):
            raise ValueError(f"{val} out of range [{self._lo}, {self._hi}]")
        self.value = val
```

Both forms are equivalent:

```python
x = ValueRange[float, 0.0, 1.0](0.73)  # explicit subscript
y = ValueRange(0.73, 0.0, 1.0)         # inferred: ValueRange[float, 0.0, 1.0]
z = ValueRange(42)                      # inferred: ValueRange[int, -inf, +inf]
```

The inference chain for `ValueRange(0.73, 0.0, 1.0)`:

```
val = 0.73  → type(0.73) = float  → T = float
lo  = 0.0   → type(0.0)  = float  → Lo = Val[float] = 0.0, consistent with T ✓
hi  = 1.0   → type(1.0)  = float  → Hi = Val[float] = 1.0, consistent with T ✓
result: ValueRange[float, 0.0, 1.0]
```

Note: `lo: type[Lo] | None` rather than `lo: Lo | None` — because `Lo` is a value (instance level), and annotating a parameter as an instance of an instance is not valid. `type[Lo]` lifts it back to the class level, which is a valid annotation. Since `type[Val[T]] = T`, this resolves correctly to `T`.

### Default bounds

`Val` parameters support default values. `Lo: Val[T] = -inf` means "if Lo is not provided, assume negative infinity." This allows:

```python
z = ValueRange(42)  # → ValueRange[int, -inf, +inf]
```

Without defaults, every use site would require explicit bounds — defeating the ergonomic goal.

---

## `Val` + `__type_checking__`

`Val` and `__type_checking__` are independent proposals that compose naturally. `__type_checking__: type[T]` tells the checker to substitute `T` at use sites, while `Val` bounds tell the checker what values are valid for each parameter:

```python
x: ValueRange[float, 0.0, 1.0] = 0.73
#  checker sees: float (via __type_checking__: type[T], T = float)
#  runtime carries: _lo = 0.0, _hi = 1.0
```

Neither proposal requires the other:
- `__type_checking__` is useful for any transparent wrapper, even without `Val` bounds.
- `Val` bounds are useful for expressing value constraints, even without transparency.

Together they make classes that are both semantically rich at runtime and clean to use statically.

---

## What `Val` enables

| without Val | with Val |
|---|---|
| no way to bound a generic to a value | `Lo: Val[T]` — Lo must be an instance of T |
| `Literal` restricted to scalar primitives | `Val[X]` works for any type |
| value constraints require runtime checks only | static inference from call arguments |
| `ValueRange` needs cast + TYPE_CHECKING split | clean generic class, honest annotations |

---

## Relation to other proposals

- **01 — `__type_checking__`**: composes with `Val` for transparent value-carrying classes. Neither depends on the other.
- **03 — slot notation**: `int[3]` as sugar for `Val[int]` where the value is `3`. Builds directly on `Val`.
- **04 — bounded TypeVarTuple**: `*V: Val` — all elements of a variadic are value-kinded. Requires `Val` to exist first.
- **05 — Map**: `Map[type, V]` — apply a type operator across a TypeVarTuple. Uses `Val` bounds internally.
- **09 — propositions**: `Gt[X: Val, Y: Val]` — propositions are parametrised by values. Requires `Val`.