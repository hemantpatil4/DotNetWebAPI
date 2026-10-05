# .NET Web API — Part 10: Global Exception Handling

## 1. What problem are we solving?

In a real Web API, exceptions can occur at many layers:

```text
HTTP Request
    ↓
Middleware
    ↓
Controller
    ↓
Service
    ↓
Repository
    ↓
Database / External API
    ↓
Exception
```

For example:

```text
GET /api/users/100
        ↓
UsersController
        ↓
UserService
        ↓
UserRepository
        ↓
Database
        ↓
User not found
        ↓
Exception
```

We need a consistent way to:

- catch unexpected exceptions
- convert known exceptions into appropriate HTTP status codes
- log the exception
- return a safe response to the client
- avoid repeating `try/catch` in every controller
- avoid exposing stack traces, SQL details, connection strings, file paths, etc.

This is the purpose of **global exception handling**.

---

# 2. The bad approach: try/catch in every controller

You could write:

```csharp
[HttpGet("{id}")]
public IActionResult GetUser(int id)
{
    try
    {
        var user = _service.GetUser(id);

        if (user == null)
            return NotFound();

        return Ok(user);
    }
    catch (Exception ex)
    {
        return StatusCode(500);
    }
}
```

Then another controller:

```csharp
[HttpPost]
public IActionResult CreateUser(CreateUserRequest request)
{
    try
    {
        var user = _service.CreateUser(request);
        return Ok(user);
    }
    catch (Exception ex)
    {
        return StatusCode(500);
    }
}
```

And another:

```csharp
[HttpDelete("{id}")]
public IActionResult DeleteUser(int id)
{
    try
    {
        _service.DeleteUser(id);
        return NoContent();
    }
    catch (Exception ex)
    {
        return StatusCode(500);
    }
}
```

This creates duplication.

More importantly, exception handling is a **cross-cutting concern**.

Other cross-cutting concerns include:

- logging
- authentication
- authorization
- correlation IDs
- metrics
- tracing
- rate limiting

Middleware is a natural place for concerns that should apply across many/all endpoints.

---

# 3. Global exception handling

Instead of:

```text
Controller 1 → try/catch
Controller 2 → try/catch
Controller 3 → try/catch
Controller 4 → try/catch
```

we want:

```text
                  Global Exception Middleware
                           │
                           ↓
                       Controller
                           │
                           ↓
                        Service
                           │
                           ↓
                       Repository
                           │
                           ↓
                       Exception
                           │
                           ↑
                  Middleware catches it
                           │
                           ↓
                    HTTP Error Response
```

The controller does not need to know how exceptions are converted into HTTP responses.

---

# 4. The key middleware concept

From Part 6 we learned:

```csharp
await _next(context);
```

`_next` represents the rest of the pipeline.

Therefore:

```csharp
try
{
    await _next(context);
}
catch (Exception ex)
{
    // Handle exception
}
```

means:

```text
Start middleware
      ↓
try
      ↓
run everything downstream
      ↓
controller
      ↓
service
      ↓
repository
      ↓
exception occurs
      ↓
exception propagates backward
      ↓
catch
      ↓
create HTTP response
```

This is the central idea behind global exception middleware.

---

# 5. Basic global exception middleware

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;

    public ExceptionHandlingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }

    private static async Task HandleExceptionAsync(
        HttpContext context,
        Exception exception)
    {
        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        context.Response.ContentType = "application/json";

        await context.Response.WriteAsJsonAsync(new
        {
            message = "An unexpected error occurred."
        });
    }
}
```

---

# 6. Registering the middleware

In `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

var app = builder.Build();

app.UseMiddleware<ExceptionHandlingMiddleware>();

app.MapControllers();

app.Run();
```

The important line is:

```csharp
app.UseMiddleware<ExceptionHandlingMiddleware>();
```

It places our exception handler into the request pipeline.

---

# 7. Why the middleware catches controller exceptions

Suppose the controller does:

```csharp
[HttpGet]
public IActionResult Test()
{
    throw new Exception("Something went wrong!");
}
```

Request flow:

```text
GET /api/test
       ↓
ExceptionHandlingMiddleware
       ↓
await _next(context)
       ↓
Controller
       ↓
throw Exception
       ↓
Exception propagates backward
       ↑
catch(Exception ex)
       ↓
HandleExceptionAsync()
       ↓
HTTP 500
```

The important mental model is:

```csharp
try
{
    await downstream;
}
catch
{
    handle;
}
```

The middleware is effectively wrapping everything downstream in a `try/catch`.

---

# 8. Why logging is important

The client should generally NOT receive:

```text
System.NullReferenceException:
Object reference not set to an instance of an object.

at MyCompany.Services.UserService.GetUser(...)
at MyCompany.Controllers.UsersController.GetUser(...)
...
```

But developers need the full exception for debugging.

Therefore:

```text
Client
   ↓
Safe error response

Application logs
   ↓
Full exception + stack trace + context
```

Inject `ILogger<T>`:

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(
                ex,
                "Unhandled exception while processing {Path}",
                context.Request.Path);

            await HandleExceptionAsync(context, ex);
        }
    }

    private static async Task HandleExceptionAsync(
        HttpContext context,
        Exception exception)
    {
        context.Response.StatusCode = 500;
        context.Response.ContentType = "application/json";

        await context.Response.WriteAsJsonAsync(new
        {
            message = "An unexpected error occurred."
        });
    }
}
```

`LogError(ex, ...)` is important because the exception object contains the stack trace.

---

# 9. Not every exception means HTTP 500

This is extremely important.

Consider:

```text
User doesn't exist
```

This is not necessarily a server failure.

It could map to:

```text
404 Not Found
```

Another example:

```text
Trading limit exceeded
```

This may be a business rule failure:

```text
422 Unprocessable Entity
```

Another:

```text
Unexpected database failure
```

This may become:

```text
500 Internal Server Error
```

So global exception handling often contains an exception-to-status-code mapping.

---

# 10. Custom exception #1 — UserNotFoundException

Create a domain/application exception:

```csharp
public class UserNotFoundException : Exception
{
    public UserNotFoundException(int userId)
        : base($"User with ID {userId} was not found.")
    {
    }
}
```

Service:

```csharp
public User GetUser(int userId)
{
    var user = _repository.GetUser(userId);

    if (user == null)
    {
        throw new UserNotFoundException(userId);
    }

    return user;
}
```

The service now communicates the actual business/application situation:

```text
User does not exist
```

It does not need to know that this should eventually become HTTP 404.

That separation is useful.

---

# 11. Custom exception #2 — TradingLimitExceededException

In an FX/trading system, suppose a dealer attempts a transaction above the configured limit.

```csharp
public class TradingLimitExceededException : Exception
{
    public decimal RequestedAmount { get; }
    public decimal AllowedLimit { get; }

    public TradingLimitExceededException(
        decimal requestedAmount,
        decimal allowedLimit)
        : base(
            $"Trading limit exceeded. " +
            $"Requested: {requestedAmount}, " +
            $"Allowed: {allowedLimit}.")
    {
        RequestedAmount = requestedAmount;
        AllowedLimit = allowedLimit;
    }
}
```

Service:

```csharp
public void ValidateTradingLimit(
    decimal amount,
    decimal limit)
{
    if (amount > limit)
    {
        throw new TradingLimitExceededException(
            amount,
            limit);
    }
}
```

This is a domain-specific exception.

The controller doesn't need:

```csharp
if (amount > limit)
{
    return UnprocessableEntity(...);
}
```

The service can raise the domain condition and the global handler can translate it into the API contract.

---

# 12. Custom exception #3 — InvalidCurrencyPairException

For an FX system:

```csharp
public class InvalidCurrencyPairException : Exception
{
    public string CurrencyPair { get; }

    public InvalidCurrencyPairException(string currencyPair)
        : base($"Invalid currency pair: {currencyPair}.")
    {
        CurrencyPair = currencyPair;
    }
}
```

Service:

```csharp
public void ValidateCurrencyPair(string currencyPair)
{
    var supportedPairs = new[]
    {
        "EURUSD",
        "USDJPY",
        "GBPUSD",
        "USDCHF"
    };

    if (!supportedPairs.Contains(currencyPair))
    {
        throw new InvalidCurrencyPairException(currencyPair);
    }
}
```

Possible API mapping:

```text
InvalidCurrencyPairException
        ↓
400 Bad Request
```

---

# 13. Exception mapping

We can now map:

```text
Exception                         HTTP Status
------------------------------------------------
UserNotFoundException             404
InvalidCurrencyPairException      400
TradingLimitExceededException     422
Unexpected Exception              500
```

Example:

```csharp
private static int GetStatusCode(Exception exception)
{
    return exception switch
    {
        UserNotFoundException =>
            StatusCodes.Status404NotFound,

        InvalidCurrencyPairException =>
            StatusCodes.Status400BadRequest,

        TradingLimitExceededException =>
            StatusCodes.Status422UnprocessableEntity,

        _ =>
            StatusCodes.Status500InternalServerError
    };
}
```

This is a very useful C# pattern:

```csharp
exception switch
{
    SomeException => statusCode,
    AnotherException => statusCode,
    _ => defaultStatusCode
};
```

---

# 14. ProblemDetails

A production API should preferably return a standardized structured error response instead of random anonymous JSON.

Example:

```json
{
    "type": "https://api.example.com/errors/user-not-found",
    "title": "User Not Found",
    "status": 404,
    "detail": "User with ID 100 was not found.",
    "instance": "/api/users/100"
}
```

Important fields:

### type

Identifies the type of error.

### title

Short human-readable summary.

### status

HTTP status code.

### detail

More specific explanation.

### instance

The request/resource where the problem occurred.

---

# 15. Why ProblemDetails is better

Without a standard:

```json
{
    "message": "User not found"
}
```

Another endpoint might return:

```json
{
    "error": "Invalid request",
    "code": 123
}
```

Another:

```json
{
    "errorMessage": "Something failed"
}
```

The client has to understand multiple formats.

With a consistent contract:

```json
{
    "type": "...",
    "title": "...",
    "status": 404,
    "detail": "...",
    "instance": "..."
}
```

the client has a predictable structure.

---

# 16. Production-style middleware using ProblemDetails

Example:

```csharp
using Microsoft.AspNetCore.Mvc;

public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(
                ex,
                "Unhandled exception. TraceId: {TraceId}, Path: {Path}",
                context.TraceIdentifier,
                context.Request.Path);

            await HandleExceptionAsync(context, ex);
        }
    }

    private static async Task HandleExceptionAsync(
        HttpContext context,
        Exception exception)
    {
        var statusCode = exception switch
        {
            UserNotFoundException =>
                StatusCodes.Status404NotFound,

            InvalidCurrencyPairException =>
                StatusCodes.Status400BadRequest,

            TradingLimitExceededException =>
                StatusCodes.Status422UnprocessableEntity,

            _ =>
                StatusCodes.Status500InternalServerError
        };

        var problem = new ProblemDetails
        {
            Status = statusCode,
            Title = GetTitle(statusCode),
            Detail = GetSafeDetail(exception),
            Instance = context.Request.Path
        };

        context.Response.StatusCode = statusCode;
        context.Response.ContentType = "application/problem+json";

        await context.Response.WriteAsJsonAsync(problem);
    }

    private static string GetTitle(int statusCode)
    {
        return statusCode switch
        {
            400 => "Bad Request",
            404 => "Resource Not Found",
            422 => "Business Rule Violation",
            500 => "Internal Server Error",
            _ => "Error"
        };
    }

    private static string GetSafeDetail(Exception exception)
    {
        return exception switch
        {
            UserNotFoundException =>
                exception.Message,

            InvalidCurrencyPairException =>
                exception.Message,

            TradingLimitExceededException =>
                exception.Message,

            _ =>
                "An unexpected error occurred."
        };
    }
}
```

Notice the important security decision:

```csharp
_ => "An unexpected error occurred."
```

For unknown exceptions, we do not expose:

```text
SQL exception
connection string
database server
stack trace
file system path
internal class names
```

---

# 17. Full FX example

Suppose we have:

```text
POST /api/fx/orders
```

Request:

```json
{
    "currencyPair": "EURUSD",
    "amount": 5000000
}
```

Controller:

```csharp
[ApiController]
[Route("api/fx/orders")]
public class FXOrdersController : ControllerBase
{
    private readonly IFXOrderService _service;

    public FXOrdersController(IFXOrderService service)
    {
        _service = service;
    }

    [HttpPost]
    public IActionResult Create(FXOrderRequest request)
    {
        var order = _service.CreateOrder(request);

        return Ok(order);
    }
}
```

Service:

```csharp
public FXOrder CreateOrder(FXOrderRequest request)
{
    ValidateCurrencyPair(request.CurrencyPair);

    var limit = 1_000_000m;

    if (request.Amount > limit)
    {
        throw new TradingLimitExceededException(
            request.Amount,
            limit);
    }

    return _repository.Create(request);
}
```

Flow:

```text
POST /api/fx/orders
        ↓
Exception Middleware
        ↓
Routing
        ↓
FXOrdersController
        ↓
FXOrderService
        ↓
Validate trading limit
        ↓
5,000,000 > 1,000,000
        ↓
TradingLimitExceededException
        ↑
Exception Middleware catches
        ↓
Map exception
        ↓
HTTP 422
        ↓
ProblemDetails
```

---

# 18. Controller stays clean

Without global exception handling:

```csharp
try
{
    ...
}
catch (TradingLimitExceededException)
{
    ...
}
catch (UserNotFoundException)
{
    ...
}
catch (Exception)
{
    ...
}
```

Every controller becomes polluted with exception-handling logic.

With global handling:

```csharp
[HttpPost]
public IActionResult Create(FXOrderRequest request)
{
    var order = _service.CreateOrder(request);

    return Ok(order);
}
```

The controller focuses on HTTP interaction.

The service focuses on business logic.

The exception middleware focuses on translating exceptions into HTTP errors.

This separation is one of the most important architectural ideas here.

---

# 19. Exception responsibilities by layer

A useful mental model:

```text
Controller
    ↓
HTTP concerns

Service
    ↓
Business/application concerns

Repository
    ↓
Data access concerns

Exception Middleware
    ↓
HTTP error translation + centralized logging
```

Example:

```text
Repository
    ↓
Database failure
    ↓
Service may translate/wrap if appropriate
    ↓
Exception propagates
    ↓
Global handler
    ↓
500 ProblemDetails
```

---

# 20. Domain exceptions vs HTTP exceptions

Avoid making every business class return:

```csharp
IActionResult
```

For example, this is usually undesirable:

```csharp
public IActionResult ValidateTrade(...)
```

inside a service.

Instead:

```csharp
public void ValidateTrade(...)
{
    if (...)
        throw new TradingLimitExceededException(...);
}
```

Then the API layer decides:

```text
TradingLimitExceededException
        ↓
HTTP 422
```

This keeps the service less coupled to ASP.NET Core HTTP concepts.

---

# 21. Built-in `UseExceptionHandler`

ASP.NET Core provides built-in exception handling support.

Example:

```csharp
var app = builder.Build();

app.UseExceptionHandler("/error");

app.MapControllers();

app.Run();
```

Or it can be configured with a lambda.

The important idea is:

```text
ASP.NET Core already provides infrastructure
for centralized exception handling.
```

You should understand custom middleware because it teaches the mechanics.

For production applications, evaluate the built-in ASP.NET Core exception-handling infrastructure rather than automatically implementing everything yourself.

---

# 22. Modern `IExceptionHandler`

Modern ASP.NET Core also provides `IExceptionHandler` for centralized exception handling.

Conceptually:

```text
HTTP Request
      ↓
ASP.NET Core exception handling
      ↓
IExceptionHandler
      ↓
Exception-specific handling
      ↓
ProblemDetails
      ↓
HTTP Response
```

A handler can implement:

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(
        ILogger<GlobalExceptionHandler> logger)
    {
        _logger = logger;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(
            exception,
            "Unhandled exception");

        httpContext.Response.StatusCode =
            StatusCodes.Status500InternalServerError;

        await httpContext.Response.WriteAsJsonAsync(
            new
            {
                message = "An unexpected error occurred."
            },
            cancellationToken);

        return true;
    }
}
```

Registration:

```csharp
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
```

Pipeline:

```csharp
app.UseExceptionHandler();
```

The exact production design can be expanded with multiple handlers, ProblemDetails, logging, tracing, and exception classification.

---

# 23. Middleware ordering

Exception handling should be early enough to catch exceptions from downstream components.

Typical structure:

```csharp
var app = builder.Build();

app.UseMiddleware<ExceptionHandlingMiddleware>();

app.UseAuthentication();

app.UseAuthorization();

app.MapControllers();

app.Run();
```

Think:

```text
Exception Handler
      ↓
Authentication
      ↓
Authorization
      ↓
Controller
      ↓
Service
      ↓
Repository
```

If something below the exception handler throws, the exception can travel back up to it.

---

# 24. What if the response has already started?

This is an important edge case.

Suppose some middleware has already written part of the response:

```csharp
await context.Response.WriteAsync("Some data");

throw new Exception();
```

At this point:

```csharp
context.Response.HasStarted
```

may be `true`.

Changing the status code may no longer be possible.

Therefore production exception handling needs to consider:

```csharp
if (context.Response.HasStarted)
{
    _logger.LogError(
        exception,
        "Exception occurred after response started.");

    throw;
}
```

The key idea:

```text
Before response starts
    ↓
Can usually create error response

After response starts
    ↓
HTTP response may already be committed
    ↓
Cannot safely replace it with a normal error response
```

---

# 25. Cancellation and `OperationCanceledException`

Not every exception represents an application failure.

A request can be cancelled because:

- client disconnected
- request timed out
- cancellation token was triggered

For example:

```csharp
await _service.ProcessAsync(
    context.RequestAborted);
```

This can result in cancellation-related exceptions.

Do not blindly classify every exception as:

```text
500 Internal Server Error
```

Exception handling should distinguish:

```text
Business exception
Validation exception
Authentication/authorization failure
Infrastructure failure
Cancellation
Unexpected application failure
```

---

# 26. Correlation / Trace IDs

When debugging production systems, you need to connect:

```text
Client request
        ↓
API logs
        ↓
Service logs
        ↓
Database/external service logs
```

A request identifier is useful.

ASP.NET Core provides:

```csharp
context.TraceIdentifier
```

Example:

```csharp
_logger.LogError(
    ex,
    "Unhandled exception. TraceId: {TraceId}",
    context.TraceIdentifier);
```

Response can optionally include a safe identifier:

```json
{
    "title": "Internal Server Error",
    "status": 500,
    "traceId": "0H..."
}
```

This lets support teams say:

```text
Please provide the trace ID.
```

Then developers can search logs for that request.

In distributed systems, this concept extends into distributed tracing and correlation IDs propagated across services.

---

# 27. Structured logging

Prefer:

```csharp
_logger.LogError(
    ex,
    "Failed to create FX order for {CurrencyPair}",
    request.CurrencyPair);
```

over:

```csharp
_logger.LogError(
    $"Failed to create FX order for {request.CurrencyPair}");
```

Structured logging keeps fields separately queryable.

For example:

```text
CurrencyPair = EURUSD
OrderId = 12345
TraceId = abc123
```

This is much better for production observability systems.

---

# 28. What should NOT be returned to clients?

Avoid exposing:

```text
Stack trace
SQL query
Database connection string
Internal IP address
Server file path
Class names
Assembly names
Secret values
Authentication tokens
Internal exception chain
```

Bad:

```json
{
    "error": "SqlException: Login failed for user 'sa'",
    "stackTrace": "...",
    "connectionString": "Server=..."
}
```

Good:

```json
{
    "title": "Internal Server Error",
    "status": 500,
    "detail": "An unexpected error occurred."
}
```

The detailed information belongs in logs, not in the public API response.

---

# 29. Exception hierarchy

You can also introduce a common application exception.

```csharp
public abstract class ApplicationExceptionBase : Exception
{
    protected ApplicationExceptionBase(string message)
        : base(message)
    {
    }
}
```

Then:

```csharp
public class UserNotFoundException
    : ApplicationExceptionBase
{
    public UserNotFoundException(int userId)
        : base($"User {userId} was not found.")
    {
    }
}
```

And:

```csharp
public class TradingLimitExceededException
    : ApplicationExceptionBase
{
    public TradingLimitExceededException(
        decimal amount,
        decimal limit)
        : base(
            $"Trading limit exceeded. " +
            $"Requested: {amount}, Allowed: {limit}.")
    {
    }
}
```

This can help classify application/domain exceptions.

However, don't create dozens of exception classes without a meaningful distinction. Exceptions should represent meaningful failure categories.

---

# 30. Exception mapping strategy

A scalable design can separate mapping from middleware.

Instead of putting everything here:

```csharp
switch (exception)
{
    ...
}
```

you can introduce:

```csharp
public interface IExceptionToProblemDetailsMapper
{
    ProblemDetails Map(
        Exception exception,
        HttpContext context);
}
```

Then:

```text
Exception Middleware
        ↓
Exception Mapper
        ↓
ProblemDetails
        ↓
HTTP Response
```

This becomes useful as the application grows.

For a smaller application, a switch expression is perfectly reasonable.

---

# 31. Complete request flow

Let's put everything together.

Request:

```text
POST /api/fx/orders
```

↓

```text
Kestrel
```

↓

```text
Middleware Pipeline
```

↓

```text
Global Exception Handler
```

↓

```text
Authentication
```

↓

```text
Authorization
```

↓

```text
Routing / Endpoint
```

↓

```text
FXOrdersController
```

↓

```text
FXOrderService
```

↓

```text
Trading limit validation
```

↓

```text
TradingLimitExceededException
```

↓

Exception propagates backward:

```text
FXOrderService
      ↑
Controller
      ↑
Authorization
      ↑
Exception Handler catches
```

↓

```text
Log exception
```

↓

```text
Map exception → HTTP 422
```

↓

```text
Create ProblemDetails
```

↓

```text
HTTP 422
```

Example:

```json
{
    "type": "https://api.example.com/errors/trading-limit-exceeded",
    "title": "Business Rule Violation",
    "status": 422,
    "detail": "Trading limit exceeded.",
    "instance": "/api/fx/orders"
}
```

---

# 32. Complete custom middleware example

A clean learning implementation:

```csharp
using Microsoft.AspNetCore.Mvc;

public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception exception)
        {
            await HandleExceptionAsync(context, exception);
        }
    }

    private async Task HandleExceptionAsync(
        HttpContext context,
        Exception exception)
    {
        _logger.LogError(
            exception,
            "Unhandled exception. TraceId: {TraceId}, Path: {Path}",
            context.TraceIdentifier,
            context.Request.Path);

        if (context.Response.HasStarted)
        {
            _logger.LogWarning(
                "Cannot write exception response because response has already started.");

            throw;
        }

        var statusCode = exception switch
        {
            UserNotFoundException =>
                StatusCodes.Status404NotFound,

            InvalidCurrencyPairException =>
                StatusCodes.Status400BadRequest,

            TradingLimitExceededException =>
                StatusCodes.Status422UnprocessableEntity,

            _ =>
                StatusCodes.Status500InternalServerError
        };

        var detail = exception switch
        {
            UserNotFoundException =>
                exception.Message,

            InvalidCurrencyPairException =>
                exception.Message,

            TradingLimitExceededException =>
                exception.Message,

            _ =>
                "An unexpected error occurred."
        };

        var problem = new ProblemDetails
        {
            Status = statusCode,
            Title = statusCode switch
            {
                400 => "Bad Request",
                404 => "Not Found",
                422 => "Business Rule Violation",
                _ => "Internal Server Error"
            },
            Detail = detail,
            Instance = context.Request.Path
        };

        problem.Extensions["traceId"] =
            context.TraceIdentifier;

        context.Response.StatusCode = statusCode;
        context.Response.ContentType =
            "application/problem+json";

        await context.Response.WriteAsJsonAsync(problem);
    }
}
```

Registration:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

var app = builder.Build();

app.UseMiddleware<ExceptionHandlingMiddleware>();

app.UseAuthentication();

app.UseAuthorization();

app.MapControllers();

app.Run();
```

---

# 33. Global exception handling vs validation

Do not confuse Part 9 and Part 10.

## Validation

Example:

```json
{
    "currencyPair": "",
    "amount": -100
}
```

The request is invalid.

Usually:

```text
Model Binding
      ↓
Validation
      ↓
ModelState invalid
      ↓
400 Bad Request
```

## Exception

Example:

```text
Database connection failed
```

or:

```text
UnexpectedNullReference
```

or:

```text
TradingLimitExceededException
```

Then:

```text
Service
   ↓
Exception
   ↓
Global exception handler
   ↓
HTTP error response
```

Mental model:

```text
Validation
    ↓
"The input is invalid."

Exception
    ↓
"Something went wrong while processing the request."
```

---

# 34. Exception handling vs business result

Not every business outcome needs to be an exception.

For example:

```text
GET user
```

If user doesn't exist, an exception such as:

```text
UserNotFoundException
```

can be reasonable depending on the application's conventions.

But don't use exceptions for ordinary control flow everywhere.

For example, avoid:

```csharp
try
{
    var user = repository.GetUser(id);
}
catch (UserNotFoundException)
{
    return null;
}
```

if absence is an expected result that can simply be represented by:

```csharp
var user = repository.GetUser(id);

if (user == null)
{
    ...
}
```

The right boundary depends on the application architecture.

---

# 35. Interview question: Why global exception handling?

Answer:

> Global exception handling centralizes error handling for the entire API. Instead of putting repetitive try/catch blocks in every controller, exceptions can propagate through the middleware pipeline to one centralized handler. The handler logs the exception, maps known application exceptions to appropriate HTTP status codes, and returns a consistent error contract such as ProblemDetails while hiding sensitive internal information.

---

# 36. Interview question: How does middleware catch controller exceptions?

Answer:

> The exception middleware calls `await _next(context)` inside a try/catch. `_next` represents the rest of the pipeline, including routing, controllers, services, and repositories. If anything downstream throws an exception, it propagates back through the awaited call and is caught by the middleware.

Short version:

```csharp
try
{
    await _next(context);
}
catch (Exception ex)
{
    // handle
}
```

---

# 37. Interview question: Why should exceptions be mapped to different status codes?

Because:

```text
UserNotFoundException
```

is different from:

```text
Database connection failure
```

and different from:

```text
TradingLimitExceededException
```

Possible mapping:

```text
UserNotFoundException
        ↓
404

InvalidCurrencyPairException
        ↓
400

TradingLimitExceededException
        ↓
422

Unexpected exception
        ↓
500
```

This gives API consumers meaningful semantics.

---

# 38. Interview question: Why not return the exception message directly?

Because exception messages can contain sensitive implementation details.

For example:

```text
SqlException
connection string
table names
file paths
internal service names
stack trace
```

Therefore:

```text
Known safe business exception
    ↓
May expose a carefully designed message

Unexpected infrastructure exception
    ↓
Generic client message
    ↓
Detailed server-side log
```

---

# 39. Interview question: What is ProblemDetails?

Answer:

> ProblemDetails is a standardized structure for representing HTTP API errors. It provides fields such as `type`, `title`, `status`, `detail`, and `instance`, allowing clients to consume errors consistently.

---

# 40. Interview question: Where should logging happen?

A common production approach is to log unhandled exceptions at the centralized exception boundary.

Example:

```csharp
_logger.LogError(
    exception,
    "Unhandled exception. TraceId: {TraceId}",
    context.TraceIdentifier);
```

This prevents every controller from duplicating the same logging behavior.

Business-level events can still be logged at appropriate service/application boundaries.

---

# 41. Important distinction: exception handling does not mean swallowing exceptions

Bad:

```csharp
catch (Exception ex)
{
    // ignore
}
```

This is dangerous because the application may appear successful when it actually failed.

Global handling should:

```text
Catch
 ↓
Log
 ↓
Classify
 ↓
Create safe response
```

not:

```text
Catch
 ↓
Ignore
```

---

# 42. Final architecture

```text
                       CLIENT
                          |
                          | HTTP Request
                          v
                    +-----------+
                    |  Kestrel  |
                    +-----------+
                          |
                          v
              +-----------------------+
              | Global Exception      |
              | Handler               |
              +-----------------------+
                          |
                          v
              +-----------------------+
              | Authentication       |
              +-----------------------+
                          |
                          v
              +-----------------------+
              | Authorization        |
              +-----------------------+
                          |
                          v
              +-----------------------+
              | Controller            |
              +-----------------------+
                          |
                          v
              +-----------------------+
              | Application / Service |
              +-----------------------+
                          |
                          v
              +-----------------------+
              | Repository             |
              +-----------------------+
                          |
                          v
                     DATABASE
```

On failure:

```text
DATABASE
   |
   X
Exception
   |
   ↑
Repository
   |
   ↑
Service
   |
   ↑
Controller
   |
   ↑
Global Exception Handler
   |
   +---- Log full exception
   |
   +---- Determine exception type
   |
   +---- Determine HTTP status
   |
   +---- Build ProblemDetails
   |
   +---- Add TraceId
   |
   +---- Return safe response
   |
   v
CLIENT
```

---

# 43. Part 10 mental model

Remember this one picture:

```text
                 HTTP REQUEST
                       |
                       v
          +-------------------------+
          | Global Exception Handler|
          |                         |
          | try                     |
          |   await _next(context)  |
          | catch(Exception ex)     |
          |   log                   |
          |   map                   |
          |   ProblemDetails        |
          |   response              |
          +-------------------------+
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

The most important line is:

```csharp
await _next(context);
```

and the most important surrounding pattern is:

```csharp
try
{
    await _next(context);
}
catch (Exception ex)
{
    // Centralized handling
}
```

---

# 44. One-minute revision

### Why?

Avoid repeated try/catch blocks.

### Where?

Middleware / centralized exception infrastructure.

### How?

```csharp
try
{
    await _next(context);
}
catch (Exception ex)
{
    ...
}
```

### What do we do?

```text
Log
 ↓
Classify
 ↓
Map
 ↓
ProblemDetails
 ↓
HTTP response
```

### Example mapping

```text
UserNotFoundException
    → 404

InvalidCurrencyPairException
    → 400

TradingLimitExceededException
    → 422

Unexpected Exception
    → 500
```

### Security

Never expose:

```text
stack traces
SQL details
connection strings
internal paths
secrets
```

### Modern ASP.NET Core

Know:

```text
UseExceptionHandler
IExceptionHandler
ProblemDetails
```

### Core interview statement

> Global exception handling uses centralized ASP.NET Core exception infrastructure or middleware to catch exceptions that propagate through the request pipeline, log them, map known exceptions to meaningful HTTP status codes, and return a consistent, safe error response such as ProblemDetails.

---

# 45. What comes next?

After Part 10, the next major topic is:

**Part 11 — async/await in ASP.NET Core**

Important areas:

```text
async
await
Task
Task<T>
I/O-bound work
Thread pool
.Result
.Wait()
.GetAwaiter().GetResult()
deadlocks
thread starvation
cancellation tokens
```

This is particularly important for Web APIs because database calls, HTTP calls, Redis calls, file operations, and other I/O operations are normally asynchronous.
