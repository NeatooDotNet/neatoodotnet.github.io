---
title: "Rules Engine Reference"
layout: single
permalink: /reference/rules/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

The Neatoo rules engine provides trigger-based business rule execution for validation and data transformation. Rules are classes that declare dependencies on properties and execute automatically when those properties change.

## Rule Base Classes

Neatoo provides two abstract base classes for creating rules:

### RuleBase

For synchronous rules that do not require async operations:

```csharp
public class AgeValidationRule : RuleBase<Person>
{
    public AgeValidationRule() : base(p => p.Age) { }

    protected override IRuleMessages Execute(Person target)
    {
        if (target.Age < 0)
            return (nameof(target.Age), "Age cannot be negative").AsRuleMessages();

        if (target.Age > 150)
            return (nameof(target.Age), "Age value seems unrealistic").AsRuleMessages();

        return None;
    }
}
```

Key characteristics:

- Inherits from `RuleBase<T>` where `T` is the entity interface type
- Override `Execute(T target)` to implement logic
- Return `None` when validation passes
- Return `IRuleMessages` when validation fails

### AsyncRuleBase

For rules requiring database access, API calls, or other async operations:

```csharp
public class UniqueEmailRule : AsyncRuleBase<Person>
{
    private readonly IEmailService _emailService;

    public UniqueEmailRule(IEmailService emailService)
        : base(p => p.Email)
    {
        _emailService = emailService;
    }

    protected override async Task<IRuleMessages> Execute(
        Person target, CancellationToken? token = null)
    {
        if (string.IsNullOrEmpty(target.Email))
            return None;

        var exists = await _emailService.EmailExistsAsync(
            target.Email, token ?? CancellationToken.None);

        return exists
            ? (nameof(target.Email), "Email already in use").AsRuleMessages()
            : None;
    }
}
```

Key characteristics:

- Inherits from `AsyncRuleBase<T>`
- Override `Execute(T target, CancellationToken? token)` for async logic
- Sets `IsBusy = true` on the entity while executing
- Supports cancellation through the token parameter

## Defining Trigger Properties

Trigger properties determine when a rule executes. Specify them in the constructor.

### Using Lambda Expressions (Recommended)

```csharp
public class FullNameRule : RuleBase<Person>
{
    // Runs when FirstName OR LastName changes
    public FullNameRule()
        : base(p => p.FirstName, p => p.LastName) { }

    protected override IRuleMessages Execute(Person target)
    {
        target.FullName = $"{target.FirstName} {target.LastName}";
        return None;
    }
}
```

### Using Property Names

```csharp
public class FullNameRule : RuleBase<Person>
{
    public FullNameRule()
        : base(nameof(Person.FirstName), nameof(Person.LastName)) { }

    protected override IRuleMessages Execute(Person target)
    {
        target.FullName = $"{target.FirstName} {target.LastName}";
        return None;
    }
}
```

### Adding Triggers After Construction

```csharp
public class DynamicRule : RuleBase<Order>
{
    public DynamicRule()
    {
        AddTriggerProperties(o => o.Subtotal);
        AddTriggerProperties(o => o.TaxRate);
        AddTriggerProperties(o => o.DiscountPercent);
    }

    protected override IRuleMessages Execute(Order target)
    {
        target.Total = target.Subtotal * (1 + target.TaxRate) *
                       (1 - target.DiscountPercent);
        return None;
    }
}
```

## The Execute() Method

The `Execute()` method is where rule logic lives.

### Synchronous Execute

```csharp
protected override IRuleMessages Execute(Person target)
{
    // Access entity properties
    var firstName = target.FirstName;
    var lastName = target.LastName;

    // Perform validation
    if (string.IsNullOrEmpty(firstName) && string.IsNullOrEmpty(lastName))
        return (nameof(target.FirstName), "At least one name is required")
            .AsRuleMessages();

    // Transform data
    target.FullName = $"{firstName} {lastName}".Trim();

    // Return None when validation passes
    return None;
}
```

### Asynchronous Execute

```csharp
protected override async Task<IRuleMessages> Execute(
    Person target, CancellationToken? token = null)
{
    var ct = token ?? CancellationToken.None;

    // Perform async operations
    var isAvailable = await _userService.CheckUsernameAsync(
        target.Username, ct);

    if (!isAvailable)
        return (nameof(target.Username), "Username is taken").AsRuleMessages();

    // Async data transformation
    target.UserDetails = await _userService.GetDetailsAsync(target.Id, ct);

    return None;
}
```

### Accessing Parent Entities

Child entities can access their parent:

```csharp
protected override IRuleMessages Execute(OrderLine target)
{
    var order = target.Parent as IOrder;
    if (order == null)
        return (nameof(target.ProductId), "Line must belong to an order")
            .AsRuleMessages();

    // Use parent data in validation
    if (target.ShipDate < order.OrderDate)
        return (nameof(target.ShipDate), "Ship date cannot be before order date")
            .AsRuleMessages();

    return None;
}
```

## Returning Rule Messages

### No Errors (Validation Passed)

```csharp
return None;
// or
return RuleMessages.None;
```

### Single Error Message

```csharp
// Tuple syntax
return (nameof(target.Email), "Invalid email format").AsRuleMessages();

// Explicit construction
return new RuleMessage(nameof(target.Email), "Invalid email format")
    .AsRuleMessages();
```

### Multiple Error Messages

```csharp
// Array syntax
return new[]
{
    (nameof(target.FirstName), "First name is required"),
    (nameof(target.LastName), "Last name is required")
}.AsRuleMessages();

// Collection builder
var messages = new RuleMessages();
messages.Add(nameof(target.FirstName), "First name is required");
messages.Add(nameof(target.LastName), "Last name is required");
return messages;
```

### Conditional (Fluent) Syntax

```csharp
// Single condition
return RuleMessages.If(
    string.IsNullOrEmpty(target.Name),
    nameof(target.Name),
    "Name is required");

// Chained conditions
return RuleMessages
    .If(string.IsNullOrEmpty(target.Email),
        nameof(target.Email), "Email is required")
    .ElseIf(() => !target.Email.Contains("@"),
        nameof(target.Email), "Email format is invalid")
    .ElseIf(() => target.Email.Length > 100,
        nameof(target.Email), "Email is too long");
```

### Clear Previous Messages for Property

When a property becomes valid, previous messages are automatically cleared. To explicitly clear:

```csharp
return new RuleMessage(nameof(target.Email)).AsRuleMessages();
// Clears any existing message for Email, adds no new message
```

## Cascading Rules

Rules can trigger other rules by modifying properties:

```csharp
// Rule 1: Calculate line total
public class LineTotalRule : RuleBase<OrderLine>
{
    public LineTotalRule() : base(l => l.Quantity, l => l.UnitPrice) { }

    protected override IRuleMessages Execute(OrderLine target)
    {
        target.Total = target.Quantity * target.UnitPrice;
        return None;
    }
}

// Rule 2: Calculate order subtotal (triggers when Lines collection changes)
public class OrderSubtotalRule : RuleBase<Order>
{
    public OrderSubtotalRule() : base(o => o.Lines) { }

    protected override IRuleMessages Execute(Order target)
    {
        target.Subtotal = target.Lines?.Sum(l => l.Total) ?? 0;
        // Setting Subtotal triggers OrderTotalRule
        return None;
    }
}

// Rule 3: Calculate order total
public class OrderTotalRule : RuleBase<Order>
{
    public OrderTotalRule() : base(o => o.Subtotal, o => o.TaxRate) { }

    protected override IRuleMessages Execute(Order target)
    {
        target.Tax = target.Subtotal * target.TaxRate;
        target.Total = target.Subtotal + target.Tax;
        return None;
    }
}
```

When `orderLine.UnitPrice` changes:

1. `LineTotalRule` runs, sets `orderLine.Total`
2. `OrderSubtotalRule` runs (Lines collection changed), sets `order.Subtotal`
3. `OrderTotalRule` runs (Subtotal changed), sets `order.Tax` and `order.Total`

### Preventing Infinite Loops

The framework prevents a rule from triggering itself. If Rule A sets Property X, and Rule A triggers on Property X, it will not re-run from its own change.

## Dependency Injection

Rules support constructor injection:

```csharp
public class UniqueNameRule : AsyncRuleBase<Person>
{
    private readonly IPersonRepository _repository;
    private readonly ILogger<UniqueNameRule> _logger;

    public UniqueNameRule(
        IPersonRepository repository,
        ILogger<UniqueNameRule> logger)
        : base(p => p.FirstName, p => p.LastName)
    {
        _repository = repository;
        _logger = logger;
    }

    protected override async Task<IRuleMessages> Execute(
        Person target, CancellationToken? token = null)
    {
        _logger.LogDebug("Checking name uniqueness for {Name}",
            $"{target.FirstName} {target.LastName}");

        var exists = await _repository.NameExistsAsync(
            target.FirstName, target.LastName, target.Id);

        return exists
            ? (nameof(target.FirstName), "This name already exists").AsRuleMessages()
            : None;
    }
}
```

### Registration

Register rules in your DI container:

```csharp
// Individual registration
builder.Services.AddScoped<IUniqueNameRule, UniqueNameRule>();

// Convention-based registration
builder.Services.RegisterMatchingName(
    typeof(IUniqueNameRule).Assembly,
    ServiceLifetime.Scoped);
```

### Creating Rule Interfaces

Define interfaces for rules to support DI:

```csharp
public interface IUniqueNameRule : IRule<IPerson> { }

public class UniqueNameRule : AsyncRuleBase<Person>, IUniqueNameRule
{
    // ...
}
```

## RuleManager Methods

The `RuleManager` is accessed through `ValidateBase<T>` and provides rule management.

### AddRule

Add a rule instance:

```csharp
public Person(IEntityBaseServices<Person> services,
              IUniqueNameRule uniqueNameRule,
              IAgeValidationRule ageRule) : base(services)
{
    RuleManager.AddRule(uniqueNameRule);
    RuleManager.AddRule(ageRule);
}
```

### AddValidation (Inline Sync Validation)

Add simple validation without a separate class:

```csharp
// Single trigger property
RuleManager.AddValidation(
    nameof(Email),
    (Person p) => string.IsNullOrEmpty(p.Email) || p.Email.Contains("@")
        ? RuleMessage.None
        : RuleMessage.Error("Invalid email format"));

// Multiple trigger properties
RuleManager.AddValidation(
    (Person p) =>
    {
        if (p.StartDate > p.EndDate)
            return (nameof(p.StartDate), "Start must be before end").AsRuleMessages();
        return RuleMessages.None;
    },
    p => p.StartDate, p => p.EndDate);
```

### AddValidationAsync (Inline Async Validation)

```csharp
RuleManager.AddValidationAsync(
    async (Person p, CancellationToken ct) =>
    {
        var exists = await emailService.ExistsAsync(p.Email, ct);
        return exists
            ? (nameof(p.Email), "Email in use").AsRuleMessages()
            : RuleMessages.None;
    },
    p => p.Email);
```

### AddAction (Inline Sync Transformation)

For rules that transform data without validation:

```csharp
// Calculate full name
RuleManager.AddAction(
    (Person p) => p.FullName = $"{p.FirstName} {p.LastName}",
    p => p.FirstName, p => p.LastName);

// Normalize data
RuleManager.AddAction(
    (Person p) => p.Email = p.Email?.ToLowerInvariant(),
    p => p.Email);
```

### AddActionAsync (Inline Async Transformation)

```csharp
RuleManager.AddActionAsync(
    async (Order o, CancellationToken ct) =>
    {
        o.TaxRate = await taxService.GetRateAsync(o.ShipToZip, ct);
    },
    o => o.ShipToZip);
```

## Data Annotation Attributes

Neatoo integrates with standard .NET `System.ComponentModel.DataAnnotations` attributes. These are automatically converted to synchronous validation rules that execute when the decorated property changes.

### Supported Attributes

The following DataAnnotations are fully supported:

```csharp
// Required - value must not be null or empty
[Required(ErrorMessage = "First name is required")]
public partial string? FirstName { get; set; }

// StringLength - string length constraints
[Required]
[StringLength(100, MinimumLength = 2, ErrorMessage = "Name must be 2-100 characters")]
public partial string? LastName { get; set; }

// EmailAddress - email format validation
[EmailAddress(ErrorMessage = "Invalid email format")]
public partial string? Email { get; set; }

// Range - numeric range validation
[Range(0, 150, ErrorMessage = "Age must be between 0 and 150")]
public partial int Age { get; set; }

// RegularExpression - pattern matching
[RegularExpression(@"^\d{5}(-\d{4})?$", ErrorMessage = "Invalid ZIP code")]
public partial string? ZipCode { get; set; }

// MinLength/MaxLength - collection and string size
[MinLength(1, ErrorMessage = "At least one item required")]
public partial IList<string>? Tags { get; set; }

// Compare - cross-property comparison
[Compare(nameof(Password), ErrorMessage = "Passwords must match")]
public partial string? ConfirmPassword { get; set; }
```

### Complete Attribute Reference

| Attribute | Purpose | Applies To |
|-----------|---------|------------|
| `[Required]` | Value must not be null or empty | Any type |
| `[StringLength]` | Min/max string length | `string` |
| `[MinLength]` | Minimum length | `string`, collections |
| `[MaxLength]` | Maximum length | `string`, collections |
| `[Range]` | Numeric range (min, max) | Numeric types, `DateTime` |
| `[EmailAddress]` | Email format validation | `string` |
| `[RegularExpression]` | Regex pattern matching | `string` |
| `[Compare]` | Equality with another property | Any type |

### How Attribute Validation Works

When you apply a validation attribute to a partial property:

1. **Source Generator Detects** - The Neatoo source generator finds attributes on partial properties
2. **Rule Created** - A sync validation rule is generated for each attribute
3. **Trigger Set** - The rule triggers when the decorated property changes
4. **Validation Runs** - On property change, the attribute validation executes
5. **Message Set** - If validation fails, the error message is added to `PropertyMessages`

```csharp
// What you write
[Required(ErrorMessage = "Name is required")]
[StringLength(50, ErrorMessage = "Name too long")]
public partial string? Name { get; set; }

// What Neatoo effectively generates (conceptually)
// Rule 1: Required check
// Rule 2: StringLength check
// Both trigger on Name property changes
```

### Custom Error Messages

All attributes support custom error messages via the `ErrorMessage` property:

```csharp
[Required(ErrorMessage = "Please enter your email address")]
[EmailAddress(ErrorMessage = "That doesn't look like a valid email")]
public partial string? Email { get; set; }
```

If you omit `ErrorMessage`, a default message is used.

### Combining Attributes with Custom Rules

Attributes work alongside custom rules. Use attributes for simple format validation and custom rules for complex business logic:

```csharp
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    public Person(
        IEntityBaseServices<Person> services,
        IUniqueEmailRule uniqueEmailRule) : base(services)
    {
        // Custom async rule for database validation
        RuleManager.AddRule(uniqueEmailRule);
    }

    // Attribute handles format, custom rule handles uniqueness
    [Required(ErrorMessage = "Email is required")]
    [EmailAddress(ErrorMessage = "Invalid email format")]
    public partial string? Email { get; set; }
}
```

Execution order:
1. Attribute-based rules run first (sync)
2. Custom sync rules run next
3. Custom async rules run last

### DisplayName Attribute

The `[DisplayName]` attribute provides a user-friendly name for UI binding:

```csharp
[DisplayName("First Name*")]
[Required(ErrorMessage = "First name is required")]
public partial string? FirstName { get; set; }
```

This is used by UI components like MudNeatoo for labels:

```razor
@* Uses DisplayName for the label *@
<MudNeatooTextField For="() => person.FirstName" />
```

### Limitations of Attribute Validation

Attribute validation has inherent limitations:

| Limitation | Solution |
|------------|----------|
| No async operations | Use `AsyncRuleBase<T>` for database checks |
| No cross-property complex logic | Use `RuleBase<T>` with multiple triggers |
| No contextual/conditional logic | Use custom rules with DI services |
| Single error message per attribute | Return multiple messages from custom rules |

### Example: Comprehensive Property Validation

Combining attributes with inline and custom rules:

```csharp
[Factory]
internal partial class Registration : EntityBase<Registration>, IRegistration
{
    public Registration(
        IEntityBaseServices<Registration> services,
        IUsernameAvailableRule usernameRule) : base(services)
    {
        // Async rule for database check
        RuleManager.AddRule(usernameRule);

        // Inline rule for password strength
        RuleManager.AddValidation(
            nameof(Password),
            (Registration r) =>
            {
                if (string.IsNullOrEmpty(r.Password)) return RuleMessage.None;
                if (!r.Password.Any(char.IsUpper))
                    return RuleMessage.Error("Password needs an uppercase letter");
                if (!r.Password.Any(char.IsDigit))
                    return RuleMessage.Error("Password needs a number");
                return RuleMessage.None;
            });
    }

    // Attribute: format and length
    [Required(ErrorMessage = "Username is required")]
    [StringLength(30, MinimumLength = 3, ErrorMessage = "Username must be 3-30 characters")]
    [RegularExpression(@"^[a-zA-Z0-9_]+$", ErrorMessage = "Letters, numbers, and underscores only")]
    public partial string? Username { get; set; }

    // Attribute: basic requirement
    [Required(ErrorMessage = "Password is required")]
    [MinLength(8, ErrorMessage = "Password must be at least 8 characters")]
    public partial string? Password { get; set; }

    // Attribute: cross-property comparison
    [Required(ErrorMessage = "Please confirm your password")]
    [Compare(nameof(Password), ErrorMessage = "Passwords do not match")]
    public partial string? ConfirmPassword { get; set; }

    // Attribute: format
    [Required(ErrorMessage = "Email is required")]
    [EmailAddress(ErrorMessage = "Invalid email format")]
    public partial string? Email { get; set; }
}
```

## Running Rules Manually

### RunRules for Specific Property

```csharp
await person.RunRules(nameof(person.Email));
```

### RunRules with Flags

```csharp
// Run all rules
await person.RunRules(RunRulesFlag.All);

// Run only sync rules
await person.RunRules(RunRulesFlag.SyncOnly);

// Run rules for all properties
await person.RunRules(RunRulesFlag.CheckAllProperties);
```

### When to Run Rules Manually

Typically rules run automatically. Manual execution is useful for:

- Initial validation when an entity loads
- Re-validation after external data changes
- Testing specific rule behavior

```csharp
// Before save, ensure all rules have run
[Insert]
public async Task Insert([Service] IDbContext db)
{
    await RunRules(RunRulesFlag.All);

    if (!IsValid)
        throw new ValidationException("Entity has validation errors");

    // Proceed with insert...
}
```

## The IsBusy State

When async rules execute, the entity's `IsBusy` property becomes `true`:

```csharp
// Entity level
bool isBusy = person.IsBusy;      // Any async operation in progress
bool selfBusy = person.IsSelfBusy; // This entity specifically

// Property level
bool emailBusy = person[nameof(person.Email)].IsBusy;
```

### UI Integration

```razor
<input @bind="person.Email"
       disabled="@person[nameof(person.Email)].IsBusy" />

@if (person[nameof(person.Email)].IsBusy)
{
    <span class="spinner">Checking...</span>
}

<button @onclick="Save"
        disabled="@(person.IsBusy || !person.IsSavable)">
    @(person.IsBusy ? "Validating..." : "Save")
</button>
```

### WaitForTasks

Wait for all async operations to complete:

```csharp
person.Email = "test@example.com";  // Triggers async validation
await person.WaitForTasks();        // Wait for completion
if (person.IsValid)
{
    await person.Save();
}
```

## Unit Testing Rules

Rules are regular classes that you can unit test:

### Testing Synchronous Rules

```csharp
[Fact]
public void AgeValidationRule_WhenNegative_ReturnsError()
{
    // Arrange
    var rule = new AgeValidationRule();
    var person = new MockPerson { Age = -5 };

    // Act
    var messages = rule.Execute(person);

    // Assert
    Assert.Contains(messages,
        m => m.PropertyName == nameof(IPerson.Age));
}

[Fact]
public void AgeValidationRule_WhenValid_ReturnsNone()
{
    // Arrange
    var rule = new AgeValidationRule();
    var person = new MockPerson { Age = 30 };

    // Act
    var messages = rule.Execute(person);

    // Assert
    Assert.Empty(messages);
}
```

### Testing Asynchronous Rules

```csharp
[Fact]
public async Task UniqueEmailRule_WhenExists_ReturnsError()
{
    // Arrange
    var mockService = new Mock<IEmailService>();
    mockService.Setup(s => s.EmailExistsAsync("taken@example.com", It.IsAny<CancellationToken>()))
        .ReturnsAsync(true);

    var rule = new UniqueEmailRule(mockService.Object);
    var person = new MockPerson { Email = "taken@example.com" };

    // Act
    var messages = await rule.Execute(person);

    // Assert
    Assert.Contains(messages,
        m => m.Message.Contains("already in use"));
}

[Fact]
public async Task UniqueEmailRule_WhenAvailable_ReturnsNone()
{
    // Arrange
    var mockService = new Mock<IEmailService>();
    mockService.Setup(s => s.EmailExistsAsync("new@example.com", It.IsAny<CancellationToken>()))
        .ReturnsAsync(false);

    var rule = new UniqueEmailRule(mockService.Object);
    var person = new MockPerson { Email = "new@example.com" };

    // Act
    var messages = await rule.Execute(person);

    // Assert
    Assert.Empty(messages);
}
```

### Creating Test Doubles

For testing, create minimal implementations:

```csharp
public class MockPerson : IPerson
{
    public Guid? Id { get; set; }
    public string? FirstName { get; set; }
    public string? LastName { get; set; }
    public string? Email { get; set; }
    public int Age { get; set; }

    // Implement required interface members...
}
```

## Complete Rule Example

Here is a complete rule implementation from the Neatoo example project:

```csharp
public interface IUniquePhoneNumberRule : IRule<IPersonPhone> { }

public class UniquePhoneNumberRule : RuleBase<PersonPhone>, IUniquePhoneNumberRule
{
    public UniquePhoneNumberRule()
        : base(p => p.PhoneNumber, p => p.PhoneType) { }

    protected override IRuleMessages Execute(PersonPhone target)
    {
        // Access parent through the aggregate
        var parent = target.Parent as IPerson;
        if (parent?.PersonPhoneList == null)
            return None;

        // Check for duplicates among siblings
        var isDuplicate = parent.PersonPhoneList
            .Where(p => p != target)
            .Any(p => p.PhoneNumber == target.PhoneNumber);

        return RuleMessages
            .If(isDuplicate,
                nameof(target.PhoneNumber),
                "Phone number must be unique");
    }
}
```

Usage in entity:

```csharp
[Factory]
internal partial class PersonPhone : EntityBase<PersonPhone>, IPersonPhone
{
    public PersonPhone(
        IEntityBaseServices<PersonPhone> services,
        IUniquePhoneNumberRule uniquePhoneRule,
        IUniquePhoneTypeRule uniqueTypeRule) : base(services)
    {
        RuleManager.AddRule(uniquePhoneRule);
        RuleManager.AddRule(uniqueTypeRule);
    }

    public partial Guid? Id { get; set; }

    [Required]
    public partial PhoneType PhoneType { get; set; }

    [Required]
    public partial string? PhoneNumber { get; set; }

    public IPerson? ParentPerson => Parent as IPerson;
}
```

## Common Patterns

### Cross-Property Validation

```csharp
public class DateRangeRule : RuleBase<Event>
{
    public DateRangeRule() : base(e => e.StartDate, e => e.EndDate) { }

    protected override IRuleMessages Execute(Event target)
    {
        if (target.StartDate > target.EndDate)
            return (nameof(target.EndDate),
                "End date must be after start date").AsRuleMessages();

        return None;
    }
}
```

### Conditional Validation

```csharp
public class ShippingAddressRule : RuleBase<Order>
{
    public ShippingAddressRule()
        : base(o => o.RequiresShipping, o => o.ShippingAddress) { }

    protected override IRuleMessages Execute(Order target)
    {
        if (target.RequiresShipping && string.IsNullOrEmpty(target.ShippingAddress))
            return (nameof(target.ShippingAddress),
                "Shipping address is required").AsRuleMessages();

        return None;
    }
}
```

### Format Normalization

```csharp
public class PhoneNormalizationRule : RuleBase<Contact>
{
    public PhoneNormalizationRule() : base(c => c.Phone) { }

    protected override IRuleMessages Execute(Contact target)
    {
        if (!string.IsNullOrEmpty(target.Phone))
        {
            // Remove non-digits
            target.Phone = new string(target.Phone.Where(char.IsDigit).ToArray());
        }
        return None;
    }
}
```

## Related Topics

- [Rules Philosophy](/concepts/rules-philosophy/) - Conceptual overview
- [EntityBase Reference](/reference/entity-base/) - RuleManager on entities
- [DDD Concepts](/concepts/ddd-overview/) - Business rules in DDD
- [Person Example](/example/person/) - Complete working example
