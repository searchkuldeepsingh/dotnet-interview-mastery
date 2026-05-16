# 🚀 Fundamentals — Interview Master Guide

> 🧠 Structured for senior-level interview preparation  
> ⚡ Fast revision + deep understanding  

---

## 📌 Overview
> This section covers key concepts required for real-world interviews.

---

# .NET Fundamentals

## Overview

The .NET platform is a comprehensive ecosystem for building applications. Understanding its foundational concepts is critical for any .NET developer, especially for senior-level interviews where you'll be asked to explain how the runtime, languages, and frameworks work together.

---

## Key Concepts

### CLR (Common Language Runtime)
- **Definition:** The execution engine that manages running .NET applications
- Handles **memory management** via garbage collection
- Provides **thread control** and **exception handling**
- Verifies type safety at runtime

### CTS (Common Type System)
- **Definition:** Defines how types are declared, managed, and enforced
- Standardizes data types across all .NET languages
- Enables **language interoperability** - C#, VB, F# can share types

### CLS (Common Language Specification)
- **Definition:** Set of rules for language interoperability
- Subset of CTS that all .NET languages must support
- Ensures code written in one language can be used by another

---

## .NET Evolution

### .NET Framework vs .NET Core vs .NET 5+

| Aspect | .NET Framework | .NET Core | .NET 5+ |
|--------|---------------|-----------|---------|
| **Platform** | Windows only | Cross-platform | Cross-platform |
| **Source** | Partial open-source | Fully open-source | Fully open-source |
| **Release** | Service packs | Monthly, LTS | Monthly, LTS |
| **Performance** | Good | Better | Best |
| **API Scope** | Windows-focused | Subset | Unified |

- **.NET Core:** Complete rewrite, optimized for modern workloads
- **.NET 5+:** Unifies .NET (removed "Core" suffix), single platform

---

## Assemblies

### What is an Assembly?
- **Definition:** Fundamental unit of deployment in .NET
- Contains **IL (Intermediate Language)** code and **metadata**
- Has a **manifest** (metadata about version, dependencies, types)

### Types of Assemblies

- **EXE (Executable)** - Has entry point, runs as process
- **DLL (Library)** - Reusable code, loaded by other applications
- **Strong-named** - Signed with cryptographic key for uniqueness
- **Dynamic** - Created programmatically at runtime

---

## Managed vs Unmanaged Code

### Managed Code
- Runs under **CLR supervision**
- CLR handles **memory (GC), type safety, exceptions**
- Examples: C#, VB.NET, F#

### Unmanaged Code
- Runs **outside CLR control**
- Developer handles memory manually
- Examples: COM components, C/C++ via P/Invoke, Win32 APIs

### Key Difference
- **Managed:** Automatic memory via GC
- **Unmanaged:** Explicit memory handling required

---

## GAC (Global Assembly Cache)

### What is GAC?
- Machine-wide code cache in Windows
- Stores shared assemblies for multiple applications

### When to Use
- Multiple apps need same assembly version
- Framework-level assemblies (System.NET.Http)
- Global deployment scenarios

### When NOT to Use
- Application-specific assemblies → use bin folder
- When version flexibility needed → complex side-by-side versioning

### Modern Best Practice
- **Avoid GAC** unless absolutely necessary
- Use **NuGet packages** or **bin-deploy assemblies** instead

---

## IL (Intermediate Language)

### What is IL?
- **Definition:** Low-level, platform-agnostic instruction set
- All .NET languages compile to IL (also called CIL)
- CLR interprets/compiles IL

### Why IL Matters

- **Language Interoperability** - Any .NET language can consume any other
- **Platform Independence** - Same IL runs on any OS with CLR
- **Security** - CLR verifies IL before execution
- **Performance** - JIT optimizes based on runtime conditions
- **Reflection** - Metadata enables runtime inspection

---

## JIT vs AOT Compilation

### JIT (Just-In-Time)
- Compiles IL to native code **at runtime**
- Method compiled **on-demand** and cached
- Optimizes based on **actual runtime conditions**

### Advantages over AOT

| JIT | AOT |
|-----|-----|
| Smaller deployment size | Larger (pre-compiled) |
| Platform optimization | Less platform-specific |
| Better runtime optimization | Less runtime data |
| Faster initial startup | Slower initial startup |

### .NET Native & ReadyToRun
- **.NET Native** (UWP): Pure AOT for faster startup
- **.NET 7+ ReadyToRun:** AOT hybrid for improved startup

---

## Process vs AppDomain

### Process
- **OS-level** execution unit
- Full isolation (memory, resources)
- Created by OS

### AppDomain (Deprecated in .NET Core)
- **.NET runtime** isolation container
- Logical isolation **within a process**
- Could load different assembly versions side-by-side

| Aspect | Process | AppDomain |
|--------|---------|-----------|
| Created By | OS | CLR |
| Isolation | Full (memory) | Logical |
| Communication | IPC | Direct reference |

### Modern Replacement
- **Process isolation** and **Assembly Load Contexts** in .NET 5+

---

## Value Types vs Reference Types

### Value Types
- Stored on **stack** (or inline in objects)
- **Copy on assignment**
- Inherit from **System.ValueType**
- Examples: `int`, `float`, `struct`, `enum`
- Cannot be null (unless nullable with `?`)
- Small, fixed memory footprint

### Reference Types
- **Heap** storage (with stack reference)
- **Reference copied** on assignment
- Inherit from **System.Object**
- Examples: `class`, `interface`, `delegate`, `array`
- Can be null
- Variable memory size

### Memory Management
- **Value types:** Cleaned up on stack unwinding
- **Reference types:** Managed by **GC**

### Boxing/Unboxing
- **Boxing:** Value type → Reference type (heap allocation)
- **Unboxing:** Reference type → Value type
- **Overhead:** Both operations have performance cost

---

## Execution Pipeline

### Pipeline Stages

1. **Source Code** → C#, VB, F#
2. **Language Compiler** → IL + Metadata (Assembly)
3. **Loader** → Loads assembly, validates IL, security checks
4. **JIT Compiler** → Converts IL → Native machine code
5. **CPU Execution** → Runs native code

### Loader Responsibilities
- Reads assembly **metadata**
- Resolves **type references**
- Sets up internal structures
- Applies **security checks**
- Prepares method stubs for JIT

### JIT Compiler Responsibilities
- Converts IL to native code **per-method**
- Applies **runtime optimizations**
- Caches compiled code
- Handles verification

---

## Code Examples

### Value Type vs Reference Type

```csharp
// Value Type (struct) - copied by value
struct Point { public int X, Y; }
var p1 = new Point { X = 1, Y = 2 };
var p2 = p1;  // p2 is a COPY
p2.X = 10;    // p1.X still = 1

// Reference Type (class) - copied by reference
class Person { public string Name; }
var p1 = new Person { Name = "John" };
var p2 = p1;  // p2 points to SAME object
p2.Name = "Jane";  // p1.Name also = "Jane"
```

### Boxing/Unboxing

```csharp
int i = 42;
object o = i;       // Boxing - heap allocation
int j = (int)o;     // Unboxing - performance cost
```

### JIT Compilation

```csharp
// At runtime, JIT compiles this to native machine code
public int Add(int a, int b)
{
    return a + b;
}
```

---

## Key Takeaways

- **CLR** is the execution engine; **CTS** is the type system; **CLS** is the interoperability contract
- **.NET Core** is the cross-platform rewrite; **.NET 5+** unifies the platform
- **Assemblies** are the unit of deployment; GAC is for shared system assemblies
- **Managed code** has GC; **unmanaged code** requires manual memory handling
- **IL** enables language interoperability; **JIT** compiles at runtime with optimizations
- **AppDomain** is deprecated; use **Process isolation** instead
- **Value types** = stack (copy); **Reference types** = heap (reference)
- **Boxing** converts value to reference; adds performance overhead
- **Loader** validates and prepares; **JIT** compiles to native code

---

*End of Section 1: Fundamentals*

---

## 🎯 Quick Revision
- Focus on WHY + HOW  
- Practice explaining verbally  
- Think in real-world scenarios  

---

## 🎤 Interview Tip
> Don’t just answer — explain trade-offs and real-world usage.
