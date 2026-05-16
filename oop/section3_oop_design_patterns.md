# 🚀 Section3 Oop Design Patterns — Interview Master Guide

> 🧠 Styled for fast revision + deep understanding

---

## 📌 Overview
# .NET Interview Preparation - Section 3: OOP & Design Patterns

## 13 Years Experience Candidate Answers

---

## Question 1: Explain each of the SOLID principles with real-world examples: Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion.

**Answer:**

**Single Responsibility Principle (SRP):**
- A class should have only one reason to change
- Example: Order class should only handle order data, not printing or database operations
- Separate into Order (domain), OrderPrinter (presentation), OrderRepository (persistence)

**Open/Closed Principle (OCP):**
- Open for extension, closed for modification
- Example: Use strategy pattern to add new payment types without modifying existing code
- Achieve via inheritance, composition, polymorphism

**Liskov Substitution Principle (LSP):**
- Subtypes must be substitutable for their base types
- Example: If Bird can fly, don't have Penguin inherit from Bird - it breaks substitution
- Derived classes should not strengthen preconditions or weaken postconditions

**Interface Segregation Principle (ISP):**
- Clients shouldn't depend on interfaces they don't use
- Example: Instead of one fat IMachine with Print/Scan/Fax, have IPrinter, IScanner, IFax
- Smaller, focused interfaces are better

**Dependency Inversion Principle (DIP):**
- Depend on abstractions, not concretions
- Example: OrderService depends on IOrderRepository, not SqlOrderRepository
- High-level modules shouldn't depend on low-level modules; both depend on abstractions

---

## Question 2: What is the Singleton design pattern? Implement it in a thread-safe way in C#. What are its pros and cons?

**Answer:**

Singleton ensures only one instance exists:

```csharp
public sealed class Singleton {
    private static readonly Lazy<Singleton> _instance = 
        new Lazy<Singleton>(() => new Singleton());
    
    public static Singleton Instance => _instance.Value;
    
    private Singleton() { } // Prevent external instantiation
}
```

**Thread-safe implementation:**
- Lazy<T> provides thread-safe initialization
- sealed prevents inheritance
- Private constructor prevents external creation

**Pros:**
- Single instance saves resources
- Global access point
- Lazy initialization possible

**Cons:**
- Tight coupling - hard to test
- Violates SRP (manages itself + its responsibility)
- Can mask design problems
- Not suitable for distributed systems

**Alternatives:** DI container Singleton, dependency injection with scoped lifetime.

---

## Question 3: Explain the Factory Method and Abstract Factory patterns. When would you use each?

**Answer:**

**Factory Method:**
- Defines interface for creating objects, lets subclasses decide the class
- Uses inheritance, relies on subclass to handle desired object creation

```csharp
public abstract class Logistics {
    public abstract ITransport CreateTransport();
    
    public void PlanDelivery() {
        var transport = CreateTransport();
        transport.Deliver();
    }
}

public class SeaLogistics : Logistics {
    public override ITransport CreateTransport() => new Ship();
}
```

**When to use Factory Method:** When a class can't anticipate the class of objects it must create, or when subclasses should specify objects they create.

**Abstract Factory:**
- Provides interface for creating families of related objects without specifying concrete classes

```csharp
public interface IUIFactory {
    IButton CreateButton();
    ITextBox CreateTextBox();
}

public class DarkThemeFactory : IUIFactory {
    public IButton CreateButton() => new DarkButton();
    public ITextBox CreateTextBox() => new DarkTextBox();
}
```

**When to use Abstract Factory:** When you need to create families of related objects that must be used together, or when you want to provide a library without exposing implementations.

**Key difference:** Factory Method uses inheritance, Abstract Factory uses composition. Factory Method creates one product, Abstract Factory creates families of products.

---

## Question 4: What is the Builder pattern? How does it differ from the Factory pattern?

**Answer:**

**Builder Pattern:**
- Separates construction of complex objects from representation
- Step-by-step object creation with fluent interface

```csharp
public class PizzaBuilder {
    private Pizza _pizza = new Pizza();
    
    public PizzaBuilder WithDough(string dough) {
        _pizza.Dough = dough;
        return this;
    }
    
    public PizzaBuilder WithTopping(string topping) {
        _pizza.Toppings.Add(topping);
        return this;
    }
    
    public Pizza Build() => _pizza;
}

// Usage
var pizza = new PizzaBuilder()
    .WithDough("thin")
    .WithTopping("pepperoni")
    .WithTopping("mushroom")
    .Build();
```

**Differences from Factory:**

| Aspect | Factory | Builder |
|--------|---------|---------|
| Purpose | Creates one product | Constructs complex object step-by-step |
| Complexity | Simple objects | Complex objects with many parts |
| Approach | Returns immediately | Fluent interface, multiple steps |
| Immutability | Usually creates ready-to-use | Builds incrementally |

**When to use Builder:** Objects have many optional parameters, fluent API needed, immutable objects.

---

## Question 5: Explain the Adapter pattern. How does it help integrate incompatible interfaces?

**Answer:**

Adapter converts one interface to another:

```csharp
// Existing interface
public interface ILogger {
    void Log(string message);
}

// New incompatible interface
public class LegacyLogger {
    public void WriteLog(string level, string msg) { }
}

// Adapter
public class LoggerAdapter : ILogger {
    private readonly LegacyLogger _legacy;
    
    public LoggerAdapter(LegacyLogger legacy) => _legacy = legacy;
    
    public void Log(string message) {
        _legacy.WriteLog("INFO", message);
    }
}
```

**How it helps:**
- Allows existing incompatible code to work with new systems
- Wraps legacy code without modifying it
- Promotes reusability of existing code

**Types:**
- **Object Adapter** - Uses composition (has-a)
- **Class Adapter** - Uses inheritance (is-a) - requires multiple inheritance

**Real-world:** Third-party libraries, legacy systems integration, testing with mocks.

---

## Question 6: What is the Decorator pattern? How does it differ from inheritance?

**Answer:**

Decorator adds behavior dynamically:

```csharp
public interface ICoffee {
    string GetDescription();
    decimal GetCost();
}

public class SimpleCoffee : ICoffee {
    public string GetDescription() => "Coffee";
    public decimal GetCost() => 2.00m;
}

public abstract class CoffeeDecorator : ICoffee {
    protected ICoffee _coffee;
    public CoffeeDecorator(ICoffee coffee) => _coffee = coffee;
    
    public virtual string GetDescription() => _coffee.GetDescription();
    public virtual decimal GetCost() => _coffee.GetCost();
}

public class MilkDecorator : CoffeeDecorator {
    public MilkDecorator(ICoffee coffee) : base(coffee) { }
    
    public override string GetDescription() 
        => _coffee.GetDescription() + ", Milk";
    
    public override decimal GetCost() 
        => _coffee.GetCost() + 0.50m;
}
```

**How it differs from inheritance:**

| Aspect | Inheritance | Decorator |
|--------|-------------|-----------|
| At compile time | Static, fixed | Runtime, dynamic |
| Object count | One class | Wraps original |
| Behavior addition | Override method | Wrap and extend |
| Flexibility | Limited | Very flexible |
| Testing | Harder | Easier (single responsibility) |

**Use Decorator when:** You need to add behavior at runtime, combine multiple behaviors, avoid class explosion from inheritance.

---

## Question 7: Explain the Facade pattern. When would you use it?

**Answer:**

Facade provides simplified interface to complex subsystem:

```csharp
public class ComputerFacade {
    private readonly CPU _cpu;
    private readonly Memory _memory;
    private readonly HardDrive _hardDrive;
    
    public ComputerFacade() {
        _cpu = new CPU();
        _memory = new Memory();
        _hardDrive = new HardDrive();
    }
    
    public void Start() {
        _cpu.Freeze();
        _memory.Load(0, _hardDrive.Read());
        _cpu.Jump(0);
    }
}

// Client uses simple interface
var computer = new ComputerFacade();
computer.Start();
```

**When to use:**
- When you need a simplified interface to complex subsystem
- To layer your application (facade over multiple subsystems)
- When there are many dependencies between clients and implementation classes
- When you want to hide complexity from other parts of the system

**Benefits:** Simplicity for clients, reduces dependencies, promotes weak coupling.

**Real-world:** .NET HttpClient over underlying socket management, ORM over ADO.NET.

---

## Question 8: What is the Observer pattern? How does C# events implement this pattern?

**Answer:**

Observer defines one-to-many dependency:

```csharp
public interface IObserver {
    void Update(string stockSymbol, decimal price);
}

public interface ISubject {
    void Attach(IObserver observer);
    void Detach(IObserver observer);
    void Notify();
}

public class Stock : ISubject {
    private List<IObserver> _observers = new();
    private string _symbol;
    private decimal _price;
    
    public void Attach(IObserver observer) => _observers.Add(observer);
    public void Detach(IObserver observer) => _observers.Remove(observer);
    
    public void Notify() {
        foreach(var o in _observers) o.Update(_symbol, _price);
    }
    
    public void SetPrice(decimal price) {
        _price = price;
        Notify();
    }
}
```

**C# Events implement Observer:**
- event keyword = subject
- += = attach
- -= = detach
- Invoke() = notify

---

## Question 9: Explain the Strategy pattern. How does it enable runtime behavior change?

**Answer:**

Strategy defines algorithms family, makes them interchangeable:

```csharp
public interface ISortStrategy<T> {
    void Sort(List<T> items);
}

public class QuickSort<T> : ISortStrategy<T> {
    public void Sort(List<T> items) { /* quicksort impl */ }
}

public class MergeSort<T> : ISortStrategy<T> {
    public void Sort(List<T> items) { /* mergesort impl */ }
}

public class Sorter<T> {
    private ISortStrategy<T> _strategy;
    
    public Sorter(ISortStrategy<T> strategy) => _strategy = strategy;
    
    public void ChangeStrategy(ISortStrategy<T> strategy) 
        => _strategy = strategy;
    
    public void Sort(List<T> items) => _strategy.Sort(items);
}
```

**Runtime behavior change:**
```csharp
var sorter = new Sorter<int>(new QuickSort<int>());
sorter.Sort(data); // Uses quicksort

// Change at runtime based on data size
if (data.Count < 1000)
    sorter.ChangeStrategy(new MergeSort<int>());
```

**Benefits:** Algorithms can vary independently from clients using them, swap strategies at runtime, follows OCP.

---

## Question 10: What is the Chain of Responsibility pattern? How does it differ from Decorator?

**Answer:**

Chain passes request along chain of handlers:

```csharp
public abstract class Handler {
    protected Handler _next;
    
    public Handler SetNext(Handler next) {
        _next = next;
        return next;
    }
    
    public abstract void Handle(Request request);
}

public class AuthHandler : Handler {
    public override void Handle(Request request) {
        if (!request.IsAuthenticated) {
            Console.WriteLine("Auth failed");
            return;
        }
        _next?.Handle(request);
    }
}
```

**Decorator vs Chain of Responsibility:**

| Aspect | Decorator | Chain of Responsibility |
|--------|-----------|--------------------------|
| Purpose | Add behavior | Pass to next handler |
| Termination | Stays in chain | May stop propagation |
| Same interface | Yes | Yes |
| Use case | Enhance object | Handle request processing |
| Order | Fixed by wrapping | Defined by chain |

---

## Question 11: Explain the Repository pattern. Why is it important for testability?

**Answer:**

Repository abstracts data access:

```csharp
public interface IUserRepository {
    User GetById(int id);
    IEnumerable<User> GetAll();
    void Add(User user);
    void Update(User user);
    void Delete(int id);
}

public class UserRepository : IUserRepository {
    private readonly DbContext _context;
    
    public UserRepository(DbContext context) => _context = context;
    
    public User GetById(int id) => _context.Users.Find(id);
    public IEnumerable<User> GetAll() => _context.Users.ToList();
}
```

**Why important for testability:**
1. **Abstraction** - Code depends on interface, not concrete implementation
2. **Mockable** - Easy to swap with mock/fake for unit tests
3. **Test data control** - Create in-memory fake repositories with test data
4. **Isolation** - Business logic can be tested without database

```csharp
// In tests
var mockRepo = new Mock<IUserRepository>();
mockRepo.Setup(r => r.GetById(1)).Returns(new User { Id = 1 });
var service = new UserService(mockRepo.Object);
```

---

## Question 12: What is the Unit of Work pattern? How does it work with multiple repositories?

**Answer:**

Unit of Work maintains list of changes, commits at once:

```csharp
public interface IUnitOfWork {
    IUserRepository Users { get; }
    IOrderRepository Orders { get; }
    void Commit();
    void Rollback();
}

public class UnitOfWork : IUnitOfWork {
    private readonly DbContext _context;
    
    private UserRepository _userRepo;
    private OrderRepository _orderRepo;
    
    public UnitOfWork(DbContext context) => _context = context;
    
    public IUserRepository Users => 
        _userRepo ??= new UserRepository(_context);
    
    public IOrderRepository Orders => 
        _orderRepo ??= new OrderRepository(_context);
    
    public void Commit() => _context.SaveChanges();
}
```

**How it works with multiple repositories:**
- All repositories share same DbContext
- Changes tracked by context
- Single SaveChanges() commits all changes atomically
- If any fails, all roll back

---

## Question 13: Explain the Dependency Injection patterns: Constructor Injection, Property Injection, Method Injection. Which is preferred and why?

**Answer:**

**Constructor Injection:**
```csharp
public class OrderService {
    private readonly IOrderRepository _repo;
    
    public OrderService(IOrderRepository repo) {
        _repo = repo;
    }
}
```
- Dependencies provided at object creation
- Object created with all dependencies (immutable)

**Property Injection (Setter):**
```csharp
public class OrderService {
    public IOrderRepository Repository { get; set; }
}
```
- Dependencies set after construction
- Optional dependencies

**Method Injection:**
```csharp
public class OrderService {
    public void Process(IOrderRepository repo) { }
}
```
- Dependency provided per method call

**Preferred: Constructor Injection**

**Why:**
- Object always in valid state (all dependencies available)
- Clear dependencies (visible in constructor)
- Immutable design possible
- Easy to test (pass mocks in constructor)
- DI containers work best with constructor injection

---

## Question 14: What is the difference between Composition over Inheritance? When is inheritance appropriate?

**Answer:**

**Composition over Inheritance:**
- Prefer "has-a" over "is-a"
- Compose behavior using objects, not class hierarchy

```csharp
// Instead of inheritance
class Car : Vehicle { } // Bad if Car needs different behavior

// Use composition
class Car {
    private readonly IEngine _engine;
    private readonly IWheels _wheels;
}
```

**When to use Composition:**
- When you want runtime behavior change
- When classes share common interface but different implementations
- When inheritance creates fragile hierarchies
- For code reuse without tight coupling

**When Inheritance is appropriate:**
- True "is-a" relationship (Dog is Animal)
- Shared identity (not just behavior)
- Liskov Substitution holds (can substitute anywhere base is used)
- No need to change behavior at runtime
- Framework requires it (Exception, Attribute)

**13-year insight:** Inheritance creates tight coupling. Start with composition, use inheritance only when inheritance is clearly the right model.

---

## Question 15: Explain the Proxy pattern. What are the different types of proxies?

**Answer:**

Proxy provides surrogate for another object:

```csharp
public interface IImage {
    void Display();
}

public class RealImage : IImage {
    private string _fileName;
    public RealImage(string fileName) {
        _fileName = fileName;
        LoadFromDisk();
    }
    public void Display() => Console.WriteLine($"Display {_fileName}");
}

public class ProxyImage : IImage {
    private RealImage _realImage;
    private string _fileName;
    
    public ProxyImage(string fileName) => _fileName = fileName;
    
    public void Display() {
        _realImage ??= new RealImage(_fileName);
        _realImage.Display();
    }
}
```

**Types of Proxies:**

1. **Remote** - Local representative for remote object
2. **Virtual** - Lazy initialization, caching
3. **Protection** - Access control
4. **Smart Reference** - Additional actions on access
5. **Cache** - Store results for repeated requests
6. **Logging** - Track method calls
7. **Validation** - Check preconditions

---

## Question 16: What is the Command pattern? How does it enable undo/redo functionality?

**Answer:**

Command encapsulates request as object:

```csharp
public interface ICommand {
    void Execute();
    void Undo();
}

public class LightOnCommand : ICommand {
    private Light _light;
    public LightOnCommand(Light light) => _light = light;
    
    public void Execute() => _light.On();
    public void Undo() => _light.Off();
}

public class RemoteControl {
    private Stack<ICommand> _history = new();
    
    public void ExecuteCommand(ICommand cmd) {
        cmd.Execute();
        _history.Push(cmd);
    }
    
    public void Undo() {
        if (_history.Any()) {
            var cmd = _history.Pop();
            cmd.Undo();
        }
    }
}
```

**How undo/redo works:**
- Each command knows how to reverse itself
- History stack tracks executed commands
- Undo pops and calls command.Undo()
- Redo stack can track undone commands

**Benefits:** Encapsulates action, supports undo/redo, queuing, logging.

---

## Question 17: Explain the Singleton vs Scoped vs Transient lifetime in DI. How do you choose?

**Answer:**

**Singleton:**
- One instance for entire application lifetime
- Created once, reused everywhere
- Use for: caching, logging, configuration

**Scoped:**
- One instance per request/scope
- Same instance within single request
- Use for: DbContext, repositories per request

**Transient:**
- New instance every time
- Every injection creates new instance
- Use for: Lightweight, stateless services

**Choosing:**
```csharp
services.AddSingleton<ILogger, Logger>();
services.AddScoped<IUnitOfWork, UnitOfWork>();
services.AddTransient<IEmailService, EmailService>();
```

**Rules:**
- If service holds state - Scoped or Singleton
- If service is stateless - Transient (unless caching needed)
- If depends on Scoped - Container must be Scoped or Singleton
- DbContext - Scoped (never Singleton or Transient)

**13-year insight:** Most services should be Scoped in web apps (one per request). Don't make DbContext Singleton (connection pooling issues).

---

## Question 18: What is the Mediator pattern? How does it reduce coupling?

**Answer:**

Mediator centralizes communication between objects:

```csharp
public interface IMediator {
    void Send(string message, Colleague colleague);
}

public class ConcreteMediator : IMediator {
    public ColleagueA ColleagueA { get; set; }
    public ColleagueB ColleagueB { get; set; }
    
    public void Send(string message, Colleague colleague) {
        if (colleague == ColleagueA)
            ColleagueB.Notify(message);
        else
            ColleagueA.Notify(message);
    }
}
```

**How it reduces coupling:**
- Objects don't know about each other
- They only know the Mediator
- Communication happens through mediator
- New colleagues can be added without changing existing code

**Real-world:** ASP.NET Core MediatR, notification systems, event aggregation.

---

## Question 19: Explain the CQRS pattern. How does it differ from traditional CRUD?

**Answer:**

**CQRS (Command Query Responsibility Segregation):**
- Separate models for reading and writing
- Commands: modify state (insert, update, delete)
- Queries: read data (select)

```csharp
// Commands (Write)
public record CreateOrderCommand(OrderDto Order);
public record UpdateOrderCommand(int Id, OrderDto Order);

// Queries (Read)
public record GetOrderByIdQuery(int Id);
public record GetOrdersQuery(PaginationParams Params);

// Different handlers
public class CreateOrderHandler : IRequestHandler<CreateOrderCommand> { }
public class GetOrderHandler : IRequestHandler<GetOrderByIdQuery, OrderDto> { }
```

**Traditional CRUD:** Same model for read/write, single responsibility.

**Key differences:**

| Aspect | CRUD | CQRS |
|--------|------|------|
| Model | Single model | Separate read/write |
| Operations | Same for all | Optimized per operation |
| Scalability | Same resource | Independent |
| Complexity | Simple | Higher |

**When to use:** Complex domains, high read/write ratio differences, different scaling needs, event sourcing.

---

## Question 20: What is the difference between Inversion of Control (IoC) and Dependency Injection?

**Answer:**

**Inversion of Control (IoC):**
- General principle: framework controls program flow
- Hollywood principle: "Don't call us, we'll call you"
- Framework calls your code, not vice versa

**Dependency Injection (DI):**
- Technique to implement IoC
- Dependencies provided to class (injected) rather than class creating them
- Form of IoC where dependencies are passed in

**Relationship:**
- DI is one way to achieve IoC
- IoC is the concept, DI is the implementation
- DI container is a tool that manages DI

```csharp
// Without DI - tightly coupled
public class OrderService {
    private readonly OrderRepository _repo = new OrderRepository();
}

// With DI - loosely coupled
public class OrderService {
    private readonly IOrderRepository _repo;
    public OrderService(IOrderRepository repo) => _repo = repo;
}
```

**Other IoC techniques:** Service Locator, Factory, Events.

---

*End of Section 3: OOP & Design Patterns*

---

## 🎯 Key Takeaways
- Revise important concepts quickly
- Focus on interview-ready answers
- Practice explaining in your own words

---

## 🎤 Interview Tip
> Always explain **WHY + HOW**, not just definitions.
