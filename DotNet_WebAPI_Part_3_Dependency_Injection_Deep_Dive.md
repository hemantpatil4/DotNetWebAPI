# .NET Core Web API — Part 3
# Dependency Injection (DI) Deep Dive

> **Goal:** Understand Dependency Injection from absolute zero: dependencies, `IServiceCollection`, `ServiceDescriptor`, registration vs resolution, `IServiceProvider`, scopes, Singleton/Scoped/Transient lifetimes, controller activation, dependency graphs, disposal, and practical Web API examples.

---

# 1. Why Do We Need Dependency Injection?

Suppose we have:

```csharp
public class UserController
{
    private UserService _userService;

    public UserController()
    {
        _userService = new UserService();
    }
}
```

This works, but the controller is responsible for creating its dependency.

Now imagine:

```text
UserController
      |
      v
UserService
      |
      v
UserRepository
      |
      v
Database
```

The controller might eventually need to do:

```csharp
public UserController()
{
    var repository = new UserRepository();
    var service = new UserService(repository);

    _userService = service;
}
```

The controller is now responsible for constructing an entire dependency graph.

Conceptually:

```text
Controller
    |
    +-- creates Service
          |
          +-- creates Repository
                |
                +-- creates database dependency
```

This creates tight coupling.

---

# 2. What Is a Dependency?

Suppose:

```csharp
public class UserController
{
    private readonly IUserService _service;

    public UserController(IUserService service)
    {
        _service = service;
    }
}
```

`UserController` depends on `IUserService`.

Why?

Because it cannot perform its responsibility without it.

So:

```text
UserController
       |
       | depends on
       v
IUserService
```

That is what a dependency is.

---

# 3. What Is Dependency Injection?

Instead of the controller creating its dependency:

```csharp
_service = new UserService();
```

the dependency is provided to the controller:

```csharp
public UserController(IUserService service)
{
    _service = service;
}
```

Compare:

## Manual creation

```text
Controller
    |
    | new
    v
UserService
```

## Dependency Injection

```text
Controller
    |
    | "I need IUserService"
    v
DI Container
    |
    v
UserService
```

Definition:

> **Dependency Injection means an object receives the dependencies it needs from an external mechanism instead of constructing those dependencies itself.**

---

# 4. Where Does ASP.NET Core Get the Dependency?

In `Program.cs`:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

This tells the DI system:

```text
When someone asks for:

IUserService

provide:

UserService
```

The controller declares:

```csharp
public UserController(IUserService service)
{
    _service = service;
}
```

The framework connects them.

Conceptually:

```text
Program.cs
     |
     v
builder.Services
     |
     | Register
     v
IUserService → UserService
     |
     v
Build()
     |
     v
IServiceProvider
     |
     v
HTTP Request
     |
     v
UserController needs IUserService
     |
     v
IServiceProvider
     |
     v
UserService
```

This is the fundamental DI flow.

---

# 5. `IServiceCollection`

Consider:

```csharp
builder.Services
```

Its conceptual type is:

```csharp
IServiceCollection
```

Think of it as a **collection of service registrations**.

Example:

```csharp
builder.Services.AddScoped<IUserService, UserService>();

builder.Services.AddScoped<IUserRepository, UserRepository>();

builder.Services.AddSingleton<ICache, RedisCache>();
```

Conceptually:

```text
IServiceCollection
---------------------------------------
Service              Implementation

IUserService         UserService
IUserRepository      UserRepository
ICache               RedisCache
---------------------------------------
```

There is also lifetime information associated with each registration.

For example:

```text
IUserService
    ↓
UserService
    ↓
Scoped
```

---

# 6. `ServiceDescriptor`

This is one level deeper.

A DI registration can be represented conceptually by a `ServiceDescriptor`.

For:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

think:

```text
ServiceDescriptor
--------------------------------
ServiceType:
    IUserService

ImplementationType:
    UserService

Lifetime:
    Scoped
--------------------------------
```

So:

```text
IServiceCollection
        |
        +-- ServiceDescriptor
        |
        +-- ServiceDescriptor
        |
        +-- ServiceDescriptor
        |
        +-- ...
```

The important idea:

> `IServiceCollection` is a collection of service registration descriptors.

---

# 7. Example of Multiple Registrations

Suppose:

```csharp
builder.Services.AddScoped<IUserService, UserService>();

builder.Services.AddTransient<IEmailService, EmailService>();

builder.Services.AddSingleton<ICache, MemoryCache>();
```

Conceptually:

```text
IServiceCollection

+-----------------------------------------+
| ServiceDescriptor                       |
|                                         |
| Service: IUserService                   |
| Implementation: UserService             |
| Lifetime: Scoped                         |
+-----------------------------------------+

+-----------------------------------------+
| ServiceDescriptor                       |
|                                         |
| Service: IEmailService                  |
| Implementation: EmailService            |
| Lifetime: Transient                     |
+-----------------------------------------+

+-----------------------------------------+
| ServiceDescriptor                       |
|                                         |
| Service: ICache                         |
| Implementation: MemoryCache             |
| Lifetime: Singleton                     |
+-----------------------------------------+
```

---

# 8. What Happens at `Build()`?

Before:

```csharp
builder.Build();
```

we have:

```text
IServiceCollection
```

containing registrations.

Then:

```csharp
var app = builder.Build();
```

builds the application and service-provider infrastructure.

Conceptually:

```text
IServiceCollection
       |
       | registrations
       v
     Build()
       |
       v
IServiceProvider
```

The provider can now resolve registered services.

---

# 9. What Is `IServiceProvider`?

`IServiceProvider` is the abstraction used to resolve services.

For example:

```csharp
var service =
    serviceProvider.GetRequiredService<IUserService>();
```

Conceptually:

```text
"I need IUserService"
        |
        v
IServiceProvider
        |
        v
Look at registrations
        |
        v
IUserService → UserService
        |
        v
Create / retrieve UserService
        |
        v
Return instance
```

Remember:

### `IServiceCollection`

Stores registrations.

### `IServiceProvider`

Resolves services.

---

# 10. Registration vs Resolution

This distinction is extremely important in interviews.

## Registration

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

Meaning:

> Tell the DI system how `IUserService` should be provided.

## Resolution

```csharp
GetRequiredService<IUserService>()
```

Meaning:

> Give me an instance of `IUserService`.

Full flow:

```text
Registration
      ↓
IServiceCollection
      ↓
Build()
      ↓
IServiceProvider
      ↓
Resolution
      ↓
Object instance
```

---

# 11. Does `AddScoped()` Create the Object?

No.

Consider:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

At registration time, the important thing is that the DI system now knows:

```text
IUserService
    →
UserService
    →
Scoped
```

A `UserService` object is not necessarily created at this exact line.

Later, when something asks for:

```text
IUserService
```

the DI system creates/reuses the appropriate instance according to the lifetime.

This distinction is crucial.

---

# 12. The Three Major DI Lifetimes

ASP.NET Core has three common service lifetimes:

```text
Singleton
Scoped
Transient
```

---

# 13. Singleton

Registration:

```csharp
builder.Services.AddSingleton<ICache, MemoryCache>();
```

Conceptually:

```text
Application
     |
     v
Root Service Provider
     |
     v
Singleton Instance
     |
     +----------+
     |          |
     v          v
Request 1    Request 2
     |          |
     +----------+
          |
          v
Same instance
```

A singleton is generally associated with the lifetime of the root service provider/application.

Example:

```text
Request 1 → Cache instance A

Request 2 → Cache instance A

Request 3 → Cache instance A

Request 4 → Cache instance A
```

Important:

> Because the same object can be used by concurrent requests, singleton services must be designed for thread safety when they contain mutable state.

---

# 14. Scoped

Registration:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

For a normal Web API request, think:

> One instance per request scope.

Example:

```text
Request 1
    |
    +-- UserService A

Request 2
    |
    +-- UserService B

Request 3
    |
    +-- UserService C
```

But within the same scope:

```text
Request 1
    |
    +-- Controller
    |
    +-- Service A
    |
    +-- Another component
           |
           +-- same Service A
```

Important:

> Scoped does NOT mean "new object every time someone asks."

It means:

> One instance within a particular scope.

---

# 15. Transient

Registration:

```csharp
builder.Services.AddTransient<IEmailService, EmailService>();
```

Conceptually:

```text
Resolve #1
     ↓
EmailService A

Resolve #2
     ↓
EmailService B

Resolve #3
     ↓
EmailService C
```

Normally, each resolution creates a new instance.

---

# 16. Compare the Three Lifetimes

```text
                Singleton      Scoped       Transient

Lifetime        Application    Request      Each resolution
                /root scope    scope

Request 1       A              A            A/B/C...
Request 2       A              B            D/E/F...
Request 3       A              C            G/H/I...
```

Simpler:

```text
Singleton:

App
 |
 +----------------------+
 |                      |
 R1                     R2
 |                      |
 +------ Same A --------+


Scoped:

R1 → A

R2 → B

R3 → C


Transient:

Resolve → A
Resolve → B
Resolve → C
```

---

# 17. Why Is `DbContext` Usually Scoped?

This is a common interview question.

Typically:

```csharp
builder.Services.AddDbContext<AppDbContext>();
```

EF Core registers `DbContext` as scoped by default.

Why?

A `DbContext` represents a unit of work around database operations and is generally intended to be used within a request scope.

Conceptually:

```text
HTTP Request
     |
     v
DbContext
     |
     +-- Query
     +-- Query
     +-- Update
     +-- SaveChanges
     |
     v
Request ends
     |
     v
DbContext disposed
```

A single `DbContext` should not be concurrently shared across unrelated requests.

---

# 18. What Is a Scope?

A scope defines a lifetime boundary for scoped services.

Conceptually:

```text
Application
     |
     v
Root Service Provider
     |
     +-------------------+
     |                   |
     v                   v
 Request 1 Scope      Request 2 Scope
     |                   |
     v                   v
Scoped instances      Scoped instances
```

More detailed:

```text
Root Provider
      |
      +-- Singleton A
      |
      +-- Request Scope 1
      |       |
      |       +-- UserService A
      |       +-- DbContext A
      |
      +-- Request Scope 2
              |
              +-- UserService B
              +-- DbContext B
```

---

# 19. Why Does a Web Request Have a Scope?

ASP.NET Core creates a request scope so scoped dependencies have a meaningful lifetime.

Conceptually:

```text
HTTP Request starts
       |
       v
Create request scope
       |
       v
Resolve scoped services
       |
       v
Execute controller
       |
       v
Generate response
       |
       v
Request scope ends
       |
       v
Dispose scoped services
```

This is why scoped services naturally work well in Web APIs.

---

# 20. Example With Multiple Dependencies

Suppose:

```csharp
public class UsersController : ControllerBase
{
    private readonly IUserService _service;
    private readonly IAuditService _audit;

    public UsersController(
        IUserService service,
        IAuditService audit)
    {
        _service = service;
        _audit = audit;
    }
}
```

Registrations:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddScoped<IAuditService, AuditService>();
```

Request:

```text
GET /api/users/10
```

Conceptually:

```text
Request
   |
   v
Request Scope
   |
   +------------------+
   |                  |
   v                  v
IUserService       IAuditService
   |                  |
   v                  v
UserService        AuditService
   |                  |
   +------------------+
            |
            v
       UsersController
```

---

# 21. Dependency Graph

Suppose:

```csharp
public class UserService : IUserService
{
    private readonly IUserRepository _repository;

    public UserService(IUserRepository repository)
    {
        _repository = repository;
    }
}
```

Repository:

```csharp
public class UserRepository : IUserRepository
{
    private readonly AppDbContext _db;

    public UserRepository(AppDbContext db)
    {
        _db = db;
    }
}
```

Registrations:

```csharp
builder.Services.AddScoped<IUserService, UserService>();

builder.Services.AddScoped<IUserRepository, UserRepository>();

builder.Services.AddDbContext<AppDbContext>();
```

Dependency graph:

```text
UsersController
       |
       v
IUserService
       |
       v
UserService
       |
       v
IUserRepository
       |
       v
UserRepository
       |
       v
AppDbContext
       |
       v
Database
```

The DI container must resolve this graph.

---

# 22. What Happens When the Controller Is Requested?

Suppose:

```http
GET /api/users/10
```

Routing identifies:

```text
UsersController.GetUser
```

ASP.NET Core needs a:

```text
UsersController
```

The constructor says:

```csharp
public UsersController(IUserService service)
```

So DI must resolve:

```text
IUserService
```

The container finds:

```text
IUserService → UserService
```

But `UserService` needs:

```text
IUserRepository
```

So DI resolves:

```text
IUserRepository → UserRepository
```

which needs:

```text
AppDbContext
```

So the conceptual construction process is:

```text
UsersController
      |
      v
IUserService
      |
      v
UserService
      |
      v
IUserRepository
      |
      v
UserRepository
      |
      v
AppDbContext
```

The container recursively resolves the dependency graph.

---

# 23. Why Is DI Powerful?

Without DI:

```csharp
var db = new AppDbContext(...);

var repository = new UserRepository(db);

var service = new UserService(repository);

var controller = new UsersController(service);
```

The caller has to understand the entire construction graph.

With DI:

```csharp
public UsersController(IUserService service)
```

and:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddDbContext<AppDbContext>();
```

the application declares the graph and the DI system handles construction.

---

# 24. Interface vs Implementation

Why register:

```text
IUserService → UserService
```

instead of making the controller depend directly on:

```text
UserService
```

Because we want the consumer to depend on an abstraction.

Controller:

```csharp
public UsersController(IUserService service)
```

doesn't need to know the internal implementation.

It only needs the contract:

```text
IUserService
```

This supports:

- Loose coupling
- Testing
- Substitution
- Dependency inversion
- SOLID principles

---

# 25. Testing Benefit

Production:

```text
IUserService → UserService
```

Unit test:

```text
IUserService → MockUserService
```

The controller does not need to change.

Example:

```csharp
var mockService = new Mock<IUserService>();
```

Then the test can inject the mock into:

```csharp
new UsersController(mockService.Object);
```

This is one of the major practical benefits of DI.

---

# 26. Multiple Registrations

Suppose:

```csharp
builder.Services.AddScoped<IMessageSender, EmailSender>();

builder.Services.AddScoped<IMessageSender, SmsSender>();
```

There are now multiple registrations for the same service type.

For a normal single-service resolution:

```csharp
IMessageSender
```

the DI behavior follows its registration rules; the last registration is generally the one returned for a single-service resolution.

If you need all implementations:

```csharp
IEnumerable<IMessageSender>
```

the DI system can provide the registered implementations.

Conceptually:

```text
IMessageSender
      |
      +-- EmailSender
      |
      +-- SmsSender
```

This is useful for strategy/plugin-style designs.

---

# 27. Open Generic Registrations

DI can also register generic implementations.

Example:

```csharp
builder.Services.AddScoped(
    typeof(IRepository<>),
    typeof(Repository<>));
```

Then:

```text
IRepository<User>
      ↓
Repository<User>

IRepository<Order>
      ↓
Repository<Order>
```

The DI system can construct the appropriate closed generic implementation.

This is useful for generic repository or infrastructure patterns.

---

# 28. Singleton + Scoped — Critical Rule

Suppose:

```csharp
builder.Services.AddSingleton<MySingleton>();
builder.Services.AddScoped<MyScoped>();
```

and:

```csharp
public class MySingleton
{
    public MySingleton(MyScoped scoped)
    {
    }
}
```

This is generally invalid/problematic.

Why?

Because:

```text
Singleton lifetime
        >
Scoped lifetime
```

The singleton lives much longer than a request scope.

Conceptually:

```text
Singleton
   |
   +---- Scoped service
           |
           v
       Request ends
           |
           v
      Scoped disposed
           |
           v
Singleton still alive
```

This creates a captive dependency problem.

ASP.NET Core DI validation can detect such lifetime violations in appropriate configurations.

---

# 29. Lifetime Rule to Remember

A useful interview rule:

```text
Singleton
   ↓
Should generally depend only on services
safe for singleton lifetime.

Scoped
   ↓
Can depend on scoped/transient services,
subject to the complete dependency graph.

Transient
   ↓
Can depend on other services,
but the dependency graph still must obey
lifetime constraints.
```

The most important rule:

> **A singleton should not directly depend on a scoped service.**

---

# 30. Disposal

DI also participates in object lifetime management.

Suppose:

```csharp
public class MyService : IDisposable
{
    public void Dispose()
    {
        Console.WriteLine("Disposed");
    }
}
```

If DI creates and owns the service, it can dispose of it when the appropriate lifetime scope ends.

## Scoped

```text
Request starts
    |
    v
Create scoped service
    |
    v
Use service
    |
    v
Request ends
    |
    v
Dispose scoped service
```

## Singleton

```text
Application starts
    |
    v
Create singleton
    |
    v
Application runs
    |
    v
Application shuts down
    |
    v
Dispose singleton
```

This is another reason DI is more than simply "automatic constructor injection."

---

# 31. Root Provider vs Scope

Important distinction:

```text
                  Root IServiceProvider
                           |
             +-------------+-------------+
             |                           |
             v                           v
       Singleton                       Scope 1
                                           |
                                           +-- Scoped A
                                           +-- Scoped B

                                       Scope 2
                                           |
                                           +-- Scoped C
                                           +-- Scoped D
```

The root provider manages application-level services.

Request scopes manage request-scoped services.

---

# 32. `IServiceScopeFactory`

You may encounter:

```csharp
IServiceScopeFactory
```

It can be used to create a scope manually:

```csharp
using var scope = serviceScopeFactory.CreateScope();

var service =
    scope.ServiceProvider.GetRequiredService<IMyService>();
```

This is useful in scenarios such as:

- Background services
- Hosted services
- Explicit unit-of-work boundaries

Why?

Because a background service does not automatically have an HTTP request scope.

---

# 33. Avoid Service Locator Style

You technically can inject:

```csharp
IServiceProvider
```

and manually resolve dependencies:

```csharp
var service =
    _provider.GetRequiredService<IUserService>();
```

But this is generally undesirable as the main application style.

It is commonly associated with the **Service Locator** pattern.

Prefer explicit constructor dependencies:

```csharp
public MyClass(IUserService service)
{
    _service = service;
}
```

Why?

Because dependencies become visible in the class constructor.

---

# 34. Constructor Injection

Constructor injection is the most common DI style in ASP.NET Core.

Example:

```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;

    public OrderService(IOrderRepository repository)
    {
        _repository = repository;
    }
}
```

Registration:

```csharp
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
```

Conceptual resolution:

```text
OrderService constructor
        |
        v
IOrderRepository required
        |
        v
DI container
        |
        v
OrderRepository
```

---

# 35. Complete DI Flow

Connect everything:

```text
                Program.cs
                    |
                    v
        builder.Services.Add...
                    |
                    v
            IServiceCollection
                    |
                    | registrations
                    v
                  Build()
                    |
                    v
            IServiceProvider
                    |
                    v
             HTTP Request
                    |
                    v
              Request Scope
                    |
                    v
             Routing / Endpoint
                    |
                    v
          Controller Activation
                    |
                    v
          Constructor Dependency
                    |
                    v
             DI Resolution
                    |
                    v
           Dependency Graph
                    |
          +---------+---------+
          |                   |
          v                   v
      Service              Repository
                              |
                              v
                          DbContext
```

---

# 36. Real Project Example — FX Domain

Suppose a real application has:

```text
FXController
    ↓
FXService
    ↓
FXRepository
    ↓
AppDbContext
```

Registrations:

```csharp
builder.Services.AddScoped<IFXService, FXService>();

builder.Services.AddScoped<IFXRepository, FXRepository>();

builder.Services.AddDbContext<AppDbContext>();

builder.Services.AddControllers();
```

Controller:

```csharp
[ApiController]
[Route("api/fx")]
public class FXController : ControllerBase
{
    private readonly IFXService _service;

    public FXController(IFXService service)
    {
        _service = service;
    }

    [HttpGet("rates")]
    public async Task<IActionResult> GetRates()
    {
        var rates = await _service.GetRatesAsync();

        return Ok(rates);
    }
}
```

Conceptual request flow:

```text
GET /api/fx/rates
        |
        v
FXController required
        |
        v
Needs IFXService
        |
        v
DI
        |
        v
FXService
        |
        v
Needs IFXRepository
        |
        v
DI
        |
        v
FXRepository
        |
        v
Needs AppDbContext
        |
        v
DI
        |
        v
AppDbContext
        |
        v
Database
```

This is the exact kind of dependency graph you encounter in enterprise Web APIs.

---

# 37. The Three Most Important DI Objects

## `IServiceCollection`

Purpose:

> Registration.

```text
"Here are the services and their lifetimes."
```

---

## `IServiceProvider`

Purpose:

> Resolution.

```text
"Give me the service I need."
```

---

## `IServiceScope`

Purpose:

> Defines a lifetime boundary for scoped services.

```text
"These scoped service instances belong to this scope."
```

---

# 38. Final Mental Model

```text
              SERVICE REGISTRATION

builder.Services
       |
       v
IServiceCollection
       |
       +-- IUserService → UserService → Scoped
       +-- IUserRepo    → UserRepo    → Scoped
       +-- ICache       → Cache       → Singleton
       +-- IEmail       → Email       → Transient
       |
       v
     Build()
       |
       v
IServiceProvider
       |
       v
Application starts
       |
       v
HTTP Request
       |
       v
Request Scope
       |
       v
Controller
       |
       v
"I need IUserService"
       |
       v
IServiceProvider
       |
       v
UserService
       |
       v
"I need IUserRepository"
       |
       v
UserRepository
       |
       v
"I need AppDbContext"
       |
       v
AppDbContext
```

---

# 39. Interview Cheat Sheet

| Concept | Meaning |
|---|---|
| `IServiceCollection` | Collection of DI registrations |
| `ServiceDescriptor` | Describes a service registration |
| `IServiceProvider` | Resolves registered services |
| Registration | Tells DI how a service should be provided |
| Resolution | Obtains a service instance |
| Singleton | One instance for the root/application lifetime |
| Scoped | One instance per scope; normally one per HTTP request |
| Transient | New instance per resolution |
| `IServiceScope` | Lifetime boundary for scoped services |
| Constructor injection | Dependency supplied through constructor |
| `IServiceScopeFactory` | Creates scopes manually |
| `AddScoped<T>` | Registers a scoped service |
| `AddSingleton<T>` | Registers a singleton |
| `AddTransient<T>` | Registers a transient |

---

# 40. Interview Diagram

```text
                   Program.cs
                       |
                       v
              builder.Services
                       |
                       v
              IServiceCollection
                       |
                  registrations
                       |
                       v
                     Build()
                       |
                       v
                IServiceProvider
                       |
                       v
                 HTTP Request
                       |
                       v
                 Request Scope
                       |
                       v
                   Controller
                       |
              constructor asks for
                       |
                       v
                 IUserService
                       |
                       v
                DI Resolution
                       |
                       v
                  UserService
                       |
                       v
               IUserRepository
                       |
                       v
                UserRepository
                       |
                       v
                 AppDbContext
                       |
                       v
                    Database
```

---

# 41. Core Takeaway

The complete DI mechanism can be reduced to:

```text
REGISTER
   ↓
IServiceCollection
   ↓
BUILD
   ↓
IServiceProvider
   ↓
CREATE SCOPE
   ↓
RESOLVE
   ↓
CREATE / REUSE INSTANCE
   ↓
INJECT INTO CONSTRUCTOR
   ↓
USE
   ↓
DISPOSE WHEN LIFETIME ENDS
```

And the three lifetimes:

```text
Singleton
    → Application/root lifetime

Scoped
    → One instance per scope
    → Normally one per HTTP request

Transient
    → New instance per resolution
```

---

# 42. Next Part

## Part 4 — DI Internals & Lifetimes

We will go deeper into:

```text
IServiceCollection
        ↓
ServiceDescriptor
        ↓
ServiceProvider
        ↓
Root Provider
        ↓
ServiceScope
        ↓
Scoped Cache
        ↓
Resolution
        ↓
Dependency Graph
        ↓
Object Creation
        ↓
Disposal
```

And we will answer:

- What exactly is stored inside `IServiceCollection`?
- What does `AddScoped<T>()` actually register?
- How does the DI container construct the dependency graph?
- What exactly is the root provider?
- How is a request scope created?
- Why do two controllers in the same request get the same scoped service?
- What happens across two different requests?
- How does disposal work?
- What is a captive dependency?
- Why can't Singleton normally depend on Scoped?
- How does `IEnumerable<T>` resolution work?
- How do open generic registrations work?
- What happens when dependencies form a circular graph?
- What happens when a service is not registered?
- How does controller activation connect to DI internally?

