# ARKlin -- System Design Agreements

**Status:** Living document
**Scope:** whole project (the principle is architectural, not file-specific)

`CODE-STYLE.md` will explain how a piece of code should be shaped once
you know what it owns. This document is one level up: **who is
allowed to own what**, between the compiler-generated code path and
the runtime's dynamic dispatch path.

This isn't a style preference. `PROBLEM-STATEMENT.md` §4 and §10
already state the rule in prose; this document exists so the boundary
has one canonical home instead of being re-derived, slightly
differently, in every PR that touches codegen.

---

## The agreement

> **A JVM instruction sequence emitted for a Python expression must
> be justified by something the compiler can *prove* at compile
> time. Where it cannot prove enough, the operation is not allowed
> to guess -- it must fall through to the runtime's dynamic dispatch
> path. Where it can prove enough, the operation is not allowed to
> pay for the runtime's generality anyway.**

Concretely, for any binary operation, method call, or attribute
access, exactly one of two things owns the codegen decision:

| Owner | Trigger | Output |
|---|---|---|
| **Compiler (static path)** | Semantic analysis proved concrete types (e.g. both operands are `int` per annotation or flow-typing) | Direct JVM primitive/instruction (`iadd`, `invokevirtual` on a known class, etc.) |
| **Runtime (dynamic path)** | Types are not fully resolved, or the operation is genuinely polymorphic (`__add__` overload, duck-typed argument, `Any`) | `PyObject`-level dispatch through the runtime |

There is no third owner. A JVM instruction sequence that is *half*
specialized -- e.g. assumes `int` layout but still routes through
dynamic dispatch "just in case" -- is exactly the kind of
locally-reasonable, globally-wrong decision this document exists to
rule out. It doesn't fail as a crash; it fails as silently doubled
cost on every hot-path operation, which is much harder to notice
than a bug.

## Why this is a *system design* class of decision, not "just an
optimization pass"

The failure shape to avoid is two subsystems both partially
believing they're responsible for correctness on the same value:

- If the **optimizer** specializes an operation the **semantic
  analyzer** didn't actually prove safe (e.g. inferring `int` from
  one call site and forgetting the function is polymorphic across
  call sites), the generated `.class` file is not merely slow -- it
  is *wrong*, and wrong in a way that only shows up when a caller
  the optimizer didn't see passes something else in.
- If the **runtime** silently re-validates types the **compiler**
  already proved, every specialized path pays dynamic-dispatch cost
  anyway, and the entire reason for Core IR / static lowering
  (see `PROBLEM-STATEMENT.md` §2, §9) is defeated without anyone
  writing a bug report, because nothing is incorrect -- it's just
  quietly as slow as CPython.

Both are variants of the same mistake: a correctness or performance
guarantee that only holds if exactly one side is deciding, being
made by both sides independently, without either checking what the
other assumed.

## The rule in IR terms

```text
Core IR node
     |
     v
Did semantic analysis attach a proven, concrete type?
     |
   +-+-------------------+
   |                     |
  yes                    no
   |                     |
   v                     v
Specialize:          Lower to runtime call:
primitive JVM         PyObject dispatch
instruction           (objects/, types/, numbers/, ...)
   |                     |
   v                     v
No further type      No primitive
checks at runtime     specialization
   (compiler already   attempted at this
    proved it safe)     IR node
```

If an IR node reaches JVM lowering without an explicit answer to
"who proved this operation's shape," that is a bug in semantic
analysis, not a decision for the JVM backend to make by falling back
to whichever path is more convenient to implement first.

## What this means day to day

- A new optimizer pass may **only** specialize an operation if it
  can point to the specific semantic-analysis fact (a type
  annotation, a monomorphic call-site proof, a closed-world escape
  analysis result) that licenses it. "It looked like it was always
  an int in testing" is not a proof.
- A new runtime primitive (`runtime/numbers/`, `runtime/objects/`,
  etc.) exists to handle the *dynamic* case correctly and generally.
  It must never be reached for as a shortcut to avoid writing the
  semantic-analysis work needed to specialize -- that direction of
  laziness is how a compiler project quietly turns into an
  interpreter with extra steps, which `PROBLEM-STATEMENT.md` §11
  explicitly rules out as a goal.
- Where a construct's semantics are genuinely unresolved (see the
  roadmap's phase gates in `PROBLEM-STATEMENT.md`), the correct
  answer is "not yet supported," diagnosed as such -- not a
  best-effort guess that happens to work on the examples someone
  tested.

## Boundary this does *not* cover

This document is about the compiler/runtime split for *operation
codegen*. It does not (yet) cover Android platform-API ownership
(who owns Activity lifecycle callbacks generated from Python
`class MainActivity(Activity)` syntax, threading model for
`async`/`await` against Android's main-thread constraints, etc.).
That boundary should get its own agreement once Phase 4/6 of the
roadmap is close enough to have concrete cases to generalize from,
the same way this document only exists because ARKtube's
WebView-ownership document existed *after* two real bugs, not
before them.
