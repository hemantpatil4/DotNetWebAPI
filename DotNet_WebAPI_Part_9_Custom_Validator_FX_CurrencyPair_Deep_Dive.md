# ASP.NET Core Custom Validation Attribute — Complete Deep-Dive Example

## FX Currency Pair Validation

This document explains one complete custom validator example in ASP.NET Core Web API.

We will build a custom validation attribute:

```csharp
[CurrencyPair]
```

that validates that an FX currency pair:

- is present
- contains exactly 6 characters
- contains uppercase letters only
- contains letters only

Examples:

```text
EURUSD   -> Valid
USDINR   -> Valid
GBPJPY   -> Valid

EUR      -> Invalid
eurusd   -> Invalid
EUR/USD  -> Invalid
EUR123   -> Invalid
EURUSDX  -> Invalid
```

The goal is not just to understand the code, but to understand the complete lifecycle:

```text
HTTP Request
     |
     v
Model Binding
     |
     v
C# DTO
     |
     v
Model Validation
     |
     v
Custom Validation Attribute
     |
     v
IsValid()
     |
     +------ Valid ------> Controller Action
     |
     +------ Invalid ----> ModelState
                              |
                              v
                         [ApiController]
                              |
                              v
                         HTTP 400
```

---

# 1. Why Do We Need Custom Validation?

ASP.NET Core already provides built-in validation attributes.

Examples:

```csharp
[Required]
[Range(1, 100)]
[StringLength(50)]
[EmailAddress]
```

But real applications often have domain-specific rules.

For an FX API, suppose the request is:

```json
{
    "currencyPair": "EURUSD",
    "amount": 100000
}
```

We may want:

```text
CurrencyPair:
    - must exist
    - exactly 6 characters
    - uppercase
    - letters only
```

There is no single standard attribute that expresses this exact rule.

So we create:

```csharp
[CurrencyPair]
```

This is called a **custom validation attribute**.

---

# 2. Complete Example

Before breaking it down, here is the complete implementation.

## 2.1 Custom Attribute

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

---

## 2.2 Request DTO

```csharp
public class FXOrderRequest
{
    [CurrencyPair]
    public string CurrencyPair { get; set; } = string.Empty;

    public decimal Amount { get; set; }
}
```

---

## 2.3 Controller

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

---

# 3. Start With the DTO

Our DTO is:

```csharp
public class FXOrderRequest
{
    [CurrencyPair]
    public string CurrencyPair { get; set; } = string.Empty;

    public decimal Amount { get; set; }
}
```

The important line is:

```csharp
[CurrencyPair]
public string CurrencyPair { get; set; }
```

This says:

> Apply the CurrencyPair validation rule to this property.

The attribute itself doesn't run when the class is created.

It is metadata that the validation infrastructure can discover and execute.

Think:

```text
FXOrderRequest
     |
     +-- CurrencyPair
             |
             +-- [CurrencyPair]
                     |
                     v
             CurrencyPairAttribute
```

---

# 4. What Is `ValidationAttribute`?

Our class inherits from:

```csharp
ValidationAttribute
```

So:

```csharp
public class CurrencyPairAttribute : ValidationAttribute
```

means:

> Create a validation attribute using the standard .NET validation infrastructure.

The framework already understands `ValidationAttribute`.

Built-in examples include:

```csharp
[Required]
[Range(1, 100)]
[StringLength(50)]
[EmailAddress]
```

Our custom attribute becomes another participant in the same validation system.

Conceptually:

```text
ValidationAttribute
        |
        +----------------+
        |                |
        v                v
   [Required]       [Range]
                         ...
                         |
                         v
                  [CurrencyPair]
```

---

# 5. Why Override `IsValid()`?

`ValidationAttribute` provides validation behavior.

We override:

```csharp
protected override ValidationResult? IsValid(
    object? value,
    ValidationContext validationContext)
```

This method is where our custom rule is implemented.

Think:

```text
CurrencyPairAttribute
        |
        v
     IsValid()
        |
        v
   Our custom rules
```

The framework calls this method during validation.

We do NOT normally call it manually from the controller.

---

# 6. Understanding the `value` Parameter

The framework passes the value being validated.

Suppose the request contains:

```json
{
    "currencyPair": "EURUSD",
    "amount": 100000
}
```

After model binding:

```csharp
request.CurrencyPair
```

contains:

```text
EURUSD
```

When the validator runs:

```csharp
IsValid(...)
```

the:

```csharp
value
```

parameter represents:

```text
"EURUSD"
```

Therefore:

```csharp
var currencyPair = value as string;
```

gives:

```text
currencyPair = "EURUSD"
```

---

# 7. Why Is `value` an `object?`

The base validation infrastructure is designed to validate many different types.

For example:

```csharp
[Required]
string Name
```

or:

```csharp
[Range(1, 100)]
int Age
```

or:

```csharp
[Custom]
decimal Amount
```

Therefore the validation API cannot assume one specific type.

It uses:

```csharp
object?
```

Our particular validator knows that it expects a string:

```csharp
var currencyPair = value as string;
```

---

# 8. `ValidationContext`

The method also receives:

```csharp
ValidationContext validationContext
```

It contains contextual information about what is being validated.

For example, it can provide information such as:

```text
Object being validated
Property name
Display name
Validation services
```

This becomes especially useful for more advanced validation scenarios.

For our simple property-level validator, we don't need to use it.

But we keep it in the method signature because it is part of the validation API.

---

# 9. Validation Rule 1 — Required

Our first check is:

```csharp
if (string.IsNullOrWhiteSpace(currencyPair))
{
    return new ValidationResult(
        "Currency pair is required.");
}
```

This handles:

```text
null
""
"   "
```

For example:

```json
{
    "currencyPair": ""
}
```

The result is:

```text
Validation failed
```

We return:

```csharp
new ValidationResult(...)
```

That tells the validation system:

> This value is invalid.

---

# 10. What Is `ValidationResult`?

`ValidationResult` represents the result of validation.

There are two important concepts.

## Invalid

```csharp
return new ValidationResult(
    "Currency pair is required.");
```

Meaning:

```text
Validation FAILED
```

## Valid

```csharp
return ValidationResult.Success;
```

Meaning:

```text
Validation PASSED
```

So our validator essentially behaves like:

```text
IsValid(value)
     |
     +---- valid ------> ValidationResult.Success
     |
     +---- invalid ----> ValidationResult(error)
```

---

# 11. Validation Rule 2 — Exactly 6 Characters

Next:

```csharp
if (currencyPair.Length != 6)
{
    return new ValidationResult(
        "Currency pair must contain exactly 6 characters.");
}
```

FX currency pairs normally consist of two three-letter currency codes.

Examples:

```text
EURUSD
USDINR
GBPJPY
```

All have:

```text
6 characters
```

Invalid:

```text
EUR
EURUSDX
```

Flow:

```text
EUR
 |
 +-- Length = 3
 |
 +-- 3 != 6
 |
 v
ValidationResult(error)
```

---

# 12. Validation Rule 3 — Uppercase

We then check:

```csharp
if (!currencyPair.All(char.IsUpper))
{
    return new ValidationResult(
        "Currency pair must contain uppercase letters only.");
}
```

Consider:

```text
EURUSD
```

Each character:

```text
E -> uppercase
U -> uppercase
R -> uppercase
U -> uppercase
S -> uppercase
D -> uppercase
```

Therefore:

```text
All(...) = true
```

and:

```text
!true = false
```

The `if` does not execute.

Now:

```text
eurusd
```

contains lowercase characters.

Therefore:

```text
All(char.IsUpper) = false
```

and:

```text
!false = true
```

The validation fails.

---

# 13. Validation Rule 4 — Letters Only

Finally:

```csharp
if (!currencyPair.All(char.IsLetter))
{
    return new ValidationResult(
        "Currency pair must contain letters only.");
}
```

This prevents:

```text
EUR123
EUR/USD
EUR-US
```

because these contain characters that are not letters.

Examples:

```text
EURUSD
   |
   +-- all letters -> valid

EUR123
   |
   +-- 1,2,3 -> not letters -> invalid
```

---

# 14. Successful Validation

If every rule passes:

```csharp
return ValidationResult.Success;
```

This means:

```text
CurrencyPair
     |
     +-- Exists              YES
     +-- Length == 6         YES
     +-- Uppercase            YES
     +-- Letters only         YES
     |
     v
VALID
```

---

# 15. Complete Validator Flow

Our method can be visualized as:

```text
IsValid(value)
     |
     v
Is value empty?
     |
   YES -----------------> ERROR
     |
    NO
     |
     v
Length == 6?
     |
    NO -----------------> ERROR
     |
   YES
     |
     v
All uppercase?
     |
    NO -----------------> ERROR
     |
   YES
     |
     v
All letters?
     |
    NO -----------------> ERROR
     |
   YES
     |
     v
ValidationResult.Success
```

---

# 16. Controller

Our controller is:

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

Notice:

```csharp
Create(FXOrderRequest request)
```

does NOT manually call:

```csharp
CurrencyPairAttribute
```

The ASP.NET Core validation pipeline does it.

That is one of the most important concepts.

---

# 17. Complete Valid Request

Client sends:

```http
POST /api/fx/orders
Content-Type: application/json
```

```json
{
    "currencyPair": "EURUSD",
    "amount": 100000
}
```

Now follow the request.

## Step 1 — HTTP request

```text
Client
   |
   v
POST /api/fx/orders
```

## Step 2 — Model binding

The JSON is converted into:

```csharp
FXOrderRequest
```

with:

```text
CurrencyPair = "EURUSD"
Amount       = 100000
```

## Step 3 — Validation

ASP.NET Core sees:

```csharp
[CurrencyPair]
```

on:

```csharp
CurrencyPair
```

and runs:

```csharp
CurrencyPairAttribute.IsValid("EURUSD")
```

## Step 4 — Validator

Checks:

```text
Empty?       NO
Length == 6? YES
Uppercase?   YES
Letters?     YES
```

Therefore:

```csharp
ValidationResult.Success
```

## Step 5 — ModelState

Validation succeeds:

```text
ModelState.IsValid = true
```

## Step 6 — Controller

The action executes:

```csharp
Create(request)
```

## Step 7 — Response

```text
HTTP 200 OK
```

Complete flow:

```text
JSON
 ↓
Model Binding
 ↓
FXOrderRequest
 ↓
CurrencyPairAttribute
 ↓
IsValid("EURUSD")
 ↓
Success
 ↓
ModelState.IsValid = true
 ↓
Controller Action
 ↓
200 OK
```

---

# 18. Complete Invalid Request

Now send:

```json
{
    "currencyPair": "eurusd",
    "amount": 100000
}
```

## Step 1 — Binding

Binding succeeds.

The object is:

```text
CurrencyPair = "eurusd"
Amount = 100000
```

Important:

> The JSON is syntactically valid and can be converted into the DTO.

So this is NOT a binding failure.

---

## Step 2 — Validation

ASP.NET Core sees:

```csharp
[CurrencyPair]
```

and executes:

```csharp
IsValid("eurusd")
```

Check:

```text
Empty?       NO
Length == 6? YES
Uppercase?   NO
```

So:

```csharp
return new ValidationResult(
    "Currency pair must contain uppercase letters only.");
```

---

# 19. Validation Result Goes Into ModelState

This is the next critical step.

The validation error becomes part of:

```csharp
ModelState
```

Conceptually:

```text
ModelState
   |
   +-- CurrencyPair
         |
         +-- Error:
              "Currency pair must contain uppercase letters only."
```

Therefore:

```csharp
ModelState.IsValid
```

becomes:

```text
false
```

---

# 20. What Does `[ApiController]` Do?

Because our controller has:

```csharp
[ApiController]
```

ASP.NET Core automatically handles invalid ModelState for API controllers.

Conceptually:

```text
ModelState.IsValid?
       |
       +------ YES ------> Action executes
       |
       +------ NO -------> Automatic 400
```

Therefore:

```text
Controller action
```

may never execute for that invalid request.

The client receives:

```text
HTTP 400 Bad Request
```

---

# 21. Complete Invalid Flow

```text
Client
  |
  | currencyPair = "eurusd"
  v
HTTP Request
  |
  v
Model Binding
  |
  v
FXOrderRequest
  |
  v
Validation
  |
  v
CurrencyPairAttribute
  |
  v
IsValid("eurusd")
  |
  v
Uppercase check FAILS
  |
  v
ValidationResult(error)
  |
  v
ModelState
  |
  v
ModelState.IsValid = false
  |
  v
[ApiController]
  |
  v
HTTP 400 Bad Request
```

---

# 22. Binding vs Validation

This is one of the most important concepts from Part 8 + Part 9.

## Binding problem

Suppose:

```csharp
public int Amount { get; set; }
```

Client sends:

```json
{
    "amount": "abc"
}
```

The framework cannot convert:

```text
"abc"
   |
   X
   |
 int
```

That's a **binding/conversion problem**.

---

## Validation problem

Suppose:

```csharp
[Range(1, 1000000)]
public decimal Amount { get; set; }
```

Client sends:

```json
{
    "amount": -100
}
```

The JSON can be converted:

```text
"-100"
   |
   v
-100 decimal
```

But the value violates the rule.

That's a **validation problem**.

Remember:

```text
Binding:
    "Can I convert/map the request?"

Validation:
    "Is the resulting value acceptable?"
```

---

# 23. Why Not Put This Logic in the Controller?

Bad approach:

```csharp
[HttpPost]
public IActionResult Create(FXOrderRequest request)
{
    if (string.IsNullOrWhiteSpace(request.CurrencyPair))
        return BadRequest();

    if (request.CurrencyPair.Length != 6)
        return BadRequest();

    if (!request.CurrencyPair.All(char.IsUpper))
        return BadRequest();

    // ...
}
```

This creates several problems:

```text
Controller becomes large
Validation duplicated
Harder to reuse
Harder to test
Controller mixes responsibilities
```

Instead:

```csharp
[CurrencyPair]
public string CurrencyPair { get; set; }
```

Now the validation rule is attached to the property and handled by the validation infrastructure.

---

# 24. Why Not Put Business Logic in the Attribute?

Custom validation attributes are good for request-level validation.

For example:

```text
CurrencyPair must be 6 uppercase letters
```

But don't put complex business logic into them.

Bad:

```text
CurrencyPairAttribute
    ↓
Call database
    ↓
Check user's trading limit
    ↓
Check current exposure
    ↓
Call risk service
    ↓
Check market state
```

This is the wrong layer.

Business logic belongs in services/application logic.

For example:

```text
Controller
    ↓
FXOrderService
    ↓
RiskService
    ↓
TradingLimitService
```

The attribute should generally remain focused on validating the request itself.

---

# 25. Property-Level Custom Validator

Our attribute is attached to one property:

```csharp
[CurrencyPair]
public string CurrencyPair { get; set; }
```

Therefore it is a **property-level validator**.

Conceptually:

```text
FXOrderRequest
      |
      +-- CurrencyPair
      |       |
      |       +-- CurrencyPairAttribute
      |
      +-- Amount
```

The validator receives the value of that property.

---

# 26. Cross-Property Validation

Now imagine a rule:

```text
If Side == "Buy",
Amount must be greater than 1,000,000.
```

We need both:

```text
Side
 +
Amount
 ↓
validation
```

A property-level attribute is not ideal because:

```csharp
[SomeValidator]
public decimal Amount
```

naturally receives the Amount value, not the whole business object.

For object-level validation, approaches such as:

```csharp
IValidatableObject
```

can be used.

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

This is a different validation pattern.

---

# 27. `ValidationContext` Becomes More Useful Here

In the simple `CurrencyPairAttribute`, we didn't use:

```csharp
ValidationContext validationContext
```

But it exists because validation can require context.

It can provide information about:

```text
The object being validated
Property name
Display name
Validation services
```

This becomes useful for advanced validation scenarios.

However, avoid turning a validation attribute into a mini service layer.

---

# 28. Unit Testing the Custom Validator

One major advantage of a custom validator is that the rules are easy to test.

For example, conceptually:

```csharp
var attribute = new CurrencyPairAttribute();
```

Then test values such as:

```text
EURUSD
USDINR
GBPJPY
```

should succeed.

And:

```text
EUR
eurusd
EUR/USD
EUR123
EURUSDX
```

should fail.

A useful test matrix:

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
| `null` | Invalid |

---

# 29. One Important Improvement

The current validator checks:

```csharp
currencyPair.All(char.IsUpper)
```

and:

```csharp
currencyPair.All(char.IsLetter)
```

This is understandable for learning.

In production, you might prefer a stricter rule for ISO-style currency codes rather than simply "six uppercase Unicode letters".

For example, the actual domain rule might be:

```text
First currency = valid ISO 4217 code
Second currency = valid ISO 4217 code
```

Then you could validate against an allowed set:

```csharp
HashSet<string>
```

or a dedicated currency metadata service.

That would be a **domain-level validation** decision rather than merely checking the string format.

---

# 30. Attribute Lifecycle Mental Model

Don't think:

```text
new CurrencyPairAttribute()
```

inside the controller.

Think:

```text
DTO Metadata
     |
     v
[CurrencyPair]
     |
     v
ASP.NET Core Validation Infrastructure
     |
     v
CurrencyPairAttribute
     |
     v
IsValid(value, context)
     |
     v
ValidationResult
```

The framework orchestrates the validation.

---

# 31. The Most Important Four Objects

Understand these four concepts:

### 1. DTO

```csharp
FXOrderRequest
```

Represents the incoming request model.

### 2. Validation Attribute

```csharp
CurrencyPairAttribute
```

Contains the validation rule.

### 3. ValidationResult

```csharp
ValidationResult.Success
```

or:

```csharp
new ValidationResult("...")
```

Represents the outcome.

### 4. ModelState

Contains validation/binding errors associated with the request.

Flow:

```text
DTO
 ↓
Attribute
 ↓
IsValid()
 ↓
ValidationResult
 ↓
ModelState
 ↓
[ApiController]
 ↓
400 or Controller
```

---

# 32. Full Production-Style Example

A clean project structure could look like:

```text
FXWebAPI
│
├── Controllers
│   └── FXOrdersController.cs
│
├── DTOs
│   └── FXOrderRequest.cs
│
├── Validation
│   └── CurrencyPairAttribute.cs
│
├── Services
│   └── FXOrderService.cs
│
└── Program.cs
```

### CurrencyPairAttribute.cs

```csharp
using System.ComponentModel.DataAnnotations;

namespace FXWebAPI.Validation;

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

### FXOrderRequest.cs

```csharp
using FXWebAPI.Validation;

namespace FXWebAPI.DTOs;

public class FXOrderRequest
{
    [CurrencyPair]
    public string CurrencyPair { get; set; } = string.Empty;

    public decimal Amount { get; set; }
}
```

### FXOrdersController.cs

```csharp
using FXWebAPI.DTOs;
using Microsoft.AspNetCore.Mvc;

namespace FXWebAPI.Controllers;

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

---

# 33. Final Mental Model

This is the diagram you should remember for interviews:

```text
                     CLIENT
                       |
                       v
               HTTP JSON Request
                       |
                       v
                MODEL BINDING
                       |
                       v
                FXOrderRequest
                       |
                       v
                MODEL VALIDATION
                       |
                       v
              [CurrencyPair]
                       |
                       v
          CurrencyPairAttribute
                       |
                       v
                   IsValid()
                       |
             +---------+---------+
             |                   |
           VALID              INVALID
             |                   |
             v                   v
      ValidationResult       ValidationResult
          .Success               Error
             |                   |
             v                   v
       ModelState            ModelState
          Valid                Invalid
             |                   |
             v                   v
      Controller Action    [ApiController]
             |                   |
             v                   v
          Service            HTTP 400
             |
             v
       Business Logic
```

The key separation is:

```text
MODEL BINDING
    ↓
Request data → C# object

CUSTOM VALIDATION
    ↓
C# object → Is this request structurally valid?

BUSINESS VALIDATION
    ↓
Is this operation actually allowed?

BUSINESS LOGIC
    ↓
Perform the operation
```

That separation is extremely important in real-world ASP.NET Core APIs.
