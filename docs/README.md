# Index for ARKlin Documents

* [Foundational/PROBLEM-STATEMENT.md](Foundational/PROBLEM-STATEMENT.md) -- the design document: what ARKlin is, why it compiles full Python syntax into JVM `.class` files instead of embedding a Python interpreter, and why that specifically matters for treating Android as a real compilation target rather than an emulation environment.
* [Foundational/SYSTEM-DESIGN-AGREEMENTS.md](Foundational/SYSTEM-DESIGN-AGREEMENTS.md) -- who's allowed to own an operation's codegen: the compiler's static specialization path, or the runtime's dynamic `PyObject` dispatch path. Never both, never neither -- that ambiguity is the failure shape this doc exists to rule out before it produces a bug that only shows up as unexplained slowness.
* [Foundational/CODE-STYLE.md](Foundational/CODE-STYLE.md) -- currently a placeholder. There's no code yet to extract real conventions from; it records the package boundaries already committed to (`compiler/`, `runtime/`, `android/`, and the one-way AST → semantics → Core IR → optimizer → JVM pipeline) and leaves everything else open until Phase 0 produces something concrete to generalize from.
* [Foundational/COMPILER-IMPLEMENTATION.md](Foundational/COMPILER-IMPLEMENTATION.md) -- proposal (not yet adopted) for how the Core IR → JVM IR lowering stage should be organized: many small, independently named, single-purpose passes with ordering kept in one manifest, rather than a monolithic optimizer. Draws organizational lessons from the Kotlin compiler's IR backend and from CPython's own staged AST-to-bytecode pipeline -- architecture only, no code reused from either.
* [Foundational/IMPLEMENTATION-STAGES.md](Foundational/IMPLEMENTATION-STAGES.md) -- proposal (not yet adopted) for the vertical slices below README's own Phase 0/1: defines Stage 0 as primitive types plus user-defined functions plus a `@fn` Kotlin escape hatch, which lets a typed Python stub bind directly to a hand-written Kotlin `@JvmStatic` method via ordinary class-file linking rather than C-style FFI, so the pipeline has something real to run end to end before every code path exists natively.
* [bugs-caught/README.md](bugs-caught/README.md) -- active bug tracker. Empty for now -- bugs get caught once there's compiled output to break. Bugs stay listed here until fixed, tested, and confirmed working.

---

## Philosophy

These documents exist for the same reason the compiler pipeline
itself is specified as a chain of separately-named stages instead of
one pass that goes straight from AST to bytecode: **the project
should be legible to whoever opens it next, including a future
version of whoever wrote it.**

A few things that follow from that:

* **Explain the constraint, not just the code.** `PROBLEM-STATEMENT.md`
  exists to answer "why compile instead of interpret" and "why is
  Python syntax not the same commitment as CPython semantics" --
  answers that won't be visible just from reading generated `.class`
  files. `SYSTEM-DESIGN-AGREEMENTS.md` exists to answer "why does
  this operation get specialized and that one doesn't" before the
  answer has to be reverse-engineered from a profiler. If a future
  change makes one of these explanations stop being true, updating
  the doc is part of the change, not a follow-up.
* **Don't reach for structure the problem hasn't asked for yet.**
  `README.md`'s own "Development strategy" section says this
  directly: get `print(2 + 3)` working end to end before building an
  abstract type system nobody has needed yet. `CODE-STYLE.md` follows
  the same instinct by staying a placeholder rather than inventing
  conventions with no code behind them.
* **Assume things will fail, and make the failure observable.**
  `PROBLEM-STATEMENT.md` §12 requires the compiler to diagnose
  unsupported semantics separately from syntax errors -- a construct
  the parser accepts but the backend can't yet lower should fail
  loudly and specifically, not silently fall back to something
  wrong. The same instinct applies to documentation: a codegen
  ownership decision that isn't written down anywhere is a decision
  that will look like an accident the first time the optimizer and
  the runtime both think they specialized the same operation.
* **Stay small on purpose.** The compiler's own architecture keeps a
  hard line between "syntax support" and "semantic support" as
  separate milestones (`README.md`, "Full Python syntax") specifically
  so the parser doesn't grow scope it can't back up yet. This docs
  folder holds itself to the same standard -- a doc exists because
  something needs explaining that isn't obvious from the design docs
  it references, not as a matter of course.
* **Bugs stay visible until they're actually gone.** `bugs-caught/`
  isn't a changelog -- an entry is removed only once it's fixed,
  tested, and confirmed, not once a fix has merely been attempted.
