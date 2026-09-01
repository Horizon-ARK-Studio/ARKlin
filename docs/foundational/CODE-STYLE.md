# ARKlin -- How We Write Code

**Status:** Placeholder -- no code exists yet
**Scope:** will apply to `compiler/`, `runtime/`, `android/` once Phase 0 of
the roadmap in `PROBLEM-STATEMENT.md` produces its first source files

ARKtube's `CODE-STYLE.md` documents conventions that were extracted
*after* a real refactor (`MainActivity.kt` growing to ~1,700 lines
before being split). ARKlin doesn't have that history yet -- there is
no `.class` file, no IR, no parser output. Writing a retrospective
style guide before there's anything to retrofit would be inventing
precedent that doesn't exist, which is exactly the kind of premature
structure `README.md`'s "Development strategy" section warns against:

> Do not spend six months implementing an abstract type system before
> producing one executable class file.

This document is deliberately thin until that changes.

---

## What's already decided (from the design docs, not invented here)

These aren't code-style rules yet -- they're the structural
boundaries `README.md` and `PROBLEM-STATEMENT.md` already commit to,
restated here so future style rules have something to attach to:

```text
compiler/
├── parser/
├── ast/
├── semantics/
├── types/
├── lowering/
├── ir/
├── optimizer/
└── jvm/

runtime/
├── objects/
├── types/
├── numbers/
├── strings/
├── collections/
├── exceptions/
├── functions/
├── generators/
├── async/
├── reflection/
└── interop/

android/
├── app/
├── template/
└── build-tools/
```

The one boundary that's a hard agreement, not a suggestion: **AST →
semantics → Core IR → optimizer → JVM backend** is a one-way pipeline
(`README.md`, "Architecture"). A package upstream of Core IR must
never depend on a package downstream of it. That's checkable today,
before a single class is written, so it belongs here now rather than
waiting.

The compiler/runtime ownership split for codegen decisions --
*which* side is allowed to specialize an operation -- is covered in
`SYSTEM-DESIGN-AGREEMENTS.md`, not here. That document is about who
owns a decision; this one will be about how the resulting code is
shaped once that decision is made.

---

## What isn't decided yet

Deliberately left open until real code forces the question, per the
same "don't reach for structure the problem hasn't asked for yet"
instinct as ARKtube's version of this document:

- Kotlin style conventions (naming, nullability handling, whether
  sealed classes or visitor dispatch represent the AST/IR node
  hierarchy -- this has real performance consequences for a
  compiler's hot path and shouldn't be picked by convention before
  it's profiled)
- Error/diagnostic reporting shape (`PROBLEM-STATEMENT.md` §12 item 3
  requires diagnosing unsupported semantics separately from syntax
  errors, but not the mechanism)
- Test layout and what a `tests/` entry is required to prove for each
  roadmap phase gate

This file gets filled in once Phase 0 ("One executable `.class`")
lands and there is an actual file to point at instead of a plan.
