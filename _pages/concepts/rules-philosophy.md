---
title: "Rules Philosophy"
layout: single
permalink: /concepts/rules-philosophy/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

Neatoo treats business rules as first-class citizens in your domain model. Rules are not scattered across controllers, services, and validation attributes. They are explicit, testable classes that execute automatically when their trigger properties change, providing immediate feedback to users and consistent enforcement across your application.

## The Problem with Anemic Domain Models

In many enterprise applications, domain objects become simple data containers. All behavior lives in service classes:

```csharp
// Anemic domain model - entities are just bags of properties
public class Person
{
    public Guid Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
}

// Logic scattered in services
public class PersonService
{
    public ValidationResult ValidatePerson(Person person)
    {
        var errors = new List<string>();

        if (string.IsNullOrEmpty(person.FirstName))
            errors.Add("First name is required");

        if (string.IsNullOrEmpty(person.LastName))
            errors.Add("Last name is required");

        if (!IsValidEmail(person.Email))
            errors.Add("Invalid email format");

        if (await EmailExists(person.Email))
            errors.Add("Email already in use");

        return new ValidationResult(errors);
    }
}
```

This approach has serious problems:

1. **Logic is invisible** - Looking at `Person`, you cannot tell what rules govern it
2. **Multiple code paths** - Different callers might validate differently (or forget to validate)
3. **Testing requires integration** - You must instantiate services to test validation
4. **UI disconnection** - Client-side validation is completely separate code
5. **No automatic updates** - Changing a property does not automatically re-validate

### The "Service Layer Explosion"

As applications grow, the service layer accumulates:

- `PersonValidationService`
- `PersonCalculationService`
- `PersonEmailService`
- `PersonAuthorizationService`

Each service has dependencies. Testing one service means mocking many others. The domain model, which should be the heart of your application, becomes a passive data shuttle between these services.

## Business Rules as First-Class Citizens

Neatoo inverts this pattern. Business rules live **on the domain object** and execute **automatically**:

```csharp
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    public Person(IEntityBaseServices<Person> services,
                  IUniqueEmailRule uniqueEmailRule) : base(services)
    {
        RuleManager.AddRule(uniqueEmailRule);
    }

    [Required(ErrorMessage = "First name is required")]
    public partial string? FirstName { get; set; }

    [Required(ErrorMessage = "Last name is required")]
    public partial string? LastName { get; set; }

    public partial string? Email { get; set; }
}

public class UniqueEmailRule : AsyncRuleBase<Person>
{
    private readonly IEmailService _emailService;

    public UniqueEmailRule(IEmailService emailService)
        : base(p => p.Email) { }

    protected override async Task<IRuleMessages> Execute(
        Person target, CancellationToken? token = null)
    {
        if (string.IsNullOrEmpty(target.Email))
            return None;

        var exists = await _emailService.EmailExistsAsync(target.Email);
        return exists
            ? (nameof(target.Email), "Email already in use").AsRuleMessages()
            : None;
    }
}
```

Now:

1. **Rules are visible** - Looking at `Person`, you see exactly what governs it
2. **Single code path** - Rules execute the same way everywhere
3. **Unit testable** - Rules are classes you can instantiate and test directly
4. **UI connected** - Same rules run on client and server
5. **Automatic execution** - Changing `Email` triggers the uniqueness check immediately

## The Spreadsheet Analogy

Think of your domain model as a spreadsheet:

| Cell | Value | Formula |
|------|-------|---------|
| A1 | 100 | (input) |
| A2 | 20 | (input) |
| A3 | 120 | =A1+A2 |
| A4 | "Valid" | =IF(A3>0, "Valid", "Invalid") |

When you change A1:

1. A3 automatically recalculates
2. A4 automatically recalculates
3. The UI shows new values immediately
4. No explicit "recalculate" button needed

Neatoo rules work the same way:

```csharp
// When FirstName or LastName changes, FullName recalculates
RuleManager.AddAction(
    (Person p) => p.FullName = $"{p.FirstName} {p.LastName}",
    p => p.FirstName, p => p.LastName);

// When Age changes, validation runs
RuleManager.AddValidation(
    nameof(Age),
    (Person p) => p.Age < 0
        ? RuleMessage.Error("Age cannot be negative")
        : RuleMessage.None);
```

Change a property, and dependent properties update. Change a value, and validation runs. No explicit calls needed.

## Trigger-Based Execution

Every rule declares which properties trigger its execution:

```csharp
public class OrderTotalRule : RuleBase<Order>
{
    // This rule runs when Subtotal, TaxRate, or DiscountPercent changes
    public OrderTotalRule()
        : base(o => o.Subtotal, o => o.TaxRate, o => o.DiscountPercent) { }

    protected override IRuleMessages Execute(Order target)
    {
        target.Tax = target.Subtotal * target.TaxRate;
        target.Discount = target.Subtotal * target.DiscountPercent;
        target.Total = target.Subtotal + target.Tax - target.Discount;
        return None;
    }
}
```

The framework tracks property changes and runs the appropriate rules. You declare dependencies; the framework handles orchestration.

### When Rules Execute

Rules execute when:

1. **Property setter is called** - User types in a field, rule runs
2. **Factory operation completes** - After `Fetch`, rules run on loaded data
3. **Manual invocation** - Code calls `RunRules()` explicitly
4. **Parent/child changes** - Cascading changes trigger related rules

Rules do **not** execute when:

- Properties are loaded via `LoadProperty` (fetch operations)
- `PauseAllActions()` is active
- The same rule is already executing (prevents infinite loops)

## Cascading Rules

Rules can trigger other rules by modifying properties:

```csharp
// Rule 1: Calculate subtotal when line items change
public class SubtotalRule : RuleBase<Order>
{
    public SubtotalRule() : base(o => o.Lines) { }

    protected override IRuleMessages Execute(Order target)
    {
        target.Subtotal = target.Lines?.Sum(l => l.Total) ?? 0;
        return None;
    }
}

// Rule 2: Calculate total when subtotal changes (triggered by Rule 1)
public class TotalRule : RuleBase<Order>
{
    public TotalRule() : base(o => o.Subtotal, o => o.TaxRate) { }

    protected override IRuleMessages Execute(Order target)
    {
        target.Tax = target.Subtotal * target.TaxRate;
        target.Total = target.Subtotal + target.Tax;
        return None;
    }
}
```

When a line item changes:

1. `SubtotalRule` runs, updates `Subtotal`
2. `Subtotal` change triggers `TotalRule`
3. `TotalRule` updates `Tax` and `Total`
4. UI bound to these properties updates

This cascading happens automatically with no manual orchestration.

## Synchronous vs. Asynchronous Rules

### Synchronous Rules (RuleBase)

Use for validation and calculations that do not require I/O:

```csharp
public class AgeValidationRule : RuleBase<Person>
{
    public AgeValidationRule() : base(p => p.Age) { }

    protected override IRuleMessages Execute(Person target)
    {
        if (target.Age < 0)
            return (nameof(target.Age), "Age cannot be negative").AsRuleMessages();

        if (target.Age > 150)
            return (nameof(target.Age), "Age seems unrealistic").AsRuleMessages();

        return None;
    }
}
```

Synchronous rules:

- Execute immediately when triggered
- Block until complete
- Cannot access databases or APIs
- Best for format validation and calculations

### Asynchronous Rules (AsyncRuleBase)

Use when rules need external data:

```csharp
public class UniqueUsernameRule : AsyncRuleBase<User>
{
    private readonly IUserRepository _repository;

    public UniqueUsernameRule(IUserRepository repository)
        : base(u => u.Username)
    {
        _repository = repository;
    }

    protected override async Task<IRuleMessages> Execute(
        User target, CancellationToken? token = null)
    {
        if (string.IsNullOrEmpty(target.Username))
            return None;

        var exists = await _repository.UsernameExistsAsync(
            target.Username, target.Id, token ?? CancellationToken.None);

        return exists
            ? (nameof(target.Username), "Username already taken").AsRuleMessages()
            : None;
    }
}
```

Asynchronous rules:

- Set `IsBusy = true` while executing
- Allow the UI to remain responsive
- Support cancellation
- Best for database lookups, API calls, complex calculations

### The IsBusy Property

While async rules execute, the entity's `IsBusy` property is `true`:

```razor
<button @onclick="Save" disabled="@(person.IsBusy || !person.IsSavable)">
    @(person.IsBusy ? "Validating..." : "Save")
</button>
```

This prevents saving while validation is in progress and provides user feedback.

## Validation vs. Transformation

Rules serve two purposes:

### Validation Rules

Return messages when data is invalid:

```csharp
protected override IRuleMessages Execute(Person target)
{
    if (string.IsNullOrEmpty(target.Email))
        return None; // Empty is OK, Required attribute handles that

    if (!target.Email.Contains("@"))
        return (nameof(target.Email), "Invalid email format").AsRuleMessages();

    return None;
}
```

Validation rules:

- Check data correctness
- Return error messages
- Make `IsValid` false when they fail
- Prevent saving when violated

### Transformation Rules

Modify data without returning messages:

```csharp
protected override IRuleMessages Execute(Order target)
{
    // Calculate derived values
    target.Subtotal = target.Lines?.Sum(l => l.Total) ?? 0;
    target.Tax = target.Subtotal * target.TaxRate;
    target.Total = target.Subtotal + target.Tax;

    return None; // No validation message
}
```

Transformation rules:

- Calculate derived properties
- Normalize data formats
- Propagate changes
- Always return `None`

### Combined Rules

A single rule can do both:

```csharp
protected override IRuleMessages Execute(Order target)
{
    // Transform
    target.Total = target.Subtotal + target.Tax - target.Discount;

    // Validate
    if (target.Total < 0)
        return (nameof(target.Total), "Total cannot be negative").AsRuleMessages();

    return None;
}
```

## Integration with UI Binding

Neatoo entities implement `INotifyPropertyChanged`. When rules run and modify properties, the UI updates automatically:

```razor
@* Blazor component *@
<input @bind="person.FirstName" />
<input @bind="person.LastName" />

@* FullName updates automatically when FirstName or LastName change *@
<p>Full Name: @person.FullName</p>

@* Validation messages appear automatically *@
@foreach (var message in person.PropertyMessages)
{
    <div class="error">@message.Message</div>
}

@* Save button enables/disables based on validation state *@
<button disabled="@(!person.IsSavable)">Save</button>
```

No explicit refresh calls. No manual message aggregation. The domain model drives the UI naturally.

## Contrast with Traditional Approaches

### Data Annotations Only

Traditional:

```csharp
public class Person
{
    [Required]
    [StringLength(50)]
    public string FirstName { get; set; }

    [Required]
    [EmailAddress]
    public string Email { get; set; }
}
```

Limitations:

- No cross-property validation
- No async validation
- No transformation/calculation
- No automatic UI updates
- Client and server validation separate

### Fluent Validation

Traditional:

```csharp
public class PersonValidator : AbstractValidator<Person>
{
    public PersonValidator()
    {
        RuleFor(x => x.FirstName).NotEmpty();
        RuleFor(x => x.Email).EmailAddress();
    }
}
```

Limitations:

- Validation is separate from the domain object
- Must explicitly call `Validate()`
- No trigger-based execution
- No automatic UI binding
- Must wire up client-side separately

### Neatoo Approach

```csharp
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    public Person(IEntityBaseServices<Person> services) : base(services)
    {
        RuleManager.AddValidation(
            nameof(Email),
            (Person p) => IsValidEmail(p.Email)
                ? RuleMessage.None
                : RuleMessage.Error("Invalid email"));
    }

    [Required]
    public partial string? FirstName { get; set; }

    public partial string? Email { get; set; }
}
```

Advantages:

- Rules live on the domain object
- Automatic trigger-based execution
- Supports sync and async
- Same code runs client and server
- Direct UI binding

## Best Practices

### Keep Rules Focused

Each rule should do one thing:

```csharp
// Good - single responsibility
public class EmailFormatRule : RuleBase<Person> { }
public class UniqueEmailRule : AsyncRuleBase<Person> { }

// Avoid - doing too much
public class EmailAllTheThingsRule : AsyncRuleBase<Person>
{
    // Validates format AND checks uniqueness AND normalizes case...
}
```

### Use Dependency Injection

Rules can have dependencies:

```csharp
public class UniqueEmailRule : AsyncRuleBase<Person>
{
    private readonly IEmailService _emailService;

    public UniqueEmailRule(IEmailService emailService)
        : base(p => p.Email)
    {
        _emailService = emailService;
    }
}
```

Register rules in your DI container:

```csharp
builder.Services.AddScoped<IUniqueEmailRule, UniqueEmailRule>();
```

### Test Rules in Isolation

Rules are classes you can unit test:

```csharp
[Fact]
public async Task UniqueEmailRule_WhenEmailExists_ReturnsError()
{
    // Arrange
    var mockService = new Mock<IEmailService>();
    mockService.Setup(s => s.EmailExistsAsync("test@example.com"))
        .ReturnsAsync(true);

    var rule = new UniqueEmailRule(mockService.Object);
    var person = CreateTestPerson();
    person.Email = "test@example.com";

    // Act
    var messages = await rule.Execute(person);

    // Assert
    Assert.Contains(messages, m => m.Message.Contains("already in use"));
}
```

### Consider Rule Execution Order

When multiple rules trigger on the same property, consider their interactions. Transformation rules should typically run before validation rules that depend on the transformed values.

## Related Topics

- [Rules Engine Reference](/reference/rules/) - Complete API documentation
- [DDD Concepts](/concepts/ddd-overview/) - Domain-Driven Design patterns
- [EntityBase Reference](/reference/entity-base/) - Entity API including RuleManager
- [Introduction](/introduction/) - Framework overview
