# Implementation Stages

**Status:** Proposal, not yet adopted
**Scope:** how the pipeline described in `PROBLEM_STATEMENT.md` and
`README.md`'s "Development strategy" gets built *in order* -- vertical
slices, each one ending in an actual executed `.class` file, not a
horizontal layer with nothing runnable underneath it.

This document does not restate the phase list already in `README.md`
("Roadmap", Phase 0 -- Phase 6). Those are feature milestones. This
document is one level more granular for the earliest, riskiest part of
that list: it defines **Stage 0**, the first thing that has to exist
before "Phase 1: Executable Python subset" means anything, and sketches
what Stage 1+ looks like once Stage 0 is standing.

---

## Why a stage below Phase 0/1

`README.md` already says not to spend months on an abstract type
system before one `.class` file runs. Stage 0 takes that literally: it
is the smallest slice that forces every pipeline stage
(`parser -> AST -> semantic analysis -> Core IR -> JVM backend`) to
exist and connect, without requiring the compiler to have solved
anything it doesn't strictly need yet -- no object model, no dynamic
`PyObject` path, no optimizer. Everything in `SYSTEM-DESIGN-AGREEMENTS.md`
about who owns an operation's codegen only becomes decidable once
there is a second code path to be ambiguous with; Stage 0 deliberately
has only one path, so that document has nothing to adjudicate yet.

---

## Stage 0: primitive types, user-defined functions, and a Kotlin escape hatch

### Goal

Make `README.md`'s "First Definition of Done" --

```python
def main():
    print(2 + 3)
```

-- actually produce a running `.class`, while giving the still-unfinished
compiler a legitimate way to call real, hand-written Kotlin for
anything it can't yet generate code for itself.

### In scope

- Primitive data types: `int`, `float`, `bool`, `str`, `None` -- lowered
  directly to JVM primitives / `java.lang.String` / `Unit`. Stage 0
  lives entirely on the "specialized JVM operations" path from
  `README.md`'s central idea, not the dynamic `PyObject` path. There is
  no dynamic path yet for it to fall back to.
- Literals and the arithmetic/comparison operators those types support.
- Variable binding (`x = 10`).
- `def` functions with fully type-annotated parameters and a return
  type. No defaults, no `*args`/`**kwargs`, no closures -- those are
  later phases in `README.md`'s own list, not Stage 0's problem.
- Function calls and `return`.
- The Kotlin escape hatch described below, used for exactly one thing
  in Stage 0: `print`.

### Out of scope (deferred, not forgotten)

Classes, collections, exceptions, closures, generators, decorators
other than the escape hatch itself, the `PyObject` dynamic dispatch
path, and every optimizer pass in `COMPILER-IMPLEMENTATION.md`. An
opaque foreign call has no semantic-analysis fact backing a
specialization decision, so it is correct for it to stay outside the
optimizer's reach entirely at this stage.

---

## The Kotlin escape hatch

### The problem it solves

Even `print` needs runtime plumbing -- string conversion, access to
`System.out` -- that has nothing to do with whether the compiler's own
codegen works yet. Blocking Stage 0 on hand-writing that plumbing as
raw JVM bytecode emission would mean debugging two unfinished things
(the pipeline, and the runtime) at once. Instead, a function's
implementation can be declared to live in ordinary, hand-written
Kotlin, and the Python-syntax `def` becomes a typed stub the compiler
binds to it.

### Why this isn't really "FFI" in the C-ABI sense

The word most people reach for here is FFI, by analogy to how Kotlin
or CPython cross into C -- `ctypes`/`cffi` on the Python side, `JNI` on
the JVM side. Those exist because the two sides don't share a runtime:
crossing from JVM bytecode into a C shared library means crossing an
ABI boundary, which is why those mechanisms need explicit marshalling
layers, handles, and pinning.

That boundary doesn't exist here. ARKlin's compiled output and
Kotlin's compiled output are the same target: ordinary JVM class
files. A hand-written Kotlin `object` with a `@JvmStatic` method
already *is* a JVM method with a normal descriptor. The ARKlin JVM
backend does not need to marshal anything across a boundary -- it
emits the same `invokestatic` instruction it would emit to call a
function ARKlin generated itself. The "escape hatch" is class-file
linking, not foreign-function marshalling. That's what makes it cheap
enough to lean on for Stage 0 specifically.

### Proposed syntax

```python
@fn("kotlin_ext/Console.kt", "printLine")
def print(value: str) -> None:
    ...
```

- The decorated function's body must be exactly `...` (or a `pass` /
  docstring-only body). It is a type contract, not an alternate
  implementation -- a body that looks like it does real work is a
  compile error, since two implementations of the same function would
  contradict each other.
- First argument: path to a Kotlin source file, relative to the
  project root, under a `kotlin_ext/` directory reserved for this
  purpose.
- Second argument: the name of a top-level Kotlin function. Stage 0
  restricts the target shape to exactly one thing -- a Kotlin `object`
  singleton exposing a `@JvmStatic fun` -- so there is exactly one
  descriptor to resolve. No instance methods, no companion objects, no
  overloads, no generics. That restriction is deliberate scope-cutting,
  not a belief that it's the right shape long-term.
- The declared Python parameter and return types must structurally
  match the Kotlin function's JVM descriptor under a fixed, small
  mapping table:

  | Python annotation | Kotlin / JVM type |
  |---|---|
  | `int`   | `Int`     |
  | `float` | `Double`  |
  | `bool`  | `Boolean` |
  | `str`   | `String`  |
  | `None`  | `Unit`    |

  A mismatch is a hard compile error. Per `PROBLEM_STATEMENT.md` §12,
  this fails loudly and specifically -- it does not silently coerce
  types to make the call fit.

### Build-time linking

1. Semantic analysis discovers `@fn` declarations in a dedicated pass
   (not folded into codegen -- consistent with
   `COMPILER-IMPLEMENTATION.md`'s "one stage, one job" preference).
2. Each referenced Kotlin file is compiled with `kotlinc` as an
   ordinary build step. This adds no new dependency: a Kotlin compiler
   is already required to build ARKlin's own compiler.
3. The resulting `.class` is inspected to confirm the target method
   exists and its descriptor matches the declared Python signature
   exactly (via the mapping table above). On success, the ARKlin
   function symbol is bound to that class/method pair. On failure, the
   pass reports which side -- the Kotlin source or the Python
   declaration -- doesn't match, rather than a generic "linking
   failed."
4. Lowering emits `invokestatic` to the resolved method at every call
   site of the ARKlin function. This is the same call shape a
   compiler-generated function would receive, so nothing calling
   `print` has to change later when `print` gets a native
   implementation and the `@fn` declaration is deleted.

### Exit criteria for Stage 0

- `def main(): print(2 + 3)` compiles, links against a hand-written
  `kotlin_ext/Console.kt`, and executes, producing `5`.
- Exactly one escape-hatch function (`print`) is used to prove the
  mechanism. Stage 0 should not accumulate more `@fn` declarations than
  the minimum needed to hit that one milestone -- every additional one
  is scope the same "don't reach for structure the problem hasn't
  asked for yet" instinct from `docs/README.md` argues against.
- At least one ARKlin-native function (e.g. `add(a: int, b: int) -> int`)
  also compiles and runs without `@fn`, so Stage 0 proves both halves:
  the compiler's own codegen path, and the escape hatch, are both real
  and both wired into the same pipeline.

### What Stage 0 does not decide

- Whether `@fn` stays a permanent, public language feature or is
  retired once native codegen covers everything it was standing in
  for. Either way, the linking mechanism itself (compile the Kotlin
  file, verify the descriptor, `invokestatic`) is the same mechanism
  Phase 4's "JVM interoperability" will eventually want for
  intentional interop, not throwaway scaffolding -- only the *reason*
  a function uses it changes, from "compiler can't do this yet" to
  "this should call real Kotlin/Java on purpose."
- Overload resolution, instance methods, extension functions, or
  generics on the Kotlin side of the boundary.
- Whether escape-hatch calls ever become eligible for inlining or
  other optimizer passes. Under `SYSTEM-DESIGN-AGREEMENTS.md`'s rule
  that every specialization needs a semantic-analysis fact to justify
  it, an opaque foreign call has no such fact by default, so the
  default is that it stays opaque until a later document argues
  otherwise.

---

## Stage 1 and beyond (sketch)

Stage 0 is deliberately narrow. Once it's standing, later stages
expand along the same "vertical slice, not horizontal layer" principle
`README.md` already commits to:

- **Stage 1** -- `if`/`while`, more arithmetic, more of README's own
  "Phase 1: Executable Python subset" list, still entirely on the
  static/specialized path.
- **Stage 2** -- start shrinking the escape hatch: give `print` (and
  anything else Stage 0 delegated) a native ARKlin implementation, and
  delete its `@fn` declaration, as a concrete test that the call-site
  shape truly didn't need to change.
- **Stage 3+** -- the object model, the dynamic `PyObject` path, and
  everything else `README.md`'s Phase 2 onward already lists. At that
  point `SYSTEM-DESIGN-AGREEMENTS.md` starts doing real work, because
  there are finally two codegen paths for it to adjudicate between.

This section intentionally stays a sketch. Concretizing it further
before Stage 0 produces something to generalize from would repeat the
premature-structure mistake `CODE-STYLE.md` and
`COMPILER-IMPLEMENTATION.md` §5 both already declined to make.
