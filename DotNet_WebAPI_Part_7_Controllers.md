# .NET Web API — Part 7
# Controllers in ASP.NET Core Web API

## 1. Where We Are in the Request Flow

So far we have built this mental model:

```text
Client
   ↓
HTTP Request
   ↓
Kestrel
   ↓
Middleware Pipeline
   ↓
Routing
   ↓
Controller
   ↓
Service
   ↓
Repository
   ↓
Database
```

We have already studied:

- Kestrel
- Middleware
- RequestDelegate
- Dependency Injection
- ServiceProvider
- Scopes
- Middleware pipeline

Now we understand:

> What exactly happens when ASP.NET Core reaches a controller?

---

# 2. What Is a Controller?

A controller is a class responsible for handling HTTP requests for a particular part of an API.

Example:

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet]
    public IActionResult GetUsers()
    {
        return Ok();
    }
}
```

Here:

```text
UsersController
      ↓
handles requests related to
      ↓
/api/users
```

For example:

```http
GET /api/users
```

can be mapped to:

```csharp
GetUsers()
```

---

# 3. Controller Is Just a C# Class

At the basic level:

```csharp
public class UsersController
{
}
```

It's just a C# class.

ASP.NET Core gives it special meaning through:

- inheritance
- attributes
- routing
- MVC infrastructure
- endpoint mapping
- dependency injection

For a Web API, we commonly use:

```csharp
ControllerBase
```

---

# 4. `ControllerBase`

Typical Web API controller:

```csharp
public class UsersController : ControllerBase
{
}
```

`ControllerBase` provides useful Web API functionality such as:

```csharp
Ok()
BadRequest()
NotFound()
Unauthorized()
Forbid()
Created()
NoContent()
```

For example:

```csharp
[HttpGet]
public IActionResult GetUser()
{
    return Ok(new
    {
        Id = 1,
        Name = "Hemant"
    });
}
```

The `Ok()` method creates an HTTP 200 response.

---

# 5. `ControllerBase` vs `Controller`

You'll often see:

```csharp
ControllerBase
```

or:

```csharp
Controller
```

For Web APIs, normally:

```csharp
ControllerBase
```

is sufficient.

`Controller` derives from `ControllerBase` and adds MVC/view-related functionality.

Conceptually:

```text
Controller
    ↓
ControllerBase
```

`Controller` is useful when you're also returning Razor/MVC views.

For a pure Web API:

```csharp
ControllerBase
```

is normally preferred.

---

# 6. `[ApiController]`

Example:

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
}
```

`[ApiController]` tells ASP.NET Core that this controller follows Web API conventions.

It enables several useful API behaviors.

One particularly important feature is automatic model validation behavior, which will be studied later.

It also influences API parameter binding behavior and API-specific conventions.

For now:

```text
[ApiController]
       ↓
"This is an API controller"
```

---

# 7. `[Route]`

Consider:

```csharp
[Route("api/[controller]")]
```

The route defines the URL pattern for the controller.

If the controller is:

```csharp
UsersController
```

then:

```text
[controller]
```

is conventionally replaced by:

```text
users
```

Therefore:

```csharp
[Route("api/[controller]")]
```

becomes approximately:

```text
/api/users
```

So:

```http
GET /api/users
```

can reach:

```csharp
UsersController
```

---

# 8. Why Is It Called `UsersController`?

The controller naming convention is important.

Usually:

```csharp
public class UsersController : ControllerBase
```

The suffix:

```text
Controller
```

identifies it as a controller.

So:

```text
UsersController
```

has controller name:

```text
Users
```

which can be used by:

```text
[controller]
```

---

# 9. HTTP Verb Attributes

Suppose:

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet]
    public IActionResult GetUsers()
    {
        return Ok("All users");
    }
}
```

`[HttpGet]` means:

> This action handles HTTP GET requests matching the route.

Similarly:

```csharp
[HttpPost]
```

for POST.

```csharp
[HttpPut]
```

for PUT.

```csharp
[HttpDelete]
```

for DELETE.

```csharp
[HttpPatch]
```

for PATCH.

---

# 10. Example Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet]
    public IActionResult GetUsers()
    {
        return Ok("All users");
    }

    [HttpPost]
    public IActionResult CreateUser()
    {
        return Ok("User created");
    }

    [HttpDelete("{id}")]
    public IActionResult DeleteUser(int id)
    {
        return Ok($"Deleted {id}");
    }
}
```

The routes become approximately:

```text
GET
/api/users

POST
/api/users

DELETE
/api/users/{id}
```

---

# 11. `[HttpDelete("{id}")]`

This is an important routing example.

```csharp
[HttpDelete("{id}")]
public IActionResult DeleteUser(int id)
{
    return Ok();
}
```

Combined with:

```csharp
[Route("api/[controller]")]
```

the route becomes:

```text
/api/users/{id}
```

So:

```http
DELETE /api/users/123
```

can map to:

```csharp
DeleteUser(123)
```

---

# 12. Route Parameter

Consider:

```csharp
[HttpGet("{id}")]
public IActionResult GetUser(int id)
{
    return Ok(id);
}
```

Request:

```http
GET /api/users/100
```

Routing extracts:

```text
id = 100
```

and ASP.NET Core provides it to:

```csharp
int id
```

Conceptually:

```text
URL
 ↓
/api/users/100
 ↓
Route matching
 ↓
id = 100
 ↓
GetUser(100)
```

This is part of model binding, which will be studied in more depth later.

---

# 13. Controller Action

A method inside a controller that handles an HTTP request is commonly called an:

> Action

Example:

```csharp
[HttpGet]
public IActionResult GetUsers()
{
    return Ok();
}
```

Here:

```text
UsersController
      ↓
GetUsers()
```

`GetUsers()` is the action.

---

# 14. Controller vs Action

Keep this distinction clear:

```text
Controller
    =
Class

Action
    =
Method inside controller
```

Example:

```csharp
public class FXController : ControllerBase
{
    [HttpGet]
    public IActionResult GetRates()
    {
        ...
    }
}
```

Here:

```text
FXController
    ↓
Controller

GetRates()
    ↓
Action
```

---

# 15. What Actually Happens When Request Arrives?

Suppose client sends:

```http
GET /api/users
```

The request goes:

```text
Client
   ↓
Kestrel
   ↓
Middleware
   ↓
Routing
   ↓
Find matching endpoint
   ↓
UsersController.GetUsers()
```

But there is a very important question:

> Who creates `UsersController`?

Answer:

# Dependency Injection

This connects directly to Parts 3 and 4.

---

# 16. Controller Creation Through DI

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

The controller needs:

```text
IUserService
```

ASP.NET Core uses DI to construct the controller.

Conceptually:

```text
Routing
   ↓
Need UsersController
   ↓
DI
   ↓
Need IUserService
   ↓
Find UserService
   ↓
Create UserService
   ↓
Inject into UsersController
   ↓
Create UsersController
```

---

# 17. Full Dependency Graph

Suppose:

```csharp
public class UsersController : ControllerBase
{
    public UsersController(IUserService service)
    {
    }
}
```

and:

```csharp
public class UserService : IUserService
{
    public UserService(IUserRepository repository)
    {
    }
}
```

and:

```csharp
public class UserRepository : IUserRepository
{
    public UserRepository(AppDbContext db)
    {
    }
}
```

DI resolves:

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
```

This is the same dependency graph studied in DI.

---

# 18. Controller Activation

The process of creating a controller is often referred to as:

> Controller activation.

Conceptually:

```text
Endpoint selected
      ↓
MVC infrastructure
      ↓
DI resolves controller
      ↓
Constructor dependencies resolved
      ↓
Controller instance created
      ↓
Action invoked
```

You don't normally manually write:

```csharp
new UsersController(...)
```

ASP.NET Core handles this.

---

# 19. Why We Don't Use `new` in Controllers

Bad:

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

This tightly couples the controller to the implementation.

Better:

```csharp
public UsersController(IUserService service)
{
    _service = service;
}
```

Now:

```text
Controller
    ↓
IUserService
```

and DI decides which implementation to use.

---

# 20. Returning `IActionResult`

A common action signature:

```csharp
public IActionResult GetUsers()
```

Then:

```csharp
return Ok(users);
```

or:

```csharp
return NotFound();
```

or:

```csharp
return BadRequest();
```

This gives the action flexibility to return different HTTP responses.

Example:

```csharp
[HttpGet("{id}")]
public IActionResult GetUser(int id)
{
    var user = _service.GetUser(id);

    if (user == null)
        return NotFound();

    return Ok(user);
}
```

Possible responses:

```text
User exists
    ↓
200 OK

User doesn't exist
    ↓
404 Not Found
```

---

# 21. Common Action Results

Some common helpers from `ControllerBase`:

```csharp
Ok()
```

HTTP:

```text
200
```

---

```csharp
Created()
```

HTTP:

```text
201
```

---

```csharp
NoContent()
```

HTTP:

```text
204
```

---

```csharp
BadRequest()
```

HTTP:

```text
400
```

---

```csharp
Unauthorized()
```

HTTP:

```text
401
```

---

```csharp
Forbid()
```

HTTP:

```text
403
```

---

```csharp
NotFound()
```

HTTP:

```text
404
```

---

# 22. Example

```csharp
[HttpGet("{id}")]
public IActionResult GetUser(int id)
{
    var user = _service.GetUser(id);

    if (user == null)
    {
        return NotFound();
    }

    return Ok(user);
}
```

Flow:

```text
GET /api/users/10
       ↓
Route matching
       ↓
GetUser(10)
       ↓
Service
       ↓
User found?
    /      \
  No        Yes
  ↓          ↓
404         200
```

---

# 23. Strongly Typed Action Results

You'll also see:

```csharp
ActionResult<UserDto>
```

Example:

```csharp
[HttpGet("{id}")]
public ActionResult<UserDto> GetUser(int id)
{
    var user = _service.GetUser(id);

    if (user == null)
        return NotFound();

    return Ok(user);
}
```

This communicates that the successful response contains:

```text
UserDto
```

while still allowing:

```text
404
```

or other HTTP results.

This becomes particularly useful in API development.

---

# 24. HTTP Verb + Route = Endpoint

An endpoint isn't just a URL.

Consider:

```csharp
[HttpGet]
public IActionResult GetUsers()
{
    ...
}
```

and:

```csharp
[HttpPost]
public IActionResult CreateUser()
{
    ...
}
```

Both may use:

```text
/api/users
```

but they are different endpoints because the HTTP method differs.

```text
GET /api/users
    ↓
GetUsers()

POST /api/users
    ↓
CreateUser()
```

---

# 25. Same Route, Different HTTP Method

Example:

```csharp
[HttpGet]
public IActionResult GetUsers()
{
    ...
}

[HttpPost]
public IActionResult CreateUser()
{
    ...
}
```

Both:

```text
/api/users
```

but:

```text
GET
 ↓
GetUsers

POST
 ↓
CreateUser
```

The HTTP method participates in endpoint matching.

---

# 26. Route Template

Consider:

```csharp
[Route("api/users")]
```

Then:

```csharp
[HttpGet]
```

means:

```text
GET /api/users
```

And:

```csharp
[HttpGet("{id}")]
```

means:

```text
GET /api/users/{id}
```

So:

```http
GET /api/users/100
```

maps to the second action.

---

# 27. Attribute Routing

ASP.NET Core Web API commonly uses attribute routing.

Example:

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    [HttpGet]
    public IActionResult GetUsers()
    {
        ...
    }

    [HttpGet("{id}")]
    public IActionResult GetUser(int id)
    {
        ...
    }
}
```

Routes:

```text
GET /api/users
GET /api/users/100
```

This is called:

> Attribute routing

because routes are defined using attributes.

---

# 28. Route Tokens

This:

```csharp
[Route("api/[controller]")]
```

uses a route token:

```text
[controller]
```

For:

```csharp
UsersController
```

it becomes:

```text
users
```

For:

```csharp
OrdersController
```

it becomes:

```text
orders
```

Therefore:

```text
UsersController
   ↓
/api/users

OrdersController
   ↓
/api/orders
```

This reduces repetition.

---

# 29. Real FX Controller

Example from an FX-style domain:

```csharp
[ApiController]
[Route("api/[controller]")]
public class FXRatesController : ControllerBase
{
    private readonly IFXRateService _service;

    public FXRatesController(IFXRateService service)
    {
        _service = service;
    }

    [HttpGet("{currencyPair}")]
    public IActionResult GetRate(string currencyPair)
    {
        var rate = _service.GetRate(currencyPair);

        if (rate == null)
        {
            return NotFound();
        }

        return Ok(rate);
    }
}
```

Request:

```http
GET /api/fxrates/EURUSD
```

Flow:

```text
HTTP Request
      ↓
Middleware
      ↓
Routing
      ↓
FXRatesController
      ↓
GetRate("EURUSD")
      ↓
IFXRateService
      ↓
FX rate source
      ↓
Response
```

---

# 30. Controller Should Not Contain Business Logic

A common architecture is:

```text
Controller
    ↓
Service
    ↓
Repository
    ↓
Database
```

Controller should generally focus on HTTP concerns:

```text
Request
 ↓
Receive/bind input
 ↓
Call service
 ↓
Choose HTTP response
```

Business logic should generally live in services/domain components.

Bad:

```csharp
[HttpGet]
public IActionResult GetRate(string pair)
{
    // 200 lines of FX pricing logic
    // database queries
    // spread calculations
    // external connectivity
    // etc.
}
```

Better:

```csharp
[HttpGet("{pair}")]
public IActionResult GetRate(string pair)
{
    var result = _service.GetRate(pair);

    return Ok(result);
}
```

---

# 31. Complete Request Flow

Let's combine everything learned so far.

Client sends:

```http
GET /api/fxrates/EURUSD
```

## Step 1 — Kestrel

Kestrel receives the HTTP request.

```text
Client
 ↓
Kestrel
```

## Step 2 — Middleware

```text
Kestrel
 ↓
Exception Middleware
 ↓
Logging Middleware
 ↓
Authentication
 ↓
Authorization
```

## Step 3 — Routing

ASP.NET Core determines:

```text
/api/fxrates/EURUSD
        ↓
FXRatesController.GetRate("EURUSD")
```

## Step 4 — Controller Activation

DI resolves:

```text
FXRatesController
      ↓
IFXRateService
      ↓
FXRateService
```

## Step 5 — Action

```csharp
GetRate("EURUSD")
```

executes.

## Step 6 — Service

```text
FXRateService
      ↓
business logic
```

## Step 7 — Repository/External Source

Possibly:

```text
Repository
   ↓
SQL
```

or:

```text
FX Service
   ↓
Redis
```

or:

```text
FX Service
   ↓
Reuters/Bloomberg stream
```

## Step 8 — Response

Controller returns:

```csharp
return Ok(rate);
```

Then:

```text
Controller
   ↓
Endpoint
   ↓
Middleware unwinds
   ↓
Kestrel
   ↓
Client
```

---

# 32. Complete Architecture

At this point, the mental model is:

```text
                         CLIENT
                           │
                           ↓
                        KESTREL
                           │
                           ↓
                  MIDDLEWARE PIPELINE
                           │
              ┌────────────┼────────────┐
              ↓            ↓            ↓
          Exception     Logging    Authentication
                           │
                           ↓
                      Authorization
                           │
                           ↓
                         ROUTING
                           │
                           ↓
                     CONTROLLER
                           │
                           ↓
                       SERVICE
                           │
                           ↓
                     REPOSITORY
                           │
                           ↓
                       DATABASE
                           │
                           ↓
                      HTTP RESPONSE
```

---

# 33. What We Have NOT Covered Yet

There is much more to controllers, but these topics should be studied separately.

## Model Binding

```text
FromRoute
FromQuery
FromBody
FromHeader
FromServices
```

## Model Validation

```text
[Required]
[Range]
[StringLength]
Custom validation
```

## DTOs

```text
Request models
Response models
```

## Action Results

```text
IActionResult
ActionResult<T>
IResult
```

## Controller Activation Internals

```text
ObjectFactory
DI
Constructor selection
```

## Filters

```text
Authorization filters
Action filters
Exception filters
Resource filters
```

These will be covered separately rather than mixing everything into this part.

---

# 34. Interview Cheat Sheet

### What is a controller?

A class that handles HTTP requests and exposes actions/endpoints.

### What is an action?

A method inside a controller that handles a request.

### Why `ControllerBase`?

It provides Web API-specific functionality such as HTTP response helpers without MVC view functionality.

### What does `[ApiController]` do?

It marks a controller as an API controller and enables API-specific conventions and behaviors, including automatic validation response behavior.

### What does `[Route]` do?

Defines the route template for the controller/action.

### What does `[HttpGet]` do?

Constrains an action to HTTP GET requests.

### How is a controller created?

ASP.NET Core's MVC infrastructure uses DI to activate the controller and resolve its constructor dependencies.

### Should controllers contain business logic?

Generally no. Controllers should remain thin and delegate business logic to services/domain components.

### How does:

```text
/api/users/100
```

reach:

```csharp
GetUser(100)
```

?

Through endpoint routing and model binding:

```text
URL
 ↓
Route matching
 ↓
Endpoint selection
 ↓
Route value id=100
 ↓
Action invocation
 ↓
GetUser(100)
```

---

# 35. Final Mental Model for Part 7

Remember:

```text
HTTP Request
     ↓
Kestrel
     ↓
Middleware
     ↓
Routing
     ↓
Find Endpoint
     ↓
Controller Activation
     ↓
DI Resolves Dependencies
     ↓
Controller Instance
     ↓
Action Method
     ↓
Service
     ↓
Repository / External System
     ↓
Action Result
     ↓
HTTP Response
```

And:

```text
Controller
    =
Class

Action
    =
Method

[ApiController]
    =
API controller conventions/behavior

[Route]
    =
URL template

[HttpGet]/[HttpPost]/...
    =
HTTP method constraint

ControllerBase
    =
Web API controller base class

DI
    =
Creates controller + dependencies

Service
    =
Business logic

Controller
    =
HTTP/API orchestration
```
