# .NET Web API — Part 8: Model Binding & Request Data

## Table of Contents

1. [What is Model Binding?](#1-what-is-model-binding)
2. [Where can request data come from?](#2-where-can-request-data-come-from)
3. [Route Binding](#3-route-binding)
4. [Query String Binding](#4-query-string-binding)
5. [Request Body Binding](#5-request-body-binding)
6. [Simple Types vs Complex Types](#6-simple-types-vs-complex-types)
7. [Default Binding Behavior](#7-default-binding-behavior)
8. [FromRoute](#8-fromroute)
9. [FromQuery](#9-fromquery)
10. [FromBody](#10-frombody)
11. [FromHeader](#11-fromheader)
12. [FromForm](#12-fromform)
13. [FromServices](#13-fromservices)
14. [Collections and Nested Objects](#14-collections-and-nested-objects)
15. [Binding Failures and Type Conversion](#15-binding-failures-and-type-conversion)
16. [ModelState](#16-modelstate)
17. [ApiController and Automatic 400 Responses](#17-apicontroller-and-automatic-400-responses)
18. [JSON Deserialization vs Model Binding](#18-json-deserialization-vs-model-binding)
19. [Complete FX Example](#19-complete-fx-example)
20. [End-to-End Request Flow](#20-end-to-end-request-flow)
21. [Common Mistakes](#21-common-mistakes)
22. [Interview Questions](#22-interview-questions)
23. [Quick Revision](#23-quick-revision)

---

# 1. What is Model Binding?

Model binding is the ASP.NET Core mechanism that takes data from an HTTP request and converts it into values that can be passed to a controller action.

For example:

```http
GET /api/users/123?active=true
```

Controller:

```csharp
[HttpGet("{id}")]
public IActionResult GetUser(int id, bool active)
{
    // ...
}
```

The request contains text:

```text
"123"
"true"
```

but the action expects:

```text
int
bool
```

Model binding performs the required extraction and conversion.

Conceptually:

```text
HTTP Request
     |
     +-- Route values
     +-- Query string
     +-- Headers
     +-- Body
     +-- Form data
     |
     v
Model Binding
     |
     v
C# parameters / objects
     |
     v
Controller Action
```

The most important mental model is:

```text
Routing:
    "Which endpoint should handle this request?"

Model Binding:
    "What values should I pass to that endpoint?"
```

---

# 2. Where Can Request Data Come From?

An HTTP request can contain information in multiple places.

Example:

```http
POST /api/fx/rates/EURUSD?source=Reuters
Content-Type: application/json
X-Correlation-ID: abc-123

{
    "bid": 1.1725,
    "ask": 1.1728
}
```

Data exists in:

```text
Route:
    EURUSD

Query string:
    source=Reuters

Header:
    X-Correlation-ID=abc-123

Body:
    {
        "bid": 1.1725,
        "ask": 1.1728
    }
```

ASP.NET Core can bind these to action parameters.

Common binding attributes:

| Attribute | Data source |
|---|---|
| `[FromRoute]` | Route values |
| `[FromQuery]` | Query string |
| `[FromBody]` | HTTP request body |
| `[FromHeader]` | HTTP headers |
| `[FromForm]` | Form fields / multipart form |
| `[FromServices]` | Dependency Injection |

---

# 3. Route Binding

Suppose the controller is:

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetUser(int id)
    {
        return Ok(id);
    }
}
```

Request:

```http
GET /api/users/100
```

Route template:

```text
/api/users/{id}
```

Routing identifies:

```text
id = "100"
```

Model binding converts it:

```text
"100"
  |
  v
100
```

and the action effectively receives:

```csharp
GetUser(100);
```

You can make the source explicit:

```csharp
[HttpGet("{id}")]
public IActionResult GetUser([FromRoute] int id)
{
    return Ok(id);
}
```

## Multiple route parameters

```csharp
[HttpGet("{userId}/orders/{orderId}")]
public IActionResult GetOrder(
    [FromRoute] int userId,
    [FromRoute] int orderId)
{
    return Ok();
}
```

Request:

```text
GET /api/users/10/orders/500
```

Values:

```text
userId = 10
orderId = 500
```

---

# 4. Query String Binding

Query string example:

```http
GET /api/users?page=2&pageSize=20
```

Controller:

```csharp
[HttpGet]
public IActionResult GetUsers(
    [FromQuery] int page,
    [FromQuery] int pageSize)
{
    return Ok();
}
```

Values:

```text
page = 2
pageSize = 20
```

Another example:

```http
GET /api/users?name=Hemant&department=FX
```

```csharp
[HttpGet]
public IActionResult Search(
    [FromQuery] string name,
    [FromQuery] string department)
{
    return Ok();
}
```

Result:

```text
name = "Hemant"
department = "FX"
```

## Optional query parameters

Use nullable/reference types when absence is valid:

```csharp
[HttpGet]
public IActionResult Search(
    [FromQuery] string? name,
    [FromQuery] int? page)
{
    // page can be null
    return Ok();
}
```

Request:

```text
GET /api/users
```

can result in:

```text
name = null
page = null
```

Whereas a non-nullable value type such as:

```csharp
int page
```

requires a value that can be successfully converted.

---

# 5. Request Body Binding

For JSON APIs, request bodies commonly contain complex objects.

Request:

```http
POST /api/users
Content-Type: application/json

{
    "name": "Hemant",
    "age": 26
}
```

DTO:

```csharp
public class CreateUserRequest
{
    public string Name { get; set; } = string.Empty;
    public int Age { get; set; }
}
```

Controller:

```csharp
[HttpPost]
public IActionResult CreateUser(
    [FromBody] CreateUserRequest request)
{
    return Ok(request);
}
```

Conceptually:

```text
JSON
 |
 v
Input Formatter
 |
 v
CreateUserRequest
 |
 +-- Name = "Hemant"
 +-- Age = 26
 |
 v
Controller Action
```

The request body is not normally treated like a query string.

For JSON, ASP.NET Core uses an input formatter to read and deserialize the body.

With the default setup, this commonly involves `System.Text.Json`.

---

# 6. Simple Types vs Complex Types

This distinction is important.

## 6.1 Simple types

Examples:

```csharp
int
long
decimal
double
bool
string
Guid
DateTime
DateTimeOffset
enum
```

Example:

```csharp
public IActionResult GetUser(int id)
```

`id` is a simple value.

Example:

```http
GET /api/users/100
```

Model binding can convert:

```text
"100" -> 100
```

---

## 6.2 Complex types

Examples:

```csharp
public class CreateUserRequest
{
    public string Name { get; set; }
    public int Age { get; set; }
}
```

The object has multiple properties.

JSON:

```json
{
    "name": "Hemant",
    "age": 26
}
```

can become:

```text
CreateUserRequest
    |
    +-- Name = "Hemant"
    +-- Age = 26
```

Complex types are especially common with `[FromBody]`.

---

# 7. Default Binding Behavior

You should understand that ASP.NET Core can infer binding sources in many common cases.

With `[ApiController]`, ASP.NET Core has conventions for determining where parameters come from.

A useful simplified mental model is:

```text
Complex object
    |
    +--> usually request body

Simple parameter
    |
    +--> route/query depending on available route values and conventions
```

For example:

```csharp
[HttpGet("{id}")]
public IActionResult GetUser(int id)
```

The route has `{id}`, so:

```text
id <- route
```

Similarly:

```csharp
[HttpGet]
public IActionResult Search(string name)
```

normally binds:

```text
name <- query string
```

For explicit APIs, attributes make the contract obvious:

```csharp
public IActionResult Search(
    [FromQuery] string name)
```

and:

```csharp
public IActionResult GetUser(
    [FromRoute] int id)
```

## Important note about `[ApiController]`

`[ApiController]` enables API-specific behavior including binding-source inference and automatic handling of invalid model state.

It is therefore common to see:

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
}
```

---

# 8. [FromRoute]

Use `[FromRoute]` when a parameter should come from the route.

Example:

```csharp
[HttpGet("{id}")]
public IActionResult GetUser(
    [FromRoute] int id)
{
    return Ok(id);
}
```

Request:

```text
GET /api/users/123
```

Result:

```text
id = 123
```

## Matching route name

Suppose:

```csharp
[HttpGet("{userId}")]
public IActionResult GetUser(
    [FromRoute] int id)
{
}
```

The route key is `userId`, while the parameter is named `id`.

To explicitly map them:

```csharp
[HttpGet("{userId}")]
public IActionResult GetUser(
    [FromRoute(Name = "userId")] int id)
{
}
```

Now:

```text
route userId -> parameter id
```

---

# 9. [FromQuery]

Use `[FromQuery]` for query string values.

```csharp
[HttpGet]
public IActionResult Search(
    [FromQuery] string name,
    [FromQuery] int page)
{
    return Ok();
}
```

Request:

```text
GET /api/users?name=Hemant&page=2
```

Result:

```text
name = "Hemant"
page = 2
```

## Different parameter name

```csharp
[HttpGet]
public IActionResult Search(
    [FromQuery(Name = "q")] string searchText)
{
    return Ok();
}
```

Request:

```text
GET /api/users?q=Hemant
```

Result:

```text
searchText = "Hemant"
```

## Collections from query

Request:

```text
GET /api/users?ids=10&ids=20&ids=30
```

Action:

```csharp
[HttpGet]
public IActionResult GetUsers(
    [FromQuery] int[] ids)
{
    return Ok(ids);
}
```

Conceptually:

```text
ids = [10, 20, 30]
```

---

# 10. [FromBody]

Use `[FromBody]` for request body data.

DTO:

```csharp
public class CreateUserRequest
{
    public string Name { get; set; } = string.Empty;
    public int Age { get; set; }
}
```

Controller:

```csharp
[HttpPost]
public IActionResult Create(
    [FromBody] CreateUserRequest request)
{
    return Ok(request);
}
```

Request:

```http
POST /api/users
Content-Type: application/json

{
    "name": "Hemant",
    "age": 26
}
```

Result:

```text
request.Name = "Hemant"
request.Age = 26
```

## Content-Type matters

For JSON:

```http
Content-Type: application/json
```

The server needs to know how to interpret the body.

Conceptually:

```text
HTTP Body
   |
   +-- Content-Type: application/json
   |
   v
JSON Input Formatter
   |
   v
C# object
```

## Important: one body

A request generally has one body stream.

Therefore, don't design an action expecting multiple unrelated parameters each independently read from the JSON body.

Prefer one request DTO:

```csharp
public class CreateTradeRequest
{
    public string CurrencyPair { get; set; } = string.Empty;
    public decimal Amount { get; set; }
    public string Side { get; set; } = string.Empty;
}
```

Then:

```csharp
[HttpPost]
public IActionResult CreateTrade(
    [FromBody] CreateTradeRequest request)
{
    return Ok();
}
```

This is much cleaner than trying to represent one JSON body as several unrelated body parameters.

---

# 11. [FromHeader]

Headers are useful for metadata.

Request:

```http
GET /api/users/100
X-Correlation-ID: abc-123
X-Client-Version: 5.2
```

Controller:

```csharp
[HttpGet("{id}")]
public IActionResult GetUser(
    [FromRoute] int id,
    [FromHeader(Name = "X-Correlation-ID")] string correlationId)
{
    return Ok();
}
```

Result:

```text
id = 100
correlationId = "abc-123"
```

Another example:

```csharp
[FromHeader(Name = "X-Client-Version")]
string clientVersion
```

Headers are commonly used for:

- correlation IDs
- client versions
- feature flags
- custom metadata
- content negotiation-related information

Authentication headers are generally processed by the authentication infrastructure rather than manually extracting tokens in every controller.

---

# 12. [FromForm]

Used when data is submitted as form data.

Example:

```csharp
[HttpPost]
public IActionResult Create(
    [FromForm] string name,
    [FromForm] int age)
{
    return Ok();
}
```

For file upload:

```csharp
[HttpPost("upload")]
public IActionResult Upload(
    [FromForm] IFormFile file)
{
    return Ok();
}
```

Client might send:

```text
multipart/form-data
```

Common use cases:

```text
File upload
Profile picture
Form submission
File + metadata
```

Example:

```csharp
public class UploadRequest
{
    public string Description { get; set; } = string.Empty;
    public IFormFile File { get; set; } = default!;
}
```

Controller:

```csharp
[HttpPost("upload")]
public IActionResult Upload(
    [FromForm] UploadRequest request)
{
    return Ok();
}
```

---

# 13. [FromServices]

`[FromServices]` is different.

It does not obtain data from the HTTP request.

It tells ASP.NET Core:

> Resolve this parameter from the dependency injection container.

Suppose:

```csharp
public interface IClock
{
    DateTimeOffset Now { get; }
}
```

Registered:

```csharp
builder.Services.AddSingleton<IClock, SystemClock>();
```

Action:

```csharp
[HttpGet]
public IActionResult GetTime(
    [FromServices] IClock clock)
{
    return Ok(clock.Now);
}
```

Here:

```text
HTTP Request
    |
    X
    |
    No request data
```

Instead:

```text
DI Container
    |
    v
IClock
    |
    v
Action parameter
```

## Constructor injection is normally preferred

Instead of:

```csharp
public IActionResult GetTime(
    [FromServices] IClock clock)
```

usually prefer:

```csharp
private readonly IClock _clock;

public UsersController(IClock clock)
{
    _clock = clock;
}
```

`[FromServices]` is useful for occasional/specific action-level dependencies, but constructor injection is generally the cleaner default.

---

# 14. Collections and Nested Objects

Model binding can handle more than primitive values.

## 14.1 Query collections

Request:

```text
GET /api/users?ids=10&ids=20&ids=30
```

Controller:

```csharp
[HttpGet]
public IActionResult GetUsers(
    [FromQuery] List<int> ids)
{
    return Ok(ids);
}
```

Conceptually:

```text
ids
 |
 +-- 10
 +-- 20
 +-- 30
```

---

## 14.2 Complex JSON object

DTO:

```csharp
public class Address
{
    public string City { get; set; } = string.Empty;
    public string Country { get; set; } = string.Empty;
}

public class CreateUserRequest
{
    public string Name { get; set; } = string.Empty;
    public Address Address { get; set; } = new();
}
```

JSON:

```json
{
    "name": "Hemant",
    "address": {
        "city": "Pune",
        "country": "India"
    }
}
```

Result:

```text
CreateUserRequest
    |
    +-- Name = "Hemant"
    |
    +-- Address
          |
          +-- City = "Pune"
          +-- Country = "India"
```

---

## 14.3 Arrays inside JSON

```csharp
public class CreateOrderRequest
{
    public string Customer { get; set; } = string.Empty;
    public List<OrderItem> Items { get; set; } = [];
}

public class OrderItem
{
    public string Product { get; set; } = string.Empty;
    public int Quantity { get; set; }
}
```

JSON:

```json
{
    "customer": "Hemant",
    "items": [
        {
            "product": "Laptop",
            "quantity": 1
        },
        {
            "product": "Mouse",
            "quantity": 2
        }
    ]
}
```

The resulting object graph contains:

```text
Order
 |
 +-- Customer = Hemant
 |
 +-- Items
      |
      +-- Item 1
      |     Product = Laptop
      |     Quantity = 1
      |
      +-- Item 2
            Product = Mouse
            Quantity = 2
```

---

# 15. Binding Failures and Type Conversion

This is extremely important.

Suppose:

```csharp
[HttpGet]
public IActionResult GetUsers(
    [FromQuery] int page)
{
    return Ok(page);
}
```

Client sends:

```text
GET /api/users?page=abc
```

The value is:

```text
"abc"
```

but ASP.NET Core needs:

```text
int
```

Conversion fails:

```text
"abc"
   X
 int
```

The failure is captured as a model-binding error.

With `[ApiController]`, this can lead to an automatic HTTP 400 response.

---

## Missing value

Request:

```text
GET /api/users
```

Action:

```csharp
public IActionResult GetUsers(int page)
```

There is no `page`.

For optional values, prefer:

```csharp
int? page
```

Then:

```text
page = null
```

---

## Invalid enum example

```csharp
public enum OrderSide
{
    Buy,
    Sell
}
```

Action:

```csharp
public IActionResult Get(
    [FromQuery] OrderSide side)
{
    return Ok();
}
```

Request:

```text
?side=Invalid
```

can cause a binding/conversion error because `"Invalid"` cannot be converted to the enum value.

---

# 16. ModelState

`ModelState` stores information about model binding and validation results.

You can inspect it:

```csharp
if (!ModelState.IsValid)
{
    return BadRequest(ModelState);
}
```

Conceptually:

```text
Request
   |
   v
Model Binding
   |
   +-- Successful conversion
   |
   +-- Binding errors
   |
   v
ModelState
```

Example:

```csharp
[HttpGet]
public IActionResult GetUsers(
    [FromQuery] int page)
{
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState);
    }

    return Ok(page);
}
```

Request:

```text
?page=abc
```

The conversion:

```text
abc -> int
```

fails.

The model-binding infrastructure records the error.

---

# 17. [ApiController] and Automatic 400 Responses

This is one of the most useful features for Web APIs.

Controller:

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
}
```

With `[ApiController]`, invalid model state can automatically produce an HTTP 400 response.

Therefore, you often don't need to write this repeatedly:

```csharp
if (!ModelState.IsValid)
{
    return BadRequest(ModelState);
}
```

ASP.NET Core can handle it automatically.

Conceptually:

```text
HTTP Request
     |
     v
Model Binding
     |
     v
ModelState
     |
     +---- valid ------> Controller Action
     |
     +---- invalid ----> Automatic 400
```

This is one reason `[ApiController]` is important.

---

## Example

DTO:

```csharp
public class CreateUserRequest
{
    public string Name { get; set; } = string.Empty;
    public int Age { get; set; }
}
```

Controller:

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(
        [FromBody] CreateUserRequest request)
    {
        return Ok(request);
    }
}
```

Suppose the client sends a body that cannot be converted into the expected model or contains invalid binding data.

The API-controller infrastructure can reject the request before the action executes.

This becomes even more important once we introduce **validation attributes** such as:

```csharp
[Required]
[Range]
[StringLength]
```

That is the subject of Part 9.

---

# 18. JSON Deserialization vs Model Binding

These terms are often mixed together in interviews.

They are related but not identical.

## JSON deserialization

Converts JSON:

```json
{
    "name": "Hemant",
    "age": 26
}
```

into:

```csharp
CreateUserRequest
```

Conceptually:

```text
JSON
 ↓
System.Text.Json
 ↓
C# Object
```

---

## Model binding

Is the broader ASP.NET Core mechanism that supplies action parameters from request data.

It handles things such as:

```text
Route
Query
Headers
Form
Body
Services
Conversion
ModelState
```

For a body parameter, the body processing uses an input formatter, commonly backed by `System.Text.Json`.

Therefore:

```text
Model Binding
      |
      +-- Route values
      +-- Query string
      +-- Headers
      +-- Form
      +-- Body
              |
              v
        Input Formatter
              |
              v
       JSON Deserialization
```

A strong interview answer:

> Model binding is the ASP.NET Core mechanism that maps request data to action parameters. For JSON request bodies, the body is processed by an input formatter, typically using System.Text.Json, to deserialize the JSON into the target .NET type.

---

# 19. Complete FX Example

Let's combine everything.

Request:

```http
POST /api/fx/rates/EURUSD?source=Reuters
Content-Type: application/json
X-Correlation-ID: abc-123

{
    "bid": 1.1725,
    "ask": 1.1728,
    "timestamp": "2026-10-05T10:00:00Z"
}
```

DTO:

```csharp
public class FXRateRequest
{
    public decimal Bid { get; set; }
    public decimal Ask { get; set; }
    public DateTimeOffset Timestamp { get; set; }
}
```

Controller:

```csharp
[ApiController]
[Route("api/fx/rates")]
public class FXRatesController : ControllerBase
{
    [HttpPost("{currencyPair}")]
    public IActionResult UpdateRate(
        [FromRoute] string currencyPair,
        [FromQuery] string source,
        [FromHeader(Name = "X-Correlation-ID")]
        string correlationId,
        [FromBody] FXRateRequest request)
    {
        return Ok(new
        {
            currencyPair,
            source,
            correlationId,
            request.Bid,
            request.Ask,
            request.Timestamp
        });
    }
}
```

Now map every value:

```text
POST /api/fx/rates/EURUSD?source=Reuters
                     │        │
                     │        └──────────── Query
                     │
                     └───────────────────── Route
```

Headers:

```text
X-Correlation-ID = abc-123
```

Body:

```json
{
    "bid": 1.1725,
    "ask": 1.1728,
    "timestamp": "2026-10-05T10:00:00Z"
}
```

After binding:

```text
currencyPair
    = "EURUSD"

source
    = "Reuters"

correlationId
    = "abc-123"

request
    |
    +-- Bid = 1.1725
    +-- Ask = 1.1728
    +-- Timestamp = 2026-10-05T10:00:00Z
```

Action receives all of these as normal C# values.

---

# 20. End-to-End Request Flow

Let's put everything we've learned so far together.

Client:

```http
POST /api/fx/rates/EURUSD?source=Reuters
Content-Type: application/json
X-Correlation-ID: abc-123

{
    "bid": 1.1725,
    "ask": 1.1728
}
```

## Step 1 — Kestrel

The HTTP request arrives at the ASP.NET Core application.

```text
Client
   |
   v
Kestrel
```

## Step 2 — Middleware

```text
Kestrel
   |
   v
Exception Middleware
   |
   v
Logging Middleware
   |
   v
Authentication
   |
   v
Authorization
```

## Step 3 — Routing

Routing determines:

```text
/api/fx/rates/{currencyPair}
```

matches:

```text
/api/fx/rates/EURUSD
```

Therefore:

```text
currencyPair = "EURUSD"
```

## Step 4 — Endpoint/controller

ASP.NET Core identifies the controller action:

```csharp
UpdateRate(...)
```

## Step 5 — Model Binding

Now it determines the action arguments.

```text
Route
    ↓
currencyPair = "EURUSD"

Query
    ↓
source = "Reuters"

Header
    ↓
correlationId = "abc-123"

Body
    ↓
FXRateRequest
```

## Step 6 — Controller action

Now:

```csharp
UpdateRate(
    "EURUSD",
    "Reuters",
    "abc-123",
    request);
```

## Step 7 — Service

Controller should normally delegate:

```text
Controller
    ↓
FXRateService
```

## Step 8 — Repository / external system

Potentially:

```text
FXRateService
    ↓
Repository
    ↓
Database
```

or:

```text
FXRateService
    ↓
Redis
```

or:

```text
FXRateService
    ↓
Market data / Reuters / Bloomberg
```

## Step 9 — Response

The result travels back through the middleware pipeline.

```text
Controller
   ↓
Middleware unwind
   ↓
Kestrel
   ↓
Client
```

---

# 21. Common Mistakes

## Mistake 1 — Thinking routing and binding are the same

Wrong:

```text
Routing = gets all parameter values
```

Better:

```text
Routing
    → identifies endpoint

Model Binding
    → creates action arguments
```

---

## Mistake 2 — Manually reading everything

You generally don't need:

```csharp
Request.Query["id"]
```

for normal controller parameters.

Prefer:

```csharp
public IActionResult Get([FromQuery] int id)
```

---

## Mistake 3 — Putting business logic into model binding

Model binding should map request data to objects.

Don't use controllers/model binding for business logic such as:

```text
Calculate FX spread
Validate trading limits
Book trade
Call Reuters
Persist order
```

Those belong in appropriate application/service layers.

---

## Mistake 4 — Confusing validation with binding

Suppose:

```text
age = "abc"
```

when expecting:

```csharp
int age
```

That's a **binding/conversion problem**.

Suppose:

```text
age = -5
```

and your business/API rule says:

```text
Age must be >= 0
```

That's generally a **validation** concern.

This distinction is important.

```text
Binding
    ↓
Can I convert/map this request data?

Validation
    ↓
Is the resulting data acceptable?
```

Part 9 will focus heavily on this distinction.

---

## Mistake 5 — Using entities directly as request models

Avoid:

```csharp
[HttpPost]
public IActionResult Create(User entity)
```

when `User` is your database entity.

Prefer:

```csharp
public class CreateUserRequest
{
    public string Name { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
}
```

Then:

```csharp
[HttpPost]
public IActionResult Create(CreateUserRequest request)
```

Why?

```text
API contract
     ≠
Database schema
```

DTOs give you a clean API boundary.

---

# 22. Interview Questions

## Q1. What is model binding?

**Answer:**

> Model binding is the ASP.NET Core mechanism that extracts data from an HTTP request, converts it to the required .NET types, and supplies it to controller action parameters.

---

## Q2. Where can model binding get data from?

Common sources:

```text
Route
Query string
Headers
Request body
Form data
Services
```

---

## Q3. Difference between `[FromRoute]` and `[FromQuery]`?

```text
[FromRoute]
    /users/123
        ↓
    id = 123

[FromQuery]
    /users?id=123
        ↓
    id = 123
```

The values are located in different parts of the HTTP request.

---

## Q4. What does `[FromBody]` do?

It tells ASP.NET Core to obtain the parameter from the HTTP request body.

For JSON APIs, an input formatter processes the body and typically uses `System.Text.Json` for deserialization.

---

## Q5. What is ModelState?

`ModelState` contains information about model binding and validation results, including conversion/binding errors.

---

## Q6. What does `[ApiController]` do?

Among other API behaviors, it enables binding-source inference and automatic HTTP 400 responses when model state is invalid.

---

## Q7. What happens if `page=abc` but the parameter is `int page`?

The string cannot be converted to an integer.

The binding infrastructure records an error in ModelState.

With `[ApiController]`, this can result in an automatic 400 response.

---

## Q8. What is the difference between model binding and JSON deserialization?

JSON deserialization is specifically converting JSON into a .NET object.

Model binding is the broader mechanism that obtains action parameter values from request data and coordinates the binding process.

---

## Q9. What is `[FromServices]`?

It tells ASP.NET Core to resolve that action parameter from the dependency injection container rather than from HTTP request data.

---

## Q10. Why use DTOs?

DTOs:

- define the API contract
- prevent exposing database entities
- reduce over-posting risks
- allow independent API/database evolution
- make validation cleaner
- make requests/responses explicit

---

# 23. Quick Revision

## The six important attributes

```text
[FromRoute]
    ↓
URL route

[FromQuery]
    ↓
?key=value

[FromBody]
    ↓
JSON/XML/etc. request body

[FromHeader]
    ↓
HTTP headers

[FromForm]
    ↓
Form/multipart data

[FromServices]
    ↓
DI container
```

---

## The most important distinction

```text
                    HTTP REQUEST
                         |
       +-----------------+-----------------+
       |                 |                 |
     Route             Query             Body
       |                 |                 |
       v                 v                 v
 [FromRoute]         [FromQuery]       [FromBody]
       |                 |                 |
       +-----------------+-----------------+
                         |
                         v
                  MODEL BINDING
                         |
                         v
                C# Action Parameters
                         |
                         v
                  Controller Action
```

---

## Routing vs Model Binding

```text
ROUTING
"What endpoint?"

             ↓

MODEL BINDING
"What arguments?"

             ↓

CONTROLLER ACTION
"Execute application logic"
```

---

## Binding vs Validation

```text
Request
   |
   v
Model Binding
   |
   | Can I map/convert the request?
   |
   v
C# Model
   |
   v
Validation
   |
   | Is the model acceptable?
   |
   v
Business Logic
```

This distinction becomes extremely important in Part 9.

---

# Final Mental Model

When you see:

```csharp
[HttpPost("{currencyPair}")]
public IActionResult UpdateRate(
    [FromRoute] string currencyPair,
    [FromQuery] string source,
    [FromHeader(Name = "X-Correlation-ID")] string correlationId,
    [FromBody] FXRateRequest request)
{
    // ...
}
```

think:

```text
HTTP Request
      |
      +---- /EURUSD
      |        ↓
      |   [FromRoute]
      |        ↓
      |   currencyPair
      |
      +---- ?source=Reuters
      |        ↓
      |   [FromQuery]
      |        ↓
      |   source
      |
      +---- X-Correlation-ID
      |        ↓
      |   [FromHeader]
      |        ↓
      |   correlationId
      |
      +---- JSON body
               ↓
          [FromBody]
               ↓
        JSON Formatter
               ↓
        FXRateRequest
               |
               v
        Controller Action
```

The complete Web API pipeline now looks like:

```text
Client
  |
  v
HTTP Request
  |
  v
Kestrel
  |
  v
Middleware Pipeline
  |
  v
Routing
  |
  v
Endpoint / Controller
  |
  v
MODEL BINDING
  |
  +--> Route
  +--> Query
  +--> Headers
  +--> Body
  +--> Form
  +--> Services
  |
  v
Action Parameters
  |
  v
Controller
  |
  v
Service
  |
  v
Repository / External Systems
  |
  v
Response
```

**Part 8 is now complete through 8.13.**

The next part is **Part 9 — Model Validation, `ModelState`, validation attributes, custom validation, and how `[ApiController]` handles validation errors.**
