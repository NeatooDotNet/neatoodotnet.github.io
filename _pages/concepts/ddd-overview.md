---
title: "DDD Concepts in Neatoo"
layout: single
permalink: /concepts/ddd-overview/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

Domain-Driven Design (DDD) provides a set of patterns and practices for building software that closely models complex business domains. Neatoo implements these patterns with a focus on simplicity and developer productivity, eliminating the boilerplate that typically accompanies DDD implementations.

## What is Domain-Driven Design?

Domain-Driven Design is an approach to software development that centers the project on the core business domain and its logic. Rather than starting with database schemas or UI wireframes, DDD begins with understanding the business problem and creating a model that represents it in code.

The key insight of DDD is that **the code should speak the language of the business**. When a domain expert talks about "placing an order" or "approving a loan application," those concepts should exist as first-class citizens in your codebase, not be scattered across controllers, services, and database tables.

### The Challenge Without Proper Tooling

Implementing DDD in practice often leads to:

- **Excessive boilerplate**: Creating entities, value objects, and aggregates requires writing property change notifications, validation tracking, and modification state by hand
- **Scattered business logic**: Rules end up in service classes, controllers, and even the UI layer
- **Impedance mismatch**: Constant mapping between domain objects, DTOs, and persistence entities
- **Complexity fatigue**: Teams abandon DDD patterns because the overhead exceeds the benefit

Neatoo addresses these challenges by providing infrastructure that makes DDD patterns natural to implement and maintain.

## The Ubiquitous Language

The Ubiquitous Language is a shared vocabulary between developers and domain experts. Every term used in code should have a precise meaning understood by everyone involved in the project.

Consider an e-commerce system. Terms like "Order," "LineItem," "Customer," and "ShippingAddress" should mean exactly the same thing in:

- Conversations with stakeholders
- Requirements documents
- Code classes and methods
- Database tables
- API endpoints

Neatoo encourages Ubiquitous Language by making your domain classes the single source of truth. The same `Order` class that domain experts recognize in discussions is the same class that:

- Binds to your Blazor UI
- Serializes to the server
- Validates business rules
- Persists to the database

There is no translation layer where meaning gets lost.

## Tactical Patterns

DDD defines several tactical patterns for structuring domain logic. Neatoo provides base classes and infrastructure for each.

### Entities

An **Entity** is an object defined primarily by its identity. Two entities are equal if they have the same identity, regardless of their other attributes.

**Real-world analogy**: Think of a driver's license. Your license has a unique number that identifies it. If you change your address or even your name, it is still the same license. The identity persists through changes.

In code, entities have:

- A unique identifier (often a GUID or database-generated ID)
- Mutable state that changes over the entity's lifetime
- Business logic that governs how the state can change
- A lifecycle (created, modified, persisted, deleted)

Neatoo's `EntityBase<T>` provides:

```csharp
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    public Person(IEntityBaseServices<Person> services) : base(services) { }

    public partial Guid? Id { get; set; }           // Identity
    public partial string? FirstName { get; set; } // Mutable state
    public partial string? LastName { get; set; }
}
```

The framework automatically tracks:

- **IsNew**: Created but not yet persisted
- **IsModified**: Changed since last save
- **IsDeleted**: Marked for deletion
- **IsValid**: Passes all business rules

See [EntityBase Reference](/reference/entity-base/) for complete details.

### Value Objects

A **Value Object** is defined by its attributes, not identity. Two value objects are equal if all their attributes are equal.

**Real-world analogy**: Think of a $20 bill. You do not care which specific $20 bill you have in your wallet. Any $20 bill is interchangeable with any other. The value matters, not the identity.

Common examples include:

- Money (amount + currency)
- Addresses (street + city + postal code)
- Date ranges (start + end)
- Measurements (quantity + unit)

Value objects should be immutable after creation. In Neatoo, use plain classes with the `[Factory]` attribute:

```csharp
[Factory]
public class Address
{
    public string? Street { get; private set; }
    public string? City { get; private set; }
    public string? PostalCode { get; private set; }
    public string? Country { get; private set; }

    [Create]
    public void Create(string street, string city, string postalCode, string country)
    {
        Street = street;
        City = city;
        PostalCode = postalCode;
        Country = country;
    }
}
```

See [Value Objects Reference](/reference/base-value-objects/) for complete details.

### Aggregates

An **Aggregate** is a cluster of entities and value objects treated as a single unit for data changes. The aggregate has a root entity (the **Aggregate Root**) through which all external access must pass.

**Real-world analogy**: Think of an Order form on paper. The form contains the customer information, line items, totals, and payment details. When you submit the form, you submit everything together. You cannot modify individual line items independently; you work with the whole form.

Key aggregate rules:

1. **Single entry point**: External objects reference only the Aggregate Root
2. **Transactional boundary**: The entire aggregate saves or fails together
3. **Invariant enforcement**: The root ensures business rules across all contained objects

In Neatoo, aggregates form naturally through entity relationships:

```csharp
[Factory]
internal partial class Order : EntityBase<Order>, IOrder
{
    public partial Guid? Id { get; set; }
    public partial DateTime OrderDate { get; set; }
    public partial IOrderLineList? Lines { get; set; }  // Child collection
    public partial decimal Total { get; set; }
}
```

When you call `Save()` on the Order, all modifications to the Order and its LineItems persist together. This is covered in depth in [Aggregates and Entity Graphs](/concepts/aggregates/).

### Business Rules

**Business Rules** encode the logic that governs your domain. They answer questions like:

- Can this order be placed?
- Is this email address already in use?
- Does this discount apply to this customer?

In traditional architectures, business rules scatter across:

- Validation attributes on DTOs
- Service class methods
- Controller actions
- Database constraints
- JavaScript in the UI

This scattering makes rules hard to find, test, and modify consistently.

Neatoo treats rules as first-class citizens. Rules are classes that:

- Declare which properties trigger them
- Execute automatically when those properties change
- Return validation messages or transform data
- Support both synchronous and asynchronous operations

```csharp
public class UniqueEmailRule : AsyncRuleBase<Person>
{
    private readonly IEmailService _emailService;

    public UniqueEmailRule(IEmailService emailService)
        : base(p => p.Email)  // Trigger property
    {
        _emailService = emailService;
    }

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

See [Rules Philosophy](/concepts/rules-philosophy/) and [Rules Engine Reference](/reference/rules/) for complete coverage.

## How Neatoo Maps to DDD Patterns

| DDD Concept | Implementation |
|-------------|---------------|
| **Entity** | `EntityBase<T>` - Full lifecycle tracking, persistence awareness |
| **Value Object** | Plain class with `[Factory]` |
| **Validated Object** | `ValidateBase<T>` - Validation without persistence (wizard steps, criteria) |
| **Aggregate** | Entity graph with parent-child relationships |
| **Aggregate Root** | Top-level `EntityBase<T>` (where `IsChild == false`) |
| **Business Rule** | `RuleBase<T>` or `AsyncRuleBase<T>` |
| **Factory** | Generated from `[Factory]` attribute |
| **Authorization** | `[AuthorizeFactory<T>]` attribute |
| **Repository** | Factory methods combined with EF Core DbContext |

### Class Hierarchy

Neatoo provides base classes in ascending order of capability:

```
Plain class with [Factory]  (Value Objects)

ValidateBase<T>  (Validation without persistence)
    └── EntityBase<T>  (Full persistence tracking)
```

| Approach | Use When |
|----------|----------|
| Plain class + `[Factory]` | Value Objects, simple factory-created objects |
| `ValidateBase<T>` | Objects needing validation but not persistence (wizard steps, search criteria) |
| `EntityBase<T>` | Full domain entities with identity, modification tracking, and persistence |

### Critical Hierarchy Rule

When using Neatoo's entity system, you cannot place an `EntityBase` under an object that does not propagate modification tracking. Valid nesting patterns:

- `EntityBase` containing `EntityBase` (parent-child entities)
- `EntityBase` containing Value Objects (entity containing value objects as leaf nodes)

Invalid pattern:

- Value Object containing `EntityBase` - modification tracking cannot propagate through value objects

## The Neatoo Advantage

Without Neatoo, implementing these DDD patterns requires:

1. **Manual property change notification**: Implementing `INotifyPropertyChanged` for every property
2. **Manual modification tracking**: Recording which properties changed and when
3. **Manual validation orchestration**: Running rules at the right times and aggregating messages
4. **Manual parent-child relationships**: Propagating state changes through the object graph
5. **Manual serialization**: Creating DTOs and mapping code for client-server communication

Neatoo handles all of this through source generators and base class infrastructure. You define your domain model, and the framework provides the plumbing.

### Define Once, Use Everywhere

The same entity class:

- **Binds to UI**: Implements `INotifyPropertyChanged`, exposes validation messages
- **Travels to server**: Serializes through the Neatoo endpoint without DTOs
- **Validates consistently**: Same rules execute on client and server
- **Persists cleanly**: Maps directly to your data model

This is DDD without the ceremony.

## Next Steps

- [Aggregates and Entity Graphs](/concepts/aggregates/) - Deep dive into aggregate structure and behavior
- [Rules Philosophy](/concepts/rules-philosophy/) - Understanding the rules engine approach
- [EntityBase Reference](/reference/entity-base/) - Complete API reference
- [Introduction](/introduction/) - Framework overview and positioning
