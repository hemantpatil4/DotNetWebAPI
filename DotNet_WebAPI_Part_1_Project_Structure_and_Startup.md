# .NET Core Web API — Part 1
## Project Structure & Startup Foundation

> **Goal:** Build a strong mental model of what a modern ASP.NET Core Web API project contains, how the application starts, and how an HTTP request eventually reaches a controller.

---

## 1. What is a .NET Web API?

A .NET Web API is a **server application** that:

1. Starts a process.
2. Opens a network endpoint.
3. Listens for HTTP requests.
4. Processes those requests.
5. Executes application code.
6. Produces HTTP responses.

High-level flow:

```text
Client
   |
   | GET /api/users/10
   v
.NET Web API
   |
   |-- Controller
   |-- Service
   |-- Repository
   |-- Database
   |
   v
HTTP Response
```

Example request:

```http
GET /api/users/10
```

Example response:

```json
{
  "id": 10,
  "name": "Hemant"
}
```

A critical distinction:

> When the application starts, the controller is not "running." The application process is running. A controller is instantiated/executed when a matching request reaches its endpoint.

---

# 2. Creating a Web API Project

A typical project can be created with:

```bash
dotnet new webapi -n MyWebApi
cd MyWebApi
dotnet run
```

You may see:

```text
Building...
info: Microsoft.Hosting.Lifetime
      Now listening on: https://localhost:7001

info: Microsoft.Hosting.Lifetime
      Now listening on: http://localhost:5001
```

At this point:

> The .NET application is a running operating-system process that is listening for HTTP requests.

---

# 3. Typical Project Structure

A basic project may look like:

```text
MyWebApi/
│
├── Controllers/
│   └── WeatherForecastController.cs
│
├── Properties/
│   └── launchSettings.json
│
├── appsettings.json
├── appsettings.Development.json
├── Program.cs
├── MyWebApi.csproj
└── MyWebApi.http
```

A real enterprise application may evolve into:

```text
MyWebApi/
│
├── Controllers/
├── Services/
├── Repositories/
├── Models/
├── DTOs/
├── Middleware/
├── Validators/
├── Extensions/
├── Infrastructure/
├── Configuration/
│
├── Program.cs
├── appsettings.json
├── appsettings.Production.json
└── MyWebApi.csproj
```

Do not assume every folder is required by ASP.NET Core. Most of these are **application architecture choices**.

---

# 4. `Program.cs`

In modern .NET, `Program.cs` is the application's **entry point and startup composition location**.

A minimal Web API can look like:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

var app = builder.Build();

app.MapControllers();

app.Run();
```

Although this looks small, it performs a lot of work.

Conceptually:

```text
CreateBuilder()
       |
       v
Create application configuration
       |
Create logging
       |
Create Host
       |
Create DI service collection
       |
       v
Register services
       |
       v
Build application
       |
Create service provider/container
       |
Configure request pipeline
       |
Map endpoints
       |
       v
Start server
```

---

# 5. What is `builder`?

Consider:

```csharp
var builder = WebApplication.CreateBuilder(args);
```

`builder` is a `WebApplicationBuilder`.

It is used to **configure the application before it is built**.

A useful analogy is constructing a house:

```text
Builder
   |
   v
Configure everything
   |
   v
Build
   |
   v
Finished house
```

Similarly:

```text
WebApplicationBuilder
        |
        |-- Configuration
        |-- Logging
        |-- Environment
        |-- Host
        |-- Services
        |
        v
      Build()
        |
        v
WebApplication
```

So:

```csharp
var builder = WebApplication.CreateBuilder(args);
```

roughly means:

> "Give me a builder with which I can configure my Web API application."

---

# 6. `builder.Services`

You will frequently see:

```csharp
builder.Services.AddControllers();
```

or:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

or:

```csharp
builder.Services.AddSingleton<ICache, Cache>();
```

This is where **Dependency Injection registrations** are made.

For example:

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

conceptually means:

```text
When something asks for IUserService
               |
               v
provide UserService
```

### Important distinction

`builder.Services` is **not yet the actual DI container**.

It is an `IServiceCollection` containing service registrations.

Think:

```text
IServiceCollection

"I need IUserService -> UserService"
"I need IOrderService -> OrderService"
"I need DbContext -> AppDbContext"
...
```

Later:

```csharp
builder.Build();
```

constructs the application and its service provider.

We will cover this deeply in the DI section.

---

# 7. `builder.Build()`

Consider:

```csharp
var app = builder.Build();
```

This is a major transition.

Before:

```text
builder
```

After:

```text
app
```

Conceptually:

```text
          BUILDER
             |
             | Configure
             v
     +------------------+
     | Configuration    |
     | Logging          |
     | Services         |
     | Host             |
     | Environment      |
     +------------------+
             |
             | Build()
             v
     +------------------+
     | WebApplication    |
     |                  |
     | ServiceProvider  |
     | Pipeline         |
     | Endpoints        |
     +------------------+
```

So:

```csharp
var app = builder.Build();
```

roughly means:

> "Take everything I configured and construct the application."

---

# 8. Why do registrations happen before `Build()`?

Normal flow:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IUserService, UserService>();

var app = builder.Build();
```

The configuration/registration phase happens before the application is built.

Conceptually:

```text
Create Builder
       |
       v
Configure/Register
       |
       v
Build
       |
       v
Configure application pipeline
       |
       v
Run
```

This separation is important because it distinguishes **application construction/configuration** from **request pipeline configuration and execution**.

---

# 9. What is `app`?

After:

```csharp
var app = builder.Build();
```

we have a `WebApplication`.

Now we configure the application's HTTP request pipeline and endpoints.

Examples:

```csharp
app.UseHttpsRedirection();

app.UseAuthentication();

app.UseAuthorization();

app.MapControllers();

app.Run();
```

The important distinction is:

### `builder`

Primarily used to **construct/configure** the application.

### `app`

The **constructed WebApplication**, used to configure its request pipeline/endpoints and run it.

---

# 10. `app.MapControllers()`

Suppose we have:

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetUser(int id)
    {
        return Ok(new
        {
            Id = id,
            Name = "Hemant"
        });
    }
}
```

Then:

```csharp
app.MapControllers();
```

maps controller actions into the application's endpoint routing system.

A request such as:

```http
GET /api/users/10
```

can eventually reach:

```csharp
GetUser(10)
```

Conceptually:

```text
HTTP Request
     |
     v
Routing
     |
     v
/api/users/10
     |
     v
UsersController
     |
     v
GetUser(10)
```

---

# 11. `app.Run()`

This line is extremely important:

```csharp
app.Run();
```

It tells the application to **start hosting and process incoming requests**.

Conceptually:

```text
Program.cs
    |
    v
app.Run()
    |
    v
Start hosting
    |
    v
Kestrel starts listening
    |
    v
WAIT
    |
    +-- Request 1
    +-- Request 2
    +-- Request 3
    +-- Request 4
    +-- ...
```

It does **not** mean:

> Execute every controller.

It means:

> Start the server/application and wait for incoming requests.

---

# 12. Kestrel

When your Web API runs, something needs to listen on a TCP port and handle HTTP connections.

That server is **Kestrel**.

Conceptually:

```text
Internet / Browser
       |
       | HTTP
       v
    KESTREL
       |
       v
ASP.NET Core HTTP Pipeline
       |
       v
Controller
```

Kestrel is the cross-platform HTTP server used by ASP.NET Core.

It is responsible for low-level web-server responsibilities such as:

- Accepting network connections
- Processing HTTP traffic
- Connection lifecycle
- HTTPS/TLS integration
- HTTP/1.1
- HTTP/2
- HTTP/3 support depending on configuration/platform

Later we will trace:

```text
Kestrel
   |
   v
HttpContext
   |
   v
Middleware
   |
   v
Routing
   |
   v
Endpoint
```

---

# 13. `appsettings.json`

`appsettings.json` is a configuration source.

Example:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

You might configure connection strings:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "..."
  }
}
```

Or application settings:

```json
{
  "Jwt": {
    "Issuer": "my-api",
    "Audience": "my-client"
  }
}
```

These settings can later be consumed through:

```csharp
IConfiguration
```

or strongly typed options such as:

```csharp
IOptions<JwtOptions>
```

Configuration will be covered separately.

---

# 14. `appsettings.Development.json`

You may have:

```text
appsettings.json
appsettings.Development.json
appsettings.Production.json
```

The general idea is:

```text
Base configuration
       +
Environment-specific configuration
       |
       v
Final configuration
```

For example:

### `appsettings.json`

```json
{
  "ConnectionStrings": {
    "Database": "DefaultConnection"
  }
}
```

### `appsettings.Development.json`

```json
{
  "ConnectionStrings": {
    "Database": "LocalDatabase"
  }
}
```

When the application runs in Development, the Development configuration can override matching base values.

Common environments include:

```text
Development
Staging
Production
```

---

# 15. `launchSettings.json`

This file is usually under:

```text
Properties/
    launchSettings.json
```

Example:

```json
{
  "profiles": {
    "MyWebApi": {
      "commandName": "Project",
      "applicationUrl": "https://localhost:7001;http://localhost:5001"
    }
  }
}
```

It is primarily useful for **local development and launch profiles**.

It can define:

- Local URLs
- Launch profiles
- Environment variables
- Development HTTPS settings

For example:

```json
"environmentVariables": {
  "ASPNETCORE_ENVIRONMENT": "Development"
}
```

This can result in:

```text
Environment = Development
```

when launching locally.

### Important

`launchSettings.json` should not be treated as your production configuration mechanism.

---

# 16. `.csproj`

The project file is another critical piece:

```text
MyWebApi.csproj
```

Example:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

</Project>
```

## `Microsoft.NET.Sdk.Web`

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
```

This identifies the project as using the .NET Web SDK and supplies web-specific build behavior/defaults.

## Target framework

```xml
<TargetFramework>net8.0</TargetFramework>
```

means the application targets .NET 8.

For another version:

```xml
<TargetFramework>net10.0</TargetFramework>
```

## Nullable

```xml
<Nullable>enable</Nullable>
```

enables nullable reference type analysis.

For example:

```csharp
string name = null;
```

produces a compiler warning under nullable analysis.

Whereas:

```csharp
string? name = null;
```

explicitly indicates that null is allowed.

## ImplicitUsings

```xml
<ImplicitUsings>enable</ImplicitUsings>
```

allows common namespaces to be automatically included by the SDK.

This is one reason modern `Program.cs` files can be very small.

---

# 17. `bin` and `obj`

After:

```bash
dotnet build
```

you will commonly see:

```text
bin/
obj/
```

## `bin`

Contains build output.

For example:

```text
bin/
  Debug/
    net8.0/
```

You may find artifacts such as:

```text
MyWebApi.dll
MyWebApi.deps.json
MyWebApi.runtimeconfig.json
```

among others.

## `obj`

Contains intermediate build information and generated/intermediate artifacts used during the build.

Normally you do not manually modify either directory.

---

# 18. What actually runs?

After building:

```bash
dotnet build
```

you may have:

```text
MyWebApi.dll
```

You can run it with:

```bash
dotnet MyWebApi.dll
```

Conceptually:

```text
Your C# source code
       |
       v
C# Compiler
       |
       v
IL / .NET assembly
       |
       v
MyWebApi.dll
       |
       v
.NET Runtime
       |
       v
Application executes
```

The .NET runtime executes the compiled application using the CLR/runtime and JIT infrastructure as appropriate.

---

# 19. Complete Startup Picture

Given:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

var app = builder.Build();

app.MapControllers();

app.Run();
```

Think about it in these stages.

## Stage 1 — Process starts

The operating system starts the application process.

```text
MyWebApi process
```

The .NET runtime starts executing the application's entry point.

---

## Stage 2 — Create builder

```csharp
var builder = WebApplication.CreateBuilder(args);
```

The framework prepares infrastructure including concepts such as:

```text
Configuration
Logging
Environment
Host
Service collection
Web hosting configuration
```

---

## Stage 3 — Register services

```csharp
builder.Services.AddControllers();
```

Services/components required by the application are registered.

A realistic application might have:

```csharp
builder.Services.AddScoped<IOrderService, OrderService>();

builder.Services.AddScoped<IOrderRepository, OrderRepository>();

builder.Services.AddDbContext<AppDbContext>();

builder.Services.AddControllers();
```

At this stage, we are primarily saying:

> "These are the dependencies and framework/application services my application needs."

---

## Stage 4 — Build

```csharp
var app = builder.Build();
```

The configured application is constructed.

The DI registrations are used to build the service-provider infrastructure used for dependency resolution.

---

## Stage 5 — Configure pipeline/endpoints

```csharp
app.MapControllers();
```

Controller endpoints are added to endpoint routing.

Middleware can also be configured:

```csharp
app.UseAuthentication();

app.UseAuthorization();
```

---

## Stage 6 — Start

```csharp
app.Run();
```

The application starts hosting and waits for HTTP requests.

---

# 20. What happens when a request arrives?

Suppose a client sends:

```http
GET https://localhost:7001/api/users/10
```

At a high level:

```text
Browser / Client
       |
       | HTTP request
       v
Operating System
       |
       v
Kestrel
       |
       v
ASP.NET Core
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
Controller
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

Then the response travels back:

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
Endpoint
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

This flow is the backbone of the rest of the course.

---

# 21. Three Concepts You Must Separate

## 21.1 Builder

```csharp
var builder = WebApplication.CreateBuilder(args);
```

Purpose:

> Configure and prepare the application.

---

## 21.2 Application

```csharp
var app = builder.Build();
```

Purpose:

> Represents the constructed Web application.

---

## 21.3 Run

```csharp
app.Run();
```

Purpose:

> Start hosting and wait for/process incoming requests.

---

# 22. The Core Mental Model

```text
                 APPLICATION STARTUP

                       Program.cs
                           |
                           v
              CreateBuilder(args)
                           |
                           v
                WebApplicationBuilder
                           |
              +------------+------------+
              |            |            |
              v            v            v
        Configuration   Logging     Services
                                      |
                               IServiceCollection
                                      |
                                      v
                                    Build()
                                      |
                                      v
                                WebApplication
                                      |
                          +-----------+-----------+
                          |                       |
                          v                       v
                     Middleware              Endpoints
                     Pipeline                 Mapping
                          |                       |
                          +-----------+-----------+
                                      |
                                      v
                                    Run()
                                      |
                                      v
                                   Kestrel
                                      |
                                      v
                              WAIT FOR REQUEST
                                      |
                                      v
                               HTTP REQUEST
                                      |
                                      v
                            Middleware Pipeline
                                      |
                                      v
                                   Routing
                                      |
                                      v
                                 Controller
```

---

# 23. A Critical Correction to the Mental Model

Do **not** think:

```text
Program.cs
    |
    v
Controller executes
```

The better model is:

```text
Program.cs
    |
    v
Application is constructed
    |
    v
Application starts
    |
    v
Application waits
    |
    v
HTTP request arrives
    |
    v
Pipeline processes request
    |
    v
Matching endpoint/controller executes
```

This distinction becomes extremely important when we study:

- Middleware
- Dependency Injection
- Controller activation
- Routing
- Async/await
- Exception handling
- Request lifetime
- Scoped services

---

# 24. Where Dependency Injection Fits

For now, only remember its position:

```text
Program.cs
    |
    v
builder.Services
    |
    v
Service registrations
    |
    v
Build()
    |
    v
ServiceProvider / DI infrastructure
    |
    v
Request arrives
    |
    v
Controller needs IUserService
    |
    v
DI resolves IUserService
    |
    v
UserService
    |
    v
Repository
```

The exact service-registration and resolution process will be covered in the dedicated DI section.

---

# 25. Where Middleware Fits

Eventually, the request flow will look roughly like:

```text
Request
   |
   v
Kestrel
   |
   v
Middleware 1
   |
   v
Middleware 2
   |
   v
Middleware 3
   |
   v
Routing
   |
   v
Authentication
   |
   v
Authorization
   |
   v
Controller
```

The response can travel back through the pipeline:

```text
Controller
   |
   v
Authorization
   |
   v
Authentication
   |
   v
Middleware 3
   |
   v
Middleware 2
   |
   v
Middleware 1
   |
   v
Response
```

This is why ASP.NET Core middleware forms a **pipeline**.

---

# 26. Interview Revision

### Q: What is `Program.cs`?

**Answer:**

`Program.cs` is the startup/composition entry point of a modern ASP.NET Core application. It is where application services and infrastructure are configured, the WebApplication is built, the request pipeline/endpoints are configured, and the application is started.

---

### Q: What is `builder`?

**Answer:**

`builder` is a `WebApplicationBuilder` used to configure the application before it is built.

---

### Q: What is `builder.Services`?

**Answer:**

`builder.Services` is an `IServiceCollection` containing dependency-injection service registrations before the application is built.

---

### Q: Is `builder.Services` the DI container?

**Answer:**

No.

It is the collection of service registrations. The service-provider/container infrastructure is built as part of constructing the application.

---

### Q: What does `Build()` do?

**Answer:**

`Build()` constructs the configured `WebApplication` and its supporting infrastructure, including the service-provider infrastructure used for dependency resolution.

---

### Q: What is `app`?

**Answer:**

`app` is the constructed `WebApplication`, used to configure the HTTP request pipeline, endpoints, and application execution.

---

### Q: What does `app.Run()` do?

**Answer:**

It starts the application's hosting/request-processing loop so the application can accept and process incoming HTTP requests.

---

### Q: What is Kestrel?

**Answer:**

Kestrel is the cross-platform HTTP server used by ASP.NET Core to accept network connections and process HTTP traffic.

---

### Q: When does the controller execute?

**Answer:**

Not when the application starts. The application starts and waits for requests. When a request matches a controller endpoint, ASP.NET Core executes the corresponding controller action.

---

# 27. Final Part 1 Cheat Sheet

```text
dotnet run
    |
    v
.NET process starts
    |
    v
Program.cs
    |
    v
WebApplication.CreateBuilder(args)
    |
    v
WebApplicationBuilder
    |
    +-- Configuration
    +-- Logging
    +-- Environment
    +-- Host
    +-- IServiceCollection
    |
    v
builder.Services.Add...
    |
    v
builder.Build()
    |
    v
WebApplication
    |
    +-- ServiceProvider / DI infrastructure
    +-- Middleware configuration
    +-- Endpoint configuration
    |
    v
app.MapControllers()
    |
    v
app.Run()
    |
    v
Kestrel
    |
    v
WAIT
    |
    v
HTTP Request
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
Controller
    |
    v
Service
    |
    v
Repository
    |
    v
Database
    |
    v
Response
```

---

## Part 1 — Key Takeaways

1. **`Program.cs` is the startup/composition point.**
2. **`builder` is used to configure the application.**
3. **`builder.Services` contains DI registrations.**
4. **`Build()` constructs the application.**
5. **`app` represents the constructed Web application.**
6. **Middleware/endpoints are configured on `app`.**
7. **`app.Run()` starts the application.**
8. **Kestrel handles HTTP server responsibilities.**
9. **The application starts before any particular controller action executes.**
10. **A controller executes only after a request reaches its matching endpoint.**
11. **`appsettings*.json` provides configuration sources.**
12. **`launchSettings.json` is primarily for local development/launch configuration.**
13. **`.csproj` defines project/build/target-framework settings.**
14. **`bin` contains build output; `obj` contains intermediate build artifacts.**
15. **The complete request lifecycle will be built piece by piece in the next parts.**

---

# Next Part

**Part 2 — `Program.cs` Deep Dive**

We will take this tiny code:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

var app = builder.Build();

app.MapControllers();

app.Run();
```

and go **line by line and internally**:

```text
WebApplication.CreateBuilder(args)
            |
            +-- WebApplicationBuilder
            +-- Host
            +-- Configuration
            +-- Logging
            +-- Environment
            +-- Services
            |
            v
builder.Services
            |
            v
Build()
            |
            +-- ServiceProvider
            +-- Host construction
            +-- Web server setup
            |
            v
WebApplication
            |
            v
app
```

That is where we start understanding what ASP.NET Core is actually doing underneath the simple-looking `Program.cs`.
