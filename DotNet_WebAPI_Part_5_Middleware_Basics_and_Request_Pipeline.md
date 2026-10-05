# .NET Web API — Part 5
# Middleware in ASP.NET Core

## 1. What Problem Does Middleware Solve?

A simple Web API might look like:

```text
Client
   ↓
GET /api/users
   ↓
UsersController
   ↓
Database
```

But real applications need many things before or after the controller:

- Request logging
- Authentication
- Authorization
- Exception handling
- CORS
- Rate limiting
- Security headers
- Correlation IDs
- Execution-time measurement
- Request/response processing

We do not want to put all of this logic inside every controller.

Instead, ASP.NET Core provides:

# Middleware

Middleware allows us to put cross-cutting request/response processing into an HTTP pipeline.

---

# 2. Basic Middleware Idea

Instead of:

```text
HTTP Request
     ↓
Controller
```

we have:

```text
HTTP Request
     ↓
Middleware
     ↓
Middleware
     ↓
Middleware
     ↓
Controller
```

For example:

```text
HTTP Request
     ↓
Exception Middleware
     ↓
Logging Middleware
     ↓
Authentication Middleware
     ↓
Authorization Middleware
     ↓
Controller
```

The controller does not need to know that all these things happened.

---

# 3. Middleware as a Chain

Think about an airport:

```text
Passenger
   ↓
Security
   ↓
Passport Check
   ↓
Boarding Check
   ↓
Gate
```

Each stage performs some work and passes the passenger to the next stage.

Middleware behaves similarly:

```text
HTTP Request
     ↓
Middleware A
     ↓
Middleware B
     ↓
Middleware C
     ↓
Controller
```

Each middleware can:

1. Inspect the request.
2. Modify the request.
3. Execute logic.
4. Call the next middleware.
5. Inspect or modify the response.
6. Stop/short-circuit the pipeline.

---

# 4. The Most Important Middleware Concept

Suppose:

```text
Middleware A
Middleware B
Middleware C
Controller
```

The request travels forward:

```text
Request
   ↓
A
   ↓
B
   ↓
C
   ↓
Controller
```

But after the controller produces a response, execution unwinds:

```text
Controller
   ↓
C
   ↓
B
   ↓
A
   ↓
Response
```

So middleware surrounds downstream processing.

Visual model:

```text
             REQUEST
                ↓
        ┌──────────────┐
        │ Middleware A │
        │              │
        │   ┌──────────┴─────────┐
        │   │ Middleware B       │
        │   │                    │
        │   │   ┌────────────────┴────────┐
        │   │   │ Middleware C            │
        │   │   │                         │
        │   │   │      Controller         │
        │   │   │                         │
        │   │   └─────────────────────────┘
        │   │             ↑
        │   └─────────────┘
        │           ↑
        └───────────┘
                ↑
             RESPONSE
```

This is the core idea of ASP.NET Core middleware.

---

# 5. Middleware in Program.cs

Example:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

var app = builder.Build();

app.UseMiddleware<MyMiddleware>();

app.MapControllers();

app.Run();
```

Now the conceptual pipeline is:

```text
HTTP Request
     ↓
MyMiddleware
     ↓
Controller
```

---

# 6. What Does `app.Use(...)` Mean?

You'll frequently see:

```csharp
app.UseSomething();
```

Examples:

```csharp
app.UseHttpsRedirection();

app.UseAuthentication();

app.UseAuthorization();
```

`Use` generally means:

> Add something into the middleware pipeline.

For example:

```csharp
app.UseAuthentication();
```

does not mean:

> Authenticate immediately while Program.cs is executing.

It means:

> Add authentication middleware to the pipeline so that it participates when HTTP requests arrive.

This distinction is important.

---

# 7. Program.cs Builds the Pipeline

Consider:

```csharp
app.UseMiddleware<A>();

app.UseMiddleware<B>();

app.UseMiddleware<C>();

app.MapControllers();
```

At startup, ASP.NET Core constructs the request pipeline conceptually as:

```text
A
 ↓
B
 ↓
C
 ↓
Endpoints
```

Then:

```csharp
app.Run();
```

starts the application.

Actual HTTP requests travel through the pipeline later.

---

# 8. Startup Time vs Request Time

This distinction is extremely important.

## Startup

```text
Program.cs
   ↓
Register services
   ↓
Build application
   ↓
Configure middleware
   ↓
Start server
```

## Request

Later:

```text
HTTP Request
   ↓
Middleware A
   ↓
Middleware B
   ↓
Middleware C
   ↓
Controller
```

Therefore:

```csharp
app.UseMiddleware<MyMiddleware>();
```

does not execute the middleware for one request immediately.

It registers the middleware into the pipeline.

---

# 9. First Simple Middleware

You can write middleware inline:

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("Before");

    await next();

    Console.WriteLine("After");
});
```

Then:

```csharp
app.MapControllers();
```

The flow becomes:

```text
Request
   ↓
"Before"
   ↓
next()
   ↓
Controller
   ↓
Response
   ↓
"After"
```

---

# 10. What Is `next()`?

This is one of the most important middleware concepts.

Consider:

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("Before");

    await next();

    Console.WriteLine("After");
});
```

`next` represents:

> The next component in the HTTP request pipeline.

Therefore:

```csharp
await next();
```

means:

> Continue processing the request with the next component.

---

# 11. Example with Two Middleware Components

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("Middleware A - Before");

    await next();

    Console.WriteLine("Middleware A - After");
});

app.Use(async (context, next) =>
{
    Console.WriteLine("Middleware B - Before");

    await next();

    Console.WriteLine("Middleware B - After");
});

app.MapControllers();
```

Request execution:

### Step 1 — A starts

```text
A Before
```

A calls:

```csharp
await next();
```

Execution moves to B.

### Step 2 — B starts

```text
B Before
```

B calls:

```csharp
await next();
```

Execution continues to the controller.

### Step 3 — Controller executes

Controller generates the response.

### Step 4 — B resumes

B continues after:

```csharp
await next();
```

So:

```text
B After
```

### Step 5 — A resumes

A continues after:

```csharp
await next();
```

So:

```text
A After
```

Final order:

```text
A Before
B Before
Controller
B After
A After
```

This order is extremely important.

---

# 12. Think of `await next()` as a Door

Imagine middleware A:

```text
A Before

    ┌───────────────┐
    │   await next  │
    └───────┬───────┘
            ↓
       Next Pipeline

A After
```

When A calls:

```csharp
await next();
```

it temporarily hands control to the rest of the pipeline.

When downstream processing finishes, execution comes back to A.

Therefore:

```text
Before next()
```

runs while the request is moving inward.

And:

```text
After next()
```

runs while the response is unwinding outward.

---

# 13. Middleware Is Similar to Nested Functions

Conceptually:

```text
A(B(C(Controller)))
```

Request:

```text
A
 └── B
      └── C
           └── Controller
```

Response:

```text
Controller
   ↑
   C
   ↑
   B
   ↑
   A
```

This is a useful mental model for understanding middleware.

---

# 14. What If We Don't Call `next()`?

Consider:

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("Middleware A");

    // No next()
});
```

Then:

```text
Request
   ↓
Middleware A
   ↓
STOP
```

The request does not continue to the next middleware/controller.

This is called:

# Short-Circuiting

---

# 15. Why Would We Short-Circuit?

There are many legitimate cases.

### Rate limiting

```text
Request
   ↓
Rate Limit Middleware
   ↓
Limit exceeded?
   ↓
YES
   ↓
HTTP 429
```

The controller does not need to execute.

### Authentication

```text
Request
   ↓
Authentication
   ↓
Valid identity?
   ↓
NO
   ↓
HTTP 401
```

Pipeline can stop.

---

# 16. Short-Circuit Example

```csharp
app.Use(async (context, next) =>
{
    if (context.Request.Headers.ContainsKey("X-Block"))
    {
        context.Response.StatusCode = 403;

        await context.Response.WriteAsync("Blocked");

        return;
    }

    await next();
});
```

If `X-Block` exists:

```text
Request
   ↓
Middleware
   ↓
X-Block exists
   ↓
403
   ↓
STOP
```

The controller never runs.

---

# 17. `HttpContext`

In:

```csharp
app.Use(async (context, next) =>
{
});
```

`context` is:

```csharp
HttpContext
```

`HttpContext` represents the current HTTP interaction.

It contains information about:

```text
Request
+
Response
```

Conceptually:

```text
HttpContext
│
├── Request
│   ├── Method
│   ├── Path
│   ├── Headers
│   ├── Query
│   ├── Cookies
│   └── Body
│
├── Response
│   ├── StatusCode
│   ├── Headers
│   ├── Cookies
│   └── Body
│
├── User
├── Items
├── Connection
└── RequestServices
```

For now, the key idea is:

> `HttpContext` represents the current request and response context.

---

# 18. Reading Request Information

Example:

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine(context.Request.Method);
    Console.WriteLine(context.Request.Path);

    await next();
});
```

For:

```http
GET /api/users
```

you might see:

```text
GET
/api/users
```

---

# 19. Reading Headers

```csharp
app.Use(async (context, next) =>
{
    var correlationId =
        context.Request.Headers["X-Correlation-ID"];

    Console.WriteLine(correlationId);

    await next();
});
```

This is useful for request tracing and production debugging.

---

# 20. Modifying the Response

Middleware can also modify the response.

Example:

```csharp
app.Use(async (context, next) =>
{
    await next();

    context.Response.Headers["X-Application"] = "MyApi";
});
```

Flow:

```text
Request
   ↓
Middleware
   ↓
Controller
   ↓
Response generated
   ↓
Middleware adds header
   ↓
Client
```

This is one reason the code after `await next()` is powerful.

---

# 21. Request and Response Processing

A middleware can therefore follow this pattern:

```csharp
app.Use(async (context, next) =>
{
    // Before
    // Inspect/process request

    await next();

    // After
    // Inspect/process response
});
```

Visual model:

```text
             REQUEST
                ↓
        ┌──────────────┐
        │ Middleware   │
        │              │
        │ Before       │
        │     ↓        │
        │   next()     │
        │     ↓        │
        │ After        │
        └──────────────┘
                ↓
             RESPONSE
```

---

# 22. Real Logging Middleware

Suppose we want to measure API execution time.

```csharp
app.Use(async (context, next) =>
{
    var start = DateTime.UtcNow;

    await next();

    var duration =
        DateTime.UtcNow - start;

    Console.WriteLine(
        $"{context.Request.Path} took {duration.TotalMilliseconds} ms");
});
```

Flow:

```text
Request
   ↓
Start timer
   ↓
next()
   ↓
Controller
   ↓
Database
   ↓
Response
   ↓
Stop timer
   ↓
Log duration
```

This is a real production-style middleware pattern.

---

# 23. Exception Middleware

A common middleware pattern is centralized exception handling:

```csharp
app.Use(async (context, next) =>
{
    try
    {
        await next();
    }
    catch (Exception ex)
    {
        context.Response.StatusCode = 500;

        await context.Response.WriteAsync(
            "Internal server error");
    }
});
```

Flow:

```text
Request
   ↓
Exception Middleware
   ↓
Controller
   ↓
Exception
   ↑
Middleware catches
   ↓
HTTP 500
```

This avoids putting identical `try/catch` blocks into every controller.

Later, this can be developed into a proper production-grade custom exception middleware.

---

# 24. Middleware Ordering

Suppose:

```csharp
app.Use(A);
app.Use(B);
app.Use(C);
```

The request order is:

```text
A
 ↓
B
 ↓
C
 ↓
Endpoint
```

Middleware order matters.

For example:

```text
Authentication
      ↓
Authorization
```

Authentication generally needs to establish the user's identity before authorization evaluates access.

Therefore middleware cannot simply be placed in arbitrary order.

---

# 25. Example Real Web API Pipeline

A simplified pipeline might look like:

```csharp
app.UseExceptionHandler();

app.UseHttpsRedirection();

app.UseCors();

app.UseAuthentication();

app.UseAuthorization();

app.MapControllers();
```

Conceptually:

```text
Request
   ↓
Exception Handling
   ↓
HTTPS
   ↓
CORS
   ↓
Authentication
   ↓
Authorization
   ↓
Controller
   ↓
Response
```

The exact ordering depends on the middleware and application configuration, but the key principle is:

> Middleware order determines execution order and can affect correctness.

---

# 26. `Use` vs `Run` vs `Map`

You will frequently see:

```csharp
app.Use(...)
app.Run(...)
app.Map(...)
```

For now, remember the basic distinction.

## `Use`

Adds middleware that can call the next component:

```csharp
app.Use(async (context, next) =>
{
    await next();
});
```

Mental model:

```text
Use
 ↓
Can continue
```

---

## `Run`

Adds terminal middleware:

```csharp
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello");
});
```

It does not receive `next`.

Mental model:

```text
Run
 ↓
Terminal
```

---

## `Map`

Branches the pipeline based on a path or condition.

Mental model:

```text
Map
 ↓
Branch
```

We'll go much deeper into `Use`, `Run`, and `Map` later.

---

# 27. First Custom Middleware Class

Instead of inline middleware:

```csharp
app.Use(async (context, next) =>
{
    // ...
});
```

we can create a class.

```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;

    public RequestLoggingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        Console.WriteLine(
            $"Request: {context.Request.Method} {context.Request.Path}");

        await _next(context);

        Console.WriteLine(
            $"Response: {context.Response.StatusCode}");
    }
}
```

Register:

```csharp
app.UseMiddleware<RequestLoggingMiddleware>();
```

---

# 28. What Is `RequestDelegate`?

This is one of the most important concepts in middleware.

```csharp
private readonly RequestDelegate _next;
```

Conceptually, `RequestDelegate` represents a method that can process an HTTP request.

Conceptually:

```csharp
delegate Task RequestDelegate(HttpContext context);
```

Therefore:

```csharp
await _next(context);
```

means:

> Execute the next request-processing component.

So:

```text
RequestDelegate
      ↓
Represents the next component
      ↓
Continue the HTTP pipeline
```

---

# 29. Custom Middleware Flow

Our middleware:

```csharp
public async Task InvokeAsync(HttpContext context)
{
    Console.WriteLine("Before");

    await _next(context);

    Console.WriteLine("After");
}
```

Flow:

```text
HTTP Request
      ↓
RequestLoggingMiddleware
      │
      ├── Before
      │
      ├── _next(context)
      │          ↓
      │      Next Middleware
      │          ↓
      │      Controller
      │          ↓
      │      Response
      │
      └── After
      ↓
HTTP Response
```

This is exactly the same concept as inline middleware.

---

# 30. Why `_next` Is Passed to the Middleware

When you write:

```csharp
app.UseMiddleware<RequestLoggingMiddleware>();
```

ASP.NET Core builds the middleware pipeline.

Conceptually:

```text
Middleware A
     ↓
RequestDelegate for B
     ↓
Middleware B
     ↓
RequestDelegate for C
     ↓
Middleware C
```

So `_next` is essentially the continuation of the pipeline.

The important mental model is:

> Middleware is a chain of request delegates.

---

# 31. Full Middleware Chain

Imagine:

```text
LoggingMiddleware
       ↓
AuthMiddleware
       ↓
ExceptionMiddleware
       ↓
Endpoint
```

Conceptually:

```text
LoggingMiddleware
       │
       │ _next
       ↓
AuthMiddleware
       │
       │ _next
       ↓
ExceptionMiddleware
       │
       │ _next
       ↓
Endpoint
```

The response unwinds in reverse:

```text
Endpoint
   ↑
ExceptionMiddleware
   ↑
AuthMiddleware
   ↑
LoggingMiddleware
```

---

# 32. Interview Answer — What Is Middleware?

If an interviewer asks:

> What is middleware in ASP.NET Core?

A strong answer is:

> Middleware is a component in the ASP.NET Core HTTP request pipeline that can inspect or modify the request and response, execute logic before and after downstream components, call the next component through a `RequestDelegate`, or short-circuit the pipeline.

Then show:

```text
Request
 ↓
Middleware A
 ↓
Middleware B
 ↓
Controller
 ↑
Middleware B
 ↑
Middleware A
 ↓
Response
```

That is much stronger than simply saying:

> Middleware handles HTTP requests.

---

# 33. Core Concepts to Remember

```text
Middleware
    ↓
Component in HTTP pipeline

app.Use(...)
    ↓
Adds middleware

next()
    ↓
Continues pipeline

HttpContext
    ↓
Current request + response

RequestDelegate
    ↓
Represents the next request-processing component

No next()
    ↓
Short-circuit

Before next()
    ↓
Request-side processing

After next()
    ↓
Response-side processing
```

---

# 34. Most Important Mental Model

Consider:

```csharp
app.Use(A);
app.Use(B);
app.Use(C);
app.MapControllers();
```

Conceptually:

```text
             REQUEST
                ↓
          ┌───────────┐
          │     A     │
          │  Before   │
          │     ↓     │
          │    next   │
          │     ↓     │
          │     B     │
          │  Before   │
          │     ↓     │
          │    next   │
          │     ↓     │
          │     C     │
          │  Before   │
          │     ↓     │
          │ Controller │
          │     ↑     │
          │  C After  │
          │     ↑     │
          │  B After  │
          │     ↑     │
          │  A After  │
          └───────────┘
                ↓
             RESPONSE
```

The most important concept is:

> **Request goes forward through the middleware pipeline; response unwinds backward through the middleware pipeline.**

Once this is clear, the rest of ASP.NET Core middleware becomes much easier.

---

# 35. Part 5 Summary

You should now understand:

- What middleware is
- Why middleware exists
- HTTP request pipeline
- `app.Use(...)`
- Startup time vs request time
- `next()`
- Request-side vs response-side processing
- Middleware ordering
- Short-circuiting
- `HttpContext`
- Reading request information
- Modifying responses
- Logging middleware
- Exception middleware concept
- `Use` vs `Run` vs `Map` at a high level
- Custom middleware class
- `RequestDelegate`
- `_next(context)`
- Reverse/unwinding response flow

The core chain to remember is:

```text
Program.cs
   ↓
Build middleware pipeline
   ↓
HTTP Request
   ↓
Middleware A
   ↓
Middleware B
   ↓
Middleware C
   ↓
Controller / Endpoint
   ↓
Response
   ↑
Middleware C
   ↑
Middleware B
   ↑
Middleware A
```

This is the foundation for the deeper middleware topics that follow.
