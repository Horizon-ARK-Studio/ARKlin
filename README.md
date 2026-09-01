# \[Project Name\]

> Python syntax. JVM classes. Android apps.

Most language projects begin by inventing syntax.

This one doesn't.

Python already has syntax for functions, classes, lambdas,
comprehensions, decorators, exceptions, generators, pattern matching,
`async`, context managers, unpacking, slicing, and the other little
grammatical treasures people have spent years learning.

So the project keeps the syntax and moves the interesting work
underneath it.

**Write Python. Compile to `.class`. Let the JVM do the rest.**

------------------------------------------------------------------------

## What is this?

A compiler and runtime that accepts **full Python syntax** and targets
the JVM.

``` text
             Python
               |
               v
             Parser
               |
               v
              AST
               |
               v
        Semantic Analysis
               |
               v
             Core IR
               |
               v
           Optimizer
               |
               v
          JVM Backend
               |
               v
            .class
               |
               v
          JVM / Android
```

An Android scaffolding layer turns the same compiler into an application
build pipeline:

``` text
app.py
  |
  v
compile
  |
  v
.class
  |
  v
Android toolchain
  |
  v
APK / AAB
```

The compiler may be implemented in Kotlin.

The language is not Kotlin.

The target is not Kotlin.

Kotlin is simply the machinery used to build the machinery. A compiler
written in Kotlin does not magically turn its language into an OOP
curriculum.

------------------------------------------------------------------------

## The central idea

There are two things that are easy to confuse:

**Python syntax**

and

**Python's runtime implementation.**

This project wants the first without blindly inheriting the second.

For example:

``` python
def add(a: int, b: int):
    return a + b
```

The compiler can know that both arguments are integers and potentially
lower the operation toward:

``` text
int + int
   |
   v
JVM iadd
```

But genuinely dynamic code can use the runtime:

``` text
PyObject
   |
   v
dynamic operation lookup
   |
   v
runtime implementation
```

This gives the project two useful paths:

``` text
predictable code
    -> specialized JVM operations

dynamic code
    -> runtime semantics
```

The compiler can gradually move more programs from the second path
toward the first as analysis improves.

------------------------------------------------------------------------

# Full Python syntax

The parser is intended to understand the full Python grammar.

That includes constructs such as:

``` python
def calculate(x, y=10, *args, **kwargs):
    return x + y
```

``` python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
```

``` python
values = [
    transform(x)
    for x in items
    if x > 10
]
```

``` python
@decorator
def function():
    ...
```

``` python
async def fetch():
    ...
```

``` python
match value:
    case 0:
        ...
    case [x, y]:
        ...
```

``` python
with resource() as r:
    process(r)
```

The parser should not become artificially incomplete simply because the
compiler has not implemented every semantic feature yet.

The architecture is:

``` text
complete Python grammar
          |
          v
       complete AST
          |
          v
   semantic capability
       analysis
          |
          v
       lowering
```

So:

> Syntax support and semantic support are separate milestones.

------------------------------------------------------------------------

# Not CPython

This is not an attempt to bolt a JVM onto CPython.

There is no requirement that:

``` python
x + y
```

must execute by reproducing every internal CPython operation.

The compiler owns the semantic model.

Where the language requires Python-compatible dynamic behavior, the
runtime supplies it.

Where the compiler can prove something more specific, it can generate
more direct JVM code.

For example:

``` text
Dynamic:

PyObject + PyObject
        |
        v
 runtime dispatch


Static:

Int + Int
   |
   v
 iadd
```

That distinction is fundamental to performance.

------------------------------------------------------------------------

# `.class` is the output

The compiler's primary concrete artifact is the JVM class file.

A Python file may produce:

``` text
Main.class
Point.class
Network.class
RuntimeSupport.class
```

The JVM then handles:

-   Execution
-   JIT compilation
-   Garbage collection
-   Threads
-   Memory management
-   Class loading
-   Verification
-   Profiling
-   Debugging

There is no reason to write another JVM merely because compiler projects
enjoy occasionally wandering into unnecessary territory.

------------------------------------------------------------------------

# Classes are supported

Python classes work because full Python syntax includes classes.

Example:

``` python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def distance(self):
        return self.x * self.x + self.y * self.y
```

The compiler can lower this into a JVM class with fields and methods.

Conceptually:

``` text
Point.class

fields:
    x
    y

methods:
    <init>
    distance
```

This does not force the compiler's own architecture to become
class-centric.

Compiler IR can remain data-oriented.

Functional code can remain function-oriented.

The JVM object model is simply one useful target representation.

------------------------------------------------------------------------

# Runtime

Dynamic Python-compatible behavior requires runtime support.

A possible runtime layout:

``` text
runtime/
├── objects/
├── types/
├── numbers/
├── strings/
├── collections/
├── functions/
├── exceptions/
├── generators/
├── async/
├── reflection/
└── interop/
```

The runtime should handle things the JVM does not natively understand as
Python semantics.

Examples:

``` text
Python object model
Attribute lookup
Dynamic dispatch
Python exceptions
Iterators
Generators
Dynamic calls
Descriptors
Selected reflection behavior
```

But runtime code should not automatically be involved in every
operation.

If the compiler knows:

``` python
x: int
y: int
```

then:

``` python
x + y
```

should have a route toward primitive JVM arithmetic.

------------------------------------------------------------------------

# Android

Android is a first-class target.

The project provides scaffolding so a Python source tree can become an
Android application.

Conceptually:

``` text
Project
├── app.py
├── resources/
└── configuration
       |
       v
    Compiler
       |
       v
   JVM classes
       |
       v
 Android build
       |
       v
      DEX
       |
       v
    APK/AAB
```

The Android layer is responsible for project generation and packaging
rather than redefining the language.

The same compiler frontend, semantic system, IR, optimizer, and JVM
backend should be reusable outside Android.

------------------------------------------------------------------------

# Architecture

A possible repository structure:

``` text
project/
├── compiler/
│   ├── parser/
│   ├── ast/
│   ├── semantics/
│   ├── types/
│   ├── lowering/
│   ├── ir/
│   ├── optimizer/
│   └── jvm/
│
├── runtime/
│   ├── objects/
│   ├── types/
│   ├── collections/
│   ├── exceptions/
│   ├── async/
│   └── interop/
│
├── android/
│   ├── app/
│   ├── template/
│   └── build-tools/
│
├── stdlib/
│
├── cli/
│
└── tests/
```

The module boundaries can change.

The important boundary is:

``` text
AST
 ↓
semantics
 ↓
Core IR
 ↓
optimizer
 ↓
JVM backend
```

------------------------------------------------------------------------

# Core IR

The compiler should not translate Python AST directly into bytecode.

Instead:

``` text
Python AST
    |
    v
Semantic model
    |
    v
Core IR
    |
    v
JVM lowering
    |
    v
.class
```

For example:

``` python
x = 10
y = x + 20
```

might become:

``` text
v0 = const_int 10
v1 = const_int 20
v2 = add_int v0, v1
return v2
```

and eventually:

``` text
bipush 10
bipush 20
iadd
ireturn
```

The IR is what allows the compiler to reason about the program
independently from JVM bytecode details.

------------------------------------------------------------------------

# Optimization

Optimization comes after correct semantics.

Potential passes include:

-   Constant folding
-   Dead-code elimination
-   Inlining
-   Primitive specialization
-   Boxing elimination
-   Closure conversion
-   Allocation reduction
-   Escape analysis
-   Common subexpression elimination
-   Tail-call optimization where valid

The compiler should care about what an abstraction becomes.

A high-level expression eventually becomes:

``` text
instructions
registers / stack slots
loads
stores
branches
calls
allocations
object references
cache accesses
```

That is where performance actually lives.

------------------------------------------------------------------------

# Android scaffolding

The Android developer experience should eventually look something like:

``` text
$ pythonjvm new MyApp
$ pythonjvm build
$ pythonjvm install
$ pythonjvm run
```

Or through the Android application:

``` text
New Project
     |
     v
Python source
     |
     v
Build
     |
     v
APK
```

The exact CLI and UI are implementation details.

The important thing is that the compiler is usable as a development tool
rather than merely being a compiler demo.

------------------------------------------------------------------------

# Development strategy

The project should be built vertically.

Do not spend six months implementing an abstract type system before
producing one executable class file. That is how compiler projects
become elaborate folders containing interfaces that have never seen a
CPU.

First make this work:

``` python
def main():
    print(2 + 3)
```

End-to-end:

``` text
app.py
  ↓
parser
  ↓
AST
  ↓
semantic analysis
  ↓
Core IR
  ↓
JVM backend
  ↓
Main.class
  ↓
Android packaging
  ↓
APK
  ↓
execution
```

Then expand:

``` text
integers
  ↓
strings
  ↓
functions
  ↓
classes
  ↓
lists
  ↓
exceptions
  ↓
closures
  ↓
generators
  ↓
async
  ↓
interop
```

The parser can target full syntax early, while semantic implementation
proceeds incrementally.

------------------------------------------------------------------------

# Roadmap

## Phase 0: Compiler skeleton

-   [ ] Kotlin compiler project
-   [ ] Python parser integration
-   [ ] Complete AST representation
-   [ ] Basic diagnostics
-   [ ] Core IR
-   [ ] JVM class-file generation
-   [ ] One executable `.class`

## Phase 1: Executable Python subset

-   [ ] Integer operations
-   [ ] Boolean operations
-   [ ] Strings
-   [ ] Variables
-   [ ] `if`
-   [ ] `while`
-   [ ] Functions
-   [ ] Function calls
-   [ ] `print`
-   [ ] Exceptions

## Phase 2: Python object model

-   [ ] Objects
-   [ ] Classes
-   [ ] Attributes
-   [ ] Methods
-   [ ] Dynamic dispatch
-   [ ] `__init__`
-   [ ] Special methods
-   [ ] Attribute lookup

## Phase 3: Python language features

-   [ ] Lists
-   [ ] Tuples
-   [ ] Dictionaries
-   [ ] Sets
-   [ ] Comprehensions
-   [ ] Iterators
-   [ ] Generators
-   [ ] Decorators
-   [ ] Context managers
-   [ ] Pattern matching

## Phase 4: Advanced runtime

-   [ ] `async`
-   [ ] `await`
-   [ ] Reflection
-   [ ] Dynamic imports
-   [ ] JVM interoperability
-   [ ] Android APIs

## Phase 5: Optimization

-   [ ] Type specialization
-   [ ] Primitive lowering
-   [ ] Boxing elimination
-   [ ] Inlining
-   [ ] Escape analysis
-   [ ] Allocation optimization
-   [ ] Profile-guided optimization

## Phase 6: Android product

-   [ ] Android project scaffolding
-   [ ] Resource integration
-   [ ] Manifest generation
-   [ ] APK builds
-   [ ] AAB builds
-   [ ] Debug builds
-   [ ] Release builds
-   [ ] Install/run workflow

------------------------------------------------------------------------

# What this project is

``` text
Python syntax
       +
compiler
       +
runtime
       +
JVM
       +
Android tooling
```

# What this project is not

``` text
CPython rewritten in Kotlin
```

``` text
A Python interpreter hidden inside an Android app
```

``` text
A new language with Python-shaped syntax
```

``` text
A new virtual machine
```

The target is much more direct:

> **Use Python as the source language, compile it into JVM classes,
> provide the missing runtime semantics, and let the JVM/Android
> platform handle the rest.**

------------------------------------------------------------------------

# First Definition of Done

The project has crossed its first meaningful boundary when this:

``` python
def main():
    print(2 + 3)
```

becomes:

``` text
Python source
    ↓
AST
    ↓
Core IR
    ↓
JVM bytecode
    ↓
Main.class
    ↓
Android application
    ↓
5
```

Everything after that is language coverage, runtime engineering,
optimization, and tooling.

Which is still a ridiculous amount of work.

But at least it is the *right* ridiculous amount of work.


---

# Native Android Apps

This project does not produce a Python app that happens to run on Android.

It produces an **Android app**.

Python is the source syntax.

The compiler is the translation layer.

The Android application is the final product.

```text
             app.py
                |
                v
        Custom compiler
                |
                v
             .class
                |
                v
        Android toolchain
                |
                v
              DEX
                |
                v
        Native Android app
```

The important distinction is that the generated application does **not** fundamentally depend on shipping Python source plus a general-purpose Python interpreter and asking that interpreter to execute the application.

The runtime exists to implement the semantics required by the compiled program.

```text
Python syntax
      |
      v
compiler + runtime
      |
      v
JVM / Android representation
      |
      v
Android framework
```

That means Android APIs are intended to be reachable as platform APIs, not simulated through a Python-only environment.

For example, an eventual application could conceptually express Android behavior from Python syntax while compiling into ordinary Android/JVM structures:

```python
class MainActivity(Activity):
    def onCreate(self, state):
        super().onCreate(state)
        setContentView(R.layout.main)
```

The source looks like Python.

The resulting application is still an Android application.

---

## The rule

> **Python syntax in. Native Android application out.**

Not:

```text
Python
  ↓
Python interpreter
  ↓
Android
```

But:

```text
Python
  ↓
custom compiler
  ↓
JVM classes
  ↓
Android/Dex
  ↓
native application
```

This is one of the project's core requirements, not an optional deployment mode.
