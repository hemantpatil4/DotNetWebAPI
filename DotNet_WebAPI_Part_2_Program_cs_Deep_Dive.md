# .NET Core Web API — Part 2
## `Program.cs` Deep Dive

> **Goal:** Understand what actually happens around `WebApplication.CreateBuilder(args)`, `builder`, `builder.Services`, `Build()`, `IServiceProvider`, `app`, and `app.Run()`.

---

# 1. The Modern `Program.cs`

A typical modern ASP.NET Core Web API starts with:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

var app = builder.Build();

app.MapControllers();

app.Run();
```

Do not look at this as one continuous block.

Think of it as phases:

```text
PHASE 1 — BUILD CONFIGURATION
        |
        v
WebApplication.CreateBuilder(args)
        |
        v
PHASE 2 — REGISTER SERVICES
        |
        v
builder.Services.Add...
        |
        v
PHASE 3 — BUILD APPLICATION
        |
        v
builder.Build()
        |
        v
PHASE 4 — CONFIGURE PIPELINE
        |
        v
app.Use...
app.Map...
        |
        v
PHASE 5 — RUN
        |
        v
app.Run()
```

The main focus of this part is:

```text
CreateBuilder()
      ↓
builder
      ↓
builder.Services
      ↓
Build()
      ↓
app
```

---

# 2. `WebApplication.CreateBuilder(args)`

The first line is:

```csharp
var builder = WebApplication.CreateBuilder(args);
```

This single line establishes a significant amount of ASP.NET Core infrastructure.

At a high level:

```text
CreateBuilder(args)
        |
        +---- Configuration
        |
        +---- Logging
        |
        +---- Environment
        |
        +---- Host
        |
        +---- Web server configuration
        |
        +---- IServiceCollection
        |
        v
WebApplicationBuilder
```

The returned object is a:

```csharp
WebApplicationBuilder
```

Therefore:

```csharp
var builder = ...
```

is conceptually:

```csharp
WebApplicationBuilder builder = ...
```

---

# 3. What is `WebApplicationBuilder`?

`WebApplicationBuilder` is a high-level builder provided by ASP.NET Core for creating a web application.

Conceptually:

```text
                    WebApplicationBuilder
                             |
        +--------------------+--------------------+
        |                    |                    |
        v                    v                    v
   Configuration          Logging              Services
        |                    |                    |
        v                    v                    v
 IConfiguration          ILogger             IServiceCollection

                             |
                             v
                           Host
                             |
                             v
                       Web Application
```

Important:

> `WebApplicationBuilder` is an orchestration/configuration object. It is not the running server.

---

# 4. Why Do We Need a Builder?

A web application requires many pieces of infrastructure:

```text
Configuration
Logging
Host
Dependency Injection
Web server
Environment
Application lifetime
```

Without a convenient builder, application startup would require significantly more manual construction and wiring.

The modern hosting model lets us begin with:

```csharp
var builder = WebApplication.CreateBuilder(args);
```

The framework establishes sensible defaults, after which we customize what we need.

Conceptually:

```text
Create Host
Create Configuration
Create Logger
Create Service Collection
Configure Environment
Configure Web Hosting
        |
        v
WebApplicationBuilder
```

---

# 5. What Does `args` Mean?

The `args` variable represents command-line arguments supplied when the application starts.

Conceptually:

```text
Operating System
       |
       v
Application arguments
       |
       v
Program.cs
       |
       v
args
```

For example, an application can be started with command-line arguments such as:

```bash
dotnet MyWebApi.dll --environment Production
```

The exact interpretation depends on the configured hosting/configuration infrastructure.

The key point:

> Command-line arguments can participate in application configuration and hosting.

---

# 6. What Gets Prepared by `CreateBuilder()`?

One of the most important concepts is that `CreateBuilder()` establishes infrastructure around several major areas:

```text
                    Application
                       Builder
                         |
       +-----------------+------------------+
       |                 |                  |
       v                 v                  v
 Configuration        Logging            Services
       |                 |                  |
       v                 v                  v
 IConfiguration       ILogger         IServiceCollection
                         |
                         v
                        Host
                         |
                         v
                    Web Hosting
```

Let's examine them individually.

---

# 7. Configuration

The builder exposes application configuration through:

```csharp
builder.Configuration
```

Conceptually its type is:

```csharp
IConfiguration
```

Think of configuration as a collection of sources/providers:

```text
builder
   |
   +---- Configuration
             |
             +---- appsettings.json
             +---- appsettings.{Environment}.json
             +---- Environment variables
             +---- Command-line arguments
             +---- Other providers
```

Example:

```json
{
  "ConnectionStrings": {
    "Default": "Server=..."
  }
}
```

The application can access configuration through:

```csharp
builder.Configuration
```

---

# 8. Configuration Is Not Just `appsettings.json`

A common beginner misconception is:

```text
Configuration = appsettings.json
```

That is incomplete.

ASP.NET Core configuration can combine multiple providers.

Conceptually:

```text
appsettings.json
        +
appsettings.Development.json
        +
Environment Variables
        +
Command-line arguments
        +
Other providers
        |
        v
     IConfiguration
```

Provider ordering and precedence determine which value wins when multiple sources define the same key.

A dedicated configuration section will cover this in detail.

---

# 9. Logging

The builder also establishes logging infrastructure.

You can access builder logging configuration through:

```csharp
builder.Logging
```

Application services can later receive:

```csharp
ILogger<T>
```

Example:

```csharp
public class UserService
{
    private readonly ILogger<UserService> _logger;

    public UserService(ILogger<UserService> logger)
    {
        _logger = logger;
    }

    public void Process()
    {
        _logger.LogInformation("Processing user");
    }
}
```

Conceptually:

```text
builder
   |
   v
Logging configuration
   |
   v
Logging providers
   |
   v
ILogger<T>
```

Later we will cover:

- Log levels
- Logging providers
- Structured logging
- Logging scopes
- Correlation IDs
- Production logging

---

# 10. Environment

The builder also exposes information about the current hosting environment:

```csharp
builder.Environment
```

Conceptually:

```text
builder.Environment
       |
       +---- EnvironmentName
       |
       +---- ContentRootPath
       |
       +---- WebRootPath
       |
       +---- IsDevelopment()
       +---- IsProduction()
       +---- ...
```

For example:

```csharp
if (builder.Environment.IsDevelopment())
{
    // Development-specific behavior
}
```

A common example is:

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

Typical environments include:

```text
Development
Staging
Production
```

---

# 11. What Is a Host?

A Host provides foundational infrastructure for running a .NET application.

Conceptually:

```text
Host
 |
 +---- Application lifetime
 |
 +---- Dependency Injection
 |
 +---- Configuration
 |
 +---- Logging
 |
 +---- Environment
 |
 +---- Hosted services
 |
 +---- Server hosting
```

The .NET Generic Host is a foundational architecture used by modern .NET applications.

ASP.NET Core adds web-specific hosting capabilities around this foundation.

---

# 12. Generic Host vs Web Hosting

Historically, ASP.NET Core code often exposed concepts such as:

```text
IWebHost
WebHostBuilder
Startup
```

Modern ASP.NET Core applications use the **Generic Host** architecture.

Conceptually:

```text
             .NET Generic Host
                    |
          +---------+---------+
          |         |         |
          v         v         v
     Configuration Logging    DI
                              |
                              v
                         Application
                              |
                              v
                         ASP.NET Core
                              |
                              v
                           Kestrel
```

The Generic Host provides general application infrastructure.

ASP.NET Core adds the web-specific hosting and HTTP pipeline.

---

# 13. `builder.Services`

Now we reach one of the most important properties:

```csharp
builder.Services
```

Its conceptual type is:

```csharp
IServiceCollection
```

Think of it as a collection of dependency-injection registrations.

Example:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

Conceptually:

```text
IUserService
       |
       v
UserService
```

You are telling the DI system how to provide `IUserService`.

---

# 14. Registration Does Not Mean Immediate Object Creation

Consider:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

A beginner may think:

> "This creates a `UserService` object."

That is not the right mental model.

You are primarily registering information describing:

```text
Service:
    IUserService

Implementation:
    UserService

Lifetime:
    Scoped
```

Conceptually:

```text
IServiceCollection

+------------------------------------+
| Service registration               |
|                                    |
| Service: IUserService              |
| Implementation: UserService        |
| Lifetime: Scoped                   |
+------------------------------------+
```

The actual instance is normally created when the DI system resolves the service within an appropriate scope.

---

# 15. Three Major DI Lifetimes

Typical registrations include:

```csharp
builder.Services.AddSingleton<ICache, Cache>();

builder.Services.AddScoped<IUserService, UserService>();

builder.Services.AddTransient<IEmailSender, EmailSender>();
```

They represent different lifetime behaviors.

## Singleton

One instance is generally associated with the application's root service-provider lifetime.

```text
Application
    |
    +---- Singleton instance
    |
    +---- Request 1
    +---- Request 2
    +---- Request 3
```

## Scoped

In a normal Web API request pipeline, one instance is generally created per request scope.

```text
Request 1
   |
   +-- UserService instance A

Request 2
   |
   +-- UserService instance B
```

## Transient

A new instance is generally created each time the service is requested.

```text
Resolve #1 → Instance A
Resolve #2 → Instance B
Resolve #3 → Instance C
```

The full lifetime behavior, scopes, disposal and validation will be covered in the dedicated DI section.

---

# 16. `AddControllers()`

Consider:

```csharp
builder.Services.AddControllers();
```

This registers the controller/MVC infrastructure required for controller-based APIs.

Conceptually:

```text
AddControllers()
      |
      +---- Controller infrastructure
      +---- Model binding
      +---- Validation infrastructure
      +---- Formatting
      +---- MVC services
      +---- Controller activation support
      +---- Other framework services
```

It does not simply mean:

> "Register my controller classes."

It sets up the framework infrastructure needed to discover, activate and execute controller-based endpoints.

---

# 17. What Does `Build()` Do?

Now:

```csharp
var app = builder.Build();
```

is a major transition.

Conceptually:

```text
                 builder
                    |
       +------------+------------+
       |            |            |
       v            v            v
 Configuration   Logging      Services
                                |
                                v
                       IServiceCollection
                                |
                              Build()
                                |
              +-----------------+----------------+
              |                 |                |
              v                 v                v
        ServiceProvider      Host             Web App
              |                 |                |
              +-----------------+----------------+
                                |
                                v
                         WebApplication
```

The exact internal implementation is more detailed, but this is the correct high-level mental model.

---

# 18. `IServiceCollection` → `IServiceProvider`

This is a critical transition.

Before `Build()`:

```text
IServiceCollection
```

contains service registrations.

After the application is built, those registrations are used to construct the service-provider infrastructure used for service resolution.

Conceptually:

```text
IServiceCollection
       |
       | "How should services be registered?"
       |
       v
     Build
       |
       v
IServiceProvider
       |
       | "Give me an instance of this service"
       |
       v
Service instance
```

Example:

```csharp
var service = serviceProvider.GetRequiredService<IUserService>();
```

The provider examines the registered service information and resolves the required dependency.

---

# 19. Registration vs Resolution

This distinction is fundamental.

## Registration

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

Means:

> Register the dependency.

## Resolution

Later, something requests:

```text
IUserService
```

The DI infrastructure resolves it:

```text
Controller asks for IUserService
              |
              v
       IServiceProvider
              |
              v
       UserService instance
```

Full conceptual flow:

```text
REGISTRATION

IUserService
      |
      v
UserService
      |
      v
Scoped


             ↓ BUILD ↓


RESOLUTION

Controller asks for IUserService
              |
              v
       IServiceProvider
              |
              v
       UserService instance
```

---

# 20. Why Does the Controller Not Manually Create Services?

Without DI:

```csharp
public class UsersController : ControllerBase
{
    private readonly UserService _service;

    public UsersController()
    {
        _service = new UserService();
    }
}
```

This creates tight coupling.

With DI:

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
builder.Services.AddScoped<IUserService, UserService>();
```

The framework can resolve the dependency.

Conceptually:

```text
HTTP Request
      |
      v
Controller activation
      |
      v
Constructor requires IUserService
      |
      v
DI infrastructure
      |
      v
UserService
      |
      v
Controller created
```

---

# 21. What Happens to the Host During `Build()`?

`Build()` constructs the infrastructure required for the configured application.

Conceptually:

```text
builder.Build()
      |
      +---- Configuration
      |
      +---- Logging
      |
      +---- DI
      |
      +---- Environment
      |
      +---- Hosting
      |
      +---- Web server configuration
      |
      v
Application
```

Do not reduce `Build()` mentally to:

```csharp
new WebApplication()
```

There is substantially more hosting and dependency-injection infrastructure involved.

---

# 22. What Happens After `Build()`?

After:

```csharp
var app = builder.Build();
```

we are now in the application/pipeline configuration phase.

For example:

```csharp
app.UseHttpsRedirection();

app.UseAuthentication();

app.UseAuthorization();

app.MapControllers();
```

The important distinction is:

```text
BEFORE Build()

builder.Services.Add...
builder.Configuration...
builder.Logging...

        |
        v
      Build()

        |
        v

AFTER Build()

app.Use...
app.Map...
app.Run()
```

---

# 23. Builder Phase vs Application Phase

A useful interview diagram:

```text
              BUILDER PHASE
                   |
                   v
       WebApplicationBuilder
                   |
        +----------+----------+
        |          |          |
        v          v          v
   Services   Configuration Logging
        |
        v
      Build()
        |
        v
             APPLICATION PHASE
                   |
                   v
             WebApplication
                   |
        +----------+----------+
        |          |          |
        v          v          v
     Use...      Map...     Run()
```

---

# 24. `app.Use...` vs `app.Map...`

A first introduction to the distinction:

## `Use`

Generally participates in the middleware request pipeline.

Examples:

```csharp
app.UseHttpsRedirection();

app.UseAuthentication();

app.UseAuthorization();
```

Conceptually:

```text
Use...
   |
   v
Middleware pipeline
```

## `Map`

Maps endpoints or application branches.

Example:

```csharp
app.MapControllers();
```

Conceptually:

```text
Map...
   |
   v
Endpoint mapping
```

The exact routing/endpoint mechanics will be covered later.

---

# 25. What Happens When `app.Run()` Executes?

Think of:

```csharp
app.Run();
```

as the transition from application configuration into application execution.

Conceptually:

```text
Program.cs
    |
    v
CreateBuilder
    |
    v
Configure
    |
    v
Build
    |
    v
Configure Pipeline
    |
    v
app.Run()
    |
    v
Host starts
    |
    v
Kestrel starts/listens
    |
    v
Application waits for requests
```

---

# 26. Request Arrives

Suppose:

```http
GET /api/users/10
```

arrives.

A simplified flow is:

```text
Client
  |
  v
Network
  |
  v
Kestrel
  |
  v
ASP.NET Core pipeline
  |
  v
Middleware
  |
  v
Routing
  |
  v
Endpoint
  |
  v
Controller activation
  |
  v
DI Resolution
  |
  v
Controller action
```

Notice:

> Dependency Injection is not merely a startup concept.

It becomes important during request processing when framework/application components need their dependencies resolved.

---

# 27. Controller Activation

Suppose:

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

When ASP.NET Core needs to execute this controller, it must create a controller instance.

Conceptually:

```text
Routing identifies UsersController.GetUser
                |
                v
Need UsersController instance
                |
                v
Constructor requires IUserService
                |
                v
DI provider resolves IUserService
                |
                v
Creates/provides UserService
                |
                v
Creates UsersController
                |
                v
Calls GetUser()
```

This connects the startup configuration:

```text
builder.Services.AddScoped<IUserService, UserService>();
```

to request-time behavior.

---

# 28. A Deeper Picture of `CreateBuilder()`

You can think of:

```csharp
WebApplication.CreateBuilder(args)
```

as preparing several foundational objects and configurations:

```text
                  CreateBuilder(args)
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
 IConfiguration     IServiceCollection  Logging
        |                |                |
        v                v                v
 Configuration       Registrations      Providers
                         |
                         v
                        Host
                         |
                         v
                  Web infrastructure
```

This is a conceptual model, not a literal claim that every internal operation occurs in exactly this sequence.

The actual framework implementation is more nuanced.

---

# 29. Why `WebApplication.CreateBuilder()` Is Convenient

Without the modern builder abstraction, developers would need to deal with more hosting setup and wiring.

Modern ASP.NET Core lets us begin with:

```csharp
var builder = WebApplication.CreateBuilder(args);
```

and gives the application sensible defaults.

The minimal startup file hides much of the ceremony while the underlying infrastructure still includes:

```text
Host
Configuration
Logging
DI
Environment
Web server
Application lifetime
Request pipeline
```

---

# 30. Minimal Hosting Model

Modern ASP.NET Core uses a **minimal hosting model**.

A traditional application might expose:

```text
Program.cs
    |
    v
CreateHostBuilder()
    |
    v
Startup
    |
    +-- ConfigureServices()
    |
    +-- Configure()
```

Modern applications commonly use:

```text
Program.cs
    |
    v
WebApplication.CreateBuilder()
    |
    v
builder.Services
    |
    v
builder.Build()
    |
    v
app.Use...
app.Map...
app.Run()
```

Minimal hosting does **not** mean the framework itself has little infrastructure.

It means:

> The application developer has less startup boilerplate.

---

# 31. Old vs Modern Mental Model

### Older style

```text
Program
   |
   v
HostBuilder
   |
   v
Startup
   |
   +---- ConfigureServices()
   |
   +---- Configure()
```

### Modern style

```text
Program.cs
   |
   v
WebApplication.CreateBuilder()
   |
   v
builder.Services
   |
   v
builder.Build()
   |
   v
app.Use...
app.Map...
app.Run()
```

You may encounter the older style in enterprise/legacy systems, so understanding both is useful.

---

# 32. The Most Important Internal Relationship

Memorize this:

```text
IServiceCollection
        |
        | registrations
        v
      Build()
        |
        v
IServiceProvider
        |
        | resolution
        v
Service instances
```

Example:

```text
builder.Services
       |
       | AddScoped<IUserService, UserService>()
       v
IServiceCollection
       |
       | Build
       v
IServiceProvider
       |
       | Resolve IUserService
       v
UserService instance
```

This is the foundation for understanding dependency injection.

---

# 33. Full `Program.cs` Mental Model

```csharp
var builder = WebApplication.CreateBuilder(args);
```

### Meaning

```text
Create/configure the application's hosting infrastructure.
```

---

```csharp
builder.Services.AddControllers();
```

### Meaning

```text
Register controller/MVC infrastructure into IServiceCollection.
```

---

```csharp
var app = builder.Build();
```

### Meaning

```text
Build the configured application and its supporting infrastructure,
including the service-provider infrastructure.
```

---

```csharp
app.MapControllers();
```

### Meaning

```text
Map controller actions as endpoints.
```

---

```csharp
app.Run();
```

### Meaning

```text
Start the application and process incoming requests.
```

---

# 34. Full Startup Flow

```text
                 OPERATING SYSTEM
                        |
                        v
                 Start process
                        |
                        v
                    Program.cs
                        |
                        v
           CreateBuilder(args)
                        |
                        v
             WebApplicationBuilder
                        |
        +---------------+----------------+
        |               |                |
        v               v                v
 Configuration       Logging          Services
        |                                |
        |                         IServiceCollection
        |                                |
        +---------------+----------------+
                        |
                        v
                      Build()
                        |
        +---------------+----------------+
        |               |                |
        v               v                v
 ServiceProvider      Host          WebApplication
        |                                |
        +--------------------------------+
                        |
                        v
                 app.Use / app.Map
                        |
                        v
                    app.Run()
                        |
                        v
                     Kestrel
                        |
                        v
                  WAIT FOR REQUEST
```

---

# 35. Request Processing

```text
HTTP Request
     |
     v
   Kestrel
     |
     v
 HttpContext
     |
     v
Middleware Pipeline
     |
     v
Routing
     |
     v
Endpoint
     |
     v
Controller Activation
     |
     v
DI Resolution
     |
     v
Controller Action
     |
     v
Service
     |
     v
Repository
     |
     v
Database
```

Response:

```text
Database
   |
   v
Repository
   |
   v
Service
   |
   v
Controller
   |
   v
Result
   |
   v
Middleware
   |
   v
Kestrel
   |
   v
Client
```

---

# 36. Interview Questions

## Q1. What is `WebApplicationBuilder`?

It is the builder used by the modern ASP.NET Core minimal hosting model to configure the application's host, services, configuration, logging, environment and web-hosting infrastructure before building the application.

---

## Q2. What is `IServiceCollection`?

It is a collection of dependency-injection service registrations.

---

## Q3. Is `IServiceCollection` the DI container?

No.

It stores registrations.

The built application creates the service-provider infrastructure used to resolve services.

---

## Q4. What is `IServiceProvider`?

It provides the mechanism for resolving registered services and managing their lifetimes.

---

## Q5. What is the difference between registration and resolution?

Registration:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

Resolution:

```text
Give me IUserService
        |
        v
IServiceProvider
        |
        v
UserService instance
```

---

## Q6. Why do we call `Build()`?

To construct the configured application and its supporting hosting and dependency-injection infrastructure.

---

## Q7. Why are `builder.Services` calls before `builder.Build()`?

Because service registrations need to be included when the application's service-provider infrastructure is constructed.

---

## Q8. What is `builder.Configuration`?

The application's configuration abstraction, typically backed by multiple configuration providers.

---

## Q9. What is `builder.Environment`?

It exposes information about the application's hosting environment, paths and environment name such as Development or Production.

---

## Q10. What is `app`?

The built `WebApplication` used to configure the request pipeline, endpoints and application execution.

---

# 37. One Diagram to Memorize

```text
              WebApplication.CreateBuilder(args)
                              |
                              v
                   WebApplicationBuilder
                              |
          +-------------------+-------------------+
          |                   |                   |
          v                   v                   v
   Configuration           Logging            Services
          |                                       |
          |                                IServiceCollection
          |                                       |
          +-------------------+-------------------+
                              |
                              v
                            Build()
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
       IServiceProvider      Host       WebApplication
              |                               |
              |                               v
              |                         app.Use(...)
              |                         app.Map(...)
              |                               |
              +-------------------------------+
                              |
                              v
                           app.Run()
                              |
                              v
                           Kestrel
                              |
                              v
                       HTTP Requests
                              |
                              v
                       Middleware
                              |
                              v
                         Routing
                              |
                              v
                         Controller
                              |
                              v
                      DI Resolution
```

---

# 38. What We Have Not Learned Yet

We have intentionally not gone deep into:

```text
IServiceCollection
        ↓
ServiceDescriptor
        ↓
ServiceProvider
        ↓
Scope
        ↓
Singleton / Scoped / Transient
```

That is the next major topic.

We also have not deeply covered:

```text
Kestrel
   ↓
HttpContext
   ↓
Middleware
   ↓
RequestDelegate
   ↓
next()
```

That belongs to the middleware section.

We also haven't yet gone through:

```text
Request
   ↓
Routing
   ↓
Endpoint
   ↓
Controller activation
   ↓
Model binding
   ↓
Validation
   ↓
Action execution
```

These will be built one piece at a time.

---

# 39. Part 2 — Core Takeaway

If you remember only this:

```text
CreateBuilder()
       ↓
Configure infrastructure
       ↓
builder.Services
       ↓
Register dependencies
       ↓
Build()
       ↓
Create the application + DI/service-provider infrastructure
       ↓
app.Use(...)
app.Map(...)
       ↓
app.Run()
       ↓
Kestrel
       ↓
HTTP requests
```

then you have the basic architecture behind modern `Program.cs`.

---

# 40. Next Part

## Part 3 — Dependency Injection From Absolute Scratch

We will start from:

```text
IServiceCollection
```

and build the complete picture:

```text
IServiceCollection
       |
       v
ServiceDescriptor
       |
       v
Registration
       |
       v
Build()
       |
       v
IServiceProvider
       |
       v
ServiceScope
       |
       v
Service resolution
       |
       +---- Singleton
       +---- Scoped
       +---- Transient
       |
       v
Controller activation
```

We will also answer:

- What exactly is stored inside `IServiceCollection`?
- What is a `ServiceDescriptor`?
- What does `AddScoped<T>()` actually register?
- When is an object actually created?
- How does the container find the implementation?
- What is the root provider?
- What is a scope?
- Why is `DbContext` normally scoped?
- What happens when two classes request the same scoped service?
- What happens across two HTTP requests?
- What happens when a service implements `IDisposable`?
- Why can't a singleton normally depend on a scoped service?
- How does controller constructor injection work?

This is where DI starts becoming truly clear.
