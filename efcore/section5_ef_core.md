# 🚀 Section5 Ef Core — Interview Master Guide

> 🧠 Styled for fast revision + deep understanding

---

## 📌 Overview
# .NET Interview Preparation - Section 5: Entity Framework Core

## 13 Years Experience Candidate Answers

---

## Question 1: Explain the DbContext lifecycle in EF Core. When is it created and disposed?

**Answer:**

**DbContext lifecycle:**

```csharp
// Registration
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

// Usage in controller
public class UserController : Controller {
    private readonly AppDbContext _context;
    
    public UserController(AppDbContext context) => _context = context;
}
```

**Creation and disposal:**
1. **Created** - When request starts (for scoped) or when injected
2. **Active** - Throughout the request/scope
3. **Disposed** - At end of scope (request completion)

**Scoped lifetime (recommended for web apps):**
- One DbContext per HTTP request
- Tracks all changes during request
- Commits at end if successful

**Important for 13 years:**
- Don't create DbContext per operation (connection pool exhaustion)
- Don't make DbContext Singleton (tracking issues)
- Always dispose - use using or scoped lifetime
- DbContext is NOT thread-safe - don't share across threads

---

## Question 2: What is the difference between AsNoTracking(), AsNoTrackingWithIdentityResolution(), and default tracking?

**Answer:**

**Default tracking:**
- EF tracks entity changes for SaveChanges
- Use when you need to update entities
- Slower (memory, change detection overhead)

**AsNoTracking():**
```csharp
var users = context.Users.AsNoTracking().ToList();
```
- No change tracking - read-only scenarios
- Faster (no snapshot, no change detection)
- Good for display, reporting, cloning

**AsNoTrackingWithIdentityResolution():**
```csharp
var users = context.Users
    .AsNoTrackingWithIdentityResolution()
    .Include(u => u.Orders)
    .ToList();
```
- No tracking BUT maintains identity within query
- Use when loading related entities that might be same instance
- Slower than AsNoTracking but avoids duplicate objects

**When to use:**
- Default: When updating/deleting
- AsNoTracking: Simple reads
- AsNoTrackingWithIdentityResolution: Reads with includes where identity matters

---

## Question 3: How does EF Core change tracking work? Explain the entity states.

**Answer:**

**Entity states:**

| State | Description |
|-------|-------------|
| **Detached** | Not tracked by context |
| **Added** | New entity, will be inserted on SaveChanges |
| **Unchanged** | Tracked, not modified |
| **Modified** | Tracked, changed, will be updated |
| **Deleted** | Tracked, will be deleted |

**How tracking works:**
```csharp
var user = new User { Id = 1, Name = "John" };
context.Users.Add(user); // Added state
context.SaveChanges(); // SQL INSERT, becomes Unchanged

user.Name = "Jane"; // Modified state
context.SaveChanges(); // SQL UPDATE

context.Users.Remove(user); // Deleted state
context.SaveChanges(); // SQL DELETE
```

**Change detection:**
- Snapshot-based - original values stored
- Compares current vs original on SaveChanges
- Can manually call context.ChangeTracker.DetectChanges()

---

## Question 4: What is the N+1 query problem? How do you solve it in EF Core?

**Answer:**

**N+1 Problem:**
```csharp
// This executes 1 + N queries!
var orders = context.Orders.ToList();
foreach (var order in orders) {
    Console.WriteLine(order.Customer.Name); // Each access = query
}
```
- 1 query for orders
- N queries for each customer

**Solutions:**

**1. Eager Loading (Include):**
```csharp
var orders = context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
    .ToList();
```

**2. Split Queries (.AsSplitQuery()):**
```csharp
var orders = context.Orders
    .Include(o => o.Customer)
    .AsSplitQuery() // Separate queries per include
    .ToList();
```

**3. Explicit Loading:**
```csharp
var order = context.Orders.First();
context.Entry(order).Reference(o => o.Customer).Load();
```

**4. Projections (Best for read-only):**
```csharp
var orders = context.Orders
    .Select(o => new { o.Id, CustomerName = o.Customer.Name })
    .ToList();
```

---

## Question 5: Explain the difference between Include(), ThenInclude(), and Load().

**Answer:**

**Include():**
- Eager loads related data in single query
```csharp
var orders = context.Orders
    .Include(o => o.Customer) // Load Customer with each Order
    .ToList();
```

**ThenInclude():**
- Loads nested related data
```csharp
var orders = context.Orders
    .Include(o => o.Customer)
        .ThenInclude(c => c.Address) // Load Address of Customer
    .Include(o => o.Items)
        .ThenInclude(i => i.Product) // Load Product of each Item
    .ToList();
```

**Load():**
- Explicit loading - loads on demand (after initial query)
```csharp
var order = context.Orders.First();

// Explicitly load customer
context.Entry(order).Reference(o => o.Customer).Load();

// Explicitly load items collection
context.Entry(order).Collection(o => o.Items).Load();
```

**Key differences:**

| Method | When loads | Query count |
|--------|-----------|-------------|
| Include | With initial query | 1 (or split) |
| Load | After initial query | Separate per call |

---

## Question 6: What is lazy loading vs eager loading vs explicit loading? How do you configure each?

**Answer:**

**Eager Loading:**
- Load related data with main query
```csharp
context.Orders.Include(o => o.Customer).ToList();
```
- Best for known data needs

**Lazy Loading:**
- Load related data on first access
- Requires Microsoft.EntityFrameworkCore.Proxies or ILazyLoader
```csharp
builder.UseLazyLoadingProxies();
```
- Easy but can cause N+1

**Explicit Loading:**
- Load related data manually on demand
```csharp
context.Entry(order).Reference(o => o.Customer).Load();
```
- Most control, explicit about loading

**Configuration:**
```csharp
// In DbContext OnConfiguring
protected override void OnConfiguring(DbContextOptionsBuilder options) {
    options.UseLazyLoadingProxies()
           .UseSqlServer(connectionString);
}
```

**Recommendations:**
- Eager loading: Default for known relationships
- Avoid lazy loading in web apps (serialization issues)
- Explicit loading: When conditional loading needed

---

## Question 7: How do migrations work in EF Core? Explain the commands and how to handle schema changes.

**Answer:**

**Migration commands:**

```bash
# Add migration (creates migration files)
dotnet ef migrations add InitialCreate

# Apply migrations to database
dotnet ef database update

# Remove last migration
dotnet ef migrations remove

# Generate SQL script
dotnet ef migrations script

# List migrations
dotnet ef migrations list
```

**How it works:**
1. **Scaffold** - Compares model vs database, generates operations
2. **Up/Down** - Each migration has Up() and Down()
3. **Apply** - Executes Up() to apply, Down() to rollback

**Creating migration:**
```csharp
public partial class AddOrdersTable : Migration {
    protected override void Up(MigrationBuilder migrationBuilder) {
        migrationBuilder.CreateTable("Orders", ...);
    }
    
    protected override void Down(MigrationBuilder migrationBuilder) {
        migrationBuilder.DropTable("Orders");
    }
}
```

**Schema changes:**
- Add migration after model changes
- Never modify existing migrations
- Use [Column(Order = N)] for column ordering
- Use fluent API or data annotations for config

---

## Question 8: What is the difference between Add(), Attach(), and Update() in EF Core?

**Answer:**

**Add():**
```csharp
context.Users.Add(new User { Name = "John" });
// State: Added - will INSERT on SaveChanges
```
- For new entities that don't exist in database
- Sets state to Added

**Attach():
```csharp
var user = new User { Id = 1, Name = "John" };
context.Users.Attach(user);
// State: Unchanged - but if modified, will UPDATE
```
- Attaches existing entity to context
- Sets state to Untracked
- Use when you know entity exists but context doesn't know

**Update():
```csharp
var user = new User { Id = 1, Name = "John" };
context.Users.Update(user);
// State: Modified - will UPDATE on SaveChanges
```
- Attaches entity and sets to Modified
- Use when you have detached entity you know exists and want to update

**Key difference:**
- Add: New entity - INSERT
- Attach: Existing, unchanged - nothing (or track changes)
- Update: Existing, changed - UPDATE

---

## Question 9: How do you handle concurrency conflicts in EF Core? Explain optimistic vs pessimistic concurrency.

**Answer:**

**Optimistic Concurrency (default):**
- Assume conflicts are rare
- Check version on save, fail if changed

**Version property:**
```csharp
public class Product {
    public int Id { get; set; }
    public string Name { get; set; }
    [Timestamp] // RowVersion
    public byte[] RowVersion { get; set; }
}

// On conflict - throws DbUpdateConcurrencyException
try {
    context.SaveChanges();
}
catch (DbUpdateConcurrencyException ex) {
    // Handle - reload and retry
}
```

**Pessimistic Concurrency:**
- Lock records before editing
- Use for critical operations

```csharp
using var context = new AppDbContext();
var product = context.Products
    .FromSqlRaw("SELECT * FROM Products WITH (UPDLOCK) WHERE Id = {0}", id)
    .First();
// Lock held until context disposed
```

**Choosing:**
- Optimistic: Most cases (default in EF Core)
- Pessimistic: Long-running transactions, critical financial data

---

## Question 10: What are shadow properties in EF Core? When would you use them?

**Answer:**

**Shadow properties:**
- Properties not in entity class but tracked by EF
- Stored in change tracker, not in .NET object

**Defining:**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder) {
    modelBuilder.Entity<Order>().Property<DateTime>("CreatedDate");
    modelBuilder.Entity<Order>().Property<string>("CreatedBy");
}
```

**Using:**
```csharp
// Set value
context.Entry(order)["CreatedDate"] = DateTime.UtcNow;
context.Entry(order)["CreatedBy"] = "User1";

// Query with shadow properties
var orders = context.Orders
    .Where(e => EF.Property<string>(e, "CreatedBy") == "User1")
    .ToList();
```

**When to use:**
- Properties not part of domain model (audit fields)
- Decouple from domain (metadata, external IDs)
- When you don't want to expose in entity class

**13-year insight:** Useful for cross-cutting concerns like audit, soft delete, but consider if they belong in domain.

---

## Question 11: Explain the different concurrency token implementations in EF Core.

**Answer:**

**1. Timestamp/RowVersion:**
```csharp
public class Product {
    [Timestamp]
    public byte[] RowVersion { get; set; }
}
```

**2. ConcurrencyCheck attribute:**
```csharp
public class Product {
    [ConcurrencyCheck]
    public string Name { get; set; }
}
```

**3. Fluent API:**
```csharp
modelBuilder.Entity<Product>()
    .Property(p => p.Name)
    .IsConcurrencyToken();
```

**How they work:**
- RowVersion: Automatic, uses WHERE version = original
- ConcurrencyCheck: Manual, adds to WHERE clause
- On conflict: DbUpdateConcurrencyException

**Handling conflicts:**
```csharp
try {
    context.SaveChanges();
}
catch (DbUpdateConcurrencyException ex) {
    var entry = ex.Entries.Single();
    var current = entry.CurrentValues.GetValue<object>("Name");
    var original = entry.OriginalValues.GetValue<object>("Name");
    // Resolve - reload, merge, or reject
}
```

---

## Question 12: How does EF Core compile queries? What is the compiled query feature?

**Answer:**

**Query compilation:**
1. **Expression tree** - LINQ converted to expression tree
2. **Translation** - Expression tree - SQL
3. **Cached** - Compiled query plan cached
4. **Execution** - Parameterized SQL executed

**Compiled queries:**
```csharp
// Define compiled query
private static readonly Func<AppDbContext, int, User> 
    _getUserById = EF.CompileAsyncQuery((AppDbContext ctx, int id) =>
        ctx.Users.FirstOrDefault(u => u.Id == id));

// Use
var user = await _getUserById(context, 1);
```

**Benefits:**
- Skip LINQ parsing/compilation on each call
- Faster for frequently executed queries
- Best for hot paths

**When to use:**
- High-frequency queries
- Complex queries
- When profiling shows compilation overhead

**Note:** EF Core caches queries by default. Compiled queries are for extreme optimization.

---

## Question 13: What is the difference between split queries and single query in EF Core?

**Answer:**

**Single query (default with Include):**
```csharp
var orders = context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
    .ToList();
// Generates one SQL with JOINs - may duplicate data
```

**Split queries:**
```csharp
var orders = context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
    .AsSplitQuery() // Separate queries
    .ToList();
// Generates: Orders query, Customer query, Items query
```

**Differences:**

| Aspect | Single Query | Split Queries |
|--------|-------------|---------------|
| SQL | One with JOINs | Multiple (one per include) |
| Data | Duplicated rows | No duplication |
| Memory | Higher for large datasets | More queries, less memory |
| Performance | Good for small/medium | Better for large with many relations |

**Configuration global:**
```csharp
builder.Services.AddDbContext(options =>
    options.UseSqlServer(connectionString, 
        o => o.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)));
```

---

## Question 14: How do you optimize EF Core queries for performance? Explain query profiling techniques.

**Answer:**

**1. Use projections:**
```csharp
// Bad
var users = context.Users.ToList();
var names = users.Select(u => u.Name).ToList();

// Good - only selects needed columns
var names = context.Users.Select(u => u.Name).ToList();
```

**2. Choose correct loading:**
```csharp
// No tracking for read-only
var users = context.Users.AsNoTracking().ToList();

// Eager load known needs
var orders = context.Orders.Include(o => o.Customer).ToList();
```

**3. Avoid client-side evaluation:**
```csharp
// Bad - filters in memory
var orders = context.Orders.ToList()
    .Where(o => o.Date > DateTime.Now.AddDays(-7));

// Good - filters in database
var orders = context.Orders
    .Where(o => o.Date > DateTime.Now.AddDays(-7))
    .ToList();
```

**Profiling:**
- **Logging** - options.LogTo(Console.WriteLine)
- **EF Core Profiler** - NuGet package
- **SQL Server Profiler** - Raw SQL
- **Analyze** - ToQueryString() for generated SQL

```csharp
var sql = context.Orders
    .Where(o => o.Status == OrderStatus.Active)
    .ToQueryString();
```

---

## Question 15: What are interpolated strings (raw SQL) vs stored procedures vs raw SQL with parameters?

**Answer:**

**Interpolated strings (FromSqlInterpolated):**
```csharp
var id = 1;
var user = context.Users
    .FromSqlInterpolated($"SELECT * FROM Users WHERE Id = {id}")
    .FirstOrDefault();
```
- Parameterized automatically
- Safe from SQL injection

**Stored procedures:**
```csharp
var users = context.Users
    .FromSqlRaw("EXEC GetUsers @p1", 
        new SqlParameter("@p1", 1))
    .ToList();
```
- Precompiled in database
- Good for complex logic, security
- Less flexible

**Raw SQL with parameters:**
```csharp
var param = new SqlParameter("@name", "John");
var users = context.Users
    .FromSqlRaw("SELECT * FROM Users WHERE Name = @name", param)
    .ToList();
```

**Choosing:**
- Interpolated: Simple queries, dynamic
- Stored procedures: Complex, frequently used, security
- Raw SQL: When LINQ insufficient

---

## Question 16: Explain the different loading strategies: collection navigation vs reference navigation.

**Answer:**

**Reference navigation (one-to-one, many-to-one):**
```csharp
public class Order {
    public int CustomerId { get; set; }
    public Customer Customer { get; set; }
}

// Loading
var order = context.Orders.First();
var customer = order.Customer; // Reference loaded separately
```

**Collection navigation (one-to-many, many-to-many):**
```csharp
public class Customer {
    public ICollection<Order> Orders { get; set; }
}

// Loading
var customer = context.Customers.First();
var orders = customer.Orders; // Collection - separate query or loaded
```

**Loading methods:**
```csharp
// Reference - explicit load
context.Entry(order).Reference(o => o.Customer).Load();

// Collection - explicit load  
context.Entry(customer).Collection(c => c.Orders).Load();

// Check if loaded
var isLoaded = context.Entry(order).Reference(o => o.Customer).IsLoaded;
```

**Key:** Reference navigation always returns single entity, collection returns ICollection<T>.

---

## Question 17: How do you implement soft delete in EF Core? What are the approaches?

**Answer:**

**Approach 1: Query filters (global):**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder) {
    modelBuilder.Entity<User>().HasQueryFilter(u => !u.IsDeleted);
}

// Bypass filter for admin/reconciliation
var users = context.Users.IgnoreQueryFilters().ToList();
```

**Approach 2: Entity intercepts:**
```csharp
public override int SaveChanges() {
    foreach (var entry in ChangeTracker.Entries()) {
        if (entry.State == EntityState.Deleted) {
            entry.State = EntityState.Modified;
            ((BaseEntity)entry.Entity).IsDeleted = true;
        }
    }
    return base.SaveChanges();
}
```

**Approach 3: Separate table:**
- Move deleted records to archive table
- More complex but true deletion from main table

**Query considerations:**
- Auto-filter catches all reads
- Use IgnoreQueryFilters() for admin operations
- Consider performance for large datasets

---

## Question 18: What is the difference between owned types and complex types in EF Core?

**Answer:**

**Owned types:**
```csharp
public class Order {
    public Address ShippingAddress { get; set; }
    public Address BillingAddress { get; set; }
}

public class Address {
    public string Street { get; set; }
    public string City { get; set; }
}

// Configure as owned
modelBuilder.Entity<Order>().OwnsOne(o => o.ShippingAddress);
modelBuilder.Entity<Order>().OwnsOne(o => o.BillingAddress);
```
- Owned entity type - saved within same table
- Multiple instances per entity (like BillingAddress, ShippingAddress)
- EF Core 2.0+

**Complex types (prior EF Core):**
- Now replaced by owned types
- Same concept but older terminology

**Table structure for Order with owned Address:**
```
Orders table: Id, ShippingAddress_Street, ShippingAddress_City, BillingAddress_Street, BillingAddress_City
```

**When to use:** Value objects, embedded objects, address, money, dimensions.

---

## Question 19: How do you handle database connection pooling in EF Core?

**Answer:**

**Connection pooling (enabled by default in SQL Server):**
```csharp
// In connection string
"Server=.;Database=App;Trusted_Connection=True;Connection Timeout=30;Max Pool Size=100"

// In code - doesn't affect pooling
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

**Pool management:**
- Default max pool size: 100
- Connections reused from pool
- Disposing context returns connection to pool

**Disabling pool (rare cases):**
```csharp
options.UseSqlServer(connectionString, options => 
    options.EnablePooling(false));
```

**Best practices:**
- Always dispose DbContext properly
- Don't create new connection strings
- Pool handles concurrent requests
- Monitor with SqlConnection events for issues

**For high-scale:**
- Adjust Max Pool Size for high concurrency
- Consider Connection Pooling=false for serverless (Azure SQL)

---

## Question 20: What are global query filters in EF Core? How do they work?

**Answer:**

**Global query filters:**
- Apply automatically to all LINQ queries
- Like automatic Where clause

**Definition:**
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder) {
    // Soft delete filter
    modelBuilder.Entity<User>().HasQueryFilter(u => !u.IsDeleted);
    
    // Tenant filter (multi-tenant)
    modelBuilder.Entity<Order>().HasQueryFilter(o => o.TenantId == _tenantId);
}
```

**How they work:**
- Applied to all queries (Include, navigation, etc.)
- Can be ignored with IgnoreQueryFilters()
- For non-parameterized filters, need to use FromExpression

**Use cases:** Soft delete, multi-tenancy, active/inactive status, security filtering.

---

*End of Section 5: Entity Framework Core*

---

## 🎯 Key Takeaways
- Revise important concepts quickly
- Focus on interview-ready answers
- Practice explaining in your own words

---

## 🎤 Interview Tip
> Always explain **WHY + HOW**, not just definitions.
