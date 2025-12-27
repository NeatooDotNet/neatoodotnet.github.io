---
title: "ValidateBase Reference"
layout: single
permalink: /reference/validate-base/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

`ValidateBase<T>` provides validation and rules capabilities without persistence tracking. Use it when you need business rules and validation but do not need to track modification state for database operations.

## When to Use ValidateBase vs EntityBase

The choice between `ValidateBase<T>` and `EntityBase<T>` depends on whether you need persistence tracking.

### Use EntityBase When

- The object represents a domain entity that will be persisted
- You need `IsNew`, `IsModified`, `IsDeleted` tracking
- The object participates in `Save()` operations
- Changes need to be tracked for database updates

### Use ValidateBase When

- The object needs validation but not persistence tracking
- The object represents a transient form or wizard step
- You are building complex input forms with multi-step validation
- The object validates user input before creating a real entity
- You need a validated DTO or command object

### Decision Guide

| Need | Base Class |
|------|------------|
| Database persistence | `EntityBase<T>` |
| Parent-child in aggregate | `EntityBase<T>` |
| Modification tracking | `EntityBase<T>` |
| Validation only | `ValidateBase<T>` |
| Wizard/form steps | `ValidateBase<T>` |
| Transient input objects | `ValidateBase<T>` |
| Value objects with validation | `ValidateBase<T>` |

## Class Hierarchy

```
Base<T>
  └── ValidateBase<T>      <-- This class
        └── EntityBase<T>
```

`ValidateBase<T>` inherits from `Base<T>` (property management, UI binding) and adds validation capabilities. `EntityBase<T>` then adds persistence tracking on top.

## Class Declaration

Declare validation classes as `partial` and inherit from `ValidateBase<T>`:

```csharp
[Factory]
internal partial class PaymentForm : ValidateBase<PaymentForm>, IPaymentForm
{
    public PaymentForm(IValidateBaseServices<PaymentForm> services)
        : base(services) { }

    // Properties and rules...
}
```

Note the constructor parameter type: `IValidateBaseServices<T>` rather than `IEntityBaseServices<T>`.

## Constructor Pattern

```csharp
public PaymentForm(
    IValidateBaseServices<PaymentForm> services,
    ICreditCardValidationRule cardRule) : base(services)
{
    RuleManager.AddRule(cardRule);
}
```

The constructor:

- Accepts `IValidateBaseServices<T>` (required)
- Passes services to the base class
- Adds any injected rules to the `RuleManager`

## Use Cases

### Wizard Steps

Multi-step forms where each step needs independent validation:

```csharp
// Step 1: Personal Information
[Factory]
internal partial class PersonalInfoStep : ValidateBase<PersonalInfoStep>, IPersonalInfoStep
{
    public PersonalInfoStep(IValidateBaseServices<PersonalInfoStep> services)
        : base(services) { }

    [Required(ErrorMessage = "First name is required")]
    public partial string? FirstName { get; set; }

    [Required(ErrorMessage = "Last name is required")]
    public partial string? LastName { get; set; }

    [EmailAddress(ErrorMessage = "Valid email required")]
    public partial string? Email { get; set; }

    [Create]
    public void Create() { }
}

// Step 2: Address Information
[Factory]
internal partial class AddressStep : ValidateBase<AddressStep>, IAddressStep
{
    public AddressStep(IValidateBaseServices<AddressStep> services)
        : base(services) { }

    [Required]
    public partial string? Street { get; set; }

    [Required]
    public partial string? City { get; set; }

    [Required]
    [RegularExpression(@"^\d{5}(-\d{4})?$", ErrorMessage = "Invalid ZIP")]
    public partial string? ZipCode { get; set; }

    [Create]
    public void Create() { }
}

// Step 3: Payment Information
[Factory]
internal partial class PaymentStep : ValidateBase<PaymentStep>, IPaymentStep
{
    public PaymentStep(
        IValidateBaseServices<PaymentStep> services,
        ICreditCardRule creditCardRule) : base(services)
    {
        RuleManager.AddRule(creditCardRule);
    }

    [Required]
    [CreditCard(ErrorMessage = "Invalid card number")]
    public partial string? CardNumber { get; set; }

    [Required]
    public partial string? ExpirationDate { get; set; }

    [Required]
    [StringLength(4, MinimumLength = 3)]
    public partial string? CVV { get; set; }

    [Create]
    public void Create() { }
}
```

**Blazor Wizard Usage:**

```razor
@page "/registration"

<MudStepper @ref="_stepper" Linear="true">
    <MudStep Title="Personal Info"
             Completed="@_personalInfo.IsValid">
        <MudNeatooTextField For="() => _personalInfo.FirstName" Label="First Name" />
        <MudNeatooTextField For="() => _personalInfo.LastName" Label="Last Name" />
        <MudNeatooTextField For="() => _personalInfo.Email" Label="Email" />
    </MudStep>

    <MudStep Title="Address"
             Completed="@_address.IsValid">
        <MudNeatooTextField For="() => _address.Street" Label="Street" />
        <MudNeatooTextField For="() => _address.City" Label="City" />
        <MudNeatooTextField For="() => _address.ZipCode" Label="ZIP Code" />
    </MudStep>

    <MudStep Title="Payment"
             Completed="@_payment.IsValid">
        <MudNeatooTextField For="() => _payment.CardNumber" Label="Card Number" />
        <MudNeatooTextField For="() => _payment.ExpirationDate" Label="Expiration" />
        <MudNeatooTextField For="() => _payment.CVV" Label="CVV" />
    </MudStep>
</MudStepper>

<MudButton OnClick="Submit"
           Disabled="@(!CanSubmit)">
    Complete Registration
</MudButton>

@code {
    private IPersonalInfoStep _personalInfo = default!;
    private IAddressStep _address = default!;
    private IPaymentStep _payment = default!;

    private bool CanSubmit =>
        _personalInfo.IsValid &&
        _address.IsValid &&
        _payment.IsValid;

    protected override void OnInitialized()
    {
        _personalInfo = PersonalInfoStepFactory.Create();
        _address = AddressStepFactory.Create();
        _payment = PaymentStepFactory.Create();
    }

    private async Task Submit()
    {
        await _personalInfo.WaitForTasks();
        await _address.WaitForTasks();
        await _payment.WaitForTasks();

        if (!CanSubmit) return;

        // Create the real entity from wizard data
        var registration = RegistrationFactory.Create();
        registration.FirstName = _personalInfo.FirstName;
        registration.LastName = _personalInfo.LastName;
        // ... copy other fields
        await RegistrationFactory.Save(registration);
    }
}
```

### Complex Forms

Forms that need rich validation before creating domain entities:

```csharp
[Factory]
internal partial class OrderEntryForm : ValidateBase<OrderEntryForm>, IOrderEntryForm
{
    public OrderEntryForm(
        IValidateBaseServices<OrderEntryForm> services,
        ICustomerExistsRule customerRule,
        IInventoryCheckRule inventoryRule) : base(services)
    {
        RuleManager.AddRule(customerRule);
        RuleManager.AddRule(inventoryRule);
    }

    [Required]
    public partial string? CustomerNumber { get; set; }

    public partial string? CustomerName { get; set; }  // Populated by rule

    [Required]
    public partial string? ProductCode { get; set; }

    public partial string? ProductName { get; set; }  // Populated by rule

    [Range(1, 1000)]
    public partial int Quantity { get; set; }

    public partial bool InStock { get; set; }  // Set by rule

    public partial int AvailableQuantity { get; set; }  // Set by rule

    [Create]
    public void Create()
    {
        Quantity = 1;
    }
}

// Rule that looks up customer
public class CustomerExistsRule : AsyncRuleBase<OrderEntryForm>, ICustomerExistsRule
{
    private readonly ICustomerService _customerService;

    public CustomerExistsRule(ICustomerService customerService)
        : base(f => f.CustomerNumber)
    {
        _customerService = customerService;
    }

    protected override async Task<IRuleMessages> Execute(
        OrderEntryForm target, CancellationToken? token = null)
    {
        if (string.IsNullOrEmpty(target.CustomerNumber))
        {
            target.CustomerName = null;
            return None;
        }

        var customer = await _customerService.FindByNumberAsync(
            target.CustomerNumber, token ?? CancellationToken.None);

        if (customer == null)
        {
            target.CustomerName = null;
            return (nameof(target.CustomerNumber), "Customer not found")
                .AsRuleMessages();
        }

        target.CustomerName = customer.Name;
        return None;
    }
}
```

### Validation-Only Objects

Objects that validate input without representing persistent domain concepts:

```csharp
[Factory]
internal partial class PasswordChangeRequest
    : ValidateBase<PasswordChangeRequest>, IPasswordChangeRequest
{
    public PasswordChangeRequest(
        IValidateBaseServices<PasswordChangeRequest> services)
        : base(services)
    {
        // Password match validation
        RuleManager.AddValidation(
            (PasswordChangeRequest r) =>
            {
                if (r.NewPassword != r.ConfirmPassword)
                    return (nameof(r.ConfirmPassword), "Passwords do not match")
                        .AsRuleMessages();
                return RuleMessages.None;
            },
            r => r.NewPassword, r => r.ConfirmPassword);

        // Password strength validation
        RuleManager.AddValidation(
            nameof(NewPassword),
            (PasswordChangeRequest r) =>
            {
                if (string.IsNullOrEmpty(r.NewPassword))
                    return RuleMessage.None;

                if (r.NewPassword.Length < 8)
                    return RuleMessage.Error("Password must be at least 8 characters");

                if (!r.NewPassword.Any(char.IsUpper))
                    return RuleMessage.Error("Password must contain uppercase letter");

                if (!r.NewPassword.Any(char.IsDigit))
                    return RuleMessage.Error("Password must contain a number");

                return RuleMessage.None;
            });
    }

    [Required(ErrorMessage = "Current password is required")]
    public partial string? CurrentPassword { get; set; }

    [Required(ErrorMessage = "New password is required")]
    public partial string? NewPassword { get; set; }

    [Required(ErrorMessage = "Please confirm your password")]
    public partial string? ConfirmPassword { get; set; }

    [Create]
    public void Create() { }
}
```

## Available Meta-Properties

`ValidateBase<T>` provides these meta-properties (subset of `EntityBase<T>`):

### IsValid

```csharp
public virtual bool IsValid { get; }
```

Returns `true` when this object and all children pass validation.

```csharp
var form = FormFactory.Create();
form.Email = "invalid";

// form.IsValid == false
```

### IsSelfValid

```csharp
public virtual bool IsSelfValid { get; }
```

Returns `true` when this object's own properties pass validation, ignoring children.

### IsBusy

```csharp
public virtual bool IsBusy { get; }
```

Returns `true` when async validation rules are executing.

### IsSelfBusy

```csharp
public virtual bool IsSelfBusy { get; }
```

Returns `true` when this specific object has async operations running.

### PropertyMessages

```csharp
public IReadOnlyCollection<IPropertyMessage> PropertyMessages { get; }
```

All validation messages for this object.

```csharp
foreach (var msg in form.PropertyMessages)
{
    Console.WriteLine($"{msg.PropertyName}: {msg.Message}");
}
```

### Parent

```csharp
public IBase? Parent { get; }
```

Reference to parent object if this is a child in a hierarchy.

## Not Available on ValidateBase

These meta-properties are only available on `EntityBase<T>`:

- `IsNew` - Persistence tracking
- `IsModified` - Change tracking
- `IsSelfModified` - Change tracking
- `IsDeleted` - Deletion tracking
- `IsSavable` - Save eligibility
- `IsChild` - Child relationship (use `Parent != null` instead)
- `ModifiedProperties` - Changed property list

If you need these, use `EntityBase<T>` instead.

## RuleManager

The `RuleManager` provides full rule functionality:

### AddRule

```csharp
RuleManager.AddRule(injectedRule);
```

### AddValidation (Inline Sync)

```csharp
RuleManager.AddValidation(
    nameof(Email),
    (MyForm f) => f.Email?.Contains("@") == true
        ? RuleMessage.None
        : RuleMessage.Error("Invalid email"));
```

### AddValidationAsync (Inline Async)

```csharp
RuleManager.AddValidationAsync(
    async (MyForm f, CancellationToken ct) =>
    {
        var exists = await _service.CheckExists(f.Value, ct);
        return exists
            ? (nameof(f.Value), "Already exists").AsRuleMessages()
            : RuleMessages.None;
    },
    f => f.Value);
```

### AddAction (Transformation)

```csharp
RuleManager.AddAction(
    (MyForm f) => f.FullName = $"{f.FirstName} {f.LastName}",
    f => f.FirstName, f => f.LastName);
```

### AddActionAsync

```csharp
RuleManager.AddActionAsync(
    async (MyForm f, CancellationToken ct) =>
    {
        f.Details = await _service.LookupDetails(f.Code, ct);
    },
    f => f.Code);
```

## Integration with UI Binding

`ValidateBase<T>` implements `INotifyPropertyChanged`, enabling seamless Blazor binding:

```razor
<EditForm Model="@_form">
    <MudNeatooTextField For="() => _form.Email" Label="Email" />

    @if (!_form.IsValid)
    {
        <MudAlert Severity="Severity.Error">
            @foreach (var msg in _form.PropertyMessages)
            {
                <p>@msg.Message</p>
            }
        </MudAlert>
    }

    <MudButton OnClick="Submit"
               Disabled="@(!_form.IsValid || _form.IsBusy)">
        Submit
    </MudButton>
</EditForm>

@code {
    private IMyForm _form = default!;

    protected override void OnInitialized()
    {
        _form = MyFormFactory.Create();
    }

    private async Task Submit()
    {
        await _form.WaitForTasks();
        if (_form.IsValid)
        {
            // Process the validated form
        }
    }
}
```

## Factory Support

`ValidateBase<T>` classes support the `[Factory]` attribute and factory methods:

```csharp
[Factory]
internal partial class SearchForm : ValidateBase<SearchForm>, ISearchForm
{
    // ...

    [Create]
    public void Create()
    {
        // Initialize defaults
        StartDate = DateTime.Today.AddMonths(-1);
        EndDate = DateTime.Today;
    }
}
```

However, `[Insert]`, `[Update]`, and `[Delete]` are typically not used since `ValidateBase<T>` objects are not persisted directly. The `Save()` method is not available.

### Using Factory

```csharp
// Create a new form
var form = SearchFormFactory.Create();

// No Save() method - forms are not persisted
// Instead, use the validated data to create/update real entities
```

## Complete Example: Search Form

```csharp
public interface IProductSearchForm : IValidateBase
{
    string? SearchTerm { get; set; }
    DateTime? StartDate { get; set; }
    DateTime? EndDate { get; set; }
    decimal? MinPrice { get; set; }
    decimal? MaxPrice { get; set; }
    ProductCategory? Category { get; set; }
    bool InStockOnly { get; set; }
}

[Factory]
internal partial class ProductSearchForm
    : ValidateBase<ProductSearchForm>, IProductSearchForm
{
    public ProductSearchForm(
        IValidateBaseServices<ProductSearchForm> services) : base(services)
    {
        // Date range validation
        RuleManager.AddValidation(
            (ProductSearchForm f) =>
            {
                if (f.StartDate.HasValue && f.EndDate.HasValue
                    && f.StartDate > f.EndDate)
                {
                    return (nameof(f.EndDate), "End date must be after start date")
                        .AsRuleMessages();
                }
                return RuleMessages.None;
            },
            f => f.StartDate, f => f.EndDate);

        // Price range validation
        RuleManager.AddValidation(
            (ProductSearchForm f) =>
            {
                if (f.MinPrice.HasValue && f.MaxPrice.HasValue
                    && f.MinPrice > f.MaxPrice)
                {
                    return (nameof(f.MaxPrice), "Max price must be >= min price")
                        .AsRuleMessages();
                }
                return RuleMessages.None;
            },
            f => f.MinPrice, f => f.MaxPrice);
    }

    [StringLength(100)]
    public partial string? SearchTerm { get; set; }

    public partial DateTime? StartDate { get; set; }

    public partial DateTime? EndDate { get; set; }

    [Range(0, double.MaxValue, ErrorMessage = "Price must be positive")]
    public partial decimal? MinPrice { get; set; }

    [Range(0, double.MaxValue, ErrorMessage = "Price must be positive")]
    public partial decimal? MaxPrice { get; set; }

    public partial ProductCategory? Category { get; set; }

    public partial bool InStockOnly { get; set; }

    [Create]
    public void Create()
    {
        InStockOnly = false;
    }
}
```

**Usage in Blazor:**

```razor
@page "/products/search"

<MudPaper Class="pa-4">
    <MudText Typo="Typo.h5">Search Products</MudText>

    <MudGrid>
        <MudItem xs="12">
            <MudNeatooTextField For="() => _search.SearchTerm"
                                Label="Search"
                                Adornment="Adornment.Start"
                                AdornmentIcon="@Icons.Material.Filled.Search" />
        </MudItem>

        <MudItem xs="6">
            <MudNeatooDatePicker For="() => _search.StartDate"
                                 Label="From Date" />
        </MudItem>

        <MudItem xs="6">
            <MudNeatooDatePicker For="() => _search.EndDate"
                                 Label="To Date" />
        </MudItem>

        <MudItem xs="6">
            <MudNeatooNumericField For="() => _search.MinPrice"
                                   Label="Min Price"
                                   Format="C2" />
        </MudItem>

        <MudItem xs="6">
            <MudNeatooNumericField For="() => _search.MaxPrice"
                                   Label="Max Price"
                                   Format="C2" />
        </MudItem>

        <MudItem xs="6">
            <MudNeatooSelect For="() => _search.Category"
                             Label="Category">
                <MudSelectItem Value="@((ProductCategory?)null)">All</MudSelectItem>
                @foreach (var cat in Enum.GetValues<ProductCategory>())
                {
                    <MudSelectItem Value="@((ProductCategory?)cat)">@cat</MudSelectItem>
                }
            </MudNeatooSelect>
        </MudItem>

        <MudItem xs="6">
            <MudNeatooCheckBox For="() => _search.InStockOnly"
                               Label="In Stock Only" />
        </MudItem>
    </MudGrid>

    <MudButton OnClick="Search"
               Disabled="@(!_search.IsValid)"
               Color="Color.Primary"
               Variant="Variant.Filled"
               Class="mt-4">
        Search
    </MudButton>
</MudPaper>

@if (_results != null)
{
    <MudTable Items="_results" Class="mt-4">
        @* Results table *@
    </MudTable>
}

@code {
    private IProductSearchForm _search = default!;
    private List<Product>? _results;

    protected override void OnInitialized()
    {
        _search = ProductSearchFormFactory.Create();
    }

    private async Task Search()
    {
        await _search.WaitForTasks();

        if (!_search.IsValid) return;

        _results = await ProductService.SearchAsync(
            _search.SearchTerm,
            _search.StartDate,
            _search.EndDate,
            _search.MinPrice,
            _search.MaxPrice,
            _search.Category,
            _search.InStockOnly);
    }
}
```

## Related Topics

- [EntityBase Reference](/reference/entity-base/) - Full entity with persistence
- [Base and Value Objects](/reference/base-value-objects/) - Simplest base class
- [Rules Engine Reference](/reference/rules/) - Validation rules
- [Properties and Meta-Properties](/concepts/properties/) - Property system
- [Blazor Integration](/guides/blazor-integration/) - UI binding patterns
