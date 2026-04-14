# 01 — `__type_checking__`: a class-level transparency declaration

## Motivation

Python's type system has no way for a class to say "when the checker sees me, treat me as something else." The closest existing tools are:

- `cast(T, x)` — a lie to the checker, scattered at call sites, and structurally incapable of preserving generic parameters: there is no version of `cast` that makes `PostInitFinal[int]` check as `int`.
- `TYPE_CHECKING` split — requires shipping two definitions of the same class, one of which is a fabrication the runtime never sees.
- Returning `type[T]` from `__class_getitem__` — loses the semantic information the class exists to carry (eg. that the class was `PostInitFinal[int]`, it just sees `int`), and pyright ignores `__class_getitem__` return annotations entirely, so it does not even function as a workaround.

All three are workarounds for the same missing primitive: a class-level declaration of transparency.

### Example 1 — a custom `PostInitFinal`

```python
# runtime class — carries real semantic information
class _PostInitFinal[T]:
    ...

# checker alias — a fabrication, the only option available
if TYPE_CHECKING:
    PostInitFinal = InitVar  # pretend PostInitFinal is just InitVar for transparency
else:
    PostInitFinal = _PostInitFinal

# usage
class Person:
    name: PostInitFinal[str]
    birth_year: PostInitFinal[int]

    def __init__(self, name: str, birth_year: int) -> None:
        self.name = name
        self.birth_year = birth_year

#  checker sees InitVar[str] → type[str] ✓
#  runtime sees _PostInitFinal → no transparency
```

The real class and the checker's class are strangers.

### Example 2 — `InitVar` itself, from the stdlib

`InitVar` is hardcoded into every major type checker. Its runtime implementation is:

```python
class InitVar:
    __slots__ = ('type', )

    def __init__(self, type):
        self.type = type

    def __repr__(self):
        if isinstance(self.type, type):
            type_name = self.type.__name__
        else:
            # typing objects, e.g. List[int]
            type_name = repr(self.type)
        return f'dataclasses.InitVar[{type_name}]'

    def __class_getitem__(cls, type):
        return InitVar(type)
```

It carries no generic information, and there is no way for a checkers to understand it without special-casing it. Pyright and mypy both hardcode `InitVar` as a known symbol. Any user trying to replicate its behaviour for their own class hits a wall immediately.

## Proposal

A new dunder annotation: `__type_checking__: type[T]`.

```python
class Foo[T]:
    __type_checking__: type[T]
```

**The checker rule:** if a class declares `__type_checking__: type[T]`, replace any reference to `cls[T, ...]` with `T` during type analysis.

This is annotations-first — no need for runtime values to be assigned. The checker reads it from the class annotations and applies the substitution. Everything else about the class is preserved.

`PostInitFinal` simply becomes

```python
class PostInitFinal[T]:
    __type_checking__: type[T]
```

and `InitVar` doesn't need special casing in type checker by just adding

```python
class InitVar[T]:
    __type_checking__: type[T]
    __slots__ = ('__type_checking__',)

    def __init__(self, type: type[T]) -> None:
        self.__type_checking__ = type

    def __repr__(self) -> str:
        type_name = repr(self.__type_checking__)
        return f'dataclasses.InitVar[{type_name}]'

    def __class_getitem__(cls, type: type[T]) -> "InitVar[T]":
        return InitVar(type)
```

Note: `__type_checking__` holds `type[T]` — the class itself — not `T` (an instance). 
This mirrors `__class_getitem__`'s `key` parameter, which is always one level up from the generic declaration.

---

## Relation to `TYPE_CHECKING`

This is the class-level analogue of `if TYPE_CHECKING:` — both are declarations to the checker, invisible at runtime:

```python
if TYPE_CHECKING:
    x: int = ...    # block-level: this binding only exists for the checker

class Foo[T]:
    __type_checking__: type[T]  # class-level: this class is transparent to T for the checker
```

Same prefix, same semantics, different scope. The naming is intentional — any checker author reading `__type_checking__` understands the contract by analogy.

---

## `__class_getitem__` becomes honest

Today, `__class_getitem__` is forced to lie about its return type. With `__type_checking__`, it can return an instance and be honest about it:

```python
class Bar[T, D]:
    __type_checking__: type[T]   # checker sees T, not Bar
    __meta__: ClassVar[type[D]]

    def __class_getitem__(cls, key: tuple[type[T], type[D]]) -> Bar[T, D]:
        _, meta = key
        cls.__meta__ = meta
```

The return type `Bar[T, D]` is honest. The checker substitutes `T` at call sites. No `cast`, no split, no fabrication.

---

## Annotations are a superset of type checking

This proposal draws a line that Python currently blurs:

- **annotations** — anything written after `:`, a superset: types, values, metadata, bounds
- **type checking** — what `__type_checking__: type[T]` declares the checker should see

`__type_checking__` is the explicit bridge. A class carries rich annotated structure at runtime while exposing only the relevant type to the checker. The annotation is the full truth; the type is the checker's projected view.

Type checkers are not the arbiters of what annotations mean — they are one consumer among many. A dimensional checker, a bounds checker, a schema validator each read the same annotation and extract what they need, operating in independent layers.

---

## Backwards compatibility

- Classes without `__type_checking__` behave exactly as today — no change.
- `__type_checking__` is annotation-only, so `hasattr(obj, '__type_checking__')` is `False` at runtime unless explicitly assigned. Existing code inspecting `__annotations__` sees it but is free to ignore it.
- Old checkers that do not understand `__type_checking__` simply ignore the annotation — no errors, no behaviour change.

---

## What this replaces

| today | with `__type_checking__` |
|---|---|
| `if TYPE_CHECKING: Foo = SomeOtherClass` | `__type_checking__: type[T]` on `Foo` directly |
| `cast(T, result)` scattered at call sites | one declaration on the class |
| `__class_getitem__` forced to return `type` | honest return type, instance allowed |
| `InitVar` as a magic transparency form | regular class with `__type_checking__: type[T]` |

---

## Relation to other proposals

- **02 — Val**: `__type_checking__` pairs with `Val` to make classes like `ValueRange[T, 1, 2]` fully expressible. Neither depends on the other — `__type_checking__` is useful standalone.
- **09 — Propositions**: `__type_checking__` is the mechanism by which `If[P, A, B]` exposes `A` or `B` to the checker depending on `P`.