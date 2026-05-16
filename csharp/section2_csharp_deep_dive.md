# 🚀 Section2 Csharp Deep Dive — Interview Master Guide

> 🧠 Styled for fast revision + deep understanding

---

## 📌 Overview
# .NET Interview Preparation - Section 2: C# Deep Dive

## 13 Years Experience Candidate Answers

---

## Question 1: Explain the difference between class, struct, record, and enum in C#. When would you choose one over the other?

**Answer:**

- **Class** - Reference type, stored on heap, supports inheritance, can have null references. Use for complex objects requiring polymorphism and reference semantics.

- **Struct** - Value type, stored on stack (or inline), no inheritance, copied by value. Use for small data containers, performance-critical scenarios where allocation overhead matters (like Point, KeyValuePair).

- **Record** - Reference type with built-in value equality, immutability support, positional syntax for init. Use for DTOs, value objects, API response models where you want value semantics.

- **Enum** - Value type representing a set of named constants. Use for fixed sets of options, flags, status codes.

**Key point:** For 13 years experience, I'd emphasize choosing struct for small, frequently created objects (like coordinates, ranges) to avoid GC pressure, and records for modern C# immutable data transfer objects.

---

## Question 2: How does memory allocation work for value types vs reference types in C#? What is the stack and heap?

**Answer:**

Memory in .NET is divided into stack and heap:

**Stack:**
- Fast, LIFO allocation
- Stores value types and method local variables
- Memory allocated at method entry, deallocated at exit
- Small, typically 1MB per thread

**Heap:**
- Large, managed by GC
- Stores reference types and objects
- Allocation via new, deallocation by GC (non-deterministic)

**Allocation process:**
- Value type in method - stack
- Reference type - reference on stack, object on heap
- Class field - heap (part of the containing object)
- Struct inside class - inline in the heap object

**Important:** Stack allocation is virtually free (just moving stack pointer). Heap allocation involves GC overhead. This is why value types can be more performant for small, short-lived data.

---

## Question 3: Explain garbage collection generations in .NET. Why is it designed this way?

**Answer:**

.NET GC uses generations because most objects are short-lived:

**Generations:**
- **Gen 0** - Newly allocated objects, most die young. Fastest to collect.
- **Gen 1** - Objects that survived Gen 0 collection. Longer-lived.
- **Gen 2** - Long-lived objects (static data, cached objects).

**Why generations?**
1. **Performance** - Most objects die young (80-90%). Gen 0 collection is frequent and fast.
2. **Efficiency** - Only scan relevant portions, not entire heap.
3. **Pause time** - Gen 0 is quick, reducing blocking time.

**How it works:**
- Allocation in Gen 0
- Survives - promoted to next generation
- Gen 2 collected only when Gen 0/1 can't free enough memory

**GC Modes:**
- Workstation (default, for client apps)
- Server (for server apps, multiple threads)

With 13 years experience, I'd mention tuning GC for high-throughput scenarios and using GC notifications for monitoring.

---

## Question 4: What is the difference between async/await, Task, and Task<T>? How does the TAP (Task-based Asynchronous Pattern) work internally?

**Answer:**

**Task vs Task<T>:**
- **Task** - Represents an async operation without return value
- **Task<T>** - Represents an async operation that returns value of type T

**async/await:**
- **async** - Compiler transforms method into state machine
- **await** - Suspends method until operation completes, doesn't block thread

**How TAP works internally:**
1. Compiler transforms async method into a state machine class
2. Each await becomes a state in the state machine
3. When await completes, continuation runs on captured synchronization context
4. Uses IAsyncStateMachine interface

**Key points for 13-year experience:**
- Async methods run synchronously until first await (synchronous phase)
- Continuation may run on different thread
- ConfigureAwait(false) avoids context capture
- Don't use Task.Run for I/O-bound work (it wastes thread pool)
- Task is not parallel; it enables efficient async I/O without thread blocking

---

## Question 5: How does LINQ work under the hood? Explain the difference between LINQ to Objects, LINQ to SQL, and LINQ to Entities.

**Answer:**

**LINQ architecture:**
- **IEnumerable<T>** - Base for LINQ to Objects
- **IQueryable<T>** - Base for LINQ to SQL/Entities, enables expression trees

**LINQ to Objects:**
- Works on in-memory collections
- Uses Enumerable extension methods
- Executes immediately or via deferred execution
- No translation, direct .NET code execution

**LINQ to SQL (LINQ to Entities/EF):**
- Uses expression trees (Expression<Func<T>>)
- Converts to SQL via IQueryable provider
- Deferred execution by default
- Generates SQL: SELECT, JOIN, WHERE at database level

**Key differences:**

| Aspect | LINQ to Objects | LINQ to SQL/Entities |
|--------|-----------------|---------------------|
| Provider | Enumerable | Queryable |
| Execution | In-memory | Database |
| Translation | None | Expression tree to SQL |
| Performance | Depends on collection size | Can optimize with SQL |

**Important:** IQueryable delays execution until enumeration. Use .ToList() or .FirstOrDefault() to trigger execution. Always profile generated SQL.

---

## Question 6: What is the difference between IEnumerable<T> and IQueryable<T>? When would you use each?

**Answer:**

**IEnumerable<T>:**
- In-memory enumeration
- LINQ to Objects provider
- Extension methods return IEnumerable<T>
- Executes immediately for immediate operators, deferred for deferred
- Data already loaded or from in-memory sources

**IQueryable<T>:**
- Out-of-memory data sources
- LINQ to SQL, EF Core, OData provider
- Expression tree-based, not delegate-based
- Builds query at provider level (generates SQL)
- Deferred execution always

**When to use IEnumerable:**
- Working with in-memory collections
- Processing already-fetched data
- LINQ to Objects operations
- When you don't need provider-specific translation

**When to use IQueryable:**
- Database queries (EF Core, LINQ to SQL)
- Remote services that support query expression
- When you want query translation to other language
- Pagination, filtering at database level

**Performance tip:** Convert IQueryable to IEnumerable only after all database-side filtering is done to avoid loading unnecessary data.

---

## Question 7: Explain covariance and contravariance in C#. How do in and out keywords affect generic type parameters?

**Answer:**

**Covariance (out):**
- Allows using more derived type than originally specified
- IEnumerable<Dog> can be assigned to IEnumerable<Animal>
- Only works with output positions (return types)

**Contravariance (in):**
- Allows using less derived type than originally specified
- IComparer<Animal> can be assigned to IComparer<Dog>
- Only works with input positions (method parameters)

**in/out keywords:**
- **out** - Covariant, used for output positions (T in return types)
- **in** - Contravariant, used for input positions (T in parameters)

**Example:**
```csharp
IEnumerable<Derived> derived = new List<Derived>();
IEnumerable<Base> base = derived; // Covariance works with 'out'
```

**Real-world use:**
- IEnumerable<T> uses out for read-only collections
- IComparer<T> uses in for comparison
- IFunc<T, TResult> - covariant on TResult, contravariant on T

**Limitation:** Only works with interfaces and delegates, not classes. Value types don't support variance.

---

## Question 8: What is the difference between ref, out, and in parameters? When would you use each?

**Answer:**

**ref:**
- Allows passing value by reference (read/write)
- Caller must initialize the variable before passing
- Called method can read and modify the value
- Changes reflect back to caller

**out:**
- Like ref but caller doesn't need to initialize
- Called method MUST assign a value before returning
- Primarily used for returning multiple values from method
- Cannot use uninitialized variable

**in:**
- Read-only reference (C# 7.2+)
- Like ref but prevents modification
- Provides performance benefit for large structs (no copy)
- Caller can pass uninitialized, but method can't modify
- Good for large value types where copying is expensive

**When to use:**
- **ref** - When you need to modify caller's variable
- **out** - For method with multiple return values (TryParse pattern)
- **in** - For large structs you want to pass efficiently without modification

**Note:** With 13 years experience, I'd emphasize avoiding ref/in for reference types (confusing semantics) and preferring them for large value types only.

---

## Question 9: How does the new keyword work? Explain the difference between new, override, and virtual.

**Answer:**

**new:**
- Hides an inherited member from base class
- Compiler produces warning if you hide without new
- Base class method still accessible via (Base)derivedInstance.Method()

**override:**
- Replaces base class implementation
- Requires base method to be virtual or abstract
- Polymorphic call always reaches derived implementation

**virtual:**
- Declares method can be overridden in derived classes
- Base implementation runs if not overridden
- Must be implemented (abstract) or have default behavior (virtual)

**Key difference:**
```csharp
class Base { public virtual void Foo() { } }
class Derived : Base { 
    public override void Foo() { } // Replaces Base.Foo()
    public new void Bar() { } // Hides Base.Bar() - not polymorphic
}
```

**When to use:**
- Use virtual when designing for extensibility
- Use override when you must change behavior
- Use new only when you must hide (usually indicates design problem)

**13-year insight:** Hiding methods often indicates a design flaw. Consider interface segregation or composition instead.

---

## Question 10: What are extension methods in C#? How are they resolved at compile time?

**Answer:**

Extension methods allow adding methods to existing types without modifying them:

**Definition:**
```csharp
public static class StringExtensions
{
    public static bool IsNullOrEmpty(this string value) 
        => string.IsNullOrEmpty(value);
}
```

**How they work:**
- Static methods with first parameter marked this
- Appear as instance methods on the extended type
- Resolution at compile time, not runtime

**Compile-time resolution:**
1. Compiler looks for instance methods on the type first
2. If no instance method matches, looks for static extension methods in scope
3. Converts call to static method call: str.IsNullOrEmpty() - StringExtensions.IsNullOrEmpty(str)

**Important rules:**
- Only accessible if namespace is imported (using directive)
- Can't override instance methods - instance methods take precedence
- Can be used on null (null is passed to the method)

**Common usage:** LINQ extension methods, fluent interfaces, framework helpers.

---

## Question 11: Explain the difference between string and StringBuilder. When should you use each?

**Answer:**

**string (immutable):**
- Immutable - any modification creates new string
- Each modification allocates new memory
- Good for small, infrequent modifications
- Thread-safe (no synchronization needed)

**StringBuilder (mutable):**
- Mutable - modifies internal buffer
- Allocates only when buffer needs to grow
- Good for many concatenations, loops with string building
- Not thread-safe (need external synchronization)

**Performance comparison:**
```csharp
// Bad - creates 1000 string objects
string result = "";
for(int i=0; i<1000; i++) result += i;

// Good - one StringBuilder allocation
var sb = new StringBuilder();
for(int i=0; i<1000; i++) sb.Append(i);
string result = sb.ToString();
```

**When to use:**
- **string**: Simple assignments, small concatenations, when value won't change
- **StringBuilder**: Loops, many concatenations, building dynamic strings

**Modern C#:** String interpolation compiles to StringBuilder for multiple interpolations. Use string.Create() for specialized scenarios.

---

## Question 12: What is the purpose of nameof operator in C#? How does it help with maintainability?

**Answer:**

**nameof returns the name of a symbol as a string:**
```csharp
nameof(Person.Name) // returns "Name"
nameof(person) // returns "person"
nameof(Console.WriteLine) // returns "WriteLine"
```

**Benefits:**
1. **Refactoring-safe** - If you rename a property, nameof updates automatically
2. **Compile-time checked** - Unlike magic strings, compiler verifies the symbol exists
3. **Error messages** - Better for logging, exceptions, validation

**Common uses:**
```csharp
throw new ArgumentNullException(nameof(param));
ValidateRequired(property, nameof(MyClass.Property));
[JsonProperty(nameof(EmailAddress))]
```

**Why it matters for maintainability:**
- Rename refactoring works across the codebase
- No runtime errors from typos in property names
- Self-documenting code - names stay in sync

**Version note:** Available since C# 6.0 (VS 2015). Now standard practice for avoiding magic strings.

---

## Question 13: How does pattern matching work in C#? Explain is, switch expressions, and property patterns.

**Answer:**

**Type pattern (is):**
```csharp
if (obj is string s && s.Length > 5) { /* use s */ }
if (obj is null) { /* null check */ }
```

**Switch expressions (C# 8+):**
```csharp
var result = shape switch {
    Circle c => c.Radius * c.Radius * Math.PI,
    Rectangle r => r.Width * r.Height,
    _ => 0
};
```

**Property patterns:**
```csharp
if (person is { Name: "John", Age: > 18 }) { }
if (order is { Status: OrderStatus.Shipped, Total: > 1000 }) { }
```

**Positional patterns:**
```csharp
if (point is (int x, int y) when x > 0 && y > 0) { }
```

**Pattern combinations:**
- And (and), Or (or), Not (not)
- Relational patterns (>, <=, etc.)

**Benefits:**
- More readable than traditional type checks and casts
- Eliminates temporary variables
- Works with records for deconstruction
- Reduces boilerplate code

**13-year insight:** Pattern matching dramatically reduces code in complex conditional logic, especially in domain modeling and validation scenarios.

---

## Question 14: What are tuples in C#? How do they differ from anonymous types and ValueTuple?

**Answer:**

**Classic Tuple (System.Tuple):**
```csharp
var tuple = Tuple.Create(1, "hello"); // tuple.Item1, tuple.Item2
```
- Reference type, immutable
- Named items via Item1, Item2 (not meaningful)
- Pre-C# 7, limited

**Anonymous Types:**
```csharp
var anon = new { Name = "John", Age = 30 }; // anon.Name, anon.Age
```
- Compiler-generated class
- Read-only properties
- Type inferred, can't be returned from methods (easily)
- Scope-limited (local usage)

**ValueTuple (C# 7+):**
```csharp
var vt = (Name: "John", Age: 30); // vt.Name, vt.Age
(string Name, int Age) person = ("John", 30);
```
- Value type, mutable
- Meaningful names for elements
- Can be returned from methods
- More efficient than Tuple

**Key differences:**

| Feature | Tuple | ValueTuple | Anonymous Type |
|---------|-------|------------|----------------|
| Type | Reference | Value | Reference |
| Names | Item1, Item2 | Custom | Custom |
| Returnable | Yes | Yes | No (easily) |
| Performance | Slow | Fast | Moderate |

**Modern recommendation:** Use ValueTuple for any multi-value return scenarios.

---

## Question 15: Explain nullable reference types in C# 8.0+. How does this improve code safety?

**Answer:**

**Nullable Reference Types (NRT):**
Enabled via <Nullable>enable</Nullable> in csproj or #nullable enable

**How it works:**
- By default, reference types are non-nullable
- Use ? to indicate nullable: string?
- Compiler warns when:
  - Assigning null to non-nullable
  - Dereferencing nullable without null check

**Benefits:**
```csharp
string name = null; // Warning: can't assign null
string? optional = null; // OK
string nonNull = optional; // Warning: may be null
optional!.Value; // Suppress warning (use carefully)

if (optional != null) { // Compiler knows it's safe
    Console.WriteLine(optional.Length); // No warning
}
```

**Improvements:**
1. **NullReferenceExceptions reduction** - Catch at compile time
2. **Intent clarity** - Explicit nullable vs non-nullable
3. **Better API design** - Libraries can express nullability

**Interoperability:**
- Use #nullable disable for legacy code
- Treat null as unknown, not as "no value" semantically
- Use null for "not yet set", consider null object pattern for "no value"

---

## Question 16: What is the difference between Equals(), ==, and ReferenceEquals()?

**Answer:**

**ReferenceEquals(object, object):**
- Static method
- Compares reference identity only (are they same object in memory?)
- Always uses reference comparison
- Useful for checking if two references point to same object

**Equals(object) virtual:**
- Virtual method, can be overridden
- Default: reference equality (same as ReferenceEquals)
- Can be overridden for value equality (string, ValueTuple)
- Check for null: obj?.Equals(other) pattern

**== operator:**
- Can be overloaded (static)
- Default for reference types: reference equality
- String overloads to value equality
- For structs: requires operator overload for custom comparison

**Best practice for 13-year experience:**
- Use == for value types and when you want value equality semantics
- Use Equals() when overriding for custom equality
- Use ReferenceEquals() for specific reference identity checks
- Implement IEquatable<T> for type-specific equality in collections

---

## Question 17: How does yield keyword work? What are its performance implications?

**Answer:**

**How yield works:**
- Transforms method into a state machine via compiler generation
- Method becomes implementation of IEnumerable<T> or IEnumerator<T>
- Each yield return becomes a state in the state machine
- Method doesn't run to completion - resumes from last yield on each MoveNext()

```csharp
IEnumerable<int> GetNumbers() {
    for (int i = 0; i < 3; i++) {
        yield return i; // Pauses here, returns value
    }
}
```

**Performance implications:**

**Benefits:**
- **Lazy evaluation** - Items produced on-demand, not all upfront
- **Memory efficient** - Don't need entire collection in memory
- **No intermediate allocations** - Streams data as needed

**Drawbacks:**
- **State machine overhead** - Compiler generates additional code
- **Cannot use in async** - Use IAsyncEnumerable with async streams instead
- **Execution flow complexity** - Debugging can be tricky
- **Thread safety** - Enumerable may be used concurrently

**Use cases:**
- Processing large datasets
- Infinite sequences
- Pipelines (filtering, transforming without allocation)

**Modern alternative:** For async scenarios, use IAsyncEnumerable<T> with async yield return (C# 8+).

---

## Question 18: What are the different kinds of delegates in C#? Explain Action, Func, Predicate, and custom delegates.

**Answer:**

**Action<T> (System.Action):**
- Returns void
- Can have 0-16 generic parameters

```csharp
Action<string> print = Console.WriteLine;
Action<int, int> add = (a, b) => Console.WriteLine(a + b);
```

**Func<T, TResult> (System.Func):**
- Returns TResult
- 0-16 input parameters, last is return type

```csharp
Func<int, int, int> add = (a, b) => a + b;
Func<string, bool> isLong = s => s.Length > 10;
```

**Predicate<T>:**
- Returns bool
- Equivalent to Func<T, bool>

```csharp
Predicate<int> isEven = n => n % 2 == 0;
```

**Custom delegates:**
```csharp
public delegate int Calculate(int a, int b);
public delegate void EventHandler(object sender, EventArgs e);
```

**Best practice:** Prefer Action/Func over custom delegates unless you need specific signature. Use EventHandler<T> for events.

---

## Question 19: Explain events and delegates in C#. How does the event model work?

**Answer:**

**Delegate:**
- Type-safe function pointer
- Holds reference to methods
- Can be invoked, combined (+=, -=), removed

**Event:**
- Special delegate with access control
- Publisher controls when event can be raised
- Subscribers can only add/remove handlers (+=, -=)
- Cannot be invoked from outside declaring class

**Event pattern:**
```csharp
public class Publisher {
    public event EventHandler<MyEventArgs>? MyEvent;
    
    protected virtual void OnMyEvent(MyEventArgs args) {
        MyEvent?.Invoke(this, args); // Thread-safe raise
    }
}
```

**EventHandler<T>:**
- Standard delegate for events
- T is custom EventArgs
- Avoid passing custom parameters - use EventArgs-derived class

**Access modifiers with events:**
- public event - any code can subscribe
- private event - only class can raise, but external subscribes via add/remove
- Use field-like events for simple cases, custom for more control

**Thread safety:** Always check for null before invoking, or use ?.Invoke() pattern.

---

## Question 20: What is the difference between early binding and late binding in C#? How does reflection enable late binding?

**Answer:**

**Early Binding:**
- Compiler resolves method calls at compile time
- Type information known at compile time
- Faster, enables IntelliSense, compile-time error checking
- Normal C# code: obj.Method() - compiler generates direct call

**Late Binding:**
- Resolution happens at runtime
- Type information not available at compile time
- More flexible but slower
- Used for dynamic loading, plugins, COM interop

**Reflection enables late binding:**
```csharp
// Get type dynamically
Type type = Type.GetType("MyNamespace.MyClass");
object instance = Activator.CreateInstance(type);

// Late-bound method call
MethodInfo method = type.GetMethod("MyMethod");
method.Invoke(instance, null);

// Access property
PropertyInfo prop = type.GetProperty("Name");
prop.SetValue(instance, "John");
```

**Use cases:**
- Plugin systems
- Dynamic loading of assemblies
- Serialization/deserialization
- ORMs (property mapping)
- Testing frameworks

**Performance:** Reflection is slow (~100x slower than direct call). Use cached delegates or expression trees for performance-critical scenarios.

**Modern alternative:** dynamic keyword uses runtime binding (DLR), simpler syntax but same performance characteristics.

---

*End of Section 2: C# Deep Dive*

---

## 🎯 Key Takeaways
- Revise important concepts quickly
- Focus on interview-ready answers
- Practice explaining in your own words

---

## 🎤 Interview Tip
> Always explain **WHY + HOW**, not just definitions.
