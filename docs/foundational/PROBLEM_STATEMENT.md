# Problem Statement

## Project: Python-Syntax Compiler and Runtime for JVM/Android

### 1. Problem

There is a useful middle ground between running Python through a
conventional Python interpreter and inventing an entirely new
programming language.

Python already provides a mature, expressive syntax:

-   Functions
-   Classes
-   Lambdas
-   Comprehensions
-   Generators
-   Decorators
-   Exceptions
-   `async` / `await`
-   Pattern matching
-   Context managers
-   Multiple assignment
-   Slicing
-   Unpacking
-   Rich expression syntax

The project therefore does **not** need to invent a new surface syntax.

Instead, it will accept **full Python syntax** and compile Python source
into JVM class files, with a project runtime providing the Python
semantics that cannot be represented directly by JVM primitives.

The JVM and Android runtime then take responsibility for execution,
memory management, JIT compilation, threading, and platform integration.

The central idea is:

> Python syntax in. JVM `.class` files out.

The language implementation itself may be written in Kotlin, but Kotlin
is only the implementation language. It does not dictate the programming
model of the compiled language.

------------------------------------------------------------------------

## 2. Core Architecture

``` text
                     Python source
                           |
                           v
                  Python parser / AST
                           |
                           v
                 Semantic analysis
                           |
                           v
                       Core IR
                           |
                    +------+------+
                    |             |
                    v             v
                Optimizer     Runtime linker
                    |
                    v
                 JVM IR
                    |
                    v
               .class files
                    |
                    v
              JVM / Android ART
```

The Android project provides scaffolding and packaging around the
compiler:

``` text
Python project
      |
      v
Android compiler/scaffolding
      |
      +---- compile
      +---- package
      +---- runtime
      +---- resources
      |
      v
   APK / AAB
```

------------------------------------------------------------------------

## 3. Full Python Syntax

The parser should target the Python grammar rather than a deliberately
tiny Python-like subset.

Examples include:

``` python
def add(a, b):
    return a + b
```

``` python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
```

``` python
values = [x * 2 for x in numbers if x > 5]
```

``` python
@decorator
def function(x, *args, **kwargs):
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

The parser should be able to represent the complete syntax tree even
when a particular feature is not yet supported by the backend.

This creates an important separation:

``` text
Python grammar
      |
      v
complete AST
      |
      v
semantic capability analysis
      |
      v
lowering
```

A syntax feature therefore does not have to be removed from the parser
merely because its implementation is unfinished.

------------------------------------------------------------------------

## 4. Python Syntax Does Not Mean CPython Semantics

The project must explicitly distinguish syntax compatibility from
runtime compatibility.

The objective is:

> Accept Python source syntax and provide a well-defined
> Python-compatible semantic model where implemented.

It is **not**:

> Embed CPython and somehow turn every Python runtime operation into JVM
> bytecode.

This distinction is essential.

Python's dynamic object model can require runtime dispatch:

``` python
a + b
```

Conceptually:

``` text
a
 |
 +--> object/type
          |
          +--> __add__
                    |
                    v
                 result
```

Where static information proves the operation safe, the compiler may
instead lower it to primitives:

``` text
Int + Int
   |
   v
JVM iadd
```

The runtime therefore remains available for dynamic behavior while the
compiler can specialize predictable code.

------------------------------------------------------------------------

## 5. Runtime

The runtime provides the semantics that cannot be represented directly
as ordinary JVM primitives.

Potential runtime components include:

``` text
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
```

A dynamic value may conceptually be represented as:

``` text
PyObject
 ├── type
 └── payload
```

The exact representation is an implementation decision and should be
optimized through measurement.

The runtime should not blindly box every value.

For example:

``` python
def add(a: int, b: int):
    return a + b
```

may eventually lower toward:

``` text
add(int, int) -> int
```

while genuinely dynamic code may use:

``` text
add(PyObject, PyObject) -> PyObject
```

This creates a path toward specialization and boxing elimination.

------------------------------------------------------------------------

## 6. Classes and Object Support

Python classes are supported because full Python syntax requires them.

A Python class:

``` python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
```

can be lowered into a JVM class:

``` text
Point.class

fields:
    x
    y

methods:
    <init>
```

This does not mean the compiler architecture itself needs to be
object-oriented.

Functions, data structures, algebraic representations, and compiler IR
can remain independent of class-oriented design.

The JVM's object model is used as a target representation where
appropriate.

------------------------------------------------------------------------

## 7. `.class` Is the Primary Backend Boundary

The compiler's concrete output should be valid JVM class files.

Example:

``` text
program.py
   |
   v
compiler
   |
   +-- Main.class
   +-- Point.class
   +-- Runtime.class
   |
   v
JVM / Android toolchain
```

The compiler does not need to implement a JVM.

The JVM already provides:

-   Bytecode execution
-   JIT compilation
-   Garbage collection
-   Threads
-   Memory management
-   Class loading
-   Verification
-   Profiling
-   Debugging infrastructure

The compiler should exploit those capabilities instead of rebuilding
them.

------------------------------------------------------------------------

## 8. Android Application Scaffolding

The project should provide a way to create Android applications directly
from Python source.

Conceptually:

``` text
new project
    |
    +-- app.py
    +-- project configuration
    +-- resources
    |
    v
compile
    |
    v
.class
    |
    v
Android/JVM toolchain
    |
    v
DEX
    |
    v
APK / AAB
```

The Android tooling should handle:

-   Project generation
-   Compiler invocation
-   Runtime inclusion
-   Resource packaging
-   Manifest generation
-   JVM/Android class conversion
-   APK/AAB generation
-   Debug builds
-   Release builds

The generated application should execute compiled code rather than
requiring Python source to be interpreted at runtime unless a feature
explicitly requires runtime evaluation.

------------------------------------------------------------------------

## 9. Compiler Pipeline

The intended compiler pipeline is:

``` text
Python source
      |
      v
Parser
      |
      v
Python AST
      |
      v
Name resolution
      |
      v
Type / semantic analysis
      |
      v
Python-compatible semantic IR
      |
      v
Core IR
      |
      v
Optimization
      |
      v
JVM IR
      |
      v
.class
```

A future optimization pipeline may perform:

-   Constant folding
-   Dead-code elimination
-   Function inlining
-   Primitive specialization
-   Boxing elimination
-   Closure conversion
-   Allocation reduction
-   Escape analysis
-   Tail-call optimization where semantically valid
-   Common subexpression elimination

Optimization must preserve the defined language semantics.

------------------------------------------------------------------------

## 10. Design Principles

### Reuse syntax

Do not invent syntax when Python already expresses the construct
clearly.

### Separate syntax from semantics

Python's grammar is an input format, not a commitment to CPython's
implementation.

### Compile ahead of execution

Prefer generated JVM bytecode over interpreting source at runtime.

### Runtime only where necessary

Dynamic semantics belong in the runtime when static compilation cannot
resolve them.

### Optimize proven cases

Use type and flow information to specialize code when safe.

### Keep the runtime small

Do not reproduce machinery already supplied by the JVM.

### Treat generated code as a first-class artifact

Compiler decisions should ultimately be explainable in terms of JVM
instructions, allocations, calls, branches, and memory behavior.

------------------------------------------------------------------------

## 11. Initial Milestone

The first end-to-end milestone is intentionally small:

``` python
def main():
    print(2 + 3)
```

The complete path must work:

``` text
app.py
  ↓
Python parser
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
program executes
```

After this exists, language coverage and runtime compatibility can grow
incrementally.

------------------------------------------------------------------------

## 12. Success Criteria

The project succeeds when it can:

1.  Parse full Python syntax.
2.  Produce a complete AST.
3.  Diagnose unsupported semantics separately from syntax errors.
4.  Lower supported Python constructs into a common IR.
5.  Generate valid JVM `.class` files.
6.  Execute those classes on the JVM.
7.  Package them for Android.
8.  Provide a runtime for dynamic Python-compatible operations.
9.  Specialize statically known operations where beneficial.
10. Maintain a clear boundary between compiler, runtime, and Android
    scaffolding.

The goal is not to make another interpreter.

The goal is to make Python source a practical input language for a
JVM/Android compiler.


---

## 13. Native Android Application Requirement

Android output is a first-class requirement of the project.

A program written using Python syntax must ultimately become a **native Android application**, not a Python application hosted inside an embedded Python interpreter.

The distinction is:

```text
Python source
     |
     v
Custom compiler
     |
     v
JVM .class files
     |
     v
Android build toolchain
     |
     v
DEX / Android application artifacts
     |
     v
Native Android application
```

The Android application should integrate with the Android platform through normal Android mechanisms.

The fact that the source language uses Python syntax must not be visible as a requirement for the device to execute Python source.

### Native means

The generated application should be able to use the Android platform through compiled/runtime bindings:

```text
Python syntax
     |
     v
compiler/runtime
     |
     v
JVM / Android APIs
     |
     v
Android framework
```

Examples include:

- Activities
- Services
- Views
- Android lifecycle APIs
- Resources
- Intents
- Notifications
- Storage
- Networking
- Threads/coroutines where appropriate

The exact interop model is an implementation concern, but Android must remain a real compilation target rather than an emulation environment.

### No source-level interpreter requirement

The default execution model must not be:

```text
Python source
    ↓
embedded interpreter
    ↓
Android application
```

Instead it should be:

```text
Python source
    ↓
custom compiler
    ↓
compiled classes
    ↓
Android/Dex toolchain
    ↓
native Android application
```

A runtime library may exist to implement Python-compatible semantics, but that runtime is part of the compiled application's implementation. It is not the Python interpreter itself.

This distinction is fundamental to the project's architecture and performance goals.
