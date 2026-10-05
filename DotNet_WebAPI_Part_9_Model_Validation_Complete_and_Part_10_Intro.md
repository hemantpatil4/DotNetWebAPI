# .NET Web API — Part 9: Model Validation — Complete Reference

## 1. Validation Pipeline

Model validation happens after model binding.

```text
HTTP Request
     ↓
Routing
     ↓
Model Binding
     ↓
C# Model / DTO
     ↓
Model Validation
     ↓
ModelState
     ↓
[ApiController]
     ↓
400 OR Controller Action
```

### Binding vs Validation

Binding asks:

> Can the request data be converted into the target .NET type?

Validation asks:

> Is the resulting value/model acceptable according to our rules?

Example binding error:

```json
{
  "age": "abc"
}
```

when:

```csharp
public int Age { get; set; }
```

Example validation error:

```json
{
  "age": -10
}
```

when:

```csharp
[Range(18, 100)]
public int Age { get; set; }
```

---

# 2. DataAnnotations

Common built-in attributes:

```csharp
[Required]
[Range(1, 100)]
[StringLength(50)]
[MinLength(3)]
[MaxLength(50)]
[EmailAddress]
[Phone]
[Url]
[Compare(nameof(Password))]
[RegularExpression(...)]
```

Namespace:

```csharp
using System.ComponentModel.DataAnnotations;
```

Example:

```csharp
public class CreateUserRequest
{
    [Required]
    [StringLength(100, MinimumLength = 3)]
    public string Name { get; set; } = string.Empty;

    [Range(18, 100)]
    public int Age { get; set; }

    [Required]
    [EmailAddress]
    public string Email { get; set; } = string.Empty;
}
```

---

# 3. `[Required]`

Example:

```csharp
[Required]
public string Name { get; set; } = string.Empty;
```

It expresses that a value is required.

For strings, if whitespace should also be rejected, understand the exact behavior you need and consider a custom rule where appropriate.

For nullable reference types, use the C# nullability annotations alongside validation.

Example:

```csharp
[Required]
public string? Name { get; set; }
```

The nullable annotation and validation attribute serve different purposes:

```text
string?
    ↓
C# compiler/nullability intent

[Required]
    ↓
Runtime/request validation
```

---

# 4. `[Range]`

```csharp
[Range(18, 100)]
public int Age { get; set; }
```

Examples:

```text
18  → valid
25  → valid
100 → valid
17  → invalid
101 → invalid
```

For decimals:

```csharp
[Range(typeof(decimal), "0.01", "1000000")]
public decimal Amount { get; set; }
```

Be careful with numeric limits and choose the correct type/range for financial applications.

---

# 5. `[StringLength]`

```csharp
[StringLength(50)]
public string Name { get; set; } = string.Empty;
```

With minimum:

```csharp
[StringLength(50, MinimumLength = 3)]
public string Name { get; set; } = string.Empty;
```

This is useful for API contracts where the allowed length is part of the request specification.

---

# 6. `[MinLength]` and `[MaxLength]`

```csharp
[MinLength(3)]
public string CurrencyPair { get; set; } = string.Empty;
```

and:

```csharp
[MaxLength(100)]
public List<string> Tags { get; set; } = [];
```

These can be useful for strings and collections.

---

# 7. `[EmailAddress]`

```csharp
[Required]
[EmailAddress]
public string Email { get; set; } = string.Empty;
```

The important idea is that multiple validators can apply to one property:

```text
Email
  |
  +--> Required
  |
  +--> EmailAddress
```

All applicable validation rules participate in validation.

---

# 8. `[Compare]`

Useful when two values must match.

```csharp
public class RegisterRequest
{
    [Required]
    public string Password { get; set; } = string.Empty;

    [Required]
    [Compare(nameof(Password))]
    public string ConfirmPassword { get; set; } = string.Empty;
}
```

If:

```text
Password       = abc123
ConfirmPassword = abc124
```

validation fails.

This is a good example of validation involving more than one property.

---

# 9. `[RegularExpression]`

Example:

```csharp
[RegularExpression(@"^[A-Z]{3}$")]
public string Currency { get; set; } = string.Empty;
```

Valid:

```text
USD
EUR
INR
```

Invalid:

```text
US
USDX
usd
USD1
```

Regex is useful for strict formats, but don't use complicated regex when a simpler type or validation rule is more maintainable.

---

# 10. Multiple Validation Attributes

Example:

```csharp
public class FXOrderRequest
{
    [Required]
    [StringLength(6, MinimumLength = 6)]
    [RegularExpression("^[A-Z]+$")]
    public string CurrencyPair { get; set; } = string.Empty;

    [Range(typeof(decimal), "0.01", "1000000000")]
    public decimal Amount { get; set; }
}
```

Conceptually:

```text
CurrencyPair
     |
     +--> Required
     +--> StringLength
     +--> RegularExpression

Amount
     |
     +--> Range
```

For more domain-specific rules, a custom validator may be cleaner.

---

# 11. Custom Validation Attribute

A custom validation attribute inherits from:

```csharp
ValidationAttribute
```

Example:

```csharp
public class CurrencyPairAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(
        object? value,
        ValidationContext validationContext)
    {
        var currencyPair = value as string;

        if (string.IsNullOrWhiteSpace(currencyPair))
        {
            return new ValidationResult(
                "Currency pair is required.");
        }

        if (currencyPair.Length != 6)
        {
            return new ValidationResult(
                "Currency pair must contain exactly 6 characters.");
        }

        if (!currencyPair.All(char.IsUpper))
        {
            return new ValidationResult(
                "Currency pair must contain uppercase letters only.");
        }

        if (!currencyPair.All(char.IsLetter))
        {
            return new ValidationResult(
                "Currency pair must contain letters only.");
        }

        return ValidationResult.Success;
    }
}
```

Usage:

```csharp
public class FXOrderRequest
{
    [CurrencyPair]
    public string CurrencyPair { get; set; } = string.Empty;

    public decimal Amount { get; set; }
}
```

---

# 12. Understanding `ValidationAttribute`

The framework understands the standard validation abstraction.

Our class:

```csharp
public class CurrencyPairAttribute : ValidationAttribute
```

becomes another validator.

The important method is:

```csharp
IsValid(...)
```

Conceptually:

```text
ValidationAttribute
       ↓
CurrencyPairAttribute
       ↓
IsValid(value, context)
       ↓
ValidationResult
```

Valid:

```csharp
return ValidationResult.Success;
```

Invalid:

```csharp
return new ValidationResult("Error message");
```

---

# 13. `ValidationContext`

Signature:

```csharp
protected override ValidationResult? IsValid(
    object? value,
    ValidationContext validationContext)
```

`ValidationContext` contains contextual information about the validation operation.

It can expose information such as:

```text
ObjectInstance
MemberName
DisplayName
GetService(...)
```

It becomes useful for advanced scenarios.

However, don't turn a simple validation attribute into a business-service layer.

---

# 14. `IValidatableObject`

Use `IValidatableObject` when validation naturally belongs to the whole object.

Example:

```csharp
public class FXOrderRequest : IValidatableObject
{
    public string Side { get; set; } = string.Empty;

    public decimal Amount { get; set; }

    public IEnumerable<ValidationResult> Validate(
        ValidationContext validationContext)
    {
        if (Side == "Buy" && Amount <= 1_000_000)
        {
            yield return new ValidationResult(
                "Buy orders must exceed 1,000,000.",
                new[]
                {
                    nameof(Side),
                    nameof(Amount)
                });
        }
    }
}
```

This is appropriate when the rule involves multiple properties.

Mental model:

```text
Property validator
    ↓
Validates one property

IValidatableObject
    ↓
Validates the whole object
```

---

# 15. Cross-Property Validation

Example:

```text
StartDate < EndDate
```

Both values are needed.

Or:

```text
If Side = Buy,
Amount must satisfy a particular rule.
```

These are not naturally single-property rules.

Possible approaches:

```text
DataAnnotations
     ↓
Compare / custom attribute

IValidatableObject
     ↓
Object-level validation

Application/service layer
     ↓
Complex business validation
```

Choose the layer according to the rule.

---

# 16. ModelState

`ModelState` is a dictionary-like structure containing information associated with request fields/parameters and their binding/validation errors.

Example:

```csharp
if (!ModelState.IsValid)
{
    return BadRequest(ModelState);
}
```

Conceptually:

```text
ModelState
 |
 +-- CurrencyPair
 |      |
 |      +-- errors
 |
 +-- Amount
        |
        +-- errors
```

Errors can originate from:

```text
Model Binding
     OR
Model Validation
```

This is why ModelState is broader than just validation attributes.

---

# 17. `[ApiController]`

Example:

```csharp
[ApiController]
[Route("api/fx/orders")]
public class FXOrdersController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(FXOrderRequest request)
    {
        return Ok();
    }
}
```

`[ApiController]` enables API-specific behavior including:

- binding-source inference
- automatic HTTP 400 responses for invalid ModelState
- API-oriented conventions

Therefore this repeated code is often unnecessary:

```csharp
if (!ModelState.IsValid)
{
    return BadRequest(ModelState);
}
```

when `[ApiController]` is being used with its default invalid-model-state behavior.

---

# 18. Automatic 400 Flow

```text
HTTP Request
      ↓
Model Binding
      ↓
Model Validation
      ↓
ModelState
      |
      +---- Valid ------> Action
      |
      +---- Invalid ----> ApiController behavior
                              ↓
                          HTTP 400
```

This means the action may never execute when ModelState is invalid.

---

# 19. ValidationProblemDetails

Modern ASP.NET Core APIs commonly return structured problem details for API errors.

Conceptually:

```json
{
  "type": "...",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "CurrencyPair": [
      "Currency pair must contain uppercase letters only."
    ]
  }
}
```

The exact serialized representation depends on the framework configuration/version, but the key concept is:

```text
ProblemDetails
     |
     +-- status
     +-- title
     +-- detail/type where applicable
     +-- validation errors
```

This is much better than returning arbitrary strings from every controller.

---

# 20. Customize Invalid Model State Response

ASP.NET Core allows customizing the API-controller behavior.

Conceptually:

```csharp
builder.Services
    .AddControllers()
    .ConfigureApiBehaviorOptions(options =>
    {
        options.InvalidModelStateResponseFactory = context =>
        {
            return new BadRequestObjectResult(
                context.ModelState);
        };
    });
```

A production API may use a standardized error contract instead.

The important idea is:

```text
Default framework behavior
          OR
Custom API error contract
```

---

# 21. Request Validation vs Business Validation

This distinction is critical.

## Request validation

Examples:

```text
CurrencyPair must be 6 characters
Amount must be positive
Email must have valid format
Name is required
```

These can often be handled by:

```text
DataAnnotations
Custom validation attributes
IValidatableObject
Request validators
```

---

## Business validation

Examples:

```text
Does this trader have enough limit?

Is EURUSD trading currently allowed?

Is the counterparty approved?

Does this order violate exposure rules?

Can this FX product be booked for this client?
```

These generally require application/domain state.

Architecture:

```text
Controller
    ↓
FXOrderService
    ↓
RiskService
    ↓
Limit / Position / Product rules
```

Don't put database-dependent business decisions into a simple DataAnnotation attribute.

---

# 22. Validation in the Service Layer

A service can still reject an operation after request validation.

Example:

```csharp
public async Task PlaceOrderAsync(FXOrderRequest request)
{
    var limit = await _riskService.GetLimitAsync();

    if (request.Amount > limit)
    {
        throw new TradingLimitExceededException();
    }

    // Continue booking...
}
```

This is not a replacement for request validation.

It is a different layer.

```text
Request Validation
    ↓
Is the request structurally valid?

Business Validation
    ↓
Is the requested operation allowed?
```

---

# 23. Validation Order — Mental Model

A useful simplified flow:

```text
HTTP Request
     ↓
Routing
     ↓
Model Binding
     ↓
Binding Errors?
     |
     +---- yes ---> ModelState
     |
     v
Model Validation
     ↓
Validation Errors?
     |
     +---- yes ---> ModelState
     |
     v
[ApiController]
     ↓
ModelState valid?
     |
     +---- no ---> 400
     |
     +---- yes --> Controller
```

This is a conceptual model; exact internal execution details depend on the MVC pipeline and configured components.

---

# 24. Multiple Errors

A request may violate multiple rules.

Example:

```json
{
  "currencyPair": "",
  "amount": -100
}
```

Possible errors:

```text
CurrencyPair:
    required

Amount:
    must be positive
```

The API should ideally return a structured collection of errors instead of stopping at the first problem.

This is another benefit of ModelState/problem-details-based responses.

---

# 25. Nested Object Validation

Suppose:

```csharp
public class CreateUserRequest
{
    [Required]
    public string Name { get; set; } = string.Empty;

    [Required]
    public AddressRequest Address { get; set; } = new();
}

public class AddressRequest
{
    [Required]
    public string City { get; set; } = string.Empty;
}
```

Request:

```json
{
  "name": "Hemant",
  "address": {
    "city": ""
  }
}
```

The validation system can represent the nested property error.

Conceptually:

```text
Address
   |
   +-- City
         |
         +-- Required failed
```

The error key may be represented using a property path such as:

```text
Address.City
```

depending on the API error serialization.

---

# 26. Nullable Reference Types vs Validation

These are different mechanisms.

```csharp
public string? Name { get; set; }
```

means:

> The compiler should treat null as potentially valid for this reference.

It does NOT by itself mean:

> The HTTP API must reject null.

For API validation:

```csharp
[Required]
public string? Name { get; set; }
```

communicates a runtime validation rule.

So:

```text
Nullable annotations
    ↓
Compile-time developer guidance

Validation attributes
    ↓
Runtime request validation
```

---

# 27. Don't Put Everything Into Attributes

Bad design:

```csharp
[ValidateEverything]
public string CurrencyPair { get; set; }
```

where the attribute:

```text
calls DB
calls Redis
calls external API
checks user
checks permissions
checks market
checks risk
```

This makes validation:

- difficult to test
- difficult to reason about
- tightly coupled
- potentially slow
- difficult to reuse
- inappropriate for a simple attribute

Use the appropriate layer.

---

# 28. Custom Validator — Complete FX Example

```csharp
using System.ComponentModel.DataAnnotations;

public class CurrencyPairAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(
        object? value,
        ValidationContext validationContext)
    {
        var currencyPair = value as string;

        if (string.IsNullOrWhiteSpace(currencyPair))
        {
            return new ValidationResult(
                "Currency pair is required.");
        }

        if (currencyPair.Length != 6)
        {
            return new ValidationResult(
                "Currency pair must contain exactly 6 characters.");
        }

        if (!currencyPair.All(char.IsUpper))
        {
            return new ValidationResult(
                "Currency pair must contain uppercase letters only.");
        }

        if (!currencyPair.All(char.IsLetter))
        {
            return new ValidationResult(
                "Currency pair must contain letters only.");
        }

        return ValidationResult.Success;
    }
}
```

DTO:

```csharp
public class FXOrderRequest
{
    [CurrencyPair]
    public string CurrencyPair { get; set; } = string.Empty;

    public decimal Amount { get; set; }
}
```

Controller:

```csharp
[ApiController]
[Route("api/fx/orders")]
public class FXOrdersController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(FXOrderRequest request)
    {
        return Ok("Order accepted");
    }
}
```

Valid:

```json
{
  "currencyPair": "EURUSD",
  "amount": 100000
}
```

Invalid:

```json
{
  "currencyPair": "eurusd",
  "amount": 100000
}
```

---

# 29. Testing Strategy

For validators, test the rule independently.

Test cases:

| Input | Expected |
|---|---|
| `EURUSD` | Valid |
| `USDINR` | Valid |
| `GBPJPY` | Valid |
| `EUR` | Invalid |
| `EURUSDX` | Invalid |
| `eurusd` | Invalid |
| `EUR/USD` | Invalid |
| `EUR123` | Invalid |
| `""` | Invalid |
| whitespace | Invalid |
| `null` | Invalid |

Then separately test the API endpoint:

```text
HTTP request
    ↓
Model binding
    ↓
Validation
    ↓
400 / action
```

This separates unit testing from integration testing.

---

# 30. Validation Libraries

For larger applications, teams may use dedicated validation libraries such as FluentValidation.

The architectural idea is:

```text
DataAnnotations
    ↓
Simple built-in rules

Custom ValidationAttribute
    ↓
Small domain/request-specific rule

IValidatableObject
    ↓
Object-level rules

Dedicated validator/service
    ↓
Larger/complex validation
```

Do not introduce a library just because a simple `[Required]` or custom attribute is sufficient.

---

# 31. Interview Cheat Sheet

### What is model validation?

> Model validation checks whether the bound model satisfies defined validation rules.

### Binding vs validation?

> Binding converts request data into .NET values. Validation checks whether those values satisfy the application's validation rules.

### What is ModelState?

> ModelState stores information about binding and validation errors associated with request data.

### What does `[ApiController]` do?

> It enables API-specific conventions including binding-source inference and automatic 400 responses when ModelState is invalid.

### How do you create custom validation?

> Create a class inheriting from `ValidationAttribute` and override `IsValid`, returning `ValidationResult.Success` for valid input or a `ValidationResult` containing an error for invalid input.

### When use `IValidatableObject`?

> When validation needs to consider the object as a whole, especially relationships between multiple properties.

### Should business validation go into DataAnnotations?

> Generally no. Simple request constraints belong in validation; stateful business decisions belong in application/domain services.

---

# 32. Final Part 9 Mental Model

```text
                  HTTP REQUEST
                       |
                       v
                 MODEL BINDING
                       |
                       v
                   C# DTO
                       |
                       v
              MODEL VALIDATION
                       |
        +--------------+--------------+
        |                             |
     Valid                          Invalid
        |                             |
        v                             v
   ModelState                    ModelState
     valid                         errors
        |                             |
        v                             v
   [ApiController]              [ApiController]
        |                             |
        v                             v
   Controller                   HTTP 400
        |
        v
   Application Service
        |
        v
 Business Validation
        |
        v
 Business Operation
```

The most important boundary is:

```text
Request Validation
        ≠
Business Validation
```

---

# Part 10 — Global Exception Handling

## 1. Why Do We Need Global Exception Handling?

Suppose we have:

```csharp
[HttpPost]
public IActionResult Create(FXOrderRequest request)
{
    try
    {
        _service.Create(request);
        return Ok();
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
public IActionResult Book(FXBookRequest request)
{
    try
    {
        _service.Book(request);
        return Ok();
    }
    catch (Exception ex)
    {
        return StatusCode(500);
    }
}
```

And another:

```csharp
try
{
    ...
}
catch
{
    ...
}
```

This is repetitive and becomes difficult to maintain.

We want:

```text
Any controller/service exception
            ↓
Global exception handler
            ↓
Log exception
            ↓
Convert exception to HTTP response
            ↓
Return standardized error
```

---

# 2. The Core Idea

Instead of:

```text
Controller A
   try/catch

Controller B
   try/catch

Controller C
   try/catch
```

we want:

```text
                 Request
                    |
                    v
          Global Exception Middleware
                    |
                    v
               Controller
                    |
                    v
                 Service
                    |
                    X
                 Exception
                    |
                    |
             propagates upward
                    |
                    v
          Global Exception Middleware
                    |
                    v
             HTTP error response
```

This works because middleware wraps the downstream pipeline.

---

# 3. Middleware Connection

Remember Part 5 and Part 6.

We learned:

```csharp
app.Use(async (context, next) =>
{
    await next();
});
```

This creates a wrapper around everything downstream.

So exception handling middleware can do:

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

This is one of the most important practical uses of middleware.

---

# 4. Request/Response Flow

Normal request:

```text
Request
  ↓
Exception Middleware
  ↓
Controller
  ↓
Service
  ↓
Repository
  ↓
Success
  ↓
Response
```

Exception:

```text
Request
  ↓
Exception Middleware
  ↓
Controller
  ↓
Service
  ↓
Repository
  ↓
Exception
  ↑
Exception Middleware catches
  ↓
Log
  ↓
Create HTTP error response
  ↓
500 / appropriate status
```

---

# 5. Basic Custom Exception Middleware

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
        context.Response.StatusCode = 500;
        context.Response.ContentType = "application/json";

        await context.Response.WriteAsJsonAsync(new
        {
            message = "An unexpected error occurred."
        });
    }
}
```

The central idea is:

```csharp
await _next(context);
```

is inside:

```csharp
try
```

Therefore exceptions thrown downstream can travel back to this middleware.

---

# 6. Register Middleware

In `Program.cs`:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

var app = builder.Build();

app.UseMiddleware<ExceptionHandlingMiddleware>();

app.MapControllers();

app.Run();
```

Now:

```text
Request
  ↓
ExceptionHandlingMiddleware
  ↓
Controller
```

Any unhandled exception downstream can be caught by the middleware.

---

# 7. Why Does This Work?

Think of middleware as nested functions.

Conceptually:

```text
ExceptionMiddleware(
    LoggingMiddleware(
        Authentication(
            Controller
        )
    )
)
```

If the controller throws:

```text
Controller
   |
   X Exception
   |
   v
Authentication
   |
   v
Logging
   |
   v
ExceptionMiddleware
```

The exception propagates back up until something catches it.

The global exception middleware is deliberately positioned to catch it.

---

# 8. Important Middleware Ordering

Suppose:

```csharp
app.UseMiddleware<ExceptionHandlingMiddleware>();

app.UseAuthentication();

app.UseAuthorization();

app.MapControllers();
```

The exception middleware is outside the downstream pipeline.

Conceptually:

```text
Exception Handler
     |
     +-- Authentication
     |
     +-- Authorization
     |
     +-- Controller
```

Therefore it can catch exceptions from downstream middleware/endpoints.

Placement matters.

---

# 9. Never Expose Raw Exceptions

Don't return:

```json
{
    "message": "System.NullReferenceException...",
    "stackTrace": "...",
    "connectionString": "..."
}
```

to clients.

This can leak:

- internal implementation details
- class names
- database information
- file paths
- secrets
- stack traces

Production APIs should return safe error information.

Log the detailed exception internally.

Return a sanitized response externally.

---

# 10. Logging

The middleware should normally use:

```csharp
ILogger<ExceptionHandlingMiddleware>
```

Example:

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

            await HandleExceptionAsync(context);
        }
    }

    private static async Task HandleExceptionAsync(
        HttpContext context)
    {
        context.Response.StatusCode = 500;

        await context.Response.WriteAsJsonAsync(new
        {
            message = "An unexpected error occurred."
        });
    }
}
```

The important rule:

```text
Client:
    safe error

Logs:
    detailed exception
```

---

# 11. Exception Types

Not every exception should necessarily become 500.

For example:

```text
ArgumentException
UnauthorizedAccessException
Domain exception
NotFound exception
Validation exception
Database exception
Unexpected exception
```

A production exception handler can map known exceptions.

Example:

```text
OrderNotFoundException
       ↓
404 Not Found

TradingLimitExceededException
       ↓
400 / 422 depending on API contract

Unauthorized operation
       ↓
401 / 403

Unexpected exception
       ↓
500
```

The exact status code should follow the API contract and semantics.

---

# 12. Exception Handling vs Validation

Don't confuse:

```text
Validation failure
```

with:

```text
Unexpected exception
```

Validation:

```text
Invalid input
     ↓
400
```

Exception:

```text
Unexpected failure
     ↓
Global handler
     ↓
500
```

Business/domain exceptions may be mapped to deliberate client-facing status codes.

---

# 13. Better Production Approach

ASP.NET Core also provides built-in exception handling infrastructure, including:

```csharp
app.UseExceptionHandler(...)
```

and newer exception-handler abstractions such as:

```text
IExceptionHandler
```

These can be preferable to writing everything manually.

The custom middleware is still valuable because it teaches exactly how exception propagation through the pipeline works.

