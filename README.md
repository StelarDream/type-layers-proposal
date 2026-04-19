# Type Layers Proposal

A comprehensive set of draft proposals to extend Python's type system with powerful new primitives, making it more expressive and enabling advanced type-level programming patterns.

## Overview

This repository contains **four interconnected PEP draft proposals** that progressively build on each other to enhance Python's typing capabilities. These proposals aim to close gaps in the current type system and enable new patterns in static type checking.

### Proposals at a Glance

| Proposal | Title | Purpose | Status |
|----------|-------|---------|--------|
| **A** | TypeVar resolution from `__class_getitem__` signatures | Enable better type inference for generic classes | Foundational |
| **B** | The `__annotated_*__` protocol | Support custom annotation protocols for richer type metadata | Independent |
| **C** | `Val` and value-level TypeVars | Bridge value and type domains for advanced type constraints | Builds on A |
| **D** | `Map` and bounded `TypeVarTuple` | Enable type-level mapping operations and constrained type variable tuples | Builds on A & C |

## Getting Started

**Start with PEP A** — it's the most targeted, immediately actionable, and foundational to understanding the later proposals.

## Key Benefits

-  **More Precise Types**: Better capture of generic class constraints and relationships
-  **Value-Type Integration**: Connect runtime values with compile-time type information
-  **Advanced Patterns**: Enable type-level programming patterns previously impossible in Python
-  **Backward Compatible**: All proposals are designed with compatibility in mind

## Proposal Details

### PEP A: TypeVar Resolution from `__class_getitem__` Signatures
Improves how TypeVars are resolved when accessing generic classes. This is the foundation for proposals C and D.

**File**: `Pep_A_DRAFT class_getitem resolution.md`

### PEP B: The `__annotated_*__` Protocol
Introduces a protocol for custom annotation handling, enabling more flexible and extensible type metadata.

**File**: `Pep_B_DRAFT annotated protocol.md`

### PEP C: `Val` and Value-Level TypeVars
Extends TypeVars to work with actual runtime values, creating new possibilities for generic programming.

**File**: `Pep_C_DRAFT val value level typevars.md`

### PEP D: `Map` and Bounded `TypeVarTuple`
Enables functional-style type operations and more constrained type variable tuples for advanced generic code.

**File**: `Pep_D_DRAFT map bounded typevartuple.md`

## For Type System Enthusiasts

These proposals are designed for:
- Python type checking tool maintainers (mypy, Pyright, Pyre, etc.)
- Library authors working with advanced generics
- Developers interested in Python's type system evolution
- Contributors to Python Enhancement Proposals (PEPs)

## Contributing & Feedback

We welcome feedback, questions, and discussions on these proposals:

-  **Open an Issue** for questions, suggestions, or alternative approaches
-  **Comment on existing issues** to join ongoing discussions
-  **Share use cases** that demonstrate the need for these features

## Disclaimer

This proposal was written with assistance from an LLM to help structure and refine ideas, but every technical claim, design decision, and example has been carefully verified and reflected upon. The author (Stelar) has dyslexia and uses LLMs as a tool for clear communication.

## Related Resources

- Python Enhancement Proposals (PEPs): https://www.python.org/dev/peps/
- Python typing module: https://docs.python.org/3/library/typing.html
- PEP 484 - Type Hints: https://www.python.org/dev/peps/pep-0484/

---

**License**: See LICENSE.md for details.

**Repository**: [StelarDream/type-layers-proposal](https://github.com/StelarDream/type-layers-proposal)
