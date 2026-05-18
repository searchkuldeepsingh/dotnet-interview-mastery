# 🚀 Fundamentals — Interview Master Guide (Enhanced)

> 🧠 Senior-level .NET interview preparation  
> ⚡ Deep concepts + real-world scenarios + advanced Q&A  

---

## 🎤 Advanced Interview Questions (Senior Level)

### ❓ Q1: How does CLR manage memory internally?

**Answer:**

How CLR Manages Memory Internally
The Execution Layer
When you write C#:

```csharp
public class OrderService
{
    private readonly decimal _taxRate = 0.18m;

    public decimal CalculateTotal(Order order)
    {
        var subtotal = order.Items.Sum(i => i.Price * i.Quantity);
        var tax = subtotal * _taxRate;
        return subtotal + tax;
    }
}
```

C# compiler → IL (Intermediate Language) → CLR's JIT (Just-In-Time) compiler → native machine code.

The CLR has three memory regions:

**1. The Stack (Thread-Local, LIFO)**

Each thread gets its own stack (~1MB default). Used for:
- Value types (locals and method parameters)
- Method call frames (return address, saved registers, exception handling info)

```csharp
public int Add(int a, int b)
{
    int result = a + b;  // a, b, result all on stack
    return result;
}
```

Stack layout when Add(3, 5) is called:
```
┌─────────────────────┐
│   return address     │ ← pushed by CALL instruction
│   a = 3              │ ← pushed as argument
│   b = 5              │ ← pushed as argument
│   result = 8         │ ← locals
├─────────────────────┤
│   ... previous frame │
```

Stack is self-cleaning: when `Add` returns, the stack pointer moves back up. No cleanup needed. That's why value types are cheap — allocate and free in a single pointer move.

```
Before call:     SP → ┌───────┐
                      │frame N│
After call:      SP → ┌───────┐
                      │frame N│
                      │a = 3  │
                      │b = 5  │
                      │result │
After return:    SP → ┌───────┐
                      │frame N│ ← result is gone. No GC needed.
```

Stack vs Heap allocation for value types:

```csharp
public void Demo()
{
    int x = 42;            // STACK — just a pointer bump
    
    object o = x;          // HEAP — boxing allocates on heap
                           // The value 42 is copied into a heap object
}
```

**2. The Heap (Managed Heap, Shared Across Threads)**

All reference types live here: class instances, arrays, strings, delegates, boxed value types.

```csharp
public class Order
{
    public int Id { get; set; }
    public List<Item> Items { get; set; }
}

public void CreateOrder()
{
    Order order = new Order(); // 'order' ref on stack, the Order object on heap
}
```

```
Stack                          Heap
┌──────────┐                 ┌──────────────────────┐
│ order ────────→            │ Order object          │
│           │                │ [SyncBlock]           │
│           │                │ [TypeHandle] → Order  │
│           │                │ Id: 0                 │
│           │                │ Items: null           │
│           │                └──────────────────────┘
```

Three sections within the managed heap:

| Generation | Size threshold | Collection frequency | Contains |
|------------|---------------|---------------------|----------|
| Gen 0 | ~256KB - 4MB (adjusts) | Very frequent (every ~1s under load) | Short-lived objects (locals, iterators) |
| Gen 1 | ~2MB - 10MB | Moderate | Objects that survived 1 collection |
| Gen 2 | Dynamic | Rare (expensive) | Long-lived objects (static data, cache, singletons) |
| Large Object Heap (LOH) | Objects ≥ 85,000 bytes | Rare | Arrays, strings, large buffers, byte[] |

Why Generations Exist — Most objects die young. 90% of objects are never alive for a second Gen 0 collection.

```csharp
public string ProcessOrder(int orderId)
{
    var order = _db.Orders.Find(orderId);  // ← Gen 0
    var summary = $"Order #{order.Id}";     // ← Gen 0 (string)
    var items = order.Items.ToList();       // ← Gen 0
    
    SendEmail(summary, items);
    
    return summary;                         // ← summary survives → Gen 1
}
```

Gen 0 collects tiny objects rapidly without touching Gen 1 or Gen 2. That's the performance secret.

**3. The Garbage Collector (GC)**

*When Does GC Trigger?*
1. Gen 0 fills up (most common)
2. `GC.Collect()` called explicitly
3. System is low on memory
4. AppDomain unloads

*What GC Actually Does (3 Phases)*

**Phase 1: Mark** — The GC starts from roots: static fields, thread stack locals, CPU registers, GC handles.

```csharp
public class OrderProcessor
{
    private static List<Order> _activeOrders = new(); // ROOT (static field)
    
    public void Process(Order order)
    {
        var calculator = new TaxCalculator();          // ROOT (stack local)
        var result = calculator.Calculate(order);      
        _activeOrders.Add(order);                      // order is reachable
    } // calculator dies here
}
```

GC walks the object graph from all roots:
- `_activeOrders` → `List<Order>` → each `Order` → each `Order.Items` → each `Item`
- Mark every reachable object. Anything NOT marked = garbage

```
Before Mark (heap state):
┌─────────────────────────────┐
│ [Order A] ← reachable        │ → Mark: ✓
│ [Item 1]  ← reachable        │ → Mark: ✓
│ [TaxCalc] ← NOT reachable    │ → Mark: ✗ (GARBAGE)
│ [CachedOrder] ← NOT reachable│ → Mark: ✗ (GARBAGE)
└─────────────────────────────┘
```

**Phase 2: Sweep / Compact**
- **Sweep-only (Gen 2, LOH)**: Build free list of dead gaps. Fast but fragments.
- **Sweep + Compact (Gen 0, Gen 1)**: Slide live objects together. Updates all references.

```
After Compact:
┌─────────────────────────────┐
│ Order A │ Item 1 │ Item 2   │
│ String  │                  │
└─────────────────────────────┘
      ↑ Compacted end — next allocation starts here
```

This is why `new object()` is just `ptr += sizeOfObject`. No free-list search like malloc.

**Phase 3: Bump the generation** — Survivors promote: Gen 0 → Gen 1, Gen 1 → Gen 2.

*GC Modes*

| Mode | Description | Best for |
|------|-------------|----------|
| Workstation GC | Per-process, one GC thread | Desktop apps |
| Server GC | Per-core heap + GC thread | ASP.NET Core |
| Background GC | Concurrent collections | Low-latency APIs |

ASP.NET Core defaults to **Server + Background GC**.

*Object Header (Every Heap Object)*

Every object on the heap has 8-16 bytes of overhead:
```
┌──────────────────────────────┐
│ SyncBlock Index (4 bytes)    │ ← Thread sync, hash code cache
│ TypeHandle Pointer (4/8 bytes)│ ← Points to method table (vtable)
│ Instance fields ...          │
└──────────────────────────────┘
```

```csharp
public class Point { public int X; public int Y; }
// Memory: [SyncBlock: 0] [TypeHandle → Point] [X: 1] [Y: 2]
// Total: 4 + 8 + 4 + 4 = 20 bytes (aligned to 24)
```

SyncBlock is lazy — only allocated when you `lock(obj)` or call `GetHashCode()`.

*Real-World Memory Leaks*

1. **Static references** — objects never die
```csharp
public static class Cache
{
    private static List<byte[]> _data = new();
    public static void Add(byte[] data) => _data.Add(data); // permanent leak
}
```

2. **Event handlers** — subscriber keeps publisher alive
```csharp
public class Subscriber
{
    public void Subscribe(Publisher p)
    {
        p.SomethingHappened += OnSomethingHappened;
        // 'this' is now referenced by Publisher → won't be collected
    }
}
```

3. **Captured variables in lambdas**
```csharp
public class Service
{
    private byte[] _largeBuffer = new byte[1_000_000];
    public Action CreateAction() => () => Console.WriteLine(_largeBuffer[0]);
    // closure captures 'this', keeping _largeBuffer alive
}
```

4. **Improper finalizers**
```csharp
public class ResourceHolder
{
    ~ResourceHolder() // FINALIZER — object survives an extra collection
    {
        // cleanup
    }
}
```
A finalizable object: alloc→Finalization Queue→GC moves to Freachable Queue→finalizer thread runs→THEN freed. Always prefer `IDisposable`:

```csharp
public class ResourceHolder : IDisposable
{
    private IntPtr _nativeResource;
    public void Dispose()
    {
        ReleaseNativeResource();
        GC.SuppressFinalize(this);
    }
}
```

*Large Object Heap (LOH)*

Objects ≥ 85,000 bytes go to the LOH:
```csharp
byte[] buffer = new byte[100_000]; // 100KB → LOH
string huge = new string('x', 50_000); // 100KB → LOH
```

Problems: LOH is never compacted → fragmentation → OOM despite free space.
Solution:
```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(90_000);
// use it
ArrayPool<byte>.Shared.Return(buffer); // reused
```

*How GC Decides When to Collect*

Gen 0 budget starts at ~256KB. If allocation rate > survival rate → budget grows. If survival > allocation → budget shrinks. Monitor with:
```bash
dotnet counters monitor --process-id 1234
```

| Counter | What it shows |
|---------|---------------|
| gen-0-gc-count | Frequent = healthy |
| gen-1-gc-count | Moderate |
| gen-2-gc-count | Rare — red flag if frequent |
| time-in-gc | Should be < 5-10% |

*Summary*

```
Stack (1MB per thread)
├── Value type locals
├── Method call frames
└── Self-cleaning (pointer bump)

Heap (Shared, managed by GC)
├── Gen 0 (256KB - 4MB) → collected every ~1s
├── Gen 1 (~2-10MB) → moderate frequency
├── Gen 2 (Dynamic) → rare, expensive
├── LOH (≥85KB) → rarely collected, never compacted
│
└── Allocation: new T() → ptr += sizeof(T) — O(1)

GC Cycle
├── MARK: Trace from roots, flag reachable
├── SWEEP/COMPACT: Free or slide
└── PROMOTE: Survivors to next generation

Performance traps
├── Static collections → permanent leak
├── Event handlers → subscribers stay alive
├── Frequent Gen 2 collections → performance crater
└── LOH fragmentation → OOM despite free space
```

---

### ❓ Q2: What happens when you create an object in .NET?

**Answer:**

The full lifecycle of `new Order()`:

```csharp
var order = new Order { Id = 1, Total = 100m };
```

**Step 1: Memory allocation on the managed heap**

The CLR calculates the total size: object header (8 bytes on x64, 12 on x32) + TypeHandle pointer (4/8 bytes) + instance fields.

```
Order instance memory layout:
[SyncBlock: 0 (4 bytes)] [TypeHandle (8 bytes)] [Id: 0 (4 bytes)] [Total: 0 (16 bytes)]
Total: 4 + 8 + 4 + 16 = 32 bytes (aligned)
```

The allocation is a simple pointer bump: `NextObjPtr += sizeOfObject`. On Server GC with background GC, each core has its own allocation arena — no lock contention.

```csharp
// Internally, the JIT emits something like:
// mov  ecx, [Order_TypeHandle]   ; type to allocate
// call CORINFO_HELP_NEWFAST      ; bump pointer, zero memory
```

**Step 2: Zero-initialize memory**

The allocated memory is zeroed out. All fields get their default values:
- `Id = 0`, `Total = 0.0m` (decimal), reference types = `null`

**Step 3: Store reference on stack**

The local variable `order` on the stack gets the heap address:

```
Stack:                     Heap:
┌─────────────────┐       ┌───────────────────────┐
│ order: 0x123456  │──→   │ [SyncBlk] [TypeHandle] │
└─────────────────┘       │ Id = 0                 │
                          │ Total = 0.0            │
                          └───────────────────────┘
```

**Step 4: Constructor executes**

The JIT compiles the constructor IL. The `this` pointer is set to the allocated address. Field initializers run first, then constructor body:

```csharp
// Compiler-generated constructor:
public Order()
{
    // Field initializers first
    // (none here, but if you had: public int Id = 5; it sets here)
    
    // Constructor body
    // ...
}
```

**Step 5: Object header initialization**

The TypeHandle pointer is set so the runtime knows this is an `Order`. The SyncBlock index is set to 0 (no lock taken yet).

**Step 6: GC begins tracking**

The object is now on the heap, in Gen 0. The GC knows about it. Every subsequent collection considers it for promotion.

*What the JIT actually emits (simplified)*

```asm
; new Order() JIT output (x64)
00007FFF  mov  rcx, 7FF9A3B4C8A0h       ; Order TypeHandle
00007FFF  call CORINFO_HELP_NEWFAST       ; allocate on Gen 0 heap
00007FFF  mov  qword ptr [rbp+28h], rax   ; save reference
00007FFF  mov  rcx, rax                    ; this = new object
00007FFF  call Order..ctor()               ; constructor
```

*Key insight: Allocation is cheap*

A `new object()` in .NET takes ~10-20 nanoseconds — comparable to a few CPU instructions. This is orders of magnitude faster than malloc() in C because:
1. **Bump pointer allocation** — no free-list traversal
2. **Thread-local arenas** — no lock contention (Server GC)
3. **Zero-initialization** — prefetches into CPU cache

---

### ❓ Q3: Boxing/Unboxing impact?

**Answer:**

Boxing wraps a value type inside a reference type object on the heap. Unboxing extracts it back.

```csharp
int number = 42;           // value type → stack
object boxed = number;     // BOXING → copies 42 from stack to heap
int unboxed = (int)boxed;  // UNBOXING → copies 42 from heap back to stack
```

*What happens in memory:*

```
Before boxing:
Stack:  number = 42

After boxing (object boxed = number):
Stack:          Heap:
number = 42     ┌───────────────────┐
boxed ───────→  │ SyncBlock: 0      │
                │ TypeHandle → Int32│
                │ value: 42         │
                └───────────────────┘
                ← 12 bytes overhead + 4 bytes data = 16 bytes
```

*The hidden cost:*

```csharp
var list = new ArrayList(); // ❌ BAD — pre-generics
list.Add(42);               // boxing: 42 → heap object
list.Add(99);               // boxing: 99 → heap object
int x = (int)list[0];       // unboxing + cast check
```

Each boxing allocates: 12 bytes overhead + sizeof(value). When the ArrayList is collected, each boxed value creates Gen 0 garbage that must be swept. Under high throughput, this causes:

1. **GC pressure** — more Gen 0 collections → more CPU in GC
2. **Cache pollution** — heap objects scattered, poor locality
3. **Extra indirection** — must dereference pointer to read value

*With generics (the fix):*

```csharp
var list = new List<int>(); // ✅ GOOD — no boxing
list.Add(42);               // stored directly in internal array
list.Add(99);               // no heap allocation
int x = list[0];            // direct memory read
```

```
List<int> internal array:
┌────┬────┬────┬────┬────┐
│ 42 │ 99 │  0 │  0 │  0 │
└────┴────┴────┴────┴────┘
← Values stored inline, no heap objects per element
```

*Real-world impact:*

| Operation | Time (relative) | GC allocation |
|-----------|----------------|---------------|
| `int a = b;` | 1x | 0 bytes |
| `object o = b;` (box) | ~15-20x | 16 bytes |
| `int c = (int)o;` (unbox) | ~5-10x | 0 bytes (but type check) |
| `List<int>.Add(b)` | 1x | 0 bytes (array resize if needed) |
| `ArrayList.Add(b)` (box) | ~15-20x | 16 bytes |

*Hidden boxing scenarios:*

```csharp
// 1. Enum comparisons with non-generic methods
enum Status { Active, Inactive }
Console.WriteLine(Status.Active); // boxes! (WriteLine(object))

// 2. Value types as interfaces
struct Point : IComparable { }
Point p;
IComparable c = p; // boxes!

// 3. String concatenation with structs
int id = 5;
string msg = "User: " + id; // boxes! (calls string.Concat(object))

// 4. Non-generic collections (legacy)
Hashtable table = new Hashtable(); // keys and values always boxed

// 5. Nullable<T> to object
int? maybe = 5;
object o = maybe; // boxes
```

*Prevention strategies:*

```csharp
// ✅ Use generics
List<int> vs ArrayList
Dictionary<string, int> vs Hashtable

// ✅ Use generic interfaces
struct Point : IComparable<Point> { }

// ✅ Override ToString() on structs
struct ProductId
{
    public int Value { get; }
    public override string ToString() => $"Product-{Value}";
}

// ✅ String interpolation avoids boxing on value with ToString()
string msg = $"User: {id}"; // calls id.ToString() directly, no boxing
```

---

### ❓ Q4: JIT optimization?

**Answer:**

The Just-In-Time compiler converts IL (Intermediate Language) into native machine code at runtime. Unlike ahead-of-time (AOT) compilation, JIT sees the actual execution context and applies runtime optimizations that static compilers cannot.

*The compilation pipeline:*

```
C# Source → Roslyn compiler → IL (.dll/.exe) → JIT → Native code
                            ↑                          ↑
                      Compile time                Runtime (per method)
```

*When JIT fires:*

Each method is JIT-compiled **lazily** — the first time it's called:

```csharp
static void Main()
{
    Console.WriteLine("Before call");    // JIT hasn't compiled CalculateTotal yet
    var result = CalculateTotal(100, 5); // NOW JIT compiles this method
    Console.WriteLine("After call");
}
```

*What JIT optimizes:*

**1. Method inlining**
```csharp
// Source
public int Add(int a, int b) => a + b;
public int Calculate() => Add(5, 3);

// Inlined — no call instruction, no stack frame
public int Calculate() => 5 + 3;
```

The JIT inlines methods that are small and called frequently. This eliminates call overhead and enables further optimizations across the boundary. Threshold: methods with ~32 IL bytes or less. You can force with `[MethodImpl(MethodImplOptions.AggressiveInlining)]`.

**2. Dead code elimination**
```csharp
public int Calculate(int x)
{
    if (false) // JIT sees this is constant false → removes branch entirely
        return 42;
    return x * 2;
}

// JIT output:
// return x * 2;
```

**3. Constant folding and propagation**
```csharp
const double TaxRate = 0.18;
var tax = price * TaxRate;

// JIT can compute: tax = price * 0.18
// No variable lookup needed at runtime
```

**4. Loop unrolling**
```csharp
for (int i = 0; i < 4; i++)
    sum += items[i];

// JIT may unroll to:
// sum += items[0];
// sum += items[1];
// sum += items[2];
// sum += items[3];
// No loop overhead (increment, compare, branch)
```

**5. Bounds-check elimination**
```csharp
int[] arr = new int[10];
for (int i = 0; i < arr.Length; i++)
    arr[i] = i * 2;

// JIT knows: arr.Length == 10, i < 10 always
// It ELIMINATES the bounds check on arr[i]
// Without this, each access checks: if (i >= arr.Length) throw IndexOutOfRangeException
```

But for this pattern:
```csharp
for (int i = 0; i <= arr.Length; i++) // note: <= not <
    arr[i] = i; // JIT CANNOT eliminate bounds check
```

**6. Devirtualization**
```csharp
sealed class Repository : IRepository { public void Save() { ... } }
IRepository repo = new Repository();
repo.Save();

// JIT sees the concrete type is sealed → calls Save() directly
// No virtual call overhead (vtable lookup)
```

**7. Guarded devirtualization (PGO)**

With Tiered PGO (Profile Guided Optimization), the JIT monitors actual types at runtime:

```csharp
abstract class Animal { public abstract void Speak(); }
class Dog : Animal { public override void Speak() => Console.WriteLine("Woof"); }
class Cat : Animal { public override void Speak() => Console.WriteLine("Meow"); }

Animal pet = new Dog();
pet.Speak();

// Without PGO: virtual call → look up Dog.Speak in vtable → call
// With PGO after warmup: checks if type is Dog → calls directly → fast path
// Only falls back to virtual call if type is unexpected
```

*Tiered Compilation (default in .NET Core 3.0+)*

```
Method first call → Tier 0 (quick JIT, minimal optimization)
                         ↓
                  Collects data (which methods are hot)
                         ↓
Hot method detected → Tier 1 (re-JIT with full optimization + PGO)
```

```csharp
// Tier 0: JITs in milliseconds, runs slow but starts fast
public int Add(int a, int b) => a + b; // no inlining

// Tier 1 (after ~30 calls): re-JIT with full optimization  
public int Add(int a, int b) => a + b; // inlined into caller
```

This is why .NET apps warm up — startup is fast (Tier 0), then steady-state is fast (Tier 1).

*How to see what JIT does:*

```bash
# Enable JIT dump
dotnet run --config=Release
set DOTNET_JitDisasm=CalculateTotal
# Shows the actual assembly generated
```

*JIT vs AOT (Native AOT in .NET 8+):*

| Feature | JIT | Native AOT |
|---------|-----|-----------|
| Startup | Slower (compiles at runtime) | Fastest (already native) |
| Peak performance | Higher (adaptive optimizations) | Lower (can't PGO based on actual runtime data) |
| App size | Smaller (carries IL + runtime) | Larger (strips unused code) |
| Platform-specific | Any (IL is portable) | Build per target |
| Reflection | Full support | Limited (trimming removes metadata) |

---

### ❓ Q5: Thread vs Task vs Async?

**Answer:**

| Concept | Abstraction level | What it represents | Cost |
|---------|------------------|--------------------|------|
| **Thread** | OS-level | Kernel object + 1MB stack + context | Very expensive (~1MB + 1ms context switch) |
| **Task** | .NET-level | Promise of work that may complete in future | Cheap (~100 bytes, no OS thread) |
| **Async/await** | Language-level | State machine that yields thread during I/O | Compiler transform (zero runtime overhead per se) |

**Thread**

```csharp
// Low-level — you almost never need this
var thread = new Thread(() =>
{
    Thread.Sleep(1000); // BLOCKS this thread for 1 second
    Console.WriteLine("Work done");
});
thread.Start();
thread.Join(); // blocks calling thread until this finishes
```

Costs:
- ~1MB virtual memory for stack
- ~70-100KB committed immediately
- ~1ms kernel-mode context switch
- Thread pool limits to prevent oversubscription

**Task**

```csharp
// High-level — represents work, may or may not run on its own thread
Task task = Task.Run(() =>
{
    Thread.Sleep(1000);  // CPU-bound: use dedicated thread
    return 42;
});

Task<string> ioTask = ReadFileAsync(); // I/O-bound: no thread during wait
```

Tasks can represent:
- **CPU-bound work** (backed by a thread pool thread via `Task.Run`)
- **I/O-bound work** (no thread during operation — kernel handles I/O)
- **Delegated work** (continuations that run when something completes)

**Async/await**

```csharp
// Language sugar over Task — yields thread during I/O waits
public async Task<User> GetUserAsync(int id)
{
    // Thread A enters here, starts DB query
    var user = await _db.Users.FindAsync(id);
    // DURING the 50ms DB wait: Thread A is back in pool serving OTHER requests
    // When DB responds, any available thread resumes here
    return user;
}
```

*The Problem (Thread Starvation)*

```csharp
// IN SYNC: 100 concurrent requests = 100 blocked threads
public IActionResult GetUser(int id)
{
    var user = _db.Users.Find(id); // Thread blocked for 50ms
    return Ok(user);
}
// Thread pool default: ~32767 threads max
// But each thread costs 1MB stack → 100MB for 100 requests
// Under 1000 req/s with 50ms DB: need 50 threads → 50MB of stacks
// At 10000 req/s: need 500 threads → 500MB → thread pool exhaustion → 503s
```

*The Solution (Non-blocking I/O)*

```csharp
// IN ASYNC: 100 concurrent requests = 2-3 threads doing real work
public async Task<IActionResult> GetUser(int id)
{
    var user = await _db.Users.FindAsync(id); // thread released during I/O
    return Ok(user);
}
// Under 10000 req/s with 50ms DB: still only ~10 threads needed
// Because threads are released back during I/O waits
```

*How async/await actually works (compiler transform)*

```csharp
public async Task<User> GetUserAsync(int id)
{
    var user = await _dbContext.Users.FindAsync(id);
    var orders = await _orderRepo.GetOrdersAsync(user.Id);
    return new UserWithOrders(user, orders);
}
```

The compiler generates a state machine struct:

```csharp
public Task<User> GetUserAsync(int id)
{
    var sm = new GetUserAsyncStateMachine();
    sm.id = id;
    sm.builder = AsyncTaskMethodBuilder<User>.Create();
    sm.state = -1;
    sm.builder.Start(ref sm);
    return sm.builder.Task;
}

struct GetUserAsyncStateMachine : IAsyncStateMachine
{
    public int state;
    public int id;
    public User user;
    public AsyncTaskMethodBuilder<User> builder;
    private TaskAwaiter<User> awaiter1;

    public void MoveNext()
    {
        switch (state)
        {
            case -1:
                awaiter1 = _db.Users.FindAsync(id).GetAwaiter();
                if (!awaiter1.IsCompleted)
                {
                    state = 0;
                    builder.AwaitUnsafeOnCompleted(ref awaiter1, ref this);
                    return; // THREAD RETURNS TO POOL HERE
                }
                goto case 0;
            case 0:
                user = awaiter1.GetResult();
                // ... continue to next await
                break;
        }
    }
}
```

Key: each `await` is a `return;` point — the thread goes back to the pool, not blocked.

*Task, Task\<T\>, ValueTask\<T\>*

```csharp
public async Task DoAsync() { }      // void-returning async, always allocates Task (~4KB)
public async Task<int> GetAsync() { } // value-returning async, allocates Task<int>

// ValueTask<T> — avoids allocation when result is often synchronous
private Dictionary<int, User> _cache = new();
public ValueTask<User> FindCachedAsync(int id)
{
    if (_cache.TryGetValue(id, out var user))
        return new ValueTask<User>(user); // sync path — no allocation
    return new ValueTask<User>(LoadFromDbAsync(id)); // async path
}
```

*When to use what:*

| Scenario | Use | Why |
|----------|-----|-----|
| I/O-bound (DB, HTTP, file) | `async Task` | Frees thread during I/O |
| CPU-bound (calculation, image processing) | `Task.Run` | Moves work to thread pool |
| Short CPU work on background | `await Task.Run(...)` | Frees UI/request thread |
| Result may be cached/sync | `ValueTask<T>` | Avoids allocation |
| Fire-and-forget (logging, metrics) | `Task` (store reference) | Never `async void` |
| Event handler | `async void` | Only exception (WinForms/WPF) |

*Async best practices:*
```csharp
// ✅ Async all the way — every I/O method is async
[HttpGet("{id}")]
public async Task<ActionResult<User>> Get(int id)
{
    var user = await _userRepo.GetByIdAsync(id);
    return Ok(user);
}

// ❌ NEVER block on async — causes deadlock in legacy frameworks
var user = GetUserAsync(1).Result; // DEADLOCK (in ASP.NET Framework)
var user = GetUserAsync(1).Wait(); // DEADLOCK

// ✅ Concurrent I/O where independent
var taskA = svc.GetDataAAsync();
var taskB = svc.GetDataBAsync();
var taskC = svc.GetDataCAsync();
await Task.WhenAll(taskA, taskB, taskC);

// Sequential where dependent
var user = await GetUserAsync(id);
var orders = await GetOrdersAsync(user.Id); // needs user first

// ❌ Async void = crash on exception
public async void Button_Click() // WRONG — exception crashes process
{
    await DoWorkAsync();
}

// ❌ Thread.Sleep inside async
await Task.Delay(1000); // ✅ — releases thread
Thread.Sleep(1000);    // ❌ — blocks thread
```

*Summary*

```
Thread (OS, 1MB stack)
  └─ Task.Run() → CPU-bound work on thread pool
       └─ async/await → I/O-bound, releases thread during wait
            └─ continues on any pool thread when I/O completes

Memory: Thread >> Task > async state machine
Scalability: async > Task.Run > Thread
```

---

### ❓ Q6: Memory leaks in .NET?

**Answer:**

"Memory leak" in .NET means an object is **reachable** (not garbage) but no longer needed. The GC can't collect it because there's a live reference chain. Over time, these accumulate until OOM.

*1. Static references holding objects forever*

```csharp
public static class GlobalCache
{
    public static List<Order> Orders = new(); // ROOT — never collected
}

// Somewhere else:
GlobalCache.Orders.Add(new Order()); // Order object lives forever
// Even if the Order is no longer needed, GlobalCache holds a reference
```

Why it's dangerous: static fields are GC roots. Any object reachable from a root cannot be collected. A `static List<T>` grows unbounded → heap grows → Gen 2 collections increase → app slows → OOM.

*2. Event handler leaks (subscriber keeps publisher alive)*

```csharp
public class Button
{
    public event EventHandler Click;
}

public class Window
{
    public void Attach(Button btn)
    {
        btn.Click += OnClick; // 'this' (Window) is registered in Button's event delegate
    }                         // Window CANNOT be collected until Button dies
}
```

The delegate `OnClick` captures `this`. The Button holds a reference to the delegate. So Button → delegate → Window. Window stays alive as long as Button is alive, even if Window is closed.

Fix:
```csharp
public void Detach(Button btn)
{
    btn.Click -= OnClick; // ALWAYS unsubscribe
}
```

Or use weak event pattern:
```csharp
public class WeakEventListener
{
    private WeakReference<Window> _windowRef;
    public void OnClick(object sender, EventArgs e)
    {
        if (_windowRef.TryGetTarget(out var window))
            window.HandleClick();
    }
}
```

*3. Captured variables in closures*

```csharp
public class OrderService
{
    private readonly byte[] _template = File.ReadAllBytes("large_template.pdf"); // 10MB
    
    public Func<Order, byte[]> CreateGenerator()
    {
        return order =>
        {
            // This lambda captures 'this' (to access _template)
            // The returned delegate keeps the entire OrderService alive
            return MergeTemplate(order, _template);
        };
    }
}

// Usage:
var generator = service.CreateGenerator(); // service CAN'T be GC'd while generator is alive
```

*4. Incorrect caching without eviction*

```csharp
public static class Cache
{
    private static Dictionary<string, byte[]> _cache = new();
    
    public static void Add(string key, byte[] data) => _cache[key] = data;
    // No eviction policy — grows forever
}
```

Fix:
```csharp
// Use IMemoryCache with expiration and size limits
services.AddMemoryCache(options =>
{
    options.SizeLimit = 100 * 1024 * 1024; // 100MB max
});
```

*5. Thread Local Storage (TLS) / AsyncLocal*

```csharp
[ThreadStatic]
private static byte[] _buffer = new byte[1024]; // kept alive per thread

static async Task ProcessAsync()
{
    var locals = new AsyncLocal<byte[]>();
    locals.Value = new byte[10_000_000]; // flows across await boundaries
    await Task.Yield();
    // locals still referenced even after method returns
}
```

*6. P/Invoke or COM interop without release*

```csharp
public class NativeResource
{
    private IntPtr _handle = NativeMethods.OpenResource();
    // If Dispose() is never called, the native resource leaks
}

// Normally:
var resource = new NativeResource();
// resource.Dispose() never called → native handle leaks forever
```

*Detecting memory leaks:*

```bash
# Use dotnet-counters for real-time heap size
dotnet counters monitor -p 1234

# Use dotnet-dump for analysis
dotnet dump collect -p 1234
dotnet dump analyze core_20260516.dmp
> dumpheap -stat    # see largest types by count
> gcroot <address>  # find what's holding an object alive
```

*Prevention checklist:*

| Pattern | Prevention |
|---------|-----------|
| Static collections | Use bounded caches with eviction |
| Events | Always unsubscribe; use weak events for long-lived publishers |
| Closures | Avoid capturing `this` if delegate outlives the object |
| Caching | Use `IMemoryCache` with expiration |
| ThreadStatic | Clear after use, avoid for large data |
| Native resources | `IDisposable` + `SafeHandle` |
| Large object allocations | Pool with `ArrayPool<T>` |

---

### ❓ Q7: Large Object Heap (LOH)?

**Answer:**

**What is LOH?**

Any object ≥ 85,000 bytes goes to the Large Object Heap:

```csharp
byte[] buffer = new byte[85_000]; // exactly at threshold → LOH
byte[] small = new byte[84_999];  // Gen 0
string huge = new string('x', 42_500); // 85,000 bytes (2 bytes/char + overhead) → LOH
int[,] bigArray = new int[100, 100];   // 40,000 bytes → Gen 0 (under 85K)
```

**Why a separate heap?**

Large objects are expensive to move during compaction. Moving a 100MB array means copying 100MB of memory. For Gen 0/1 compaction, 1MB objects are cheap. For LOH, the cost of compaction is linear with size.

**Problems with LOH**

*1. Fragmentation*

LOH is never compacted by default (too expensive):

```csharp
var buffers = new List<byte[]>();
for (int i = 0; i < 1000; i++)
{
    buffers.Add(new byte[100_000]); // LOH
    if (i % 5 == 0) buffers.RemoveAt(0); // creates gaps
}
```

After this loop:
```
LOH state:
┌─────────┬──────┬─────────┬──────┬─────────┐
│ 100K    │ FREE │ 100K    │ FREE │ 100K     │
└─────────┴──────┴─────────┴──────┴─────────┘
   ↑ used    ↑ 100K gap   ↑ used
```

If a new 150KB allocation arrives, it can't fit in any gap → extends LOH. Over time, LOH grows despite having free space.

*2. OOM despite available memory*

```csharp
// After fragmentation, you might have 500MB free in LOH gaps
// But no single contiguous 100MB block
byte[] big = new byte[100_000_000]; // OOM — fragmentation!
```

*3. Gen 2 collection required*

LOH is collected only during Gen 2 collections. Since Gen 2 collections are expensive (stop-the-world, entire heap scan), LOH garbage accumulates longer.

**How to monitor LOH**

```bash
dotnet counters monitor -p 1234
# Look for "gen-2-gc-count" and "loh-size"

# Or via code:
var gcInfo = GC.GetGCMemoryInfo();
Console.WriteLine($"LOH size: {gcInfo.HeapSizeBytes / 1024 / 1024} MB");
Console.WriteLine($"Fragmented: {gcInfo.FragmentedBytes / 1024 / 1024} MB");
```

**Solutions**

*1. Use ArrayPool\<T\>*

```csharp
// Rent from a shared pool — reuse instead of allocate
byte[] buffer = ArrayPool<byte>.Shared.Rent(100_000);
try
{
    await stream.ReadAsync(buffer, 0, 100_000);
    // process...
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer, clearArray: false);
    // buffer goes back to pool, no LOH allocation next time
}
```

*2. Break large objects into smaller chunks*

```csharp
// Instead of one 100MB array:
byte[] huge = new byte[100_000_000]; // LOH

// Use multiple smaller arrays:
var chunks = new byte[100][];
for (int i = 0; i < 100; i++)
    chunks[i] = new byte[1_000_000]; // Still LOH (≥85K) but easier to manage
```

However, each 1MB chunk is still LOH. Better:

```csharp
var chunks = new byte[100][];
for (int i = 0; i < 100; i++)
    chunks[i] = ArrayPool<byte>.Shared.Rent(1_000_000); // rented, not allocated
```

*3. Force LOH compaction (if needed)*

```csharp
// In .NET Core/.NET 5+, you can enable LOH compaction:
GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
GC.Collect(); // ⚠️ Expensive call — use sparingly
```

*4. Use `string.Create` for large strings*

```csharp
// Building large strings via StringBuilder creates many LOH strings
var sb = new StringBuilder(200_000);
for (int i = 0; i < 10_000; i++) sb.Append($"Line {i}\n");
string result = sb.ToString(); // 200KB LOH string

// Better: use StringBuilder with pooled arrays internally
// Or streaming output instead of holding the full string
```

*5. Configure GC for low-latency*

```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
</PropertyGroup>
```

**When LOH is OK:**

- Short-lived large arrays that are immediately freed
- Application startup pre-allocation
- Known-size collections that don't grow
- High-memory environments where fragmentation risk is low (e.g., batch processing)

---

### ❓ Q8: IEnumerable vs ICollection vs IList vs IQueryable?

**Answer:**

| Interface | Purpose | Mutation | Count | Index | Deferred | DB Query |
|-----------|---------|----------|-------|-------|----------|----------|
| `IEnumerable<T>` | Forward-only iteration | No | No (needs `.Count()`) | No | Yes | Loads ALL rows first |
| `ICollection<T>` | Mutable collection with count | Add/Remove | Yes (`.Count`) | No | No | N/A (in-memory) |
| `IList<T>` | Indexable collection | Add/Remove/Insert | Yes | Yes (`.Item[i]`) | No | N/A |
| `IQueryable<T>` | Expression tree for queries | No | No | No | Yes | Translates to SQL |

**`IEnumerable<T>`**

```csharp
IEnumerable<User> users = GetUsers(); // nothing executed yet

var filtered = users.Where(u => u.IsActive); // still lazy
foreach (var u in filtered) // NOW executes: loads ALL users, filters in memory
    Console.WriteLine(u.Name);

// Problem with EF Core:
IEnumerable<User> users = _db.Users; // implicit cast to IEnumerable
var active = users.Where(u => u.IsActive).ToList();
// SQL EXECUTED: SELECT * FROM Users  ← ALL rows loaded to memory
// Then C# filters in memory — even though only 10 rows needed
```

**`ICollection<T>`**

```csharp
ICollection<string> items = new List<string> { "A", "B" };
items.Add("C");         // ✅
items.Remove("A");      // ✅
int count = items.Count; // ✅ O(1) for List
bool has = items.Contains("B"); // ✅ O(n) for List
```

Use when: You need a mutable collection with Count, for in-memory data.

**`IList<T>`**

```csharp
IList<string> items = new List<string> { "A", "B", "C" };
items.Insert(0, "Z");        // ✅ "Z", "A", "B", "C"
items.RemoveAt(2);           // ✅ "Z", "A", "C"
string first = items[0];     // ✅ index access, O(1)
items[1] = "X";              // ✅ replace by index
```

Use when: You need random access by index in addition to add/remove.

**`IQueryable<T>`**

```csharp
IQueryable<User> query = _db.Users; // nothing executed

// These BUILD an expression tree but don't execute
query = query.Where(u => u.IsActive);
query = query.Where(u => u.CreatedAt > cutoff);
query = query.OrderBy(u => u.Name);
query = query.Take(10);

// Only ToList() / ToListAsync() executes:
var result = await query.ToListAsync();
// SQL: SELECT TOP 10 * FROM Users 
//      WHERE IsActive = 1 AND CreatedAt > @cutoff
//      ORDER BY Name
// SQL sends ONLY 10 rows over the network
```

Key: `IQueryable` builds an expression tree. The LINQ provider (EF Core) translates it to SQL. Filtering happens on the **database server**.

**Performance comparison with EF Core:**

```csharp
// BAD: IEnumerable loads everything
IEnumerable<Order> ie = _context.Orders;
var filtered = ie.Where(o => o.Total > 100 && o.Date > cutoff);
var result = filtered.OrderBy(o => o.Date).Take(20).ToList();
// SQL: SELECT * FROM Orders  → 1M rows over network
// Then C# filters in memory → keeps only 20

// GOOD: IQueryable filters on server
IQueryable<Order> iq = _context.Orders;
var filtered = iq.Where(o => o.Total > 100 && o.Date > cutoff);
var result = filtered.OrderBy(o => o.Date).Take(20).ToListAsync();
// SQL: SELECT TOP 20 * FROM Orders WHERE Total > 100 AND Date > @cutoff ORDER BY Date
// → 20 rows over network
```

**Real-world mistake:**

```csharp
// ❌ Repository returning IEnumerable — callers can't filter at DB
public class OrderRepository
{
    public IEnumerable<Order> GetAll()
    {
        return _context.Orders; // Implicit → IEnumerable. Caller gets all rows.
    }
}

// Usage — seems efficient but loads everything:
var orders = repo.GetAll().Where(o => o.Total > 100).Take(10).ToList();
// SQL: SELECT * FROM Orders (loads ALL)

// ✅ Return IQueryable so callers compose queries
public IQueryable<Order> GetAll()
{
    return _context.Orders; // IQueryable
}

var orders = repo.GetAll().Where(o => o.Total > 100).Take(10).ToList();
// SQL: SELECT TOP 10 * FROM Orders WHERE Total > 100
```

**Conversion chain:**

```
DB → IQueryable<T> (expression tree)
               ↓ .ToList()
         IEnumerable<T> (in-memory)
               ↓ .ToList()
         List<T> (implements ICollection, IList)
```

---

### ❓ Q9: Assembly Load Context?

**Answer:**

Assembly Load Contexts (ALC) replaced AppDomains in .NET Core for loading and isolating assemblies. Each ALC is a logical scope for resolving assembly dependencies independently.

**Why ALCs exist (the problem they solve):**

```csharp
// Two different plugins want different versions of the same DLL
PluginA requires Newtonsoft.Json 12.0 ✓
PluginB requires Newtonsoft.Json 13.0 ✓
Without ALCs: one version wins → the other breaks
```

**Default load contexts:**

```
Default ALC (Main app)
  ├── app.dll (your code)
  ├── Newtonsoft.Json 13.0.0
  └── System.Text.Json 8.0.0

Collectible ALC (Plugin isolation)
  ├── PluginA.dll
  ├── Newtonsoft.Json 12.0.0 ← different version!
  └── SomeLib-PLUGIN-specific.dll

Another Collectible ALC
  ├── PluginB.dll
  └── Newtonsoft.Json 13.0.0 ← same as default (shared)
```

**Creating a custom ALC (plugin system):**

```csharp
public class PluginLoadContext : AssemblyLoadContext
{
    private readonly AssemblyDependencyResolver _resolver;

    public PluginLoadContext(string pluginPath) : base(isCollectible: true)
    {
        _resolver = new AssemblyDependencyResolver(pluginPath);
    }

    protected override Assembly Load(AssemblyName assemblyName)
    {
        // Try to resolve from the plugin's folder first
        string assemblyPath = _resolver.ResolveAssemblyToPath(assemblyName);
        if (assemblyPath != null)
            return LoadFromAssemblyPath(assemblyPath);

        // Fall back to default context
        return null;
    }

    protected override IntPtr LoadUnmanagedDll(string unmanagedDllName)
    {
        string path = _resolver.ResolveUnmanagedDllToPath(unmanagedDllName);
        if (path != null)
            return LoadUnmanagedDllFromPath(path);

        return IntPtr.Zero;
    }
}

// Usage:
var pluginContext = new PluginLoadContext(pluginPath);
Assembly pluginAssembly = pluginContext.LoadFromAssemblyName(new AssemblyName("MyPlugin"));
var pluginType = pluginAssembly.GetType("MyPlugin.EntryPoint");
var plugin = (IPlugin)Activator.CreateInstance(pluginType);
plugin.Execute();
```

**Isolation and unloading:**

```csharp
// With collectible ALCs (isCollectible: true), you can unload:
var context = new PluginLoadContext(pluginPath);
// ... load and run plugin ...
context.Unload(); // Unloads ALL assemblies in this context
// BUT: only works if no references from outside keep them alive
```

**When to use ALCs:**

| Scenario | Use ALC? | Why |
|----------|---------|-----|
| Single app, no plugins | Default ALC only | No isolation needed |
| Plugins with same dependencies | Default ALC + custom resolver | Avoid duplication |
| Plugins with conflicting deps | Custom collectible ALC per plugin | True isolation |
| Hot-reload / code generation | Custom ALC | Unload old code, load new |
| Running user-submitted scripts | Custom ALC with security | Isolate malicious assembly access |

**Common pitfalls:**
```csharp
// ❌ Type from different ALCs are DIFFERENT types
// Even if both load "MyPlugin.Plugin" from the same file
var type1 = defaultContext.LoadFromAssemblyName(...).GetType("MyPlugin.Plugin");
var type2 = pluginContext.LoadFromAssemblyName(...).GetType("MyPlugin.Plugin");
type1 == type2 // FALSE — different ALCs, different type identities

// ✅ Use interfaces from a shared assembly in the Default ALC
var plugin = (IPlugin)Activator.CreateInstance(type); // IPugin from Default ALC
```

---

### ❓ Q10: Exception handling internals?

**Answer:**

*What happens when you throw:*

```csharp
public void Process(Order order)
{
    if (order is null)
        throw new ArgumentNullException(nameof(order));

    CalculateTotal(order);
}
```

**Step 1: CLR allocates exception object on heap**

```
Stack:                              Heap:
┌────────────────────────────────┐  ┌──────────────────────────────────┐
│ Process frame                  │  │ ArgumentNullException object     │
│   order: null                  │  │ [SyncBlock] [TypeHandle]         │
│   return_addr → CalculateTotal │  │ Message: "Value cannot be null"  │
│   Exception reified here       │  │ ParamName: "order"               │
└────────────────────────────────┘  │ StackTrace: (empty, to be filled)│
                                    └──────────────────────────────────┘
```

**Step 2: Stack unwinding begins**

The CLR walks the call stack looking for a matching `catch` block:

```
Call stack at throw:
Main → ProcessOrder → Process → ⚡ throw (here)
                                 ↓
CLR searches for catch...         
↓                                  ↓
Main has no catch → ProcessOrder has catch(ArgumentNullException)? No
↓                                  ↓
Top-level has catch(Exception)? Yes → FOUND!
↓
Unwind to top-level: 
- Process frame destroyed
- ProcessOrder frame destroyed  
- Main continues to catch block
```

During unwinding, `finally` blocks execute:

```csharp
public void Process(Order order)
{
    var timer = Stopwatch.StartNew();
    try
    {
        if (order is null) throw new ArgumentNullException();
        Save(order);
    }
    finally
    {
        // ALWAYS executes — even during unwinding
        Console.WriteLine($"Took {timer.Elapsed}ms");
    }
}
```

**Step 3: Exception filters (when clause)**

```csharp
try
{
    Process(order);
}
catch (DbException ex) when (ex.IsTransient)
{
    // Only catches transient DB errors — non-transient pass through
    await RetryAsync();
}
catch (DbException ex) when (!ex.IsTransient)
{
    // Catches non-transient errors separately
    LogAndAlert(ex);
}
```

Filters run **before** the stack unwinds — the stack is intact, so you can inspect local variables:

```csharp
try
{
    Save(order);
}
catch (Exception ex) when (LogWithoutCatching(ex))
{
    // This catch never actually matches — we just logged
    // The exception continues propagating
}

static bool LogWithoutCatching(Exception ex)
{
    Logger.Log(ex);
    return false; // false → catch block doesn't execute, exception keeps going
}
```

**Step 4: Exception filters vs catch+rethrow**

Filters NEVER unwind the stack before evaluation, while catch+rethrow does unwinding first:

```csharp
// ❌ catch + rethrow — loses original call stack
try { DoWork(); }
catch (Exception ex)
{
    logger.Log(ex);
    throw; // ✅ preserves stack trace, but we already hit the catch block
}

// Using filters — stack is preserved during evaluation
try { DoWork(); }
catch (Exception ex) when (Logger.ShouldLog(ex))
{
    // Only enters here if filter returns true
}
```

**Exception nesting:**

```csharp
class MyCustomException : Exception
{
    public MyCustomException(string message, Exception inner) 
        : base(message, inner) { }
}

try
{
    try { Parse(null); }
    catch (FormatException ex)
    {
        // Wraps the original, preserving it as inner exception
        throw new MyCustomException("Failed to parse", ex);
    }
}
catch (MyCustomException ex)
{
    Console.WriteLine(ex.InnerException?.Message); // original FormatException
    Console.WriteLine(ex.StackTrace); // stack trace for the outer
}
```

**When catch blocks are evaluated:**

The CLR uses a method table-based search. Each method has an exception handling table embedded in its IL metadata:

```
Method Process(ExceptionLookupTable):
┌──────────────────────────────┐
│ try block: IL 0x10 - 0x35    │
│ catch(Exception) at IL 0x40  │
│ finally at IL 0x60           │
└──────────────────────────────┘
```

The JIT generates native code with this table. When an exception occurs, the CLR uses the table to find the correct handler without re-JITting.

**Critical performance note:**

```csharp
// ❌ Using exceptions for flow control — 1000x slower
public int ParseOrZero(string input)
{
    try { return int.Parse(input); } // throws on invalid input
    catch { return 0; }
}

// ✅ Use TryParse — no exception overhead
public int ParseOrZero(string input)
{
    return int.TryParse(input, out var result) ? result : 0;
}
```

Benchmark (invalid input, 1M iterations):
| Method | Time | Allocations |
|--------|------|-------------|
| `int.Parse` + catch | 12.4s | 1.2 GB (exception objects) |
| `int.TryParse` | 0.08s | 0 MB |

Exceptions are **not** for flow control. They're for unexpected, exceptional conditions.

---

### ❓ Q11: What is Garbage Collection pause (Stop-the-world)?

**Answer:**

During a GC, the runtime suspends all managed threads to scan roots, mark live objects, and compact memory. This suspension is called **Stop-the-world (STW)**.

**What happens during STW:**

```
Normal execution:
ThreadA: ──Request1────Request2──────Request3──────
ThreadB: ──Request4────────────Request5─────────────
ThreadC: ──Request6────────────────────Request7─────

GC trigger (Gen 0 full):
ThreadA: ──Request1────Request2──║                    ║──Request3──────
ThreadB: ──Request4──────────────║  ALL THREADS        ║──Request5─────
ThreadC: ──Request6──────────────║  SUSPENDED           ║────Request7───
                                  ║   GC RUNS HERE      ║
                                  ║  (50-200μs Gen 0)   ║
                                  ╚══════════════════════╝
```

**Duration by generation:**

| Generation | Typical pause | Frequency | Impact |
|-----------|--------------|-----------|--------|
| Gen 0 | 50-200μs | Every ~1s (high throughput) | Negligible |
| Gen 1 | 200-500μs | Every ~10-30s | Low |
| Gen 2 | 10-500ms+ | Rare (minutes to hours) | Noticeable |
| Full blocking (all gens) | 100ms - several seconds | Very rare | Severe latency spikes |

**Why Gen 2 pauses are expensive:**

Gen 2 collections scan the **entire managed heap** — Gen 0, Gen 1, Gen 2, and LOH. Every object's reachability must be verified. With a 10GB heap:

```
10GB heap / 100 MB/s scan rate ≈ 100 seconds worst case
(With modern hardware and parallel GC: 100ms - 2s)
```

**Background GC (mitigation):**

Server GC in .NET Core uses **background GC** — Gen 2 collections run on a dedicated thread while application threads continue:

```
Background GC:
ThreadA: ──Request1────Request2────Request3─────────────────Request4──
ThreadB: ──Request5────────────Request6────────────────────Request7───
BG GC:    ════════Gen0(200μs)═══════Gen2 BACKGROUND═══════════════════
                                    ↑ App keeps running
                                    ↑ Only short Gen0 pauses occur
```

Background GC still has short blocking pauses for Gen 0/1, but Gen 2 compaction can happen concurrently.

**Measuring GC pauses:**

```csharp
// Using ETW events programmatically
var config = new GCConfiguration
{
    PauseTimeThreshold = TimeSpan.FromMilliseconds(200)
};

// Monitor with dotnet counters
dotnet counters monitor -p 1234
// Look for: time-in-gc (%), gen-2-gc-count, pause-time-percentage

// In code:
GCMemoryInfo info = GC.GetGCMemoryInfo();
Console.WriteLine($"Pause duration: {info.PauseDurations[0].TotalMilliseconds}ms");
Console.WriteLine($"Pause percentage: {info.PauseTimePercentage}%");
```

**When STW becomes a problem:**

| Scenario | Impact | Solution |
|----------|--------|----------|
| High-throughput API (1000+ req/s) | Gen 0 pauses are frequent but fine (~200μs). Gen 2 pauses cause latency spikes | Use Server + Background GC. Keep heap small. Pool objects. |
| Real-time trading (sub-ms latency) | Any pause > 100μs is problematic | Use `GCLatencyMode.LowLatency` or `SustainedLowLatency`. Pre-allocate. Use structs. Consider `GC.TryStartNoGCRegion()`. |
| Gaming (60 FPS = 16ms frame budget) | Pauses > 16ms cause frame drops | Use SustainedLowLatency mode during gameplay. Force collection during loading screens. |
| SignalR / WebSocket / streaming | Mid-stream pauses cause buffering delays | Background GC helps. Isolate real-time users to separate process. |

**Configuring GC for low latency:**

```csharp
// Sustained low latency (ASP.NET Core default: Background GC)
GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency;

// Low latency mode — discourages Gen 2 collections (may OOM under memory pressure)
GCSettings.LatencyMode = GCLatencyMode.LowLatency;

// No GC region — temporary critical section
if (GC.TryStartNoGCRegion(100 * 1024 * 1024)) // 100MB budget
{
    // Critical section — no GC allowed
    ProcessHighPriorityWork();
    GC.EndNoGCRegion();
}
```

**The pause time equation:**

```
Total pause time = (Live objects × Mark cost) + (Compacted objects × Move cost)
                  ──────────────────   ───────────────────────────────────
                  Proportional to        Proportional to
                  live heap size         object size × number of survivors

Minimizing: keep live heap small, reduce allocations, pool objects
```

---

### ❓ Q12: What is Finalization in .NET?

**Answer:**

Finalization is the mechanism to run cleanup code before an object is garbage collected. It's designed for objects that wrap unmanaged resources (file handles, database connections, etc.).

**How finalization works (the two-queue dance):**

```csharp
public class FileWriter
{
    private IntPtr _fileHandle = NativeMethods.OpenFile("data.txt");

    ~FileWriter() // Finalizer — called by GC, NOT by user code
    {
        NativeMethods.CloseHandle(_fileHandle);
    }
}
```

When a finalizable object is allocated, the GC adds it to the **Finalization Queue**.

```
Step 1: new FileWriter() allocates
         ┌──────────────┐    ┌───────────────────┐
         │ FileWriter    │──→ │ Finalization Queue│
         │ _fileHandle   │    └───────────────────┘
         └──────────────┘

Step 2: FileWriter becomes unreachable (no more references)
         GC detects it during Mark phase

Step 3: Instead of freeing memory, GC moves it to the fReachable Queue
         ┌──────────────┐    ┌──────────────────────┐
         │ FileWriter    │──→ │ fReachable Queue     │
         │ _fileHandle   │    │ (objects waiting for │
         └──────────────┘    │  finalization)        │
                             └──────────────────────┘

Step 4: Dedicated finalizer thread runs ~Finalizer()
         Thread picks up FileWriter → calls ~FileWriter() → closes handle

Step 5: Next GC — object is now non-finalizable → memory reclaimed
```

**The cost of finalization:**

| Stage | Without finalizer | With finalizer |
|-------|------------------|----------------|
| Allocation | ptr bump | ptr bump + add to Finalization Queue |
| GC detection | Mark as dead → free | Mark as dead → move to fReachable |
| Memory freed | Immediately | **After** finalizer runs (one MORE GC cycle) |
| Thread involvement | GC thread only | GC thread + Finalizer thread |
| Object survives | 0 extra collections | 1 extra collection |

This means: **finalizable objects survive at least one additional GC**, keeping memory alive longer.

**The IDisposable pattern (proper solution):**

```csharp
public class FileWriter : IDisposable
{
    private IntPtr _fileHandle;

    public FileWriter(string path)
    {
        _fileHandle = NativeMethods.OpenFile(path);
    }

    public void Dispose()
    {
        // 1. Clean up managed resources
        // 2. Clean up unmanaged resources
        NativeMethods.CloseHandle(_fileHandle);
        _fileHandle = IntPtr.Zero;

        // 3. Tell GC: finalizer not needed — skip it
        GC.SuppressFinalize(this);
    }

    // Finalizer as safety net (only if Dispose isn't called)
    ~FileWriter()
    {
        Dispose();
    }
}

// Usage:
using (var writer = new FileWriter("data.txt"))
{
    writer.Write(...);
} // Dispose() called here → SuppressFinalize → object collected normally
```

**Finalization vs Dispose:**

| | Finalizer (`~ClassName`) | Dispose (`IDisposable`) |
|---|---|---|
| Called by | GC (non-deterministic) | User code (deterministic) |
| When | Unknown time after object becomes unreachable | Immediately in `using` block `}` |
| Thread | Finalizer thread (single, sequential) | Caller's thread |
| Exception handling | ⚠️ Exception kills the process | Caller can catch |
| Performance | Object survives extra GC | No extra GC cycles |
| Parameter safety | Cannot use other finalizable objects (they may already be finalized) | Any object is safe |

**The finalizer thread problem:**

```csharp
~MyClass()
{
    // ❌ NEVER do this — the logger might already be finalized!
    Logger.Log("Finalizing MyClass");

    // ❌ NEVER do this — the stream might already be finalized!
    _anotherStream.Dispose();
}

// ̃MyOtherClass()
// {
//     _myStream.Dispose(); // WRONG — _myStream's finalizer may run first
// }
```

The finalizer thread runs finalizers sequentially. Thread.Sleep in one finalizer delays ALL finalization, growing the fReachable queue and causing memory pressure.

**Weak event handlers (avoiding finalization leaks):**

```csharp
public class MyService
{
    public event EventHandler DataChanged;

    ~MyService()
    {
        // ⚠️ Don't raise events or interact with managed objects here
        // They may already be finalized or in an unknown state
    }
}
```

**Modern best practice — no finalizer if you can help it:**

```csharp
// ✅ Modern pattern: only IDisposable, no finalizer
public class FileWriter : IDisposable
{
    private SafeFileHandle _handle; // SafeFileHandle handles finalization itself

    public void Dispose()
    {
        _handle?.Dispose();
    }
}
```

Use `SafeHandle` subclasses for native resources. They implement the finalization pattern correctly so you don't have to. Only implement a finalizer if you're directly holding an `IntPtr` to a native resource.

---

### ❓ Q13: What is IDisposable pattern?

**Answer:**

`IDisposable` provides a deterministic way to release unmanaged resources (file handles, sockets, database connections, etc.) immediately instead of waiting for the GC's non-deterministic finalization.

**The basic pattern:**

```csharp
public class Resource : IDisposable
{
    public void Dispose()
    {
        // Release all resources
    }
}

// Usage — deterministic cleanup:
using (var resource = new Resource())
{
    resource.DoWork();
} // Dispose() called HERE, immediately after exiting the block
```

**The full dispose pattern (with inheritance):**

```csharp
public class BaseResource : IDisposable
{
    private bool _disposed;
    private SafeHandle _handle;
    private MemoryStream _memoryStream;

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;

        if (disposing)
        {
            // Release MANAGED resources (objects that implement IDisposable)
            _memoryStream?.Dispose();
            _handle?.Dispose();
        }

        // Release UNMANAGED resources (IntPtr, raw handles, etc.)
        // (none in this example)

        _disposed = true;
    }

    ~BaseResource() => Dispose(false);
}

public class DerivedResource : BaseResource
{
    private bool _disposed;
    private HttpClient _client;

    protected override void Dispose(bool disposing)
    {
        if (_disposed) return;

        if (disposing)
        {
            _client?.Dispose();
        }

        // Release unmanaged resources specific to this class

        _disposed = true;

        // Call base — always call base after disposing own resources
        base.Dispose(disposing);
    }

    ~DerivedResource() => Dispose(false);
}
```

**The `using` statement (what it compiles to):**

```csharp
using (var file = new FileStream("data.txt", FileMode.Open))
{
    file.Read(buffer);
}

// Compiler generates:
var file = new FileStream("data.txt", FileMode.Open);
try
{
    file.Read(buffer);
}
finally
{
    if (file != null)
        file.Dispose();
}
```

For `await using` (IAsyncDisposable):

```csharp
await using (var conn = new DbConnection())
{
    await conn.OpenAsync();
}

// Compiler generates try/finally with DisposeAsync()
```

**IAsyncDisposable (for async cleanup):**

```csharp
public class DbConnection : IAsyncDisposable
{
    private NetworkStream _stream;

    public async ValueTask DisposeAsync()
    {
        await _stream.FlushAsync();
        _stream.Dispose();
    }
}

// Usage:
await using var conn = new DbConnection();
await conn.OpenAsync();
``` // DisposeAsync() called here, awaited

**Common mistakes:**

```csharp
// ❌ Not disposing — resource held until GC finalizes it (which may never come in time)
var reader = new StreamReader("file.txt");
reader.ReadToEnd(); // file handle stays open

// ✅ Always dispose
using var reader = new StreamReader("file.txt");
reader.ReadToEnd();

// ❌ Returning undisposed resource
public MemoryStream GetStream()
{
    var ms = new MemoryStream();
    // ... write to it ...
    return ms; // caller now owns responsibility
}
// ✅ Acceptable since caller will dispose

// ❌ Double dispose (usually safe, but depends on implementation)
using (var ms = new MemoryStream())
{
    ms.Dispose(); // explicit dispose
} // disposed again in '}' — MemoryStream handles it gracefully

// ⚠️ Dispose in finalizer — never call Dispose on other objects
~MyClass()
{
    // This is in the finalizer thread
    // _anotherObj might already be finalized → AccessViolationException
    _anotherObj.Dispose(); // ❌ DANGEROUS
}
```

**When to implement IDisposable:**

| You have... | Implement... |
|------------|-------------|
| A managed `IDisposable` field (Stream, SqlConnection, HttpClient) | `IDisposable` — dispose the field |
| An `IntPtr` to a native resource | `IDisposable` + finalizer (or use `SafeHandle`) |
| A class with neither managed nor unmanaged resources | Don't implement `IDisposable` (no need) |
| A base class that has resources | `protected virtual void Dispose(bool)` for inheritance |
| An async cleanup pattern | `IAsyncDisposable` |

**Reality check — most classes should NOT implement IDisposable:**

```csharp
// ❌ Unnecessary — only managed memory, no resources
public class OrderCalculator : IDisposable
{
    public void Dispose() { } // pointless — GC handles memory
}

// ✅ Only when you own a resource
public class OrderRepository : IDisposable
{
    private SqlConnection _conn; // I own the connection → I must dispose it
    
    public void Dispose()
    {
        _conn?.Dispose();
    }
}
```

---

### ❓ Q14: What is difference between GC.Collect() and automatic GC?

**Answer:**

**Automatic GC** runs when the GC determines it's needed — typically when Gen 0 fills up. The GC uses heuristics (allocation rate, survival rate, memory pressure) to decide the optimal moment.

**`GC.Collect()`** forces a collection immediately, ignoring the GC's heuristics:

```csharp
GC.Collect();          // Collects all generations (0, 1, 2)
GC.Collect(0);         // Collects Gen 0 only
GC.Collect(1);         // Collects Gen 0 + Gen 1
GC.Collect(2);         // Full collection (all generations + LOH)
GC.Collect(2, GCCollectionMode.Optimized); // Collect only if beneficial
```

**Why automatic is better:**

```csharp
// Auto GC: when Gen 0 is full (~1MB)
// Apps allocate/deallocate 1MB in ~1s → GC runs naturally
// Pause: ~200μs — negligible
// Memory: freed efficiently

// GC.Collect() at wrong time:
// Object is still alive → promoted to Gen 2 (even though it would die in 5ms)
// Gen 2 collection is expensive (full heap scan)
// You FORCED it to run when it wouldn't have → wasted CPU
```

**When would you call GC.Collect() (rare, justified scenarios):**

```csharp
// 1. After a large batch of work ends — good time to clean up
public void ProcessBatch(List<Order> orders)
{
    foreach (var order in orders)
    {
        var report = GenerateReport(order); // allocates large temp objects
    }
    // All temp objects are now dead — prime time for GC
    GC.Collect(); // ⚠️ Only if you measured this is beneficial
    GC.WaitForPendingFinalizers(); // If you have finalizable objects
}

// 2. Before entering a latency-sensitive section
GC.Collect(2); // Clean up Gen 2
GC.WaitForPendingFinalizers();
GC.Collect(2); // Clean up finalizer survivors
GCSettings.LatencyMode = GCLatencyMode.LowLatency;
// Now enter critical section with minimal GC risk
```

**The cost of unnecessary GC.Collect():**

```
Scenario: ASP.NET API serving 1000 req/s
Memory: 500MB heap, 20% live data

Strategy             | GC CPU time | P99 latency | Throughput
─────────────────────|─────────────|─────────────|───────────
Auto (no manual GC)  | 2%          | 15ms        | 1000 req/s
GC.Collect every req | 35%         | 120ms       | 650 req/s
GC.Collect every 10s | 15%         | 45ms        | 850 req/s
```

**The one acceptable use case — unit tests:**

```csharp
[Fact]
public void WeakReference_test()
{
    var obj = new MyClass();
    var weak = new WeakReference<MyClass>(obj);

    obj = null;
    GC.Collect();      // Force collection to test weak reference behavior
    GC.WaitForPendingFinalizers();

    Assert.False(weak.TryGetTarget(out _));
}
```

**When automatic GC might not be enough:**

```csharp
// High allocation rate scenario:
// Allocating 100MB/s in Gen 0 → GC runs every 10ms
// GC overhead becomes significant (10%+ CPU)

// Solutions:
// 1. Reduce allocations → fewer GCs
// 2. Use object pooling → recycle instead of allocate
// 3. Use structs → stack allocation, no GC
// NOT: calling GC.Collect() to "help" — it makes it worse
```

---

### ❓ Q15: What is ThreadPool?

**Answer:**

The ThreadPool is a managed pool of worker threads maintained by the CLR. It reuses threads to avoid the expensive cost of creating and destroying threads constantly.

**How ThreadPool works:**

```
Request comes in → TP checks for free thread
    ├── Thread available → assign work
    └── No thread → queue work
         └── Monitor thread checks queue
              ├── Thread freed → pick up queued work
              └── Queue growing → inject new thread (1 per 500ms, up to max)
```

```csharp
// You rarely use ThreadPool directly — Task.Run does it internally
ThreadPool.QueueUserWorkItem(state =>
{
    Console.WriteLine("Running on thread pool");
});

// Same as:
Task.Run(() => Console.WriteLine("Running on thread pool"));
```

**ThreadPool vs creating threads directly:**

| Aspect | `new Thread()` | `ThreadPool` |
|--------|---------------|--------------|
| Creation cost | ~100ms (OS thread creation) | ~1μs (reuse from pool) |
| Stack size | 1MB per thread | Reuses existing thread |
| Peak threads | Limited by memory/Linux settings | Controlled (min/max) |
| Oversubscription | Manual | Auto-injection heuristic |
| Ideal for | Long-running, dedicated threads | Short-lived, burst work |

**ThreadPool heuristics:**

```csharp
// Defaults (.NET Core):
ThreadPool.GetMinThreads(out int minWorker, out int minIO);
ThreadPool.GetMaxThreads(out int maxWorker, out int maxIO);
// Typically: min=8 (per core), max=32767 (worker)
// But: max is a soft limit — new threads are injected slowly (1 per 500ms)

// The "hill climbing" algorithm:
// ThreadPool monitors throughput vs threads
// If throughput ↑ as threads ↑ → add more
// If throughput ↓ as threads ↑ → reduce
// Scales to optimal thread count adaptively
```

**Thread starvation (the real problem):**

```csharp
// Problem: blocking an async method on the thread pool
public async Task<IActionResult> Get()
{
    // Thread from pool enters here
    Thread.Sleep(1000); // BLOCKS this thread — it's gone from the pool for 1s
    var data = await GetDataAsync();
    return Ok(data);
}

// Under load:
// 1. Request comes in → TP gives Thread A
// 2. Thread.Sleep(1000) → A is blocked for 1s
// 3. New request → TP gives Thread B
// 4. B also blocks...
// 5. Eventually all threads blocked → queue grows → 503s

// The fix: don't block:
public async Task<IActionResult> Get()
{
    await Task.Delay(1000); // thread returned to pool during wait
    var data = await GetDataAsync();
    return Ok(data);
}
```

**ThreadPool min/max configuration:**

```csharp
// Increase min threads to avoid slow injection under burst load
// Add this in Program.cs or Startup:
ThreadPool.SetMinThreads(
    workerThreads: Environment.ProcessorCount * 2,
    completionPortThreads: Environment.ProcessorCount * 2
);

// Best practice: set min to the number of concurrent requests you expect
// For a web server expecting 200 concurrent requests:
ThreadPool.SetMinThreads(200, 200);
```

**Dedicated thread vs ThreadPool:**

```csharp
// ThreadPool: short burst work
await Task.Run(() => CompressImage(image)); // ~50ms CPU work

// Dedicated thread: long-running
var thread = new Thread(() =>
{
    while (true) // NEVER ENDS — would block a ThreadPool thread forever
    {
        CheckForUpdates();
        Thread.Sleep(5000);
    }
});
thread.IsBackground = true;
thread.Start();

// OR: TaskCreationOptions.LongRunning
var task = Task.Factory.StartNew(() =>
{
    while (true) { WatchQueue(); }
}, TaskCreationOptions.LongRunning); // Creates dedicated thread, not TP
```

---

### ❓ Q16: What is deadlock?

**Answer:**

A deadlock occurs when two or more threads each hold a lock that the other needs, resulting in all threads being stuck indefinitely.

**The classic deadlock:**

```csharp
private static readonly object _lockA = new();
private static readonly object _lockB = new();

// Thread 1
void Method1()
{
    lock (_lockA)
    {
        Thread.Sleep(100); // ensure Thread 2 gets lockB first
        lock (_lockB)      // BLOCKS — Thread 2 holds lockB
        {
            // Never reached
        }
    }
}

// Thread 2
void Method2()
{
    lock (_lockB)
    {
        Thread.Sleep(100); // ensure Thread 1 gets lockA first
        lock (_lockA)      // BLOCKS — Thread 1 holds lockA
        {
            // Never reached
        }
    }
}
```

```
Thread 1: holds _lockA, waiting for _lockB
Thread 2: holds _lockB, waiting for _lockA
          → Both wait forever → DEADLOCK
```

**Four conditions for deadlock:**

1. **Mutual exclusion** — resources can't be shared
2. **Hold and wait** — threads hold one lock while waiting for another
3. **No preemption** — locks can't be forcibly taken away
4. **Circular wait** — circular chain of threads waiting

**Detecting deadlocks:**

```bash
# In production
dotnet-dump collect -p 1234
dotnet-dump analyze core.dmp
> threads
> clrstack
# Look for threads with Monitor.Enter waiting for each other

# In code — use timeout:
if (Monitor.TryEnter(_lock, TimeSpan.FromSeconds(5)))
{
    try { /* critical section */ }
    finally { Monitor.Exit(_lock); }
}
else
{
    logger.LogWarning("Potential deadlock detected on _lock");
}
```

**Prevention strategies:**

```csharp
// 1. Always acquire locks in the SAME order
lock (_lockA) { lock (_lockB) { ... } } // Everywhere uses A→B

// 2. Use timeout (avoids infinite wait)
if (Monitor.TryEnter(_lock, TimeSpan.FromSeconds(3)))
{
    try { /* critical */ }
    finally { Monitor.Exit(_lock); }
}

// 3. Reduce lock scope — lock less, lock later
lock (_lock)         // ❌ Lock held too long
{
    var data = ReadData();
    var transformed = ExpensiveTransformation(data);
    SaveToDb(transformed);
}

// ✅ Lock only what's necessary
var data = ReadData();
var transformed = ExpensiveTransformation(data); // no lock
lock (_lock) { SaveToDb(transformed); }

// 4. Use higher-level concurrency primitives
// SemaphoreSlim, ReaderWriterLockSlim, Channel<T>, ConcurrentDictionary
// These are less error-prone than manual locking

// 5. Avoid nested locks entirely where possible
var dbLock = new object();
var cacheLock = new object();

// ❌ Nested locks
lock (dbLock) { lock (cacheLock) { ... } }

// ✅ Single lock or restructure to avoid nesting
```

**Real-world deadlock examples:**

```csharp
// 1. UI deadlock (WinForms/WPF)
// UI thread:
var result = await Task.Run(() => ComputeAsync()).Result; // DEADLOCK
// Background task tries to marshal back to UI thread (via SynchronizationContext)
// UI thread is blocked waiting for task → task waits for UI → DEADLOCK

// 2. Thread pool deadlock
// If all thread pool threads are blocked waiting for each other
// and no more threads can be created (max reached or min not set)
// → Complete thread pool starvation

// 3. Event-driven deadlock
void OnDataReceived(object sender, EventArgs e)
{
    _resetEvent.WaitOne(); // Waiting for signal in an event handler
}
// Somewhere else, a background thread does:
dataProcessor.Process(data); // Calls Process → raises DataReceived
// But DataReceived handler blocks → Process never completes
// → Signal never set → DEADLOCK
```

**Async deadlock (common in ASP.NET Framework — fixed in ASP.NET Core):**

```csharp
// ASP.NET Framework (has SynchronizationContext):
public ActionResult Get()
{
    var user = _service.GetUserAsync(1).Result; // BLOCKS
    return Ok(user);
}
// The .Result blocks the request thread
// GetUserAsync tries to resume on the original SynchronizationContext
// But that context is the blocked request thread
// → DEADLOCK: thread waits for task, task waits for thread

// ASP.NET Core (no SynchronizationContext):
public ActionResult Get()
{
    var user = _service.GetUserAsync(1).Result; // BAD but won't deadlock
    return Ok(user);
}
// No SC → continuation runs on thread pool → doesn't wait for blocked thread
// Still BAD for scalability (thread wasted), but at least it completes
```

---

### ❓ Q17: What is async deadlock?

**Answer:**

Async deadlock happens when synchronous blocking (`.Result`, `.Wait()`, `.GetAwaiter().GetResult()`) is used on async code, and the async continuation needs to resume on the blocked thread.

**The mechanism:**

```csharp
// ASP.NET Framework (full .NET, with SynchronizationContext):
public ActionResult GetUser(int id)
{
    var user = _service.GetUserAsync(id).Result;
    //                                                 ┌────────────────┐
    // 1. .Result blocks the request thread            │ Request Thread  │
    // 2. GetUserAsync starts the DB query async       │ BLOCKED on .R. │
    // 3. DB query completes                       ╔═══╪════════════════╗
    // 4. Continuation tries to resume on original   ║  |  .Result     ║
    //    SynchronizationContext (the request thread) ║  |  hold        ║
    // 5. But that thread is BLOCKED on .Result       ║  ↓  forever     ║
    //    → DEADLOCK                              ╚═══╧════════════════╝
    return Ok(user); // never reached
}
```

**The same in WinForms/WPF:**

```csharp
private void button_Click(object sender, EventArgs e)
{
    // UI thread, on UI SynchronizationContext
    var data = LoadDataAsync().Result; // DEADLOCK
    textBox.Text = data;
}
```

**Why ASP.NET Core is safe:**

```csharp
// ASP.NET Core removed the SynchronizationContext (performance optimization):
public IActionResult GetUser(int id)
{
    var user = _service.GetUserAsync(id).Result;
    // After DB query completes, continuation runs on ANY thread pool thread
    // Not blocked, no deadlock — but thread pool thread is STILL wasted
    return Ok(user);
}
```

**All blocking patterns to avoid:**

```csharp
// ❌ All deadlock in ASP.NET Framework / WinForms / WPF
var result = GetDataAsync().Result;
var result = GetDataAsync().GetAwaiter().GetResult();
var result = GetDataAsync().Wait();
Task.WaitAll(GetDataAsync()); // WaitAll
Task.WaitAny(GetDataAsync()); // WaitAny

// ✅ Correct
var result = await GetDataAsync();

// ❌ Wrong fix — ConfigureAwait(false) helps in library code but not always in UI
var result = GetDataAsync().ConfigureAwait(false).GetAwaiter().GetResult();
// Still blocks! And in ASP.NET Core there's no SC to begin with.

// ✅ The only fix — async all the way
public async Task<IActionResult> GetUser(int id)
{
    var user = await _service.GetUserAsync(id);
    return Ok(user);
}
```

**Making sync wrappers correctly:**

```csharp
// ❌ This is problematic
public User GetUserSync(int id)
{
    return GetUserAsync(id).GetAwaiter().GetResult();
}

// ✅ Alternative — use a dedicated thread (only if you MUST have sync API)
public User GetUserSync(int id)
{
    return Task.Run(() => GetUserAsync(id)).GetAwaiter().GetResult();
    // Runs on thread pool, not blocking original context
}

// ✅ Better — don't create sync wrappers. Go async end-to-end.
```

---

### ❓ Q18: What is immutability?

**Answer:**

An immutable object cannot be changed after creation. Every "modification" creates a new object with the change applied.

```csharp
public class ImmutablePerson
{
    public string Name { get; }          // getter-only — set in constructor
    public int Age { get; }              // readonly fields

    public ImmutablePerson(string name, int age)
    {
        Name = name;
        Age = age;
    }

    // "Modification" returns NEW instance
    public ImmutablePerson WithAge(int newAge) =>
        new ImmutablePerson(Name, newAge);
}
```

**Why immutability matters:**

```csharp
var person = new ImmutablePerson("Kuldeep", 30);
// Thread A reads person — sees Name="Kuldeep", Age=30
// Thread B wants to update:
var updated = person.WithAge(31); // creates new object
// Thread A sees the SAME person — no race condition

// Vs mutable:
var mutable = new MutablePerson { Name = "Kuldeep", Age = 30 };
// Thread A reads mutable.Age at exact moment Thread B sets mutable.Age = 31
// → Thread A might see 30 or 31 — RACE CONDITION
```

**Benefits:**

```csharp
// 1. Thread-safe by design — no locks needed
public class Cache
{
    private ImmutableDictionary<string, Data> _cache = ImmutableDictionary<string, Data>.Empty;

    public Data Get(string key) => _cache.TryGetValue(key, out var data) ? data : null;

    public void Update(string key, Data value)
    {
        // Atomic swap — readers see either old or new, never intermediate
        Interlocked.Exchange(ref _cache, _cache.SetItem(key, value));
    }
}

// 2. Predictable — object always in valid state
var order = new Order(100, "USD"); // always has amount + currency
// order.Amount = -50; // ❌ Won't compile — no setter

// 3. Safe as dictionary keys / hash set members
var dict = new Dictionary<ImmutablePerson, string>();
dict.Add(new ImmutablePerson("Alice", 30), "Engineer"); // hash is permanent
// With mutable: change object → hash changes → can't find it anymore
```

**Immutable collections in .NET:**

```csharp
using System.Collections.Immutable;

ImmutableArray<int> arr = ImmutableArray.Create(1, 2, 3);
ImmutableList<int> list = ImmutableList<int>.Empty;
ImmutableDictionary<string, int> dict = ImmutableDictionary<string, int>.Empty;

// "Modifications" return new instances (structural sharing for performance)
var list1 = ImmutableList<int>.Empty;
var list2 = list1.Add(1);  // list1 is still empty, list2 has 1
var list3 = list2.Add(2);  // list1=[], list2=[1], list3=[1,2]
// Structural sharing: list2 and list3 share the internal node for 1
```

**Performance characteristics:**

| Operation | Mutable | Immutable |
|-----------|---------|-----------|
| Read | O(1) | O(1) |
| Add one item | O(1) amortized | O(log n) typically (structural sharing) |
| Memory | Same object, in-place | New object (but shares unchanged parts) |
| GC pressure | Low per operation | Higher (temporary objects) |

**When to use:**

```csharp
// ✅ Immutable records are PERFECT for DTOs
public record OrderDto(int Id, decimal Total, string Currency);
// Built-in immutability, value equality, with-expression

// ✅ Configuration objects
public class AppConfig
{
    public string DbConnection { get; }
    public int TimeoutSeconds { get; }
    // Set once, never change
}

// ❌ DON'T for hot path allocations in tight loops
public ImmutableList<int> Process(int[] data)
{
    var list = ImmutableList<int>.Empty;
    foreach (var item in data)
        list = list.Add(item * 2); // O(n²) in ImmutableList
    return list;
}
// ✅ Use mutable List then convert
public ImmutableList<int> Process(int[] data)
{
    var list = new List<int>(data.Length);
    foreach (var item in data)
        list.Add(item * 2);
    return list.ToImmutableList();
}
```

---

### ❓ Q19: What is reflection?

**Answer:**

Reflection allows inspecting and invoking types, methods, properties, and fields at runtime that aren't known at compile time.

```csharp
// Inspect a type at runtime
Type type = typeof(User);
PropertyInfo[] props = type.GetProperties();
MethodInfo[] methods = type.GetMethods();

Console.WriteLine($"Type: {type.Name}");
foreach (var prop in props)
    Console.WriteLine($"  Property: {prop.Name} ({prop.PropertyType.Name})");

// Output:
// Type: User
//   Property: Id (Int32)
//   Property: Name (String)
//   Property: Email (String)
```

**Creating objects via reflection:**

```csharp
// Instead of: var user = new User("Alice", "alice@test.com");
Type userType = typeof(User);
object user = Activator.CreateInstance(userType, "Alice", "alice@test.com");

// Or without constructor args:
object user = Activator.CreateInstance<User>();
```

**Invoking methods dynamically:**

```csharp
public class Calculator
{
    public int Add(int a, int b) => a + b;
}

// Normal: var calc = new Calculator(); calc.Add(2, 3);

// Via reflection:
Type calcType = typeof(Calculator);
object calc = Activator.CreateInstance(calcType);
MethodInfo addMethod = calcType.GetMethod("Add");
int result = (int)addMethod.Invoke(calc, new object[] { 2, 3 });
// result = 5
```

**Accessing private members (violating encapsulation):**

```csharp
public class Secrets
{
    private string _password = "supersecret";
    private void InternalMethod() => Console.WriteLine("Internal");
}

// Via reflection:
var secrets = new Secrets();
var field = typeof(Secrets).GetField("_password", BindingFlags.NonPublic | BindingFlags.Instance);
string pwd = (string)field.GetValue(secrets); // "supersecret"

var method = typeof(Secrets).GetMethod("InternalMethod", BindingFlags.NonPublic | BindingFlags.Instance);
method.Invoke(secrets, null); // "Internal"
```

**Performance cost:**

```csharp
// Test: 1M iterations
// Normal call:            5ms     (direct JIT)
// Reflection invoke:      850ms   (type checks + virtual dispatch)
// Delegate from MethodInfo: 30ms  (compiled delegate, cached)
// Expression trees:        12ms   (compiled at first use, then JIT direct)

// ⚠️ Avoid in hot paths. Cache what you can.

// ✅ Good — cache MethodInfo, don't re-fetch every time
private static readonly MethodInfo _parseMethod = typeof(int).GetMethod("Parse", new[] { typeof(string) });

public int ParseDynamic(string input) =>
    (int)_parseMethod.Invoke(null, new object[] { input });

// ✅ Even better — compile and cache as delegate
private static readonly Func<string, int> _parseFunc = (Func<string, int>)
    Delegate.CreateDelegate(typeof(Func<string, int>), typeof(int).GetMethod("Parse", new[] { typeof(string) }));

public int ParseDynamic(string input) => _parseFunc(input); // near-native speed
```

**When reflection is useful:**

```csharp
// 1. Serialization/Deserialization (JsonSerializer, EF Core, etc.)
var config = JsonSerializer.Deserialize<Config>(json); // uses reflection internally

// 2. Dependency Injection containers
services.AddScoped<IUserRepository, UserRepository>();
// DI container uses reflection to: 1) find constructor, 2) resolve params, 3) create instance

// 3. ORMs (Entity Framework)
var entity = _context.Users.Find(1);
// EF uses reflection to map columns to properties, track changes

// 4. Testing frameworks
[Fact]
public void Test() { ... }
// xUnit uses reflection to discover [Fact] methods and invoke them

// 5. Plugin/extension systems
Assembly.LoadFrom("plugin.dll").GetTypes()
    .Where(t => typeof(IPlugin).IsAssignableFrom(t))
    .Select(t => (IPlugin)Activator.CreateInstance(t));
```

**Modern alternatives (faster than raw reflection):**

| Need | Alternative | Speed |
|------|------------|-------|
| Property get/set | `Expression<Func<T, TResult>>` compiled | Near-native |
| Method invoke | `Delegate.CreateDelegate` | Near-native |
| Create instance | `ActivatorUtilities.CreateInstance` (DI-aware) | Fast |
| Type check | Pattern matching (`is`, `as`) | Native |
| Dynamic dispatch | `dynamic` keyword | Slower than native, faster than raw reflection |
| Code generation | Source Generators (compile-time) | Native — no runtime cost |

**Source Generators (the modern way):**

```csharp
// Instead of reflection at runtime:
// JSON serializer reads your properties via reflection at runtime

// Source Generators do this at COMPILE TIME:
[JsonSerializable(typeof(User))]
public partial class MyJsonContext : JsonSerializerContext { }
// Source Generator emits code like: user.Name, user.Email — direct property access
// Zero reflection at runtime
```

---

### ❓ Q20: What is Span\<T\>?

**Answer:**

`Span<T>` is a stack-only ref struct that provides a type-safe, memory-safe view over any contiguous memory — arrays, strings, unmanaged buffers, or stack memory — without allocations.

```csharp
Span<int> numbers = stackalloc int[] { 1, 2, 3, 4, 5 };
// Lives entirely on the stack — zero heap allocations
```

**What Span can wrap:**

```csharp
// 1. Array (heap)
int[] arr = new int[] { 1, 2, 3, 4, 5 };
Span<int> span1 = arr;                 // full array
Span<int> span2 = arr.AsSpan(1, 3);    // slice: [2, 3, 4]

// 2. String (heap)
string text = "Hello World";
ReadOnlySpan<char> span3 = text.AsSpan();
ReadOnlySpan<char> span4 = text.AsSpan(0, 5); // "Hello"

// 3. Stack memory (no allocation)
Span<byte> span5 = stackalloc byte[256]; // on stack, < GC pressure

// 4. Unmanaged memory
IntPtr ptr = Marshal.AllocHGlobal(100);
Span<byte> span6 = new Span<byte>(ptr.ToPointer(), 100);
Marshal.FreeHGlobal(ptr);

// 5. Arrays from other processes / native interop
Span<byte> span7 = new Span<byte>(nativeMemory, length);
```

**Why Span exists — the allocation problem:**

```csharp
// ❌ Old way — every slice allocates a new array
byte[] fullData = File.ReadAllBytes("large.bin");
byte[] header = fullData[0..64];     // NEW array copy! 64 bytes allocated
byte[] body = fullData[64..];        // NEW array copy! rest allocated
ProcessHeader(header);
ProcessBody(body);
// Total: 3x the data in memory

// ✅ Span way — zero allocations
byte[] fullData = File.ReadAllBytes("large.bin");
Span<byte> data = fullData;
Span<byte> header = data[0..64];     // VIEW only — no copy
Span<byte> body = data[64..];        // VIEW only — no copy
ProcessHeader(header);
ProcessBody(body);
// Total: 1x the data in memory (same array)
```

**Real-world performance gains:**

```csharp
// Parsing a CSV line — old way (allocations everywhere)
public (string name, int age, string city) ParseCsvOld(string line)
{
    var parts = line.Split(',');         // allocates string[] + each substring
    return (parts[0], int.Parse(parts[1]), parts[2]);
}
// GC allocations per call: 4 strings + 1 array = ~100 bytes

// Span way — zero allocations on the hot path
public (string name, int age, string city) ParseCsvSpan(ReadOnlySpan<char> line)
{
    // Find comma positions, no substring allocations
    var firstComma = line.IndexOf(',');
    var secondComma = line.Slice(firstComma + 1).IndexOf(',') + firstComma + 1;

    var nameSlice = line[..firstComma];        // view
    var ageSlice = line.Slice(firstComma + 1, secondComma - firstComma - 1); // view
    var citySlice = line[(secondComma + 1)..]; // view

    return (new string(nameSlice), int.Parse(ageSlice), new string(citySlice));
    // Only allocates the 3 result strings — intermediate slices are views
}
// GC allocations per call: 3 strings = ~60 bytes

// Benchmark (1M lines):
// Old: 450ms, 120MB GC allocations
// Span: 280ms, 60MB GC allocations
// 38% faster, 50% less GC pressure
```

**Span limitations (because it's a ref struct):**

```csharp
// ✅ Where Span can go:
Span<int> span = ...;
void Process(Span<int> data) { ... } // method parameter
ref struct Container { public Span<int> Data; } // field of ref struct

// ❌ Where Span CAN'T go:
class Wrapper
{
    public Span<int> Data { get; set; } // ❌ ref struct can't be field of class
}
Span<int>[] array = ...; // ❌ ref struct can't be array element
async Task ProcessAsync(Span<int> data) { ... } // ❌ ref struct can't cross await
Func<Span<int>> func = () => ...; // ❌ ref struct can't be in lambda (captured on heap)
dynamic d = span; // ❌ ref struct can't be boxed
```

**ReadOnlySpan\<T\> (the read-only version):**

```csharp
string text = "Hello World";
ReadOnlySpan<char> span = text.AsSpan();
// char c = span[0]; // ✅ read
// span[0] = 'h';    // ❌ compile error — readonly

// Parameters should prefer ReadOnlySpan when data isn't modified
void ProcessData(ReadOnlySpan<byte> data) { ... }
```

**Memory\<T\> (the heap-friendly Span):**

```csharp
// Memory<T> is NOT a ref struct — can live on heap, be used in async
Memory<char> memory = "Hello".AsMemory();

async Task ProcessAsync(Memory<char> data)
{
    await Task.Yield();
    Span<char> span = data.Span; // get Span from Memory inside method
    span[0] = 'h'; // safe inside sync method
}
```

**When to use Span:**

| Scenario | Use Span | Why |
|----------|----------|-----|
| Parsing strings/substrings | Always | Avoid allocations from `Substring`, `Split` |
| Binary data processing | Always | Zero-copy slicing of byte arrays |
| High-throughput APIs | Always | Less GC = more throughput |
| Hot path (called 1000+/s) | Always | Every allocation avoided = measurable GC reduction |
| Async methods | `Memory<T>` instead | Span can't be used across await |
| Class fields | `Memory<T>` or `ReadOnlyMemory<T>` | Span can't be a class field |
| Ref struct fields | Span works | Ref structs are also stack-only |

---

## 📊 GC Lifecycle

```
Allocation → Gen 0 (fast, frequent)
   │
   ├── Dead → Collected (Gen 0 collection)
   │
   └── Survives → Gen 1 (less frequent)
        │
        ├── Dead → Collected (Gen 1 collection)
        │
        └── Survives → Gen 2 (rare, expensive)
             │
             ├── Dead → Collected (Gen 2/full GC)
             │
             └── LOH (≥85KB, rarely compacted)
```

---

## 🎯 Final Revision Points

- GC pauses impact latency — monitor `time-in-gc`
- Avoid manual `GC.Collect()` — let the GC decide
- Use `async/await` correctly — never `.Result` or `.Wait()`
- Prefer immutability for thread safety
- Use `Span<T>` on hot parsing paths — zero allocation slicing
- Boxing hurts GC — use generics, not `ArrayList`/`Hashtable`
- Finalization doubles GC cost — prefer `IDisposable` + `SafeHandle`
- Thread pool starvation is the #1 scalability killer — don't block async
- LOH fragmentation → OOM — use `ArrayPool<T>` for large buffers
- `IQueryable<T>` vs `IEnumerable<T>` — filter at the DB, not in memory
