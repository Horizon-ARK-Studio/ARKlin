# Compiler Implementation Notes -- IR Lowering Architecture

**Status:** Proposal, not yet adopted
**Scope:** the Core IR → JVM IR half of the pipeline described in
`PROBLEM_STATEMENT.md` §9

This document exists to answer one narrow question: **once semantic
analysis has produced Core IR, what shape should the code that turns
it into JVM `.class` output actually take?** It does not revisit *what*
Core IR should specialize (`SYSTEM-DESIGN-AGREEMENTS.md` already owns
that) -- only *how the lowering machinery that acts on Core IR should
be organized*, so that document's compiler/runtime boundary is easy to
enforce in practice rather than something every pass has to
re-derive.

The recommendation below was formed by reading, at the architecture
level (directory layout, module boundaries, and how passes are
orchestrated -- not by copying any source), two existing systems that
already solved a version of this problem: the Kotlin compiler's IR
backend, and CPython's own AST-to-bytecode pipeline. Both are cited
for their *organizational* lessons. No code from either project is
reused here, and none should be pasted into ARKlin -- what's portable
is the shape of the solution, not the implementation.

---

## 1. The problem this addresses

`PROBLEM_STATEMENT.md` §9 lists a long, growing set of optimization
concerns: constant folding, dead-code elimination, inlining,
primitive specialization, boxing elimination, closure conversion,
escape analysis, tail-call optimization, common subexpression
elimination -- and `SYSTEM-DESIGN-AGREEMENTS.md` adds a hard
constraint on top of all of them: every one of those transformations
has to be able to point to the specific semantic-analysis fact that
licenses it, or refuse to run.

Writing that as a single "optimizer" function that walks Core IR once
and tries to do all of it inline does not scale to that constraint.
Each concern needs its own place to live, its own reasoning about
what it's allowed to assume, and its own ability to be skipped,
tested, and reasoned about independently -- otherwise the "which
subsystem proved this was safe" question `SYSTEM-DESIGN-AGREEMENTS.md`
requires an answer to becomes impossible to audit once more than one
or two transformations exist.

## 2. What Kotlin's IR backend does about this

Kotlin's own compiler faces a structurally similar problem: one IR
(their `ir.tree`) has to be lowered toward several different targets
(JVM, JS, Native, Wasm), and each target needs many small,
target-specific transformations applied to it in a specific order
before codegen can run.

The organizational answer, visible directly in the repository layout,
is:

- A **shared IR tree module** (`ir.tree`) defines the IR node types
  once, independent of any backend.
- A **shared common-lowering module** (`backend.common`) holds
  transformations that apply across targets.
- **Each backend** (`backend.jvm`, `backend.js`, `backend.native`,
  `backend.wasm`) has its own `lower/` directory containing dozens of
  small, independently named files -- one concern per file (e.g. a
  file for enum class lowering, a separate file for tailrec lowering,
  a separate file for lateinit handling, and so on for several dozen
  concerns), rather than one large pass.
- A single **phase-list file per backend** (e.g.
  `JvmLoweringPhases.kt`) does nothing but declare the fixed order
  those small passes run in. It contains no lowering logic itself --
  its only job is sequencing.

The lesson worth taking is not any specific pass. It's the structural
decision that **"what does this transformation assume, and is that
assumption still true" is answerable per-file**, because each file
owns exactly one transformation and nothing else. A reviewer (or a
future maintainer) can look at one lowering in isolation and check its
precondition without having to hold the entire optimizer in their
head at once.

## 3. What CPython's own pipeline confirms independently

CPython's `compile.c` is a useful second data point precisely because
it is not IR-based in the same sense, and arrives at a compatible
answer anyway: it describes its own AST-to-bytecode path as a fixed
sequence of *named, separately implemented* stages -- future-statement
checking, symbol-table construction, instruction-sequence generation,
control-flow-graph construction with its own optimization pass, and
final assembly -- each documented as owning one job in the pipeline,
each in its own source file, run in a fixed order.

Two independently-developed compilers, aimed at different targets and
written in different languages, both converged on "one stage, one
file, one job, fixed order" rather than a monolithic pass. That's
weak evidence but the right kind: it suggests this is a property of
the problem (many semantically distinct transformations that must
compose correctly), not an accident of Kotlin's or CPython's specific
codebase.

## 4. Recommendation for ARKlin's Core IR → JVM IR stage

Given both precedents, and given that ARKlin's own
`SYSTEM-DESIGN-AGREEMENTS.md` already requires every specialization
decision to cite its justification:

1. **Model Core IR's node types the way `ir.tree` models theirs: as a
   shared, backend-independent tree.** Core IR should not know it is
   eventually headed for the JVM specifically. This keeps the door
   open for the "optimizer vs. runtime linker" split
   `PROBLEM_STATEMENT.md` §2 already draws, and for any future backend
   target, without redesigning the IR.

2. **Implement optimization and lowering as a list of small, named,
   single-purpose passes over Core IR -- not one optimizer function.**
   Each pass should be answerable in isolation against
   `SYSTEM-DESIGN-AGREEMENTS.md`'s rule: "what semantic-analysis fact
   does this pass require before it's allowed to act, and does it
   check for that fact explicitly?" A pass that can't state its
   precondition in one sentence is a sign the precondition doesn't
   actually exist yet.

3. **Keep pass *ordering* in exactly one place**, mirroring the
   phase-list-file pattern: a single manifest that lists which passes
   run, in what order, with no lowering logic of its own. This gives
   the project one canonical place to answer "did X run before Y,"
   the same way `SYSTEM-DESIGN-AGREEMENTS.md` gives the project one
   canonical place to answer "who owns this operation's codegen."

4. **Do not adopt Kotlin's IR node vocabulary wholesale.** Kotlin's IR
   is shaped by Kotlin-specific commitments ARKlin does not share --
   null-safety types, inline classes, coroutine-specific IR nodes --
   and Python's dynamic object model, duck typing, and `PyObject`
   fallback path (§4, §5 of `PROBLEM_STATEMENT.md`) are a different
   set of problems. What transfers is the *organization* of the
   lowering machinery, not the node taxonomy.

5. **Do not adopt CPython's bytecode-level optimizer as a model for
   where in the pipeline optimization happens.** CPython's flow-graph
   optimizations run *after* a stack-machine instruction sequence
   already exists, because CPython bytecode is itself the execution
   target. ARKlin's optimizer sits earlier, at Core IR, specifically
   so that specialization decisions (§5 of `SYSTEM-DESIGN-AGREEMENTS.md`)
   can be made before anything JVM-shaped exists, which is a different
   point in the pipeline for a different reason.

## 5. What this does not decide yet

This document proposes a *shape* for the lowering stage. It does not:

- enumerate the actual list of passes ARKlin needs (that follows the
  same "don't reach for structure the problem hasn't asked for yet"
  principle from `docs/README.md` -- passes get added as the initial
  milestone in `PROBLEM_STATEMENT.md` §11 grows real cases to
  generalize from);
- specify the concrete Core IR node types;
- decide how the phase manifest is expressed in Kotlin (a plain
  ordered list is sufficient at this stage; nothing more elaborate is
  justified until there's a second concern -- e.g. conditional phases
  -- that a plain list can't express).

Those stay open until Phase 0 produces something concrete to
generalize from, consistent with how `CODE-STYLE.md` is handling the
same kind of premature-structure risk.
