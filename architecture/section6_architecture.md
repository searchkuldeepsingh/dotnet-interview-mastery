# 🚀 Section6 Architecture — Interview Master Guide

> 🧠 Styled for fast revision + deep understanding

---

## 📌 Overview
# .NET Interview Preparation - Section 6: Architecture

## 13 Years Experience Candidate Answers

---

## Question 1: Explain Clean Architecture layers: Domain, Application, Infrastructure, Presentation. What goes in each?

**Answer:**

**Clean Architecture Layers:**

```
         Presentation (API/UI)          
   Controllers, ViewModels, DTOs         
-----------------------------------------
         Application (Use Cases)         
   Services, Commands, Queries, DTOs     
-----------------------------------------
              Domain (Business)          
   Entities, Value Objects, Interfaces  
-----------------------------------------
         Infrastructure (Data/External)  
   Repositories, External Services       
```

**Domain Layer:**
- Entities, Value Objects
- Domain Services
- Repository interfaces
- Domain Events
- NO dependencies on other layers

**Application Layer:**
- Use Cases/Application Services
- Commands, Queries (CQRS)
- DTOs, Mappers
- Interfaces for external services
- Depends ONLY on Domain

**Infrastructure Layer:**
- Repository implementations
- Database context
- External API clients
- File storage, Email services
- Implements Application interfaces

**Presentation Layer:**
- API Controllers
- Razor Pages, MVC
- View Models
- Depends on Application

**Key principle:** Dependencies point inward. Outer layers depend on inner, never vice versa.

---

## Question 2: What is the difference between layered architecture and hexagonal architecture?

**Answer:**

**Layered Architecture:**

```
┌─────────────┐
│ Presentation│
├─────────────┤
│ Application │
├─────────────┤
│   Domain    │
├─────────────┤
│ Infrastructure│
└─────────────┘
     ↓
Dependencies flow down
```

- Traditional, familiar
- Coupling through layers
- Infrastructure tied to specific implementations

**Hexagonal Architecture (Ports & Adapters):**

```
        ┌─────────────┐
        │   Adapter   │ ← UI, API, Tests
        │ (Driving)   │
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │    Port     │ ← Interface
        │ (Driving)   │
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │   Domain   │ ← Core Business
        │   Logic    │
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │    Port     │ ← Interface
        │ (Driven)    │
        └──────┬──────┘
               │
        ┌──────▼──────┐
        │   Adapter   │ ← DB, External APIs
        │ (Driven)    │
        └─────────────┘
```

- Ports (interfaces) in center
- Adapters around (implementations)
- Domain isolated from everything
- Easy to swap implementations

**When to use:**
- Layered: Simple apps, teams familiar with MVC
- Hexagonal: Complex domain, testability critical, multiple adapters

---

## Question 3: Explain CQRS (Command Query Responsibility Segregation). When should you use it?

**Answer:**

**CQRS separates read and write:**

```csharp
// Commands (Write)
public record CreateOrderCommand(OrderDto Order) : IRequest<Guid>;
public record UpdateOrderCommand(Guid Id, OrderDto Order) : IRequest<bool>;

public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Guid> {
    public async Task<Guid> Handle(CreateOrderCommand request, 
        CancellationToken ct) {
        // Create and save
    }
}

// Queries (Read)
public record GetOrderByIdQuery(Guid Id) : IRequest<OrderDto>;
public record GetOrdersQuery(int Page, int PageSize) : IRequest<List<OrderDto>>;

public class GetOrderByIdHandler : IRequestHandler<GetOrderByIdQuery, OrderDto> {
    public async Task<OrderDto> Handle(GetOrderByIdQuery request, 
        CancellationToken ct) {
        // Read and return
    }
}
```

**When to use CQRS:**
- Complex domains with different read/write models
- High read-to-write ratio (reads much more than writes)
- Different scaling needs for reads vs writes
- Event sourcing scenarios
- When you need optimized queries per use case

**When NOT to use:**
- Simple CRUD operations
- Small teams, simple domains
- Overhead not justified

**Benefits:** Independent scaling, optimized queries, clear separation, better testability.

---

## Question 4: What is Event Sourcing? How does it differ from traditional CRUD?

**Answer:**

**Traditional CRUD:**
- Store current state
- Overwrite on changes
- Limited history

**Event Sourcing:**
- Store all domain events
- Rebuild state by replaying events
- Full audit trail built-in

```csharp
// Store events
public void Withdraw(decimal amount) {
    if (amount > Balance) throw new InvalidOperationException();
    
    var withdrawalEvent = new WithdrawalEvent(Id, amount, DateTime.UtcNow);
    _events.Add(withdrawalEvent);
    Apply(withdrawalEvent);
}

// Rebuild state
public void Apply(WithdrawalEvent e) {
    Balance -= e.Amount;
}
```

**Key differences:**

| Aspect | CRUD | Event Sourcing |
|--------|------|----------------|
| Storage | Current state | Events |
| History | Limited | Complete |
| Performance | Good for updates | Good for reads |
| Complexity | Lower | Higher |
| Debugging | State snapshot | Replay events |

---

## Question 5: Explain the differences between microservices, service-oriented architecture (SOA), and monolithic architecture.

**Answer:**

**Monolithic Architecture:**
```
┌────────────────────────────────────────────┐
│   Web │ Business │ Data Access │ DB       │
└────────────────────────────────────────────┘
```
- Single deployable unit
- All code together
- Simple to develop, test, deploy initially
- Hard to scale, maintain as grows

**SOA (Service-Oriented Architecture):**
```
┌───┐   ┌───┐   ┌───┐
│SVC│   │SVC│   │SVC│   ← Enterprise Services
└─┬─┘   └─┬─┘   └─┬─┘
  └───────┴───────┘
       ESB
```
- Shared services via ESB
- Often heavyweight
- Enterprise-focused
- Less granular than microservices

**Microservices:**
```
┌───┐  ┌───┐  ┌───┐  ┌───┐
│API│  │API│  │API│  │API│  ← Independent services
└─┬─┘  └─┬─┘  └─┬─┘  └─┬─┘
  │     │     │     │
  DB   DB    DB    DB  ← Own data
```
- Small, focused services
- Own database
- Independent deployment
- Technology agnostic
- Challenges: distributed systems, data consistency

**Choosing:**
- Small team, simple domain - Monolith
- Enterprise integration - SOA
- Complex domain, scaling needs - Microservices

---

## Question 6: What is the API Gateway pattern? What are its responsibilities?

**Answer:**

**API Gateway:**
- Single entry point for all clients
- Routes requests to services
- Handles cross-cutting concerns

```
Client → API Gateway → Services
         │
         ├── Authentication
         ├── Routing
         ├── Rate Limiting
         ├── Logging
         └── Caching
```

**Responsibilities:**
1. **Routing** - Route to appropriate service
2. **Authentication** - Validate tokens, auth
3. **Rate limiting** - Throttle requests
4. **Load balancing** - Distribute across instances
5. **Caching** - Cache responses
6. **Logging/Tracing** - Request tracking
7. **Protocol translation** - REST to gRPC

**Implementation options:**
- Ocelot (.NET)
- Kong (nginx-based)
- AWS API Gateway
- Azure API Management

**BFF (Backend for Frontend):**
- Separate gateway per client type
- Web, Mobile, Third-party
- Customized responses

---

## Question 7: Explain the Saga pattern for distributed transactions. What are the alternatives?

**Answer:**

**Saga pattern:**
- Sequence of local transactions
- Each step has compensating action for rollback

```csharp
public class OrderSaga {
    public async Task CreateOrder(Order order) {
        try {
            // Step 1: Create order
            await _orderService.Create(order);
            
            // Step 2: Reserve inventory
            await _inventoryService.Reserve(order.Items);
            
            // Step 3: Process payment
            await _paymentService.Process(order.Payment);
            
            // Step 4: Confirm order
            await _orderService.Confirm(order.Id);
        }
        catch (Exception ex) {
            // Compensate in reverse
            await _paymentService.Refund(order.Payment);
            await _inventoryService.Release(order.Items);
            await _orderService.Cancel(order.Id);
        }
    }
}
```

**Alternatives:**

**1. Choreography:**
- Services emit/listen to events
- No central orchestrator
- More decoupled, harder to track

**2. 2-Phase Commit (2PC):**
- Coordinator asks participants to prepare
- If all yes, commit
- Not recommended for microservices

**3. Outbox Pattern:**
- Transactional outbox table
- Separate process publishes events
- Ensures delivery

---

## Question 8: What is the difference between synchronous and asynchronous communication between services?

**Answer:**

**Synchronous (REST/gRPC):**
```csharp
// HTTP call - blocks until response
var response = await httpClient.GetAsync("http://order-service/orders/1");
var order = await response.Content.ReadAsAsync<OrderDto>();
```
- Request-response pattern
- Client waits for response
- Simple, familiar
- Tight coupling
- Availability dependent

**Asynchronous (Message Queues):**
```csharp
// Publish event, don't wait
await _messagePublisher.PublishAsync("OrderCreated", new {
    OrderId = order.Id,
    CustomerId = order.CustomerId
});
```
- Fire-and-forget
- Decoupled in time
- Better resilience
- Eventual consistency
- More complex

**Comparison:**

| Aspect | Sync | Async |
|--------|------|-------|
| Latency | Immediate | Variable |
| Coupling | Tight | Loose |
| Failure handling | Direct | Retry/DLQ |
| Complexity | Lower | Higher |
| Consistency | Strong | Eventual |

**Best practice:** Use async for cross-service communication, sync only when immediate response needed.

---

## Question 9: Explain message queues (RabbitMQ, Kafka). When would you use each?

**Answer:**

**RabbitMQ:**
- Message broker
- Queue, exchange, routing
- Supports various patterns

**Apache Kafka:**
- Distributed event streaming
- Logs-based, persistent
- High throughput

**When to use:**

| Use Case | RabbitMQ | Kafka |
|----------|----------|-------|
| Task queues | ✓ | |
| Event streaming | | ✓ |
| Complex routing | ✓ | |
| High throughput | | ✓ |
| Message persistence | | ✓ |
| Simplicity | ✓ | |

---

## Question 10: What is service mesh? How does it help with microservices communication?

**Answer:**

**Service Mesh:**
- Infrastructure layer for service communication
- Sidecar proxies alongside each service
- Central control plane

```
┌──────────────────────────────┐
│      Service Mesh            │
│  ┌─────┐    ┌─────┐         │
│  │Proxy│    │Proxy│ ← Sidecar│
│  └─────┘    └─────┘          │
│     ↓          ↓             │
│  Service A  Service B        │
│                              │
│  Control Plane (config)      │
└──────────────────────────────┘
```

**Key capabilities:**
1. **Service Discovery** - Find services automatically
2. **Load Balancing** - Distribute requests
3. **Traffic Management** - Routing, splitting
4. **Security** - mTLS, auth
5. **Observability** - Metrics, tracing
6. **Resilience** - Retries, circuit breakers

**Popular implementations:**
- **Istio** - Full-featured, complex
- **Linkerd** - Simpler, lightweight
- **Consul Connect** - HashiCorp ecosystem
- **Envoy** - Underlying proxy

**Benefits:** Offloads infrastructure concerns from code, consistent behavior across services.

---

## Question 11: Explain the Circuit Breaker pattern. How does it improve resilience?

**Answer:**

**Circuit Breaker:**
- Prevents cascading failures
- States: Closed → Open → Half-Open

```
Closed: Normal operation → Failures accumulate
   ↓ (threshold reached)
Open: Requests fail fast → No calls to failing service
   ↓ (timeout)
Half-Open: Test if service recovered
   ↓ (success)
Closed: Resume normal operation
```

**Implementation (Polly):**
```csharp
// Define circuit breaker policy
var circuitBreaker = Policy
    .Handle<HttpRequestException>()
    .CircuitBreaker(
        exceptionsAllowedBeforeBreaking: 3,
        durationOfBreak: TimeSpan.FromSeconds(30));

// Use with HTTP client
var httpClient = new HttpClient();
var pipeline = new PolicyHttpMessageHandler(circuitBreaker);
services.AddHttpClient("orders")
    .AddPolicyHandler(pipeline);
```

**Why it helps:**
- Fail fast prevents resource exhaustion
- Gives failing service time to recover
- Prevents cascading failures
- Graceful degradation

**Related patterns:**
- Retry with exponential backoff
- Bulkhead (isolation)
- Fallback (degraded response)

---

## Question 12: What is the difference between eventual consistency and strong consistency?

**Answer:**

**Strong Consistency:**
- All reads see most recent write
- Data is immediately consistent across all replicas
- Example: Bank account balance
- After write, immediately read reflects change

**Eventual Consistency:**
- Writes propagate asynchronously
- Read might see stale data temporarily
- Example: Social media likes, caching
- Write happens, but replicas update eventually

**Comparison:**

| Aspect | Strong | Eventual |
|--------|--------|----------|
| Latency | Higher (wait for sync) | Lower (async) |
| Availability | Lower (during partition) | Higher (always available) |
| Data freshness | Immediate | May lag |
| Use cases | Financial, critical | Social, caching |

**Choosing:**
- Critical data (transactions) - Strong
- High availability, non-critical - Eventual
- Most distributed systems use eventual for scale

---

## Question 13: Explain the Backends for Frontends (BFF) pattern. When is it useful?

**Answer:**

**BFF Pattern:**
- Separate API per client type
- Client-specific aggregation

```
         ┌──────────┐
         │  Web     │ → Web BFF
         └──────────┘
Client ──┤  Mobile  │ → Mobile BFF
         └──────────┘
         │  Third  │ → Third-party BFF
         └──────────┘
```

**When useful:**
- Different clients need different data shapes
- Different security requirements
- Different performance characteristics
- Team ownership per client

**Benefits:** Optimized responses, client-specific auth, easier evolution

---

## Question 14: What is the Repository pattern? How does it abstract data access?

**Answer:**

**Repository:**
- Mediates between domain and data sources
- Collection-like interface for domain

```csharp
public interface IOrderRepository {
    Order GetById(int id);
    IEnumerable<Order> GetAll();
    IEnumerable<Order> GetByCustomer(int customerId);
    void Add(Order order);
    void Update(Order order);
    void Delete(int id);
}

public class OrderRepository : IOrderRepository {
    private readonly AppDbContext _context;
    
    public Order GetById(int id) => _context.Orders.Find(id);
    public IEnumerable<Order> GetByCustomer(int customerId) 
        => _context.Orders.Where(o => o.CustomerId == customerId);
}
```

**How it abstracts:**
- Domain doesn't know about EF, Dapper, HTTP
- Testable with in-memory fake
- Swappable data sources
- Collection-like API

**Benefits:**
- Testability (mock/fake)
- Flexibility (swap storage)
- Clean domain model
- Centralized query logic

---

## Question 15: Explain the Unit of Work pattern in the context of enterprise applications.

**Answer:**

**Unit of Work:**
- Coordinates multiple repositories
- Single transaction for multiple changes

```csharp
public interface IUnitOfWork {
    IOrderRepository Orders { get; }
    IProductRepository Products { get; }
    ICustomerRepository Customers { get; }
    void Commit();
    void Rollback();
}

public class UnitOfWork : IUnitOfWork {
    private readonly AppDbContext _context;
    private IOrderRepository _orders;
    private IProductRepository _products;
    
    public UnitOfWork(AppDbContext context) => _context = context;
    
    public IOrderRepository Orders => 
        _orders ??= new OrderRepository(_context);
    
    public void Commit() => _context.SaveChanges();
}

// Usage
using (var uow = _unitOfWorkFactory.Create()) {
    var order = uow.Orders.GetById(1);
    order.Status = "Shipped";
    uow.Products.DecrementStock(order.Items);
    uow.Commit(); // Single transaction
}
```

**Benefits:**
- Atomic operations across entities
- Single SaveChanges call
- Consistent data
- Testable

**Common in:** Enterprise apps, complex business operations, database transactions.

---

## Question 16: What are the considerations for choosing between REST, GraphQL, and gRPC?

**Answer:**

**REST:**
- Resources, HTTP verbs
- Standard, widely understood
- Over-fetching/under-fetching issues

**GraphQL:**
- Query language for APIs
- Client specifies exact data needed
- Single endpoint, flexible queries

**gRPC:**
- High-performance RPC
- Protocol Buffers (binary)
- Contracts, strongly typed

**Choosing:**

| Factor | REST | GraphQL | gRPC |
|--------|------|---------|------|
| Flexibility | Low | High | Medium |
| Performance | Good | Good | Excellent |
| Learning curve | Low | Medium | High |
| Use case | CRUD, standard | Dynamic UI, mobile | Inter-service, high-perf |
| Caching | HTTP caching | Custom | Streaming |
| Tooling | Wide | Growing | Limited |

---

## Question 17: How do you handle cross-cutting concerns in a Clean Architecture application?

**Answer:**

**Cross-cutting concerns:**
- Logging, Authentication, Caching, Validation, Exception handling

**In Clean Architecture:**

**1. Middleware (ASP.NET Core):**
```csharp
app.UseMiddleware<LoggingMiddleware>();
app.UseMiddleware<ExceptionHandlingMiddleware>();
```

**2. Decorator pattern (for use cases):**
```csharp
public class LoggingBehavior<TRequest, TResponse> 
    : IPipelineBehavior<TRequest, TResponse> {
    public async Task<TResponse> Handle(TRequest request, 
        RequestHandlerDelegate<TResponse> next) {
        _logger.LogRequest(request);
        var response = await next();
        _logger.LogResponse(response);
        return response;
    }
}
```

**3. Attributes:**
```csharp
[Authorize]
[ValidateModel]
[ExceptionHandler]
public class OrdersController : Controller { }
```

**4. DI Registrations:**
```csharp
services.AddScoped(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
services.AddScoped(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
```

**Key:** Don't put cross-cutting in domain. Keep them in infrastructure/application layers.

---

## Question 18: Explain the concept of bounded contexts in Domain-Driven Design.

**Answer:**

**Bounded Context:**
- Boundary where a particular domain model applies
- Explicit ownership of domain concepts

```
┌─────────────────────┐  ┌─────────────────────┐
│   Order Context     │  │  Shipping Context  │
│  ┌───────────────┐  │  │  ┌───────────────┐  │
│  │ Order Entity  │  │  │  │ Shipment      │  │
│  │ OrderItem     │  │  │  │ Delivery      │  │
│  │ OrderService  │  │  │  │ Carrier       │  │
│  └───────────────┘  │  │  └───────────────┘  │
└─────────────────────┘  └─────────────────────┘
          │                      │
          ↓                      ↓
    ┌──────────┐           ┌──────────┐
    │   API   │           │   API    │
    └──────────┘           └──────────┘
```

**Example:**
- **Order Context** owns: Order, OrderLine, Pricing
- **Shipping Context** owns: Shipment, Delivery, Address
- **Customer Context** owns: Customer, Address (different from Shipping address)

**Communication between contexts:**
- Anticorruption Layer (translate)
- Published Language (common format)
- Integration Events

**Benefits:** Clear ownership, independent models, scalable teams, reduced complexity.

---

## Question 19: What is the difference between domain events and integration events?

**Answer:**

**Domain Events:**
- Within single bounded context
- Represent something that happened in domain
- Handled within same application

```csharp
// Domain event
public class OrderPlacedEvent : INotification {
    public Guid OrderId { get; }
    public DateTime PlacedAt { get; }
}

// In domain service
public void PlaceOrder(Order order) {
    order.Status = OrderStatus.Placed;
    _mediator.Publish(new OrderPlacedEvent(order.Id, DateTime.UtcNow));
}

// Handler in same context
public class OrderPlacedHandler : INotificationHandler<OrderPlacedEvent> {
    public Task Handle(OrderPlacedEvent notification, 
        CancellationToken ct) {
        // Update aggregates, send notifications
    }
}
```

**Integration Events:**
- Cross-boundary, cross-service
- For communication between systems
- Published to message broker

```csharp
// Integration event
public class OrderPlacedIntegrationEvent {
    public Guid OrderId { get; }
    public decimal Total { get; }
}

// Publish to message queue
await _eventBus.PublishAsync(new OrderPlacedIntegrationEvent {
    OrderId = order.Id,
    Total = order.Total
});
```

**Key difference:** Domain events handled locally; integration events for external systems.

---

## Question 20: How do you structure a solution for enterprise-scale applications?

**Answer:**

**Solution structure:**

```
Solution/
├── src/
│   ├── Project.Domain/           # Domain layer
│   │   ├── Entities/
│   │   ├── ValueObjects/
│   │   ├── Interfaces/
│   │   └── Events/
│   │
│   ├── Project.Application/      # Application layer
│   │   ├── Commands/
│   │   ├── Queries/
│   │   ├── DTOs/
│   │   ├── Interfaces/
│   │   └── Behaviors/
│   │
│   ├── Project.Infrastructure/  # Infrastructure layer
│   │   ├── Persistence/
│   │   ├── Services/
│   │   └── Migrations/
│   │
│   ├── Project.Api/              # API layer
│   │   ├── Controllers/
│   │   ├── Filters/
│   │   └── Program.cs
│   │
│   └── Project.Tests/            # Tests
│
├── docs/
├── scripts/
└── tests/
```

**Key principles:**
- One project per layer (or feature folder)
- Clear dependencies (Domain → Application → Infrastructure → API)
- Feature-based folder structure within layers
- Shared kernel for cross-cutting concepts
- Solution folders match team ownership

---

*End of Section 6: Architecture*

---

## 🎯 Key Takeaways
- Revise important concepts quickly
- Focus on interview-ready answers
- Practice explaining in your own words

---

## 🎤 Interview Tip
> Always explain **WHY + HOW**, not just definitions.
