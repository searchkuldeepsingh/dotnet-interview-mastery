# 🚀 Testing — Interview Master Guide

> 🧠 Structured for senior-level interview preparation  
> ⚡ Fast revision + deep understanding  

---

## 📌 Overview
> This section covers key concepts required for real-world interviews.

---

# .NET Interview Preparation - Section 8: Testing

## 13 Years Experience Candidate Answers

---

## Question 1: Explain the different levels of testing: Unit, Integration, End-to-End, UI. What goes in each?

**Answer:**

**Unit Tests:**
- Test single class/method in isolation
- Mock all dependencies
- Fast, focused, deterministic
- Test one thing per test

**Integration Tests:**
- Test multiple components working together
- Use real dependencies (database, file system)
- Slower than unit tests
- Test interactions

**End-to-End (E2E) Tests:**
- Test complete user workflows
- Test through the UI or API
- Slowest, most realistic
- Test like a user would

**UI Tests:**
- Use UI automation (Selenium, Playwright)
- Test in real browser
- Most expensive to maintain
- For critical user paths

**Pyramid:** Many unit tests, fewer integration, minimal E2E.

---

## Question 2: What is the difference between xUnit, NUnit, and MSTest? Which would you choose and why?

**Answer:**

**xUnit:**
- Modern, created by NUnit author
- Highly extensible
- [Theory] for data-driven tests
- Parallel by default
- Popular in .NET Core ecosystem

**NUnit:**
- Mature, feature-rich
- [TestCase] for parameters
- Rich assertion library
- Very stable
- Large community

**MSTest:**
- Microsoft-provided
- Integrated with Visual Studio
- Good for Azure DevOps
- Less flexible than others

**Which to choose:** xUnit is most popular in modern .NET development. NUnit is great for teams coming from older frameworks. All are capable - choose based on team familiarity.

---

## Question 3: How do you write effective unit tests? What makes a good unit test?

**Answer:**

**Good unit test characteristics:**
- **Fast** - completes in milliseconds
- **Isolated** - doesn't depend on other tests
- **Deterministic** - same result every time
- **Self-documenting** - test name explains what it tests

**Structure (AAA):**
- **Arrange** - setup test data, objects
- **Act** - execute the action
- **Assert** - verify expected outcome

**Best practices:**
- Test one thing per test
- Use meaningful test names
- Don't test implementation details
- Keep assertions clear
- Avoid magic numbers

---

## Question 4: Explain the Arrange-Act-Assert pattern. How do you structure your tests?

**Answer:**

**Arrange:**
- Create objects needed
- Setup mock dependencies
- Prepare test data
- Configure expected results

**Act:**
- Execute single action being tested
- Call the method under test
- Minimal code in this section

**Assert:**
- Verify the result
- Check method did what expected
- Multiple assertions OK if testing one thing

```csharp
[Fact]
public void CalculateTotal_ReturnsSum() {
    // Arrange
    var calculator = new OrderCalculator();
    var items = new[] { new Item { Price = 10 }, new Item { Price = 20 } };
    
    // Act
    var total = calculator.CalculateTotal(items);
    
    // Assert
    Assert.Equal(30, total);
}
```

---

## Question 5: What is mocking? Explain the difference between stubs and mocks.

**Answer:**

**Mocking:** Creating fake objects that replace real dependencies in tests.

**Stubs:**
- Provide canned data/responses
- Passive - just return configured values
- Used to provide test data
```csharp
var stubRepo = new Mock<IUserRepository>();
stubRepo.Setup(r => r.GetById(1)).Returns(new User { Id = 1 });
```

**Mocks:**
- Also verify interactions
- Active - track method calls, parameters
- Used to verify behavior
```csharp
mockRepo.Verify(r => r.Save(It.IsAny<User>()), Times.Once);
```

**Use stubs** for data, **mocks** for behavior verification.

---

## Question 6: How does Moq work? Explain common Moq patterns and best practices.

**Answer:**

**Moq creates mock objects:**
```csharp
var mockService = new Mock<IUserService>();
```

**Setup return values:**
```csharp
mockService.Setup(s => s.GetUser(1)).Returns(new User { Id = 1 });
mockService.Setup(s => s.GetUser(It.IsAny<int>())).Returns(new User());
```

**Verify interactions:**
```csharp
mockService.Verify(s => s.Save(It.IsAny<User>()), Times.Once);
```

**Best practices:**
- Mock interfaces, not classes
- Use strict vs loose mode appropriately
- Don't over-mock - prefer real objects when possible
- Setup only what you need
- Use It.IsAny<T> for parameters you don't care about

---

## Question 7: What is FluentAssertions? How does it improve test readability?

**Answer:**

**FluentAssertions provides chainable assertions:**
```csharp
// Traditional
Assert.AreEqual(5, result.Count);
Assert.IsTrue(result.Success);
Assert.IsNotNull(result.Data);

// FluentAssertions
result.Should()
    .HaveCount(5)
    .And.Succeed()
    .And.NotBeNull();
```

**Benefits:**
- Readable: reads like English
- Better error messages
- Chainable, expressive
- Many extension methods

**Common usage:**
```csharp
actual.Should().Be(expected);
result.Should().Contain(item);
exception.Should().BeOfType<ArgumentException>();
```

---

## Question 8: Explain the difference between Shouldly, FluentAssertions, and traditional assertions.

**Answer:**

**Traditional Assert:**
```csharp
Assert.AreEqual(expected, actual);
Assert.IsTrue(condition);
Assert.IsNotNull(obj);
```
- Limited methods
- Harder to read
- Less descriptive errors

**FluentAssertions:**
```csharp
actual.Should().Be(expected);
```
- Chainable
- Clear, readable

**Shouldly:**
```csharp
actual.ShouldBe(expected);
```
- Similar to FluentAssertions
- Slightly different syntax
- Also provides better errors

**All improve readability over traditional.** Choose based on preference.

---

## Question 9: What is dependency injection in testing? How do you mock dependencies?

**Answer:**

**Dependency injection makes testing easier:**
- Dependencies injected, can be replaced
- Constructor injection preferred
- Easy to pass mocks

**Mocking with DI:**
```csharp
public class UserService {
    private readonly IUserRepository _repo;
    
    public UserService(IUserRepository repo) {
        _repo = repo;
    }
}

// In test
var mockRepo = new Mock<IUserRepository>();
var service = new UserService(mockRepo.Object);
```

**Benefits:**
- Test without real database
- Control test data
- Verify interactions
- Fast, isolated tests

---

## Question 10: How do you test async methods? What are common pitfalls?

**Answer:**

**Testing async methods:**
```csharp
[Fact]
public async Task GetAsync_ReturnsUser() {
    var service = new UserService();
    var user = await service.GetAsync(1);
    Assert.NotNull(user);
}
```

**For older frameworks or specific cases:**
```csharp
// Use .Result with caution
var user = service.GetAsync(1).Result;

// Or GetAwaiter
var user = service.GetAsync(1).GetAwaiter().GetResult();
```

**Common pitfalls:**
- Forgetting to await (fire and forget)
- Blocking on async code (deadlocks in sync context)
- Not handling exceptions in async code
- Using async void (can't catch exceptions)

**Best practices:** Always use `async Task` return type for tests.

---

*End of Section 8: Testing*

---

## 🎯 Quick Revision
- Focus on WHY + HOW  
- Practice explaining verbally  
- Think in real-world scenarios  

---

## 🎤 Interview Tip
> Don’t just answer — explain trade-offs and real-world usage.
