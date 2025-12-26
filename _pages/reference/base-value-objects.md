---
title: "Base<T> and Value Objects Reference"
layout: single
permalink: /reference/base-value-objects/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

`Base<T>` is the foundation class in Neatoo's hierarchy. It provides property management, UI binding through `INotifyPropertyChanged`, and parent-child relationships. Use it for Value Objects and simple bindable objects that do not need validation or persistence tracking.

## The DDD Value Object Pattern

In Domain-Driven Design, a **Value Object** represents a concept defined entirely by its attributes, with no distinct identity. Two Value Objects with the same attributes are considered equal, regardless of which instance you hold.

**Classic Examples:**

- **Money** - $20 is $20, regardless of which specific bill
- **Address** - Two addresses are equal if street, city, and zip match
- **DateRange** - A period defined by start and end dates
- **Coordinates** - A point defined by latitude and longitude

**Characteristics of Value Objects:**

1. **Identity-less** - Defined by attributes, not identity
2. **Immutable** - Once created, values do not change
3. **Replaceable** - Swap one instance for another with same values
4. **Self-validating** - Invalid values cannot be constructed

**Why Value Objects Matter:**

Value Objects encapsulate domain concepts with rich behavior. Instead of passing raw primitives, you pass meaningful types that enforce business rules:

```csharp
// Primitive obsession - error prone
void SetPrice(decimal amount, string currency);

// Value Object approach - type safe and meaningful
void SetPrice(Money amount);
```

## Class Hierarchy

```
Base<T>           <-- This class (Value Objects, simple bindable objects)
  └── ValidateBase<T>
        └── EntityBase<T>
```

`Base<T>` is the simplest Neatoo base class, providing only property management and UI binding.

## Using Base<T> for Value Objects

Declare Value Object classes as `partial` inheriting from `Base<T>`:

```csharp
[Factory]
internal partial class Address : Base<Address>, IAddress
{
    public Address(IBaseServices<Address> services) : base(services) { }

    public partial string? Street { get; set; }
    public partial string? City { get; set; }
    public partial string? State { get; set; }
    public partial string? ZipCode { get; set; }
    public partial string? Country { get; set; }

    // Computed property
    public string FormattedAddress =>
        $"{Street}\n{City}, {State} {ZipCode}\n{Country}";

    [Create]
    public void Create(
        string street,
        string city,
        string state,
        string zipCode,
        string country = "USA")
    {
        Street = street;
        City = city;
        State = state;
        ZipCode = zipCode;
        Country = country;
    }
}
```

**Note:** The constructor accepts `IBaseServices<T>` rather than `IEntityBaseServices<T>` or `IValidateBaseServices<T>`.

## The [Factory] Attribute on Non-EntityBase Classes

The `[Factory]` attribute works on any Neatoo class, not just entities. For Value Objects, it generates a factory that creates instances:

```csharp
[Factory]
internal partial class Money : Base<Money>, IMoney
{
    public Money(IBaseServices<Money> services) : base(services) { }

    public partial decimal Amount { get; set; }
    public partial string? Currency { get; set; }

    [Create]
    public void Create(decimal amount, string currency = "USD")
    {
        if (amount < 0)
            throw new ArgumentException("Amount cannot be negative", nameof(amount));
        if (string.IsNullOrEmpty(currency))
            throw new ArgumentException("Currency is required", nameof(currency));

        Amount = amount;
        Currency = currency;
    }
}
```

**Generated Factory:**

```csharp
public interface IMoneyFactory
{
    IMoney? Create(decimal amount, string currency = "USD");
    // Authorization methods if applicable
}
```

**Usage:**

```csharp
var money = MoneyFactory.Create(99.99m, "USD");
```

### Factory Differences for Value Objects

Value Object factories typically only have `[Create]` methods:

| Factory Method | Entity Use | Value Object Use |
|----------------|------------|------------------|
| `[Create]` | Initialize new entity | Create value object |
| `[Fetch]` | Load from database | Rarely used |
| `[Insert]` | Persist new entity | Not applicable |
| `[Update]` | Save changes | Not applicable |
| `[Delete]` | Remove from database | Not applicable |

Value Objects are not persisted independently; they are part of their containing entity.

## Immutability Patterns in Neatoo

Traditional Value Objects are immutable - once created, their values never change. Neatoo provides patterns to achieve this.

### Constructor Validation

Validate in the `[Create]` method and throw for invalid inputs:

```csharp
[Create]
public void Create(decimal amount, string currency)
{
    if (amount < 0)
        throw new ArgumentException("Amount cannot be negative");

    if (string.IsNullOrEmpty(currency) || currency.Length != 3)
        throw new ArgumentException("Currency must be 3-letter code");

    Amount = amount;
    Currency = currency.ToUpperInvariant();
}
```

### Private Setters Pattern

Use Neatoo properties normally but treat them as immutable by convention:

```csharp
[Factory]
internal partial class Money : Base<Money>, IMoney
{
    // Properties are technically settable, but interface exposes only getters
    public partial decimal Amount { get; set; }
    public partial string? Currency { get; set; }

    // Rich behavior methods return NEW instances
    public IMoney Add(IMoney other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add different currencies");

        return _factory.Create(Amount + other.Amount, Currency!);
    }

    public IMoney Multiply(decimal factor)
    {
        return _factory.Create(Amount * factor, Currency!);
    }

    private readonly IMoneyFactory _factory;

    public Money(IBaseServices<Money> services, IMoneyFactory factory)
        : base(services)
    {
        _factory = factory;
    }
}

// Interface exposes only getters
public interface IMoney : IBase
{
    decimal Amount { get; }
    string? Currency { get; }

    IMoney Add(IMoney other);
    IMoney Multiply(decimal factor);
}
```

### Replacement Pattern

When a value changes, replace the entire Value Object:

```csharp
// In an Entity
public partial IAddress? ShippingAddress { get; set; }

// To "change" the address, replace it entirely
public void UpdateShippingCity(string newCity)
{
    if (ShippingAddress == null) return;

    // Create new address with updated city
    ShippingAddress = AddressFactory.Create(
        ShippingAddress.Street,
        newCity,  // Changed
        ShippingAddress.State,
        ShippingAddress.ZipCode,
        ShippingAddress.Country);
}
```

## Value Object Serialization

Neatoo Value Objects serialize along with their containing entities. The generated serializers handle Value Objects automatically.

### Properties Serialize Directly

When an entity contains a Value Object property, it serializes as a nested object:

```csharp
[Factory]
internal partial class Order : EntityBase<Order>, IOrder
{
    // Value Object property
    public partial IMoney? TotalAmount { get; set; }

    public partial IAddress? ShippingAddress { get; set; }
}
```

The JSON representation includes nested objects:

```json
{
  "id": "...",
  "totalAmount": {
    "amount": 150.00,
    "currency": "USD"
  },
  "shippingAddress": {
    "street": "123 Main St",
    "city": "Springfield",
    "state": "IL",
    "zipCode": "62701",
    "country": "USA"
  }
}
```

### Remote Operations

Value Objects serialize correctly across the client-server boundary:

```csharp
// Client creates Value Object
var address = AddressFactory.Create("123 Main", "City", "ST", "12345");
order.ShippingAddress = address;

// When order saves, address serializes to server
await OrderFactory.Save(order);
// Server receives the complete Value Object
```

## Using Value Objects as Entity Properties

Value Objects are properties on entities, not independent persistent objects.

### Declaration

```csharp
[Factory]
internal partial class Customer : EntityBase<Customer>, ICustomer
{
    public partial Guid? Id { get; set; }
    public partial string? Name { get; set; }

    // Value Object properties
    public partial IAddress? BillingAddress { get; set; }
    public partial IAddress? ShippingAddress { get; set; }
    public partial IMoney? CreditLimit { get; set; }
}
```

### Factory Integration

Create Value Objects within entity factory methods:

```csharp
[Create]
public void Create(
    [Service] IAddressFactory addressFactory,
    [Service] IMoneyFactory moneyFactory)
{
    BillingAddress = addressFactory.Create("", "", "", "", "USA");
    ShippingAddress = addressFactory.Create("", "", "", "", "USA");
    CreditLimit = moneyFactory.Create(0, "USD");
}

[Remote]
[Fetch]
public async Task<bool> Fetch(
    [Service] ICustomerDbContext db,
    [Service] IAddressFactory addressFactory,
    [Service] IMoneyFactory moneyFactory)
{
    var entity = await db.Customers.FindAsync(Id);
    if (entity == null) return false;

    MapFrom(entity);

    // Create Value Objects from stored data
    BillingAddress = addressFactory.Create(
        entity.BillingStreet,
        entity.BillingCity,
        entity.BillingState,
        entity.BillingZip,
        entity.BillingCountry);

    CreditLimit = moneyFactory.Create(
        entity.CreditLimitAmount,
        entity.CreditLimitCurrency);

    return true;
}
```

### Mapping Value Objects

Handle Value Object mapping in your mapper methods:

```csharp
// Manual mapping for complex Value Objects
public partial void MapFrom(CustomerEntity entity);

// Override or extend for Value Objects
[MapperIgnore]
public partial IAddress? BillingAddress { get; set; }

// Then handle in factory method:
[Fetch]
public async Task<bool> Fetch([Service] IDbContext db,
                              [Service] IAddressFactory addressFactory)
{
    var entity = await db.Customers.FindAsync(Id);
    MapFrom(entity);  // Maps simple properties

    // Manually create Value Objects
    BillingAddress = addressFactory.Create(
        entity.BillingStreet,
        entity.BillingCity,
        entity.BillingState,
        entity.BillingZip);

    return true;
}
```

### Storing Value Objects

Flatten Value Objects when persisting:

```csharp
[Insert]
public async Task Insert([Service] IDbContext db)
{
    var entity = new CustomerEntity();
    MapTo(entity);

    // Flatten Value Object to columns
    if (BillingAddress != null)
    {
        entity.BillingStreet = BillingAddress.Street;
        entity.BillingCity = BillingAddress.City;
        entity.BillingState = BillingAddress.State;
        entity.BillingZip = BillingAddress.ZipCode;
        entity.BillingCountry = BillingAddress.Country;
    }

    db.Customers.Add(entity);
    await db.SaveChangesAsync();
}
```

## Why Value Objects Cannot Contain Entities

Value Objects should not reference Entities. This is a DDD principle that Neatoo supports.

### The Problem

If a Value Object contained an Entity:

1. **Identity Confusion** - Value Objects are identity-less, but Entities have identity
2. **Lifecycle Mismatch** - Entity lifecycle (create, update, delete) does not fit Value Object semantics
3. **Persistence Complexity** - How would the nested Entity be saved?
4. **Equality Breaks** - Two Value Objects with "equal" Entity references may not truly be equal

### The Solution

Value Objects contain only:

- Primitive types (string, int, decimal, DateTime, etc.)
- Other Value Objects
- Enums
- Immutable collections of the above

```csharp
// Correct: Value Object contains primitives and other Value Objects
[Factory]
internal partial class OrderSummary : Base<OrderSummary>, IOrderSummary
{
    public partial IMoney? Subtotal { get; set; }      // Value Object
    public partial IMoney? Tax { get; set; }           // Value Object
    public partial IMoney? Total { get; set; }         // Value Object
    public partial int ItemCount { get; set; }         // Primitive
    public partial DateTime CalculatedAt { get; set; } // Primitive
}

// Incorrect: Value Object containing Entity - avoid this
// public partial IOrder? Order { get; set; }  // Don't do this!
```

### Reference Entities from Entities

When you need to reference an Entity from a Value Object concept, reference it from the containing Entity instead:

```csharp
[Factory]
internal partial class OrderLine : EntityBase<OrderLine>, IOrderLine
{
    // Entity reference - on the Entity, not Value Object
    public partial IProduct? Product { get; set; }

    // Value Object for the price snapshot
    public partial IMoney? UnitPrice { get; set; }

    // Primitive for simple values
    public partial int Quantity { get; set; }
}
```

## Complete Examples

### Address Value Object

```csharp
public interface IAddress : IBase
{
    string? Street { get; }
    string? Street2 { get; }
    string? City { get; }
    string? State { get; }
    string? ZipCode { get; }
    string? Country { get; }

    string FormattedAddress { get; }
    bool IsComplete { get; }
}

[Factory]
internal partial class Address : Base<Address>, IAddress
{
    public Address(IBaseServices<Address> services) : base(services) { }

    public partial string? Street { get; set; }
    public partial string? Street2 { get; set; }
    public partial string? City { get; set; }
    public partial string? State { get; set; }
    public partial string? ZipCode { get; set; }
    public partial string? Country { get; set; }

    public string FormattedAddress
    {
        get
        {
            var lines = new List<string>();
            if (!string.IsNullOrEmpty(Street)) lines.Add(Street);
            if (!string.IsNullOrEmpty(Street2)) lines.Add(Street2);
            if (!string.IsNullOrEmpty(City) || !string.IsNullOrEmpty(State))
            {
                var cityState = $"{City}, {State} {ZipCode}".Trim(' ', ',');
                lines.Add(cityState);
            }
            if (!string.IsNullOrEmpty(Country)) lines.Add(Country);
            return string.Join("\n", lines);
        }
    }

    public bool IsComplete =>
        !string.IsNullOrEmpty(Street) &&
        !string.IsNullOrEmpty(City) &&
        !string.IsNullOrEmpty(State) &&
        !string.IsNullOrEmpty(ZipCode);

    [Create]
    public void Create(
        string street,
        string city,
        string state,
        string zipCode,
        string? street2 = null,
        string country = "USA")
    {
        Street = street;
        Street2 = street2;
        City = city;
        State = state;
        ZipCode = zipCode;
        Country = country;
    }
}
```

### Money Value Object with Currency

```csharp
public interface IMoney : IBase
{
    decimal Amount { get; }
    string Currency { get; }

    IMoney Add(IMoney other);
    IMoney Subtract(IMoney other);
    IMoney Multiply(decimal factor);
    IMoney Round(int decimals = 2);

    string Format();
}

[Factory]
internal partial class Money : Base<Money>, IMoney
{
    private readonly IMoneyFactory _factory;

    public Money(IBaseServices<Money> services, IMoneyFactory factory)
        : base(services)
    {
        _factory = factory;
    }

    public partial decimal Amount { get; set; }
    public partial string? CurrencyCode { get; set; }

    public string Currency => CurrencyCode ?? "USD";

    public IMoney Add(IMoney other)
    {
        ValidateSameCurrency(other);
        return _factory.Create(Amount + other.Amount, Currency);
    }

    public IMoney Subtract(IMoney other)
    {
        ValidateSameCurrency(other);
        return _factory.Create(Amount - other.Amount, Currency);
    }

    public IMoney Multiply(decimal factor)
    {
        return _factory.Create(Amount * factor, Currency);
    }

    public IMoney Round(int decimals = 2)
    {
        return _factory.Create(Math.Round(Amount, decimals), Currency);
    }

    public string Format()
    {
        return Currency switch
        {
            "USD" => Amount.ToString("C", CultureInfo.GetCultureInfo("en-US")),
            "EUR" => Amount.ToString("C", CultureInfo.GetCultureInfo("de-DE")),
            "GBP" => Amount.ToString("C", CultureInfo.GetCultureInfo("en-GB")),
            _ => $"{Amount:N2} {Currency}"
        };
    }

    private void ValidateSameCurrency(IMoney other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException(
                $"Cannot combine {Currency} with {other.Currency}");
    }

    [Create]
    public void Create(decimal amount, string currency = "USD")
    {
        if (string.IsNullOrEmpty(currency) || currency.Length != 3)
            throw new ArgumentException("Currency must be 3-letter ISO code");

        Amount = amount;
        CurrencyCode = currency.ToUpperInvariant();
    }
}
```

### DateRange Value Object

```csharp
public interface IDateRange : IBase
{
    DateTime Start { get; }
    DateTime End { get; }

    int DayCount { get; }
    bool Contains(DateTime date);
    bool Overlaps(IDateRange other);
    IDateRange? Intersect(IDateRange other);
}

[Factory]
internal partial class DateRange : Base<DateRange>, IDateRange
{
    private readonly IDateRangeFactory _factory;

    public DateRange(IBaseServices<DateRange> services, IDateRangeFactory factory)
        : base(services)
    {
        _factory = factory;
    }

    public partial DateTime Start { get; set; }
    public partial DateTime End { get; set; }

    public int DayCount => (End - Start).Days + 1;

    public bool Contains(DateTime date)
    {
        return date >= Start && date <= End;
    }

    public bool Overlaps(IDateRange other)
    {
        return Start <= other.End && End >= other.Start;
    }

    public IDateRange? Intersect(IDateRange other)
    {
        if (!Overlaps(other)) return null;

        var intersectStart = Start > other.Start ? Start : other.Start;
        var intersectEnd = End < other.End ? End : other.End;

        return _factory.Create(intersectStart, intersectEnd);
    }

    [Create]
    public void Create(DateTime start, DateTime end)
    {
        if (end < start)
            throw new ArgumentException("End date must be >= start date");

        Start = start.Date;  // Normalize to date only
        End = end.Date;
    }
}
```

## Available Meta-Properties on Base<T>

`Base<T>` provides a minimal set of meta-properties:

### IsBusy

```csharp
public virtual bool IsBusy { get; }
```

Returns `true` if any async property setters are in progress.

### Parent

```csharp
public IBase? Parent { get; }
```

Reference to the containing object if this is a child.

### PropertyChanged Event

```csharp
public event PropertyChangedEventHandler? PropertyChanged;
```

Standard `INotifyPropertyChanged` implementation for UI binding.

**Not Available on Base<T>:**

- `IsValid`, `IsSelfValid` - Use `ValidateBase<T>`
- `IsNew`, `IsModified`, `IsDeleted` - Use `EntityBase<T>`
- `IsSavable` - Use `EntityBase<T>`
- `PropertyMessages` - Use `ValidateBase<T>`

## Best Practices

### Validate in Factory Methods

Since `Base<T>` has no validation engine, validate in `[Create]`:

```csharp
[Create]
public void Create(decimal amount, string currency)
{
    if (amount < 0)
        throw new ArgumentException("Amount must be non-negative");

    if (currency?.Length != 3)
        throw new ArgumentException("Currency must be 3-letter code");

    Amount = amount;
    Currency = currency;
}
```

### Keep Value Objects Small

Value Objects should be small, focused concepts:

```csharp
// Good: Focused, single concept
public interface IMoney { decimal Amount { get; } string Currency { get; } }
public interface IAddress { string Street { get; } string City { get; } ... }

// Less good: Too many unrelated concepts
public interface IOrderDetails { /* 20+ properties */ }
```

### Expose Behavior, Not Just Data

Value Objects should encapsulate behavior:

```csharp
// Just data
var total = money1.Amount + money2.Amount;

// Rich behavior
var total = money1.Add(money2);  // Validates currency match
```

### Use Factories, Not Constructors

Always use the generated factory:

```csharp
// Correct
var money = MoneyFactory.Create(100m, "USD");

// Incorrect - bypasses validation
// var money = new Money(...);  // Internal anyway
```

## Related Topics

- [ValidateBase Reference](/reference/validate-base/) - Validation without persistence
- [EntityBase Reference](/reference/entity-base/) - Full entity with persistence
- [DDD Concepts](/concepts/ddd-overview/) - Domain-Driven Design patterns
- [Factory Operations Reference](/reference/factory-operations/) - Factory system
- [Data Mapping Reference](/reference/data-mapping/) - Mapping patterns
