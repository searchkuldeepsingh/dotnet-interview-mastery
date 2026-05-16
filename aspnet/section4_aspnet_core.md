# 🚀 Section4 Aspnet Core — Interview Master Guide

> 🧠 Styled for fast revision + deep understanding

---

## 📌 Overview
# .NET Interview Preparation - Section 4: ASP.NET Core

## 13 Years Experience Candidate Answers

---

## Question 1: Explain the ASP.NET Core middleware pipeline. How does a middleware component work?

**Answer:**

Middleware is software pipeline that handles requests/responses:

```csharp
public class RequestPipeline {
    public async Task InvokeAsync(HttpContext context) {
        // Do work before next middleware
        Console.WriteLine($"Before: {context.Request.Path}");
        
        await _next(context); // Call next middleware
        
        // Do work after next middleware completes (response phase)
        Console.WriteLine($"After: {context.Response.StatusCode}");
    }
}
```

**How middleware works:**
- Each middleware receives HttpContext and RequestDelegate next
- Can decide to pass to next or short-circuit (not call next)
- Can modify request before and response after next
- Pipeline constructed in Program.cs with Use, Run, Map

**Key points for 13 years:**
- Middleware executes in order added
- Use app.Use() for middleware that calls next
- Use app.Run() for terminal middleware (doesn't call next)
- app.UseWhen() for conditional middleware branches

---

## Question 2: What is the difference between app.Use() and app.Run() in ASP.NET Core?

**Answer:**

**app.Use():**
- Adds middleware to pipeline that can optionally call next middleware
- Middleware receives HttpContext and can continue pipeline or short-circuit
```csharp
app.Use(async (context, next) => {
    // Can call next or short-circuit
    if (context.Request.Path.StartsWith("/api"))
        await next(); // Continue
    else
        await context.Response.WriteAsync("Not API"); // Short-circuit
});
```

**app.Run():**
- Terminal middleware - doesn't accept next parameter
- Pipeline ends here, no more middleware runs after
```csharp
app.Run(async context => {
    await context.Response.WriteAsync("Hello");
    // No next - pipeline ends
});
```

**Key difference:** Use can call next (continues pipeline), Run is terminal (ends pipeline).

---

## Question 3: Explain the ASP.NET Core request lifecycle from incoming request to response.

**Answer:**

**Request Lifecycle:**

1. **Kestrel receives request** - Low-level HTTP server listens
2. **Create HttpContext** - Request, Response, Features created
3. **Middleware pipeline starts** - First middleware receives context
4. **Middleware execution** - Each middleware processes request
5. **Routing** - Endpoint routing matches URL to route
6. **Controller/Endpoint selection** - MVC or minimal API handler chosen
7. **Model Binding** - Route/data/querystring bound to parameters
8. **Authorization** - Authorization filters run
9. **Action execution** - Controller action runs, business logic
10. **Result execution** - IActionResult produces response
11. **Exception handling** - If error, exception middleware handles
12. **Response middleware** - Response headers/body written
13. **Response sent** - Kestrel sends response to client

**Important:** Middleware runs before and after MVC. Use UseWhen for conditional branching. Filters provide MVC-specific pipeline.

---

## Question 4: How does Dependency Injection work in ASP.NET Core? Explain the request pipeline integration.

**Answer:**

**DI in ASP.NET Core:**

```csharp
// Register services
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddSingleton<ILogger, Logger>();

// Use in controller
public class UsersController : Controller {
    private readonly IUserService _userService;
    
    public UsersController(IUserService userService) {
        _userService = userService;
    }
}
```

**Request pipeline integration:**

1. **Service registration** - Program.cs with builder.Services.Add...
2. **Service provider** - Built at app start, manages lifecycle
3. **Request scope** - For scoped services, new instance per request
4. **Middleware injection** - Middleware can receive services via constructor
5. **Controller resolution** - DI container creates controllers with dependencies

**How it works:**
- IServiceProvider holds registered services
- HttpContext.RequestServices exposes request-scoped provider
- Scoped: New instance per HTTP request
- Transient: New instance every time
- Singleton: One instance for app lifetime

**For 13 years:** DI is fundamental to ASP.NET Core testability and Clean Architecture.

---

## Question 5: What is the difference between IActionResult, ActionResult<T>, and direct return types in ASP.NET Core?

**Answer:**

**IActionResult:**
- Base interface for all action results
- Generic return type, no compile-time type safety
```csharp
public IActionResult Get() => Ok(); // Can return any result type
```

**ActionResult<T>:**
- Type-safe wrapper around IActionResult
- Allows returning T or ActionResult<T>
```csharp
public ActionResult<User> Get() {
    if (user == null) return NotFound();
    return Ok(user);
}
```

**Direct return types (C# 7+):**
- Most concise, automatic result wrapping
```csharp
public User Get() => _service.GetUser();
public async Task<User> Get() => await _service.GetUserAsync();
```

**Comparison:**

| Return Type | Use When |
|------------|----------|
| IActionResult | Returning multiple types, complex logic |
| ActionResult<T> | Single type with error handling |
| Direct type | Simple, clean, always returns T |

**Best practice:** Use direct return types (C# 7+) for simplicity. Use ActionResult<T> when you need both T and error responses in same method.

---

## Question 6: Explain model binding in ASP.NET Core. How does it work for complex types?

**Answer:**

Model binding maps request data to action parameters:

**Simple types (from route, query, header):**
```csharp
public IActionResult Get(int id, string name) { } // id from route, name from query
```

**Complex types:**
```csharp
public IActionResult Create([FromBody] UserDto user) { } // from request body
public IActionResult Edit([FromForm] UserDto user) { } // from form data
public IActionResult Details([FromRoute] int id) { } // from route
```

**How it works:**
1. Binder identifies source (route, query, body, form)
2. Searches for property values using naming convention
3. Simple types use TypeConverter or TryParse
4. Complex types use property-based binding
5. Validation runs after binding

**Complex type binding:**
```csharp
// POST /api/users/5?name=John&Address.City=Boston
public IActionResult Create(User user) { }
// user.Id = 5, user.Name = "John", user.Address.City = "Boston"
```

**Custom binding:** Implement IModelBinder for complex scenarios.

---

## Question 7: What is the difference between routing in MVC vs minimal APIs?

**Answer:**

**MVC Routing:**
```csharp
// Convention-based
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

// Attribute-based
[Route("api/[controller]")]
[ApiController]
public class UsersController : ControllerBase {
    [HttpGet("{id}")]
    public IActionResult Get(int id) => Ok();
}
```

**Minimal APIs:**
```csharp
app.MapGet("/users/{id}", (int id, IUserService service) => 
    service.GetUser(id));

app.MapPost("/users", (UserDto user, IUserService service) => 
    service.Create(user));
```

**Key differences:**

| Aspect | MVC | Minimal APIs |
|--------|-----|--------------|
| Setup | Controllers, routing | Direct lambda/map methods |
| Model binding | Controller parameters, filters | Lambda parameters |
| Validation | DataAnnotations, FluentValidation | Manual or [FromBody] attributes |
| Filters | Full pipeline | Limited |
| OpenAPI | Automatic | Manual/codegen |

**When to use:**
- MVC: Enterprise apps, complex operations, full features
- Minimal APIs: Microservices, simple endpoints, performance-critical

---

## Question 8: How does JWT authentication work in ASP.NET Core? Explain the token flow.

**Answer:**

**JWT (JSON Web Token) Flow:**

1. **Client sends credentials** (username/password)
2. **Server validates and generates JWT** (access token)
3. **Client includes JWT in Authorization header**
4. **Server validates JWT on each request**

**Implementation:**

```csharp
// Generate token
var claims = new[] {
    new Claim(ClaimTypes.Name, user.Username),
    new Claim(ClaimTypes.Role, "Admin")
};
var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_secret));
var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
var token = new JwtSecurityToken(
    issuer: "MyApp",
    audience: "MyApp",
    claims: claims,
    expires: DateTime.Now.AddMinutes(30),
    signingCredentials: creds
);
return new JwtSecurityTokenHandler().WriteToken(token);
```

**Configure JWT:**
```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => {
        options.TokenValidationParameters = new TokenValidationParameters {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = "MyApp",
            ValidAudience = "MyApp",
            IssuerSigningKey = key
        };
    });
```

**Client request:** GET /api/data Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

---

## Question 9: What is the difference between Authorize attribute roles vs policies vs schemes?

**Answer:**

**Roles:**
```csharp
[Authorize(Roles = "Admin,Manager")]
public IActionResult AdminPanel() { }
```
- Simple role-based authorization
- User must have ANY of the listed roles

**Policies:**
```csharp
// Register policy
builder.Services.AddAuthorization(options => {
    options.AddPolicy("AdultOnly", policy => 
        policy.RequireClaim("Age", "18", "21", "25"));
    options.AddPolicy("Premium", policy => 
        policy.RequireAuthenticatedUser());
});

// Use
[Authorize(Policy = "AdultOnly")]
public IActionResult Restricted() { }
```
- More flexible, complex rules
- Can require claims, authenticate type, custom requirements

**Schemes:**
```csharp
[Authorize(AuthenticationSchemes = "Bearer,Jwt")]
public IActionResult Secure() { }
```
- Specifies which authentication scheme to use
- Multiple schemes = user can authenticate via any

**Use cases:**
- Roles: Simple, position-based access
- Policies: Complex rules, claims-based
- Schemes: Multiple auth methods (cookie + JWT)

---

## Question 10: Explain CORS in ASP.NET Core. How do you configure it properly?

**Answer:**

CORS (Cross-Origin Resource Sharing) allows cross-origin requests:

**Why needed:** Browsers block requests to different origin for security. CORS allows server to explicitly permit.

**Configuration:**

```csharp
// In Program.cs
builder.Services.AddCors(options => {
    options.AddPolicy("AllowReactApp", policy => {
        policy.WithOrigins("http://localhost:3000")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials(); // Required for credentials
    });
});

app.UseCors("AllowReactApp"); // Before UseAuthentication
app.UseAuthentication();
app.UseAuthorization();
```

**Common settings:**
- WithOrigins - Allowed origins (comma-separated or multiple calls)
- AllowAnyOrigin - Allow all (incompatible with AllowCredentials)
- AllowAnyHeader, AllowAnyMethod - Wildcard headers/methods
- AllowCredentials - Cookies, auth headers (requires specific origins)

**Important:** UseCors must be before UseAuthentication and UseAuthorization.

---

## Question 11: What is output caching in ASP.NET Core 7+? How does it differ from in-memory caching?

**Answer:**

**Output Caching:**
- Caches entire HTTP response based on route/query
- Reduces server processing for expensive operations

```csharp
// Enable output caching
builder.Services.AddOutputCache();

// Add caching to endpoint
app.MapGet("/products", async (AppDbContext db) => {
    var products = await db.Products.ToListAsync();
    return Results.Ok(products);
})
.CacheOutput(p => p.VaryByHeader("Accept")
                   .Expire(TimeSpan.FromSeconds(30)));
```

**In-Memory Caching:**
- Manual caching of data/objects in memory
- Programmer controls what and when to cache
```csharp
builder.Services.AddMemoryCache();
var cache = services.GetRequiredService<IMemoryCache>();
var data = cache.GetOrCreate("key", entry => expensiveOperation());
```

**Key differences:**

| Aspect | Output Caching | In-Memory Cache |
|--------|---------------|-----------------|
| Caches | HTTP response | Data/objects |
| Automatic | Yes | Manual |
| Vary by | Headers, query, route | Custom keys |
| Location | HTTP layer | Application layer |

**Use Output Cache for:** Stable responses, full page/endpoint caching. Use Memory Cache for: Data fragments, complex computation results.

---

## Question 12: Explain health checks in ASP.NET Core. How do you implement custom health checks?

**Answer:**

**Built-in health checks:**
```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<DbContext>("database")
    .AddUrlGroup(new Uri("https://api.external.com"), name: "external");
```

**Custom health check:**
```csharp
public class CustomHealthCheck : IHealthCheck {
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context, 
        CancellationToken cancellationToken = default) {
        
        try {
            // Check something
            var isHealthy = await CheckDatabaseAsync();
            return isHealthy 
                ? HealthCheckResult.Healthy("OK")
                : HealthCheckResult.Unhealthy("Failed");
        } catch (Exception ex) {
            return HealthCheckResult.Unhealthy(ex.Message);
        }
    }
}

// Register
builder.Services.AddHealthChecks()
    .AddCheck<CustomHealthCheck>("custom");
```

**Expose health endpoint:**
```csharp
app.MapHealthChecks("/health");
```

**Use cases:** Kubernetes liveness/readiness probes, load balancer health checks, monitoring systems.

---

## Question 13: What is the difference between AddSingleton, AddScoped, and AddTransient in ASP.NET Core?

**Answer:**

**AddSingleton:**
```csharp
services.AddSingleton<ILogger>(new Logger());
```
- One instance for entire application lifetime
- Created first time requested, reused everywhere
- Use for: Logging, configuration, caching

**AddScoped:**
```csharp
services.AddScoped<IUserService, UserService>();
```
- One instance per HTTP request/scope
- New instance per request, same instance within request
- Use for: DbContext, repositories, Unit of Work

**AddTransient:**
```csharp
services.AddTransient<IEmailService, EmailService>();
```
- New instance every time it's requested
- Never shared within or across requests
- Use for: Lightweight, stateless services

**Choosing:**
- **Singleton** if stateless and shared (logger, config)
- **Scoped** for per-request state (DbContext)
- **Transient** for stateless, lightweight (helpers, utilities)

**Important:** Never inject Scoped into Singleton (lifetime mismatch). Scoped into Scoped is fine. Transient into anything is fine.

---

## Question 14: How does exception handling work in ASP.NET Core? Explain UseExceptionHandler and exception filters.

**Answer:**

**UseExceptionHandler:**
```csharp
// Simple version
app.UseExceptionHandler("/error");

// Programmatic
app.UseExceptionHandler(errorApp => {
    errorApp.Run(async context => {
        context.Response.StatusCode = 500;
        await context.Response.WriteAsync("Error occurred");
    });
});
```

**Exception Filters (MVC):**
```csharp
public class GlobalExceptionFilter : IExceptionFilter {
    public void OnException(ExceptionContext context) {
        var logger = context.HttpContext.RequestServices
            .GetRequiredService<ILogger<GlobalExceptionFilter>>();
        
        logger.LogError(context.Exception, "Error");
        context.Result = new ObjectResult(new { error = "Error" }) {
            StatusCode = 500
        };
        context.ExceptionHandled = true;
    }
}

// Register
builder.Services.AddControllers(options => 
    options.Filters.Add<GlobalExceptionFilter>());
```

**Exception handling options:**
- UseExceptionHandler - Global middleware (catches all)
- Exception Filter - MVC-specific, can access controller context
- try-catch - Local handling within action

**Best practice:** Use global middleware for cross-cutting, filters for MVC-specific handling.

---

## Question 15: What is the difference between session and tempdata? How do you configure them?

**Answer:**

**Session:**
- Persists across requests within session lifetime
- Server-side storage (memory, distributed cache, SQL)
- Requires cookies or query string to identify session

```csharp
// Configuration
builder.Services.AddSession(options => {
    options.IdleTimeout = TimeSpan.FromMinutes(20);
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true;
});
app.UseSession();

// Usage
HttpContext.Session.SetString("Name", "John");
var name = HttpContext.Session.GetString("Name");
```

**TempData:**
- Short-lived, survives one redirect
- Used for passing data between requests (commonly post-redirect-get)
- Backed by session or cookies

```csharp
// Controller
TempData["Message"] = "Saved!";
return RedirectToAction("Index");

// Target action
var message = TempData["Message"];
```

**Key differences:**

| Aspect | Session | TempData |
|--------|---------|----------|
| Lifetime | Across requests | One redirect only |
| Use case | User preferences, cart | Flash messages |
| Storage | Server/cookies | Session or cookies |
| Cleanup | Manual | Auto after read |

---

## Question 16: Explain the different ways to handle concurrency in ASP.NET Core applications.

**Answer:**

**1. Locking (pessimistic):**
```csharp
private static readonly object _lock = new object();
public void Update() {
    lock (_lock) { /* critical section */ }
}
```

**2. SemaphoreSlim (async):**
```csharp
private static readonly SemaphoreSlim _semaphore = new(1, 1);
public async Task ProcessAsync() {
    await _semaphore.WaitAsync();
    try { /* work */ }
    finally { _semaphore.Release(); }
}
```

**3. Concurrent collections:**
```csharp
var concurrentBag = new ConcurrentBag<T>();
var concurrentDict = new ConcurrentDictionary<string, T>();
```

**4. Parallel processing:**
```csharp
await Parallel.ForEachAsync(items, ...);
var results = items.AsParallel().Select(x => Process(x));
```

**5. Distributed locks (Redis):**
```csharp
var redis = ConnectionMultiplexer.Connect("localhost");
var lock = redis.GetLock("key", TimeSpan.FromSeconds(30));
if (await lock.AcquireAsync()) {
    try { /* work */ }
    finally { await lock.ReleaseAsync(); }
}
```

**Choosing:**
- Single server: SemaphoreSlim, lock, ConcurrentDictionary
- Multiple servers: Distributed locks (Redis, SQL)
- High performance: Concurrent collections, parallel processing

---

## Question 17: What is SignalR? How does it enable real-time communication?

**Answer:**

SignalR provides real-time bidirectional communication:

**Server:**
```csharp
public class ChatHub : Hub {
    public async Task SendMessage(string user, string message) {
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }
}

// Register
builder.Services.AddSignalR();

// Map hub
app.MapHub<ChatHub>("/chat");
```

**Client (JavaScript):**
```javascript
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chat")
    .build();

connection.on("ReceiveMessage", (user, message) => {
    console.log(`${user}: ${message}`);
});

await connection.start();
await connection.invoke("SendMessage", "John", "Hello");
```

**How it works:**
- Uses WebSockets when available
- Falls back to Server-Sent Events or Long Polling
- Persistent connection from client to server
- Server can push to all or specific clients

**Use cases:** Live chat, notifications, dashboards, collaborative editing, gaming.

---

## Question 18: Explain static files handling in ASP.NET Core. How do you configure caching?

**Answer:**

**Enable static files:**
```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.UseStaticFiles(); // Enable serving static files
app.MapDefaultControllerRoute();
app.Run();
```

**Configure caching:**
```csharp
app.UseStaticFiles(new StaticFileOptions {
    OnPrepareResponse = context => {
        var headers = context.Context.Response.Headers;
        headers["Cache-Control"] = "public,max-age=3600";
        headers["Expires"] = DateTime.UtcNow.AddHours(1).ToString("R");
    }
});
```

**UseCacheTagHelper:**
```csharp
// In Razor views
<script src="~/js/site.js" asp-append-version="true"></script>
// Adds hash to filename for cache busting
```

**CDN for production:**
- Use app.UseResponseCaching() for HTTP response caching
- Configure CDN in Azure/AWS for global distribution
- Set proper cache headers for static assets

---

## Question 19: What is the difference between Razor Pages and MVC? When would you choose each?

**Answer:**

**MVC (Model-View-Controller):**
- Explicit routing with attributes [Route], [HttpGet]
- Controllers handle HTTP requests, return IActionResult
- Views are separate .cshtml files
- More flexibility, complex scenarios

**Razor Pages:**
- Page-focused, each .cshtml is its own "page"
- Uses @page directive for routing
- Code-behind inherits from PageModel
- Simpler for page-centric apps

**MVC Example:**
```csharp
[Route("users")]
public class UsersController : Controller {
    [HttpGet("{id}")]
    public IActionResult Details(int id) => View();
}
```

**Razor Pages Example:**
```csharp
// Pages/Users/Details.cshtml
@page "{id:int}"
@model DetailsModel

// Details.cshtml.cs
public class DetailsModel : PageModel {
    public void OnGet(int id) { }
}
```

**When to choose:**
- **Razor Pages:** Page-centric, simple CRUD, content sites, when you want simpler structure
- **MVC:** API + UI, complex routing, more control, team familiarity with MVC pattern

---

## Question 20: How do you implement rate limiting in ASP.NET Core?

**Answer:**

**Built-in rate limiting (.NET 7+):**
```csharp
builder.Services.AddRateLimiter(options => {
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(
        httpContext => RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: httpContext.User.Identity?.Name 
                ?? httpContext.Connection.RemoteIpAddress?.ToString(),
            factory: partition => new FixedWindowRateLimiterOptions {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1),
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                QueueLimit = 0
            }));
});
```

**Custom rate limiting middleware:**
```csharp
public class RateLimitMiddleware {
    private readonly RequestDelegate _next;
    private static int _requestCount = 0;
    private static DateTime _windowStart = DateTime.UtcNow;
    
    public async Task InvokeAsync(HttpContext context) {
        if ((DateTime.UtcNow - _windowStart).TotalSeconds > 60) {
            _requestCount = 0;
            _windowStart = DateTime.UtcNow;
        }
        
        if (_requestCount++ >= 100) {
            context.Response.StatusCode = 429;
            return;
        }
        
        await _next(context);
    }
}
```

**Common strategies:** Fixed window, sliding window, token bucket, concurrency limit.

---

*End of Section 4: ASP.NET Core*

---

## 🎯 Key Takeaways
- Revise important concepts quickly
- Focus on interview-ready answers
- Practice explaining in your own words

---

## 🎤 Interview Tip
> Always explain **WHY + HOW**, not just definitions.
