# .NET Web API — Part 4
# Dependency Injection Internals & Lifetimes — Deep Dive

> This part goes deeper into **how the built-in .NET DI container actually behaves** after you register services.
>
> The goal is not just to memorize `AddScoped`, `AddSingleton`, and `AddTransient`, but to understand:
>
> **IServiceCollection → ServiceDescriptor → Build() → IServiceProvider → Root Provider → Request Scope → Resolution → Dependency Graph → Object Creation → Caching → Disposal**

---

# 1. Where Part 4 Fits

In the previous part, we learned:

```text
builder.Services.AddScoped<IUserService, UserService>();
```

At a high level:

```text
IServiceCollection
      ↓
Service registrations
      ↓
Build()
      ↓
IServiceProvider
      ↓
Resolve dependencies
      ↓
Create objects
```

Part 4 asks:

> What exactly happens between those steps?

---

# 2. The Three Most Important DI Objects

Keep these three objects clearly separated.

```text
IServiceCollection
        ↓
     Registration
        ↓
Build()
        ↓
IServiceProvider
        ↓
    Resolution
        ↓
 IServiceScope
```

## 2.1 IServiceCollection

`IServiceCollection` is where registrations are stored.

Example:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddSingleton<ICache, MemoryCache>();
```

Conceptually:

```text
IServiceCollection
 ├── IUserService → UserService → Scoped
 ├── IUserRepository → UserRepository → Scoped
 └── ICache → MemoryCache → Singleton
```

Important:

> `IServiceCollection` is primarily a collection of service registrations. It is not the runtime object resolver.

---

# 3. ServiceDescriptor

Internally, registrations are represented by `ServiceDescriptor` objects.

Conceptually:

```csharp
ServiceDescriptor
{
    ServiceType,
    ImplementationType,
    Lifetime
}
```

For:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

the registration is conceptually:

```text
ServiceType:
    IUserService

ImplementationType:
    UserService

Lifetime:
    Scoped
```

So:

```text
IUserService
     ↓
UserService
     ↓
Scoped
```

---

# 4. Registration Does NOT Mean Object Creation

This is one of the most important concepts.

When you write:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

.NET does NOT normally do:

```csharp
var service = new UserService();
```

at that line.

Instead it records:

```text
"If someone asks for IUserService,
create UserService using Scoped lifetime."
```

So:

```text
Registration
    ≠
Object creation
```

This distinction is extremely important in interviews.

---

# 5. What Build() Changes

Consider:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IUserService, UserService>();

var app = builder.Build();
```

Before:

```text
IServiceCollection
```

contains registrations.

After:

```csharp
builder.Build();
```

ASP.NET Core builds the application's runtime infrastructure, including the DI service provider.

Conceptually:

```text
IServiceCollection
       │
       │ Build()
       ↓
IServiceProvider
```

The provider is now capable of resolving services.

---

# 6. IServiceProvider

`IServiceProvider` is the runtime abstraction used to resolve services.

Example:

```csharp
var service = serviceProvider.GetRequiredService<IUserService>();
```

The provider looks at the registrations and determines:

```text
IUserService
      ↓
UserService
      ↓
Scoped
```

Then it resolves the dependencies required by `UserService`.

---

# 7. Dependency Graph

Suppose we have:

```csharp
public class UsersController : ControllerBase
{
    private readonly IUserService _service;

    public UsersController(IUserService service)
    {
        _service = service;
    }
}
```

And:

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

And:

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

The dependency graph is:

```text
UsersController
      ↓
 IUserService
      ↓
 UserService
      ↓
 IUserRepository
      ↓
 UserRepository
      ↓
 AppDbContext
      ↓
 Database
```

DI walks this graph recursively.

---

# 8. Recursive Resolution

Suppose ASP.NET Core needs:

```text
UsersController
```

The controller requires:

```text
IUserService
```

DI asks:

```text
Who implements IUserService?
```

Answer:

```text
UserService
```

Now DI asks:

```text
What does UserService require?
```

Answer:

```text
IUserRepository
```

Then:

```text
Who implements IUserRepository?
```

Answer:

```text
UserRepository
```

Then:

```text
What does UserRepository require?
```

Answer:

```text
AppDbContext
```

Eventually:

```text
AppDbContext
    ↓
UserRepository
    ↓
UserService
    ↓
UsersController
```

Then the controller can be created.

---

# 9. Constructor Injection

This is the normal DI mechanism.

```csharp
public class FXController : ControllerBase
{
    private readonly IFXService _fxService;

    public FXController(IFXService fxService)
    {
        _fxService = fxService;
    }
}
```

You do NOT normally write:

```csharp
var fxService = new FXService();
```

Instead:

```text
Controller
    ↓
asks DI for IFXService
    ↓
DI finds FXService
    ↓
DI creates FXService
    ↓
DI injects it into controller
```

---

# 10. Request Scope

Now we reach one of the most important concepts.

In ASP.NET Core, an HTTP request normally gets its own DI scope.

Conceptually:

```text
Application
│
└── Root IServiceProvider
       │
       ├── Request 1 Scope
       │
       ├── Request 2 Scope
       │
       └── Request 3 Scope
```

Each request gets its own scope.

---

# 11. Why Does ASP.NET Core Need Scopes?

Consider:

```csharp
builder.Services.AddScoped<UserService>();
```

Suppose two requests arrive:

```text
Request A
Request B
```

We normally want:

```text
Request A
    ↓
UserService instance A

Request B
    ↓
UserService instance B
```

We do NOT generally want both requests sharing the same scoped object.

Therefore:

```text
Request A Scope
    UserService A

Request B Scope
    UserService B
```

---

# 12. Scoped Lifetime

Registration:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

Meaning:

> One instance per DI scope.

In ASP.NET Core, the normal scope corresponds to one HTTP request.

Therefore:

```text
Request 1
    ↓
Scope 1
    ↓
UserService Instance 1

Request 2
    ↓
Scope 2
    ↓
UserService Instance 2
```

---

# 13. Very Important: Same Scoped Instance Within One Request

Suppose:

```csharp
builder.Services.AddScoped<MyService>();
```

Now inside one request:

```csharp
var a = provider.GetRequiredService<MyService>();
var b = provider.GetRequiredService<MyService>();
```

For the same scope:

```text
a ─────┐
       ├── same MyService instance
b ─────┘
```

Conceptually:

```text
Scope
  │
  └── MyService Instance
          ↑
          ├── Resolution A
          └── Resolution B
```

This is why scoped services behave like per-request instances.

---

# 14. Different Requests Get Different Scoped Instances

Request 1:

```text
Scope 1
  ↓
MyService Instance A
```

Request 2:

```text
Scope 2
  ↓
MyService Instance B
```

Therefore:

```text
Instance A != Instance B
```

even though both were resolved from the same registration.

---

# 15. Scoped Instance Cache — Mental Model

A useful conceptual model is:

```text
Root Provider
    │
    ├── Singleton cache
    │
    └── Scope 1
          │
          └── Scoped instance cache
```

When a scoped service is resolved:

```text
Does this scope already have the service?
        │
     ┌──┴──┐
    Yes    No
     │      │
     │      ↓
     │   Create instance
     │      ↓
     │   Store in scope
     │      ↓
     └── Return instance
```

This explains why repeated resolution inside one scope gives the same instance.

This is a conceptual model rather than a promise about the exact internal data structures used by every runtime version.

---

# 16. Singleton Lifetime

Registration:

```csharp
builder.Services.AddSingleton<ICache, MemoryCache>();
```

Meaning:

> One instance for the application's root DI container lifetime.

Conceptually:

```text
Root Provider
     │
     └── Cache Instance
          ↑
          ├── Request 1
          ├── Request 2
          ├── Request 3
          └── Request N
```

All requests can resolve the same singleton instance.

---

# 17. Singleton Example

```csharp
public class ApplicationCache
{
    public Dictionary<string, string> Values { get; } = new();
}
```

Register:

```csharp
builder.Services.AddSingleton<ApplicationCache>();
```

Then:

```text
Request A ──┐
Request B ──┼──> Same ApplicationCache
Request C ──┘
```

This means singleton mutable state must be designed carefully.

Multiple requests can execute concurrently.

Therefore, this can be dangerous:

```csharp
public Dictionary<string, string> Values { get; }
```

if multiple threads modify it without appropriate synchronization.

Use thread-safe structures where appropriate:

```csharp
ConcurrentDictionary<string, string>
```

or proper locking/coordination.

---

# 18. Transient Lifetime

Registration:

```csharp
builder.Services.AddTransient<MyService>();
```

Meaning:

> A new instance is generally created each time the service is resolved.

Conceptually:

```text
Resolve #1 → Instance A
Resolve #2 → Instance B
Resolve #3 → Instance C
```

Unlike scoped:

```text
Scoped:
same scope → same instance
```

Transient:

```text
resolution → new instance
```

---

# 19. Comparing the Three Lifetimes

| Lifetime | Conceptual lifetime | Typical ASP.NET Core behavior |
|---|---|---|
| Singleton | Root application lifetime | One instance |
| Scoped | DI scope | One instance per HTTP request |
| Transient | Resolution | New instance per resolution |

Mental picture:

```text
Singleton
────────────────────────────
Application lifetime
      │
      └── One instance


Scoped
────────────────────────────
Request 1 → Instance A
Request 2 → Instance B
Request 3 → Instance C


Transient
────────────────────────────
Resolve 1 → A
Resolve 2 → B
Resolve 3 → C
```

---

# 20. Why DbContext Is Usually Scoped

Typical registration:

```csharp
builder.Services.AddDbContext<AppDbContext>();
```

`DbContext` is normally registered as scoped.

Why?

Because a request commonly represents a unit of work:

```text
HTTP Request
    ↓
Service
    ↓
Repository
    ↓
DbContext
    ↓
Database
```

You generally want the same context within the request's work.

Example:

```text
Request
   │
   ├── UserService
   │      ↓
   │   UserRepository
   │      ↓
   │   DbContext
   │
   └── OrderRepository
          ↓
       DbContext
```

The repositories can resolve the same scoped `DbContext` instance within that request.

---

# 21. Why DbContext Should Not Normally Be Singleton

Imagine:

```text
Request A
     │
     └── DbContext X
             ↑
Request B ────┘
```

Now unrelated concurrent requests share a database context.

This creates serious lifecycle and concurrency problems.

The normal model is:

```text
Request A → DbContext A
Request B → DbContext B
```

---

# 22. Captive Dependency

One of the most important DI interview topics.

Suppose:

```csharp
builder.Services.AddSingleton<SingletonService>();
builder.Services.AddScoped<ScopedService>();
```

And:

```csharp
public class SingletonService
{
    private readonly ScopedService _service;

    public SingletonService(ScopedService service)
    {
        _service = service;
    }
}
```

Dependency graph:

```text
SingletonService
       ↓
ScopedService
```

This is problematic.

Why?

Singleton:

```text
Application lifetime
```

Scoped service:

```text
Request lifetime
```

The singleton would try to hold a dependency that belongs to a shorter-lived scope.

This is called a:

> Captive dependency.

---

# 23. Lifetime Direction Rule

A useful mental rule:

```text
Long-lived service
      ↓
should not directly capture
      ↓
shorter-lived service
```

Think:

```text
Singleton
   ↓
Scoped     ❌

Singleton
   ↓
Transient  ⚠️
```

Transient dependencies can sometimes be valid, but if the singleton captures the transient instance in its constructor, the transient effectively becomes held for the singleton's lifetime.

Therefore the important question is:

> Does a long-lived service retain an object whose intended lifetime is shorter?

---

# 24. Scope Hierarchy

Think of the DI hierarchy like:

```text
Root Provider
│
├── Singleton objects
│
├── Scope A
│    ├── Scoped A1
│    └── Scoped A2
│
├── Scope B
│    ├── Scoped B1
│    └── Scoped B2
│
└── Scope C
     ├── Scoped C1
     └── Scoped C2
```

Singletons belong to the root lifetime.

Scoped objects belong to a scope.

Transient objects generally don't get reused simply because the same type was previously resolved.

---

# 25. Manually Creating a Scope

Sometimes code executes outside an HTTP request.

Example:

```text
Background Service
Hosted Service
Worker
Scheduled job
```

There may not be a normal HTTP request scope.

You can create a scope:

```csharp
using var scope = scopeFactory.CreateScope();

var service =
    scope.ServiceProvider.GetRequiredService<MyScopedService>();
```

Example:

```csharp
public class Worker
{
    private readonly IServiceScopeFactory _scopeFactory;

    public Worker(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public async Task ExecuteAsync()
    {
        using var scope = _scopeFactory.CreateScope();

        var service =
            scope.ServiceProvider
                  .GetRequiredService<MyScopedService>();

        await service.RunAsync();
    }
}
```

Flow:

```text
Worker
  ↓
CreateScope()
  ↓
New Scope
  ↓
Resolve ScopedService
  ↓
Use service
  ↓
Dispose scope
  ↓
Dispose scoped dependencies
```

---

# 26. Why Background Services Need Special Attention

A hosted service is commonly registered as a singleton.

For example:

```csharp
builder.Services.AddHostedService<MyWorker>();
```

So this is dangerous:

```text
Singleton Worker
      ↓
Scoped Service
```

Instead:

```text
Singleton Worker
      ↓
IServiceScopeFactory
      ↓
CreateScope()
      ↓
Scoped Service
```

This creates the scoped dependency only for the intended unit of work.

---

# 27. IServiceProvider vs IServiceScopeFactory

`IServiceProvider`:

> Resolve services.

`IServiceScopeFactory`:

> Create a new DI scope.

Example:

```csharp
var scope = scopeFactory.CreateScope();
```

Then:

```csharp
var service = scope.ServiceProvider
                   .GetRequiredService<MyService>();
```

---

# 28. IServiceProvider as Service Locator

Technically, you can write:

```csharp
public class MyController
{
    private readonly IServiceProvider _provider;

    public MyController(IServiceProvider provider)
    {
        _provider = provider;
    }

    public void Execute()
    {
        var service =
            _provider.GetRequiredService<MyService>();
    }
}
```

This works.

But this is usually discouraged for normal application dependencies.

Prefer:

```csharp
public MyController(MyService service)
{
    _service = service;
}
```

Why?

Constructor injection makes dependencies explicit.

Bad:

```text
Controller
   ↓
IServiceProvider
   ↓
???
   ↓
MyService
```

Better:

```text
Controller
   ↓
MyService
```

The dependency graph is visible.

---

# 29. Multiple Registrations

Suppose:

```csharp
builder.Services.AddScoped<INotification, EmailNotification>();
builder.Services.AddScoped<INotification, SmsNotification>();
```

There are now two registrations:

```text
INotification → EmailNotification
INotification → SmsNotification
```

When resolving a single:

```csharp
GetRequiredService<INotification>()
```

the built-in container's normal behavior is that the last registration wins.

Conceptually:

```text
INotification
      ↓
SmsNotification
```

---

# 30. Resolving All Implementations

If you want all implementations:

```csharp
IEnumerable<INotification>
```

Then:

```csharp
var notifications =
    provider.GetRequiredService<IEnumerable<INotification>>();
```

You get:

```text
[
    EmailNotification,
    SmsNotification
]
```

This is extremely useful for strategy-style designs.

---

# 31. Real FX Example — Multiple Strategies

Suppose:

```csharp
public interface ISpreadStrategy
{
    decimal CalculateSpread(decimal rate);
}
```

Implementations:

```csharp
public class RetailSpreadStrategy : ISpreadStrategy
{
    public decimal CalculateSpread(decimal rate)
    {
        return rate * 0.001m;
    }
}
```

```csharp
public class InstitutionalSpreadStrategy : ISpreadStrategy
{
    public decimal CalculateSpread(decimal rate)
    {
        return rate * 0.0005m;
    }
}
```

Register:

```csharp
builder.Services.AddScoped<ISpreadStrategy, RetailSpreadStrategy>();
builder.Services.AddScoped<ISpreadStrategy, InstitutionalSpreadStrategy>();
```

Then:

```csharp
public class SpreadResolver
{
    private readonly IEnumerable<ISpreadStrategy> _strategies;

    public SpreadResolver(IEnumerable<ISpreadStrategy> strategies)
    {
        _strategies = strategies;
    }
}
```

DI supplies both implementations.

This pattern is useful for:

```text
FX
 ├── Retail spread
 ├── Institutional spread
 ├── VIP spread
 └── Corporate spread
```

---

# 32. Open Generic Registration

Suppose:

```csharp
public interface IRepository<T>
{
    Task<T?> GetAsync(int id);
}
```

Implementation:

```csharp
public class Repository<T> : IRepository<T>
{
    public Task<T?> GetAsync(int id)
    {
        // ...
        return Task.FromResult<T?>(default);
    }
}
```

Instead of registering every type:

```csharp
AddScoped<IRepository<User>, Repository<User>>();
AddScoped<IRepository<Order>, Repository<Order>>();
AddScoped<IRepository<Product>, Repository<Product>>();
```

you can register the generic mapping:

```csharp
builder.Services.AddScoped(
    typeof(IRepository<>),
    typeof(Repository<>));
```

Now conceptually:

```text
IRepository<User>
      ↓
Repository<User>

IRepository<Order>
      ↓
Repository<Order>

IRepository<Product>
      ↓
Repository<Product>
```

This is called an:

> Open generic registration.

---

# 33. Unregistered Dependency

Suppose:

```csharp
public class UserService
{
    public UserService(IUserRepository repository)
    {
    }
}
```

But you forgot:

```csharp
builder.Services.AddScoped<IUserRepository, UserRepository>();
```

When DI tries to construct `UserService`, it cannot resolve:

```text
IUserRepository
```

The application/request will fail with a DI resolution exception.

Conceptually:

```text
UserService
     ↓
IUserRepository
     ↓
❌ No registration
```

This is why the entire dependency graph must be resolvable.

---

# 34. Circular Dependencies

Suppose:

```csharp
class A
{
    public A(B b) { }
}
```

and:

```csharp
class B
{
    public B(A a) { }
}
```

Graph:

```text
A
↓
B
↓
A
↓
B
↓
...
```

This is a circular dependency.

The built-in DI container cannot construct the graph normally and will report a circular dependency error.

---

# 35. Why Circular Dependencies Are Usually a Design Smell

Suppose:

```text
FXService
    ↓
OrderService
    ↓
FXService
```

This usually indicates the responsibilities are too tightly coupled.

Better design may be:

```text
FXService ─────┐
               ↓
        Shared Component
               ↑
               │
OrderService ──┘
```

or introduce an orchestration/application service.

---

# 36. Disposal

DI also manages the lifecycle of many disposable objects.

Suppose:

```csharp
public class MyService : IDisposable
{
    public void Dispose()
    {
        // cleanup
    }
}
```

If DI creates and owns the instance, it can dispose it when its lifetime ends.

Conceptually:

```text
Create
  ↓
Use
  ↓
Lifetime ends
  ↓
Dispose
```

For scoped:

```text
Request begins
    ↓
Scoped service created
    ↓
Request executes
    ↓
Request ends
    ↓
Scope disposed
    ↓
Scoped IDisposable disposed
```

---

# 37. Singleton Disposal

Singleton:

```text
Application starts
      ↓
Singleton created
      ↓
Application runs
      ↓
Application shuts down
      ↓
Singleton disposed
```

The exact disposal behavior depends on whether the DI container owns the instance.

---

# 38. Transient Disposal

Transient disposable objects are also managed by DI when DI creates them, but their disposal behavior is different from scoped/singleton caching because they aren't retained for the same lifetime cache.

The important interview-level point is:

> Let DI create disposable dependencies when possible so their lifecycle can be managed correctly.

Avoid manually creating DI-managed disposable objects with `new` and then expecting the container to dispose them.

---

# 39. `new` vs DI

Consider:

```csharp
public class UserService
{
}
```

Bad in a DI-based application when `UserService` has dependencies:

```csharp
var service = new UserService();
```

You bypass the container.

Better:

```csharp
var service =
    provider.GetRequiredService<UserService>();
```

Or, preferably, constructor injection:

```csharp
public class Controller
{
    private readonly UserService _service;

    public Controller(UserService service)
    {
        _service = service;
    }
}
```

---

# 40. Full ASP.NET Core Request DI Flow

This is the complete mental model to remember.

```text
Application Startup
       │
       ↓
WebApplication.CreateBuilder()
       │
       ↓
builder.Services
       │
       ├── AddScoped(...)
       ├── AddSingleton(...)
       └── AddTransient(...)
       │
       ↓
builder.Build()
       │
       ↓
IServiceProvider
       │
       ↓
Application starts
       │
       ↓
HTTP Request
       │
       ↓
ASP.NET Core creates request scope
       │
       ↓
Routing
       │
       ↓
Controller activation
       │
       ↓
DI resolves controller dependencies
       │
       ↓
Dependency graph traversal
       │
       ↓
Create/reuse objects according to lifetime
       │
       ↓
Controller action executes
       │
       ↓
Response
       │
       ↓
Request scope ends
       │
       ↓
Scoped disposable objects disposed
```

---

# 41. Example — Complete FX Dependency Graph

Registration:

```csharp
builder.Services.AddScoped<IFXService, FXService>();
builder.Services.AddScoped<IFXRepository, FXRepository>();
builder.Services.AddDbContext<FXDbContext>();
```

Classes:

```csharp
public class FXController : ControllerBase
{
    private readonly IFXService _service;

    public FXController(IFXService service)
    {
        _service = service;
    }
}
```

```csharp
public class FXService : IFXService
{
    private readonly IFXRepository _repository;

    public FXService(IFXRepository repository)
    {
        _repository = repository;
    }
}
```

```csharp
public class FXRepository : IFXRepository
{
    private readonly FXDbContext _db;

    public FXRepository(FXDbContext db)
    {
        _db = db;
    }
}
```

Graph:

```text
FXController
     │
     ↓
 IFXService
     │
     ↓
 FXService
     │
     ↓
 IFXRepository
     │
     ↓
 FXRepository
     │
     ↓
 FXDbContext
     │
     ↓
 SQL Server
```

During one HTTP request:

```text
Request
  │
  └── Scope
       │
       ├── FXService
       │
       ├── FXRepository
       │
       └── FXDbContext
```

The scoped objects belong to that request scope.

---

# 42. Important Lifetime Example

Consider:

```csharp
builder.Services.AddSingleton<A>();
builder.Services.AddScoped<B>();
builder.Services.AddTransient<C>();
```

And:

```text
A → B
B → C
```

The dependency chain is:

```text
Singleton A
     ↓
Scoped B
     ↓
Transient C
```

This is problematic because:

```text
A lives for application lifetime
B lives for request lifetime
```

`A` cannot safely capture `B`.

Correct architecture might be:

```text
Singleton A
     ↓
IServiceScopeFactory
     ↓
Create Scope
     ↓
Scoped B
     ↓
Transient C
```

---

# 43. Singleton + Thread Safety

A common interview trap:

> Is Singleton thread-safe?

Answer:

> No. Singleton lifetime does not automatically make an object thread-safe.

If many requests use the same singleton:

```text
Request 1 ──┐
Request 2 ──┼──> Singleton
Request 3 ──┘
```

they may access it concurrently.

Therefore mutable singleton state needs appropriate synchronization.

For example:

```csharp
ConcurrentDictionary<string, decimal>
```

may be more appropriate than:

```csharp
Dictionary<string, decimal>
```

for concurrent writes.

---

# 44. Singleton and Configuration

Singleton is commonly appropriate for objects that are:

- stateless
- immutable
- thread-safe
- expensive to create
- application-wide

Examples can include:

```text
Configuration-related components
Caches
Stateless helpers
Application-wide clients/factories
```

But lifetime should always follow actual behavior rather than simply "this looks reusable."

---

# 45. Scoped Is Often the Default Business-Service Choice

For many Web API business services:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddScoped<IFXService, FXService>();
```

Why?

Because these services commonly participate in request-level operations and depend on scoped resources such as:

```text
DbContext
Repositories
Request-specific state
```

This is a common pattern, not an absolute rule.

---

# 46. Transient Is Useful for Lightweight Services

Transient can be appropriate when:

```text
Object is cheap to create
Object has little/no state
Object doesn't need request-level reuse
```

Example:

```csharp
builder.Services.AddTransient<IEmailFormatter, EmailFormatter>();
```

Every resolution can produce a fresh formatter.

---

# 47. `ValidateScopes`

During development, you can enable scope validation.

Conceptually:

```csharp
builder.Host.UseDefaultServiceProvider(options =>
{
    options.ValidateScopes = true;
    options.ValidateOnBuild = true;
});
```

This can help detect invalid DI configurations early.

For example:

```text
Singleton
   ↓
Scoped
```

can be detected during validation.

Exact defaults can vary by environment/runtime configuration, so explicit validation is useful when you want predictable startup checks.

---

# 48. ValidateOnBuild

`ValidateOnBuild` asks the service provider to validate registrations when building the provider.

This can catch certain problems earlier instead of waiting until a request actually resolves the service.

Conceptually:

```text
Application startup
       ↓
Build provider
       ↓
Validate registrations
       ↓
Error early if invalid
```

This is especially useful in larger applications.

---

# 49. Key Mental Model

Remember:

```text
IServiceCollection
        │
        │ stores registrations
        ↓
ServiceDescriptor
        │
        │ service type
        │ implementation
        │ lifetime
        ↓
Build()
        │
        ↓
IServiceProvider
        │
        ├── Root lifetime
        │
        └── Creates scopes
                 │
                 ↓
           Scoped resolution
                 │
                 ↓
          Dependency graph
                 │
                 ↓
          Object construction
                 │
                 ↓
             Disposal
```

---

# 50. Interview Questions

## Q1. Does AddScoped create an object immediately?

No.

It registers a service mapping and lifetime.

Example:

```csharp
AddScoped<IUserService, UserService>();
```

Object creation occurs when the service is resolved.

---

## Q2. What is IServiceCollection?

It is the collection used to register services and their lifetimes/configuration before the service provider is built.

---

## Q3. What is IServiceProvider?

It is the runtime service resolver used to obtain registered dependencies.

---

## Q4. What is a scope?

A scope is a DI lifetime boundary.

In ASP.NET Core, an HTTP request normally has its own scope.

---

## Q5. How many instances of a scoped service are created?

Typically:

```text
One instance per scope
```

Therefore:

```text
One per HTTP request
```

for normal Web API request processing.

---

## Q6. What happens if you resolve a scoped service twice in one request?

Normally both resolutions return the same scoped instance.

---

## Q7. What is a singleton?

One instance associated with the root/application DI lifetime.

---

## Q8. Is singleton automatically thread-safe?

No.

The object must itself be designed for concurrent access.

---

## Q9. What is a captive dependency?

A longer-lived service captures a shorter-lived dependency.

Classic example:

```text
Singleton → Scoped
```

---

## Q10. Why should a BackgroundService not directly depend on a scoped service?

A hosted service is commonly singleton-like, while the dependency is request/scope-based.

Instead:

```text
BackgroundService
      ↓
IServiceScopeFactory
      ↓
CreateScope()
      ↓
Resolve scoped service
```

---

## Q11. What happens with multiple registrations?

For a single service resolution, the last registration normally wins.

For:

```csharp
IEnumerable<T>
```

all registrations are returned in registration order.

---

## Q12. What is an open generic registration?

Example:

```csharp
AddScoped(
    typeof(IRepository<>),
    typeof(Repository<>));
```

It allows DI to construct:

```text
IRepository<User>
IRepository<Order>
IRepository<Product>
```

from the generic mapping.

---

## Q13. What happens if a dependency is not registered?

When the container attempts to construct the object, resolution fails because the dependency cannot be resolved.

---

## Q14. What happens with circular dependencies?

The container detects the circular dependency and fails to construct the graph.

---

# 51. One-Minute Revision

If you only remember one section, remember this:

```text
IServiceCollection
    =
registration list

ServiceDescriptor
    =
service type + implementation + lifetime

Build()
    =
build application/service-provider infrastructure

IServiceProvider
    =
runtime resolver

Scope
    =
lifetime boundary

Singleton
    =
one instance for root/application lifetime

Scoped
    =
one instance per scope/request

Transient
    =
new instance per resolution

Constructor Injection
    =
dependencies supplied by DI

Captive Dependency
    =
long-lived service holding shorter-lived dependency

IEnumerable<T>
    =
all registrations

Open Generic
    =
generic registration pattern

Dispose
    =
container manages owned disposable instances according to their lifetime
```

---

# 52. Final Mental Picture

The entire DI system can be visualized as:

```text
                    APPLICATION STARTUP
                           │
                           ↓
                WebApplication.CreateBuilder
                           │
                           ↓
                   IServiceCollection
                           │
             ┌─────────────┼─────────────┐
             ↓             ↓             ↓
          Scoped        Singleton      Transient
             │             │             │
             └─────────────┼─────────────┘
                           ↓
                         Build()
                           │
                           ↓
                   IServiceProvider
                           │
                     Root Provider
                           │
              ┌────────────┼────────────┐
              ↓            ↓            ↓
           Request 1    Request 2    Request 3
             Scope        Scope        Scope
              │            │            │
              ↓            ↓            ↓
        Scoped objects Scoped objects Scoped objects
              │            │            │
              └────────────┼────────────┘
                           ↓
                  Dependency Resolution
                           │
                           ↓
                  Dependency Graph
                           │
                           ↓
                    Object Creation
                           │
                           ↓
                    Controller Action
                           │
                           ↓
                         Response
                           │
                           ↓
                     Scope Disposal
```

The most important transition to understand is:

```text
Registration
    ↓
Build
    ↓
Provider
    ↓
Scope
    ↓
Resolution
    ↓
Object graph
    ↓
Execution
    ↓
Disposal
```

That is the foundation required before moving into middleware, request delegates, controller activation, and the ASP.NET Core request pipeline.
