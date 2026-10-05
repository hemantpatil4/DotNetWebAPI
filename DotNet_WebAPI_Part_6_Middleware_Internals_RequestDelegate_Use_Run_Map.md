# .NET Web API — Part 6
# Middleware Internals — RequestDelegate, Use, Run, Map & Pipeline Construction

## 1. Where Part 6 Fits

In Part 5 we learned the basic middleware model:

```text
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

We learned:

- Middleware
- `app.Use(...)`
- `next()`
- `HttpContext`
- Short-circuiting
- Request/response flow
- Custom middleware
- `RequestDelegate`

Part 6 goes one level deeper.

The main goal is to understand:

```text
app.Use(A)
app.Use(B)
app.Use(C)
```

and how ASP.NET Core conceptually turns this into a connected chain of request delegates.

---

# 2. The Most Important Concept

Suppose:

```csharp
app.Use(A);
app.Use(B);
app.Use(C);
```

Conceptually:

```text
Request
   ↓
A
   ↓
B
   ↓
C
   ↓
Endpoint
```

Each middleware has a reference to the next component.

Conceptually:

```text
A
│
└── _next → B
             │
             └── _next → C
                          │
                          └── _next → Endpoint
```

This `_next` connection is the heart of the middleware pipeline.

---

# 3. What Is RequestDelegate?

`RequestDelegate` is one of the most important types in ASP.NET Core middleware.

Conceptually, it represents a method that receives an `HttpContext` and returns a `Task`.

Conceptually:

```csharp
delegate Task RequestDelegate(HttpContext context);
```

In practical terms:

```text
RequestDelegate
       ↓
A function capable of processing
an HTTP request
```

Therefore:

```csharp
await _next(context);
```

means:

> Invoke the next request-processing component.

---

# 4. Simple RequestDelegate Example

You can create a delegate directly:

```csharp
RequestDelegate handler = async context =>
{
    await context.Response.WriteAsync("Hello");
};
```

Then:

```csharp
await handler(context);
```

executes it.

So:

```text
HttpContext
     ↓
RequestDelegate
     ↓
Process request
```

A middleware pipeline is essentially a chain of these request-processing delegates.

---

# 5. Middleware as a Function

Think of middleware conceptually as:

```text
Middleware
    =
Function that receives:
    HttpContext
    +
Next RequestDelegate
```

For example:

```csharp
async Task Middleware(
    HttpContext context,
    RequestDelegate next)
{
    // Before

    await next(context);

    // After
}
```

The middleware receives:

```text
context
   ↓
Current HTTP request/response

next
   ↓
Rest of pipeline
```

---

# 6. The `_next` Field

A custom middleware often looks like:

```csharp
public class LoggingMiddleware
{
    private readonly RequestDelegate _next;

    public LoggingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        Console.WriteLine("Before");

        await _next(context);

        Console.WriteLine("After");
    }
}
```

The important line is:

```csharp
private readonly RequestDelegate _next;
```

This means:

> Store a reference to the next component in the pipeline.

Then:

```csharp
await _next(context);
```

means:

> Continue with that component.

---

# 7. Think of `_next` as a Link

Imagine a linked list:

```text
Node A → Node B → Node C → Node D
```

Each node knows the next node.

Middleware is conceptually similar:

```text
Middleware A
     ↓
    _next
     ↓
Middleware B
     ↓
    _next
     ↓
Middleware C
     ↓
    _next
     ↓
Endpoint
```

So `_next` is like the pointer to the next node.

---

# 8. How `app.Use()` Fits Into This

Consider:

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("A");

    await next();

    Console.WriteLine("A After");
});
```

`app.Use()` adds a middleware component to the pipeline.

Conceptually:

```text
Use()
  ↓
Add middleware
  ↓
Connect middleware to next component
```

When multiple `Use()` calls are made:

```csharp
app.Use(A);
app.Use(B);
app.Use(C);
```

they form a chain:

```text
A → B → C → Endpoint
```

---

# 9. `Use` Does Not Execute the Middleware Immediately

This is important.

When Program.cs executes:

```csharp
app.Use(A);
```

ASP.NET Core is configuring the pipeline.

It is not saying:

```text
Execute A right now
```

Instead:

```text
Configure pipeline
      ↓
Store/connect middleware
      ↓
Build application
      ↓
Start server
      ↓
Wait for HTTP request
```

Later, when a request arrives:

```text
HTTP Request
      ↓
A executes
```

---

# 10. Conceptual Pipeline Construction

Suppose:

```csharp
app.Use(A);
app.Use(B);
app.Use(C);

app.MapControllers();
```

Conceptually the pipeline becomes:

```text
Request
   ↓
A
   ↓
B
   ↓
C
   ↓
Controllers/Endpoints
```

Each component receives the next delegate.

Conceptually:

```text
A(next = B)
B(next = C)
C(next = Endpoint)
```

This is the most important internal mental model.

---

# 11. Nested Function Model

There is another excellent way to visualize this.

Suppose:

```text
A
 ↓
B
 ↓
C
 ↓
Endpoint
```

Conceptually think of it as:

```text
A(
   B(
      C(
         Endpoint
      )
   )
)
```

This isn't literal C# syntax for how you write the pipeline, but it is an excellent mental model.

Request enters A.

A calls B.

B calls C.

C calls Endpoint.

Then execution returns:

```text
Endpoint
   ↑
   C
   ↑
   B
   ↑
   A
```

---

# 12. Why `await next()` Creates the Reverse Flow

Consider:

```csharp
public async Task InvokeAsync(HttpContext context)
{
    Console.WriteLine("A Before");

    await _next(context);

    Console.WriteLine("A After");
}
```

The method is paused at:

```csharp
await _next(context);
```

The next component executes.

Eventually it completes.

Then execution resumes:

```csharp
Console.WriteLine("A After");
```

Therefore:

```text
A Before
   ↓
next()
   ↓
B Before
   ↓
next()
   ↓
C
   ↓
C After
   ↓
B After
   ↓
A After
```

This is why middleware has an inward and outward phase.

---

# 13. Complete Three-Middleware Example

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("A Before");

    await next();

    Console.WriteLine("A After");
});

app.Use(async (context, next) =>
{
    Console.WriteLine("B Before");

    await next();

    Console.WriteLine("B After");
});

app.Use(async (context, next) =>
{
    Console.WriteLine("C Before");

    await next();

    Console.WriteLine("C After");
});

app.MapControllers();
```

Conceptual execution:

```text
Request
  ↓
A Before
  ↓
B Before
  ↓
C Before
  ↓
Controller
  ↓
C After
  ↓
B After
  ↓
A After
  ↓
Response
```

---

# 14. `Run()` — Terminal Middleware

Now compare `Use()` with `Run()`.

Example:

```csharp
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello");
});
```

`Run()` creates terminal middleware.

It doesn't provide a `next` delegate to call.

Conceptually:

```text
A
 ↓
B
 ↓
Run
 ↓
STOP
```

Therefore:

```text
Run
  =
Terminal component
```

---

# 15. Example: `Use` Followed by `Run`

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("Before");

    await next();

    Console.WriteLine("After");
});

app.Run(async context =>
{
    Console.WriteLine("Terminal");

    await context.Response.WriteAsync("Hello");
});
```

Execution:

```text
Before
   ↓
Terminal
   ↓
After
```

Why?

Because `Run()` is the endpoint/terminal component.

There is no next component after it.

---

# 16. What Happens If You Put `Run()` First?

Consider:

```csharp
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello");
});

app.Use(async (context, next) =>
{
    Console.WriteLine("This will not be reached");
    await next();
});
```

The terminal component ends processing.

Conceptually:

```text
Request
   ↓
Run
   ↓
STOP
```

The later middleware cannot participate in that request path.

Therefore:

> Terminal middleware should be placed intentionally because it ends that pipeline branch.

---

# 17. `Map()` — Branching the Pipeline

`Map()` is used to branch the pipeline based on a request path.

Example:

```csharp
app.Map("/admin", adminApp =>
{
    adminApp.Run(async context =>
    {
        await context.Response.WriteAsync("Admin");
    });
});
```

Conceptually:

```text
                 Request
                    │
              Path starts /admin?
                 /       \
               Yes        No
                ↓          ↓
          Admin branch   Main pipeline
```

---

# 18. Example with Two Paths

```csharp
app.Map("/admin", adminApp =>
{
    adminApp.Run(async context =>
    {
        await context.Response.WriteAsync("Admin Area");
    });
});

app.Map("/public", publicApp =>
{
    publicApp.Run(async context =>
    {
        await context.Response.WriteAsync("Public Area");
    });
});
```

Requests:

```text
GET /admin
      ↓
Admin branch
```

and:

```text
GET /public
      ↓
Public branch
```

This is pipeline branching.

---

# 19. `Use`, `Run`, `Map` Mental Model

Remember:

```text
Use
 ↓
Middleware that can continue

Run
 ↓
Terminal middleware

Map
 ↓
Branch the pipeline
```

A simple picture:

```text
                 Request
                    ↓
                  Use A
                    ↓
                  Use B
                    ↓
                 ┌──┴──┐
                 │ Map │
                 └──┬──┘
                /         \
               /           \
          /admin           /public
             ↓                ↓
           Run              Run
```

---

# 20. Short-Circuiting Internally

Consider:

```csharp
app.Use(async (context, next) =>
{
    if (context.Request.Path == "/blocked")
    {
        context.Response.StatusCode = 403;
        return;
    }

    await next();
});
```

For `/blocked`:

```text
Request
   ↓
Middleware
   ↓
Condition true
   ↓
Return
   ↓
STOP
```

For `/users`:

```text
Request
   ↓
Middleware
   ↓
Condition false
   ↓
next()
   ↓
Next middleware
```

Therefore:

```text
next()
```

is effectively the decision to continue the pipeline.

---

# 21. RequestDelegate Chain — Deeper Mental Model

Suppose:

```text
A → B → C → Endpoint
```

Think of:

```text
A's _next = RequestDelegate(B)

B's _next = RequestDelegate(C)

C's _next = RequestDelegate(Endpoint)
```

So:

```text
A
│
└── RequestDelegate → B
                       │
                       └── RequestDelegate → C
                                                │
                                                └── RequestDelegate → Endpoint
```

When A executes:

```csharp
await _next(context);
```

it invokes B.

When B executes:

```csharp
await _next(context);
```

it invokes C.

When C executes:

```csharp
await _next(context);
```

it invokes the endpoint.

---

# 22. Middleware Pipeline as a Chain of Delegates

The whole pipeline can therefore be viewed as:

```text
HTTP Request
     ↓
RequestDelegate A
     ↓
RequestDelegate B
     ↓
RequestDelegate C
     ↓
RequestDelegate Endpoint
```

This is a useful interview-level understanding.

---

# 23. How Custom Middleware Fits

Custom middleware:

```csharp
public class LoggingMiddleware
{
    private readonly RequestDelegate _next;

    public LoggingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        Console.WriteLine("Before");

        await _next(context);

        Console.WriteLine("After");
    }
}
```

Registration:

```csharp
app.UseMiddleware<LoggingMiddleware>();
```

Conceptually:

```text
ASP.NET Core
     ↓
Creates/configures middleware
     ↓
Provides next RequestDelegate
     ↓
Middleware stores it in _next
     ↓
Request arrives
     ↓
InvokeAsync(context)
     ↓
_next(context)
```

---

# 24. `HttpContext` Travels Through the Pipeline

Notice:

```csharp
await _next(context);
```

The same logical `HttpContext` represents the current HTTP request as it moves through the pipeline.

Conceptually:

```text
Request
   ↓
HttpContext
   ↓
Middleware A
   ↓
HttpContext
   ↓
Middleware B
   ↓
HttpContext
   ↓
Middleware C
   ↓
HttpContext
   ↓
Endpoint
```

Each component can inspect the request/response through that context.

---

# 25. Adding Information to `HttpContext.Items`

`HttpContext.Items` can be used for request-local data.

Example:

```csharp
app.Use(async (context, next) =>
{
    context.Items["RequestStart"] =
        DateTime.UtcNow;

    await next();
});
```

Later middleware can retrieve it:

```csharp
var start =
    context.Items["RequestStart"];
```

This data belongs to the current request context.

It is not a global application dictionary.

---

# 26. Middleware Ordering Example

Suppose:

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

The logical relationship is:

```text
Authentication
      ↓
Establish identity
      ↓
Authorization
      ↓
Evaluate permissions
```

If you reverse the conceptual dependency:

```text
Authorization
      ↓
Authentication
```

authorization may not have the identity information it needs.

Therefore:

> Middleware order is part of application correctness.

---

# 27. Exception Handling and Ordering

Suppose we have:

```csharp
app.UseExceptionHandler();

app.UseAuthentication();

app.UseAuthorization();

app.MapControllers();
```

A request exception from downstream processing can travel back toward the exception handler.

Conceptually:

```text
Request
   ↓
Exception Handler
   ↓
Authentication
   ↓
Authorization
   ↓
Controller
   ↓
Exception
   ↑
Exception Handler catches/handles
```

This illustrates why middleware that is intended to observe downstream exceptions often needs to be positioned appropriately around that downstream work.

---

# 28. A Realistic Pipeline

A simplified Web API might look like:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

var app = builder.Build();

app.UseExceptionHandler();

app.UseHttpsRedirection();

app.UseCors();

app.UseAuthentication();

app.UseAuthorization();

app.MapControllers();

app.Run();
```

Conceptually:

```text
                  HTTP Request
                       ↓
                Exception Handler
                       ↓
                 HTTPS Redirect
                       ↓
                      CORS
                       ↓
                Authentication
                       ↓
                 Authorization
                       ↓
                   Routing/
                   Endpoint
                       ↓
                  Controller
                       ↓
                    Response
                       ↑
                Middleware unwind
```

The exact production pipeline can differ based on the application and framework features, but this is a useful conceptual model.

---

# 29. Important Distinction: `MapControllers()`

You previously saw:

```csharp
app.MapControllers();
```

Do not think of this as just:

```text
"execute controllers"
```

It maps controller actions into the application's endpoint routing system.

Conceptually:

```text
Middleware pipeline
       ↓
Endpoint routing
       ↓
Find matching controller action
       ↓
Execute endpoint
```

So the endpoint becomes part of the downstream request-processing chain.

---

# 30. Full Request Walkthrough

Suppose the client sends:

```http
GET /api/fx/rates
```

Pipeline:

```text
Client
  ↓
Kestrel
  ↓
Exception Middleware
  ↓
Logging Middleware
  ↓
Authentication
  ↓
Authorization
  ↓
Routing/Endpoint
  ↓
FXController
  ↓
FXService
  ↓
FXRepository
  ↓
Database
```

Response:

```text
Database
  ↑
FXRepository
  ↑
FXService
  ↑
FXController
  ↑
Endpoint
  ↑
Authorization
  ↑
Authentication
  ↑
Logging Middleware
  ↑
Exception Middleware
  ↑
Kestrel
  ↑
Client
```

This connects the middleware concepts with the DI and Web API concepts we learned earlier.

---

# 31. Middleware + DI

Middleware can also use DI.

Example:

```csharp
public class LoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<LoggingMiddleware> _logger;

    public LoggingMiddleware(
        RequestDelegate next,
        ILogger<LoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        _logger.LogInformation(
            "Request: {Path}",
            context.Request.Path);

        await _next(context);
    }
}
```

This connects Part 4:

```text
DI
 ↓
Middleware dependencies
```

with Part 5/6:

```text
Middleware
 ↓
Request pipeline
```

---

# 32. Important Lifetime Consideration

Middleware itself has lifecycle considerations.

A middleware registered through:

```csharp
app.UseMiddleware<MyMiddleware>();
```

should not be designed casually around scoped dependencies in its constructor.

A common safe pattern is to inject dependencies with appropriate lifetimes into the `InvokeAsync` method when the middleware needs request-scoped services.

Example:

```csharp
public async Task InvokeAsync(
    HttpContext context,
    IMyScopedService service)
{
    await _next(context);
}
```

This allows the scoped service to be resolved for the current request scope.

The exact activation behavior depends on how middleware is registered, so the important design principle is:

> Do not accidentally capture a request-scoped dependency inside a long-lived middleware instance.

---

# 33. Why This Matters

Suppose:

```text
Middleware
   ↓
ScopedService
```

If the middleware instance lives longer than the scoped service, holding that scoped object as a field can create a lifetime problem.

This is another application of the lifetime rule from Part 4:

```text
Long-lived object
      ↓
should not capture
      ↓
short-lived object
```

---

# 34. Pipeline Construction Mental Model

Do not try to memorize the actual internal source implementation.

Instead, remember this conceptual process:

```text
Program.cs
   ↓
app.Use(A)
   ↓
app.Use(B)
   ↓
app.Use(C)
   ↓
Map endpoint
   ↓
Pipeline gets connected
   ↓
A points to B
   ↓
B points to C
   ↓
C points to endpoint
```

Result:

```text
A → B → C → Endpoint
```

---

# 35. Interview Questions

## Q1. What is `RequestDelegate`?

`RequestDelegate` represents a function that processes an `HttpContext` asynchronously and returns a `Task`.

Conceptually:

```csharp
Task(HttpContext)
```

It is used to connect components in the HTTP request pipeline.

---

## Q2. What does `_next` represent?

`_next` is a `RequestDelegate` representing the next component in the middleware pipeline.

```csharp
await _next(context);
```

continues processing.

---

## Q3. What happens if middleware doesn't call `_next()`?

The pipeline is short-circuited unless the middleware otherwise invokes another path.

---

## Q4. What is the difference between `Use` and `Run`?

```text
Use
 ↓
Can call next

Run
 ↓
Terminal
```

---

## Q5. What does `Map` do?

It creates a branch in the middleware pipeline based on a path/condition.

---

## Q6. Why does middleware execute in reverse after `next()`?

Because each middleware awaits the downstream delegate.

Conceptually:

```text
A Before
   ↓
B Before
   ↓
Endpoint
   ↑
B After
   ↑
A After
```

When downstream processing completes, control returns to the awaiting middleware.

---

## Q7. Why does middleware ordering matter?

Because middleware executes in registration order on the way in, and its post-`next()` logic executes in reverse order on the way out.

Incorrect ordering can cause authentication, authorization, exception handling, routing, CORS, or other behavior to work incorrectly.

---

# 36. One-Minute Revision

Remember these:

```text
RequestDelegate
    =
HTTP request-processing delegate

_next
    =
next component in pipeline

await _next(context)
    =
continue downstream

app.Use(...)
    =
add non-terminal middleware

app.Run(...)
    =
terminal middleware

app.Map(...)
    =
branch pipeline

Before next()
    =
request-side processing

After next()
    =
response-side processing

No next()
    =
short-circuit
```

---

# 37. Final Mental Picture

The entire concept can be summarized as:

```text
                    HTTP REQUEST
                         ↓
                ┌────────────────┐
                │ Middleware A   │
                │                │
                │ Before         │
                │      ↓         │
                │   _next()      │
                └──────┬─────────┘
                       ↓
                ┌────────────────┐
                │ Middleware B   │
                │                │
                │ Before         │
                │      ↓         │
                │   _next()      │
                └──────┬─────────┘
                       ↓
                ┌────────────────┐
                │ Middleware C   │
                │                │
                │ Before         │
                │      ↓         │
                │   _next()      │
                └──────┬─────────┘
                       ↓
                 CONTROLLER
                       ↓
                    RESPONSE
                       ↑
                  C After
                       ↑
                  B After
                       ↑
                  A After
                       ↑
                     CLIENT
```

The most important internal chain is:

```text
A
 ↓ _next
B
 ↓ _next
C
 ↓ _next
Endpoint
```

And the most important execution pattern is:

```text
A Before
    ↓
B Before
    ↓
C Before
    ↓
Endpoint
    ↑
C After
    ↑
B After
    ↑
A After
```

This is the foundation for understanding ASP.NET Core's request pipeline.
