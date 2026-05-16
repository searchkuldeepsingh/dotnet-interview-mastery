# 🚀 Performance Optimization — Interview Master Guide

> 🧠 Structured for senior-level interview preparation  
> ⚡ Fast revision + deep understanding  

---

## 📌 Overview
> This section covers key concepts required for real-world interviews.

---

# .NET Interview Preparation - Section 7: Performance & Optimization

## 13 Years Experience Candidate Answers

---

## Question 1: What is the difference between profiling and benchmarking in .NET? What tools do you use?

**Answer:**

Profiling measures application behavior during execution:
- Analyzes CPU usage, memory allocation, method execution time
- Identifies bottlenecks and performance issues
- Tools: dotTrace, ANTS Profiler, PerfView, Visual Studio Profiler

Benchmarking measures specific operations for performance comparison:
- Runs operations repeatedly to get accurate timing
- Compares different implementations
- Tools: BenchmarkDotNet

**For 13 years:** Use profiling to find problems, benchmarking to measure improvements. Both are essential for performance optimization work.

---

## Question 2: Explain the different caching strategies: in-memory, distributed, HTTP, output caching.

**Answer:**

**In-Memory Caching:**
- Stores data in process memory
- Fastest access, single server only
- Use: IMemoryCache, application-level data
- Risk: not shared across instances

**Distributed Caching:**
- Shared across multiple servers (Redis, Memcached)
- Slower than in-memory but consistent
- Use: session data, shared state
- Examples: Redis, Memcached

**HTTP Caching:**
- Client-side/browser caching
- Uses Cache-Control, ETag headers
- Use: static assets, API responses
- Reduces server load

**Output Caching:**
- Caches entire HTTP response
- ASP.NET Core Output Caching middleware
- Use: expensive to generate, rarely changing responses

---

## Question 3: What is connection pooling? How does it work in ADO.NET and EF Core?

**Answer:**

Connection pooling reuses database connections instead of creating new ones for each request:

**How it works:**
- Pool maintains minimum and maximum connections
- When connection requested, reuse from pool if available
- If pool exhausted, create new connection up to max
- Connections returned to pool when disposed

**ADO.NET:**
```csharp
// In connection string
"Server=myserver;Database=mydb;Max Pool Size=100;"
```

**EF Core:**
- Uses DbContext-scoped pooling automatically
- Connections pooled at ADO.NET level
- DbContext disposal returns connection to pool
- Don't make DbContext static or singleton

---

## Question 4: How do you optimize LINQ queries? What are common performance pitfalls?

**Answer:**

**Optimizations:**
- Use projections: `Select(new { ... })` not full entity
- Use `AsNoTracking()` for read-only queries
- Use `ToList()` early if multiple enumerations needed
- Avoid multiple `.Include()` - use split queries
- Use compiled queries for hot paths

**Common pitfalls:**
- N+1 queries (lazy loading)
- Client-side evaluation: `.Where()` after `.ToList()`
- Multiple enumerations of IQueryable
- Missing indexes on queried columns
- Not using pagination

---

## Question 5: Explain async/await best practices. What are common mistakes developers make?

**Answer:**

**Best practices:**
- Use async/await for I/O-bound operations (not CPU-bound)
- Use `ConfigureAwait(false)` in library code
- Use proper return types: `Task`, `Task<T>`, `ValueTask<T>`
- Prefer async over blocking threads

**Common mistakes:**
- Using `Task.Run()` for I/O-bound work (wastes thread pool)
- Using `.Result` or `.Wait()` (causes deadlocks)
- `async void` except for event handlers
- Not awaiting async methods (fire-and-forget)
- Not handling exceptions in async code

---

## Question 6: What is the difference between Task.Run() and async/await? When should you use each?

**Answer:**

**Task.Run():**
- Spans work to thread pool
- For CPU-bound work that shouldn't block current thread
- Actually uses a thread (resource cost)
```csharp
await Task.Run(() => Compute());
```

**async/await:**
- Non-blocking wait for I/O operations
- Doesn't use extra thread
```csharp
await httpClient.GetAsync(url);
```

**When to use:**
- **Task.Run**: CPU-intensive work that would block UI/request thread
- **async/await**: HTTP calls, database queries, file I/O, any waiting
- Never use `Task.Run` to "make things async" - it's not async

---

## Question 7: Explain memory profiling in .NET. What are common memory issues and how to detect them?

**Answer:**

**Common memory issues:**
- Memory leaks: objects not released (event handlers, static references)
- Large Object Heap (LOH) allocations: objects > 85KB
- Excessive boxing: value types as objects
- Unbounded caches

**Detection tools:**
- **dotMemory** - visual memory profiling
- **ANTS Memory Profiler** - simple interface
- **PerfView** - detailed, free, powerful
- **Debug diagnostics** - for production issues

**Techniques:**
- Compare memory snapshots
- Monitor GC.CollectionCount for each generation
- Use MemoryCache metrics
- Check for Gen 2 collections increasing

---

## Question 8: What is the difference between GC collection modes? When do you tune GC settings?

**Answer:**

**Workstation GC (default):**
- Optimized for client apps
- Lower memory usage
- More frequent collections
- Single thread for Gen 0/1

**Server GC:**
- Optimized for server throughput
- Higher memory, better scalability
- Parallel collection
- Multiple threads

**When to tune:**
- High-throughput services (use Server GC)
- Low-latency requirements (use Latency mode)
- Memory-constrained environments
- Large heap applications

```csharp
GCSettings.LatencyMode = GCLatencyMode.Batch; // For server
```

---

## Question 9: Explain how to use Span<T> and Memory<T> for high-performance scenarios.

**Answer:**

**Span<T>:**
- Stack-only, ref struct (no heap allocation)
- Zero-copy slicing of arrays, strings
- Use for parsing, data processing
```csharp
Span<byte> data = stackalloc byte[1024];
var slice = data.Slice(start, length);
```

**Memory<T>:**
- Similar to Span<T> but for async scenarios
- Can be stored in fields
- Use for async buffer management

**Benefits:**
- Avoid allocations in hot paths
- Efficient parsing without string allocations
- Better performance for high-throughput scenarios
- Available in .NET Core 2.1+

---

## Question 10: What is ValueTask vs Task? When should you use ValueTask<T>?

**Answer:**

**Task<T>:**
- Class, allocated on heap
- Always available for async operations

**ValueTask<T>:**
- Struct, stack-allocated
- Avoids allocation when completes synchronously
- Only use when you often complete synchronously

**When to use:**
- Use `ValueTask<T>` for hot paths where allocation matters
- Don't use for long-running async operations
- Don't use as general replacement for Task
- Good for sync-over-async patterns

```csharp
public ValueTask<int> GetAsync() {
    if (cached) return new ValueTask<int>(value); // sync
    return new ValueTask<int>(FetchAsync());
}
```

---

## Question 11: How does the thread pool work in .NET? How do you tune it?

**Answer:**

**Thread Pool:**
- Maintains pool of worker threads
- Grows on demand based on work
- Reuses threads to avoid creation overhead
- Threads timeout and disappear if idle

**Tuning:**
```csharp
// Set minimum threads (pre-warm)
ThreadPool.SetMinThreads(50, 50);

// But be careful - can cause issues
```

**When to consider:**
- For I/O-bound work, thread pool handles scaling well
- For CPU-bound work with high parallelism, consider alternatives
- Don't manually create threads - use async or ThreadPool
- Monitor queue length for tuning

---

## Question 12: Explain parallel processing in .NET: Parallel.ForEach, PLINQ, Channels.

**Answer:**

**Parallel.ForEach:**
- Parallelizes for loops
- Partition work across threads
```csharp
Parallel.ForEach(items, item => Process(item));
```

**PLINQ:**
- Parallel LINQ queries
- Parallelize data transformation
```csharp
var results = items.AsParallel()
    .Where(x => Filter(x))
    .Select(x => Transform(x))
    .ToList();
```

**Channels:**
- Async producer-consumer pattern
- For pipeline processing
```csharp
var channel = Channel.CreateBounded<Data>(10);
await channel.Writer.WriteAsync(data);
```

**Choose based on:** Simple loops (Parallel.ForEach), data transformation (PLINQ), async pipelines (Channels).

---

## Question 13: What is the difference between lock, Monitor, Mutex, Semaphore, and ReaderWriterLockSlim?

**Answer:**

**lock/Monitor:**
- Simplest, mutual exclusion
- For single-process, low contention

**Mutex:**
- Named, can be system-wide
- For cross-process synchronization
```csharp
var mutex = new Mutex(false, "MyMutex");
mutex.WaitOne();
```

**Semaphore:**
- Limits concurrent access
- For resource pooling
```csharp
var semaphore = new Semaphore(3, 3);
```

**ReaderWriterLockSlim:**
- Allows multiple readers OR single writer
- For read-heavy scenarios

**Choose:** Based on scope (single/cross-process) and read/write pattern.

---

## Question 14: How do you implement request/response compression in ASP.NET Core?

**Answer:**

**Enable Response Compression:**
```csharp
builder.Services.AddResponseCompression(options => {
    options.EnableForHttps = true;
    options.MimeTypes = new[] { "application/json" };
});

app.UseResponseCompression();
```

**Compression providers:**
- Brotli (best compression)
- Gzip (widest compatibility)
- Deflate

**Best practice:** Use Brotli, enable for HTTPS. Compress API responses, static files. Don't compress already-compressed formats (images, zip).

---

## Question 15: What are the best practices for working with large collections in .NET?

**Answer:**

- Use pagination: `Skip(page * size).Take(size)`
- Use yield for streaming results
- Avoid loading entire collection into memory
- Use `HashSet<T>` for O(1) lookups
- Consider streaming APIs instead of collections
- Use database-level filtering, not in-memory
- Consider immutable collections for thread safety
- Use blockingCollection for producer-consumer

---

## Question 16: Explain how to use BenchmarkDotNet for reliable benchmarking.

**Answer:**

**Setup:**
```csharp
[MemoryDiagnoser]
public class MyBenchmarks {
    [Benchmark]
    public void MyMethod() { }
}

// Run
var summary = BenchmarkRunner.Run<MyBenchmarks>();
```

**Best practices:**
- Run in Release mode, outside IDE
- Use proper iterations (throughput mode)
- Warmup iterations included automatically
- Don't measure debug builds
- Use params for different inputs

**Features:** Provides mean, median, standard deviation, memory allocation stats.

---

## Question 17: What is the difference between heap and stack? How does this affect performance?

**Answer:**

**Stack:**
- Fast allocation (just move stack pointer)
- Limited size (~1MB per thread)
- Automatic cleanup (pop on method exit)
- Stores value types, references

**Heap:**
- Slower allocation (managed by GC)
- Larger, managed by garbage collector
- GC has overhead
- Stores reference types

**Performance impact:**
- Stack allocation nearly free
- Heap allocation costs (GC, possible compaction)
- Use struct for small, short-lived data to avoid GC
- Avoid boxing (value → reference conversion)

---

## Question 18: How do you handle database connection timeouts and retry logic?

**Answer:**

**Connection timeouts:**
- Set in connection string: `Connection Timeout=30`
- Use appropriate timeout for workload
- Handle timeout exceptions gracefully

**Retry logic with Polly:**
```csharp
var retryPolicy = Policy
    .Handle<SqlException>()
    .WaitAndRetryAsync(3, retryAttempt => 
        TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));

await retryPolicy.ExecuteAsync(() => db.QueryAsync());
```

**Circuit breaker:**
- After failures, stop trying (don't overload)
- After timeout, test if recovered

---

## Question 19: What is the difference between eager loading, lazy loading, and explicit loading performance implications?

**Answer:**

**Eager Loading:**
- Loads related data in initial query
- Single round trip when data needed is known
- Use `.Include()`
```csharp
var orders = ctx.Orders.Include(o => o.Customer).ToList();
```

**Lazy Loading:**
- Loads related data on first access
- Causes N+1 queries if not careful
- Use carefully or disable

**Explicit Loading:**
- Manually load related data when needed
- Gives control over timing
```csharp
ctx.Entry(order).Reference(o => o.Customer).Load();
```

**Performance:** Eager is best when you know what you need. Lazy convenient but problematic. Explicit gives control.

---

## Question 20: How do you identify and resolve memory leaks in .NET applications?

**Answer:**

**Identification:**
- Use memory profiler (dotMemory,ANTS)
- Compare snapshots over time
- Monitor GC.CollectionCount()
- Look for Gen 2 growing

**Common causes:**
- Event handlers not unsubscribed
- Static references holding objects
- Caches growing unbounded
- Finalizer not running (P/Invoke)
- Captured variables in lambdas

**Resolution:**
- Unsubscribe events
- Use WeakReference for caches
- Clear cache entries
- Dispose properly
- Use tools to verify

---

*End of Section 7: Performance & Optimization*

---

## 🎯 Quick Revision
- Focus on WHY + HOW  
- Practice explaining verbally  
- Think in real-world scenarios  

---

## 🎤 Interview Tip
> Don’t just answer — explain trade-offs and real-world usage.
