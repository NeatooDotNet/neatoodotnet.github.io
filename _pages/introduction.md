---
title: "Introduction to Neatoo"
layout: single
permalink: /introduction/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

Neatoo is a Domain-Driven Design (DDD) framework for .NET that enables you to build rich, interactive applications with Blazor and WPF. It provides the infrastructure for creating bindable, serializable Aggregate Entity Graphs where business logic is defined once and executed everywhere—on both client and server.

## The Problem with Traditional Architectures

Most enterprise applications follow a traditional n-tier architecture that separates presentation, business logic, and data access into distinct layers. While this sounds clean in theory, the reality often looks different:

### The DTO Trap

In a typical 3-tier web application, you end up maintaining multiple representations of the same domain concept:

- **EF Core Entity** - Your database model
- **Domain Entity** - Your business logic holder (often skipped)
- **DTO** - For API transport
- **ViewModel** - For UI binding

Each representation requires mapping code. Business rules get scattered across controllers, services, and sometimes even the UI. When requirements change, you update logic in multiple places—hoping you caught them all.

### Anemic Domain Models

Under pressure, teams often collapse to an "Anemic Domain Model" where entities become simple data containers with no behavior. Business logic migrates to service classes, leaving you with procedural code dressed in object-oriented clothing. The domain model becomes a glorified ORM mapping rather than a true representation of your business.

### The Validation Problem

Client-side validation provides responsive user experience but can be bypassed. Server-side validation is authoritative but creates friction. Teams duplicate validation logic—once in JavaScript/Blazor for immediate feedback, once in C# for security—and inevitably these drift apart.

## How Neatoo Solves These Problems

Neatoo takes a fundamentally different approach: **define once, use everywhere**.

### One Entity, Complete Behavior

Your Neatoo entity is your domain model, your API contract, and your UI binding target—all in one:

```csharp
internal partial class Person : EntityBase<Person>, IPerson
{
    public Person(IEntityBaseServices<Person> services,
                  IUniqueNameRule uniqueNameRule) : base(services)
    {
        RuleManager.AddRule(uniqueNameRule);
    }

    [Required(ErrorMessage = "First Name is required")]
    public partial string? FirstName { get; set; }

    [Required(ErrorMessage = "Last Name is required")]
    public partial string? LastName { get; set; }
}
```

This single class definition:
- Binds directly to Blazor components
- Serializes between client and server
- Validates on both tiers automatically
- Tracks its own modification state
- Knows how to persist itself

### No DTOs, No Mapping Layers

With Neatoo, your entity travels between client and server intact. The framework handles serialization transparently through a single `/api/neatoo` endpoint. You write zero mapping code between layers.

### Rules Execute Everywhere

Business rules defined on your entities execute on both client and server:

```csharp
public class UniqueNameRule : AsyncRuleBase<Person>
{
    public UniqueNameRule() : base(nameof(Person.FirstName), nameof(Person.LastName)) { }

    protected override async Task<RuleMessage> Execute(Person target, CancellationToken token)
    {
        // This validation runs on client for immediate feedback
        // AND on server for authoritative enforcement
        var exists = await CheckNameExists(target.FirstName, target.LastName);
        return exists
            ? RuleMessage.Error("A person with this name already exists")
            : RuleMessage.None;
    }
}
```

### Compile-Time Safety via Source Generators

Neatoo uses Roslyn Source Generators to create entity-specific factories at compile time. No runtime reflection. No magic strings that fail at runtime. When you change your entity, the compiler catches mismatches immediately.

## Core Components

Neatoo's architecture centers on three core concepts that work together:

### Entities

Entities are your domain objects. They inherit from `EntityBase<T>` and provide:

- **Bindable Properties**: Automatically implement `INotifyPropertyChanged`
- **Meta-Properties**: `IsNew`, `IsModified`, `IsDeleted`, `IsValid`, `IsSavable`, `IsBusy`
- **Rules Engine**: Attach validation and transformation rules
- **Data Mapping**: Declare how entities map to/from your persistence model

See [Entity Overview](/overview/entity/) for details.

### Rules

Rules encapsulate business logic as first-class citizens:

- **Trigger-Based Execution**: Rules fire when specified properties change
- **Async Support**: Seamlessly handle rules that require I/O
- **Cascading**: Rules can trigger other rules
- **Unit Testable**: Rules are injectable classes you can test in isolation

See [Rule Overview](/overview/rule/) for details.

### Factories

Factories manage entity lifecycle through the 3-tier architecture:

- **Create**: Instantiate new entities (client-side)
- **Fetch**: Load existing entities from persistence
- **Save**: Route to Insert, Update, or Delete based on entity state
- **Authorization**: Check permissions before operations execute

See [Factory Overview](/overview/factory/) for details.

## The "Define Once, Use Everywhere" Philosophy

This is Neatoo's core principle. When you define an entity with its properties, rules, and factory methods, that definition serves every layer of your application:

| Traditional Approach | Neatoo Approach |
|---------------------|-----------------|
| DTO for API transport | Entity IS the transport |
| ViewModel for UI binding | Entity binds directly |
| Service for business logic | Rules live on the entity |
| Separate client/server validation | Same rules run both places |
| Multiple mapping layers | Zero mapping code |

The result: less code, fewer bugs, faster development, and a codebase that actually represents your domain.

## When to Use Neatoo

Neatoo excels when:

- **You are building with Blazor or WPF** - These technologies support the rich data binding Neatoo provides
- **You have complex business behavior** - Rules engine handles sophisticated validation and transformation
- **You want a true domain model** - Not just an ORM wrapper, but entities with real behavior
- **You need consistent client/server logic** - Same rules, same validation, everywhere
- **You value compile-time safety** - Source generators catch errors before runtime

## When NOT to Use Neatoo

Neatoo may not be the best fit when:

- **High transaction throughput is critical** - The rich object model adds overhead
- **Many users edit the same data concurrently** - Neatoo optimizes for user-session workflows
- **You need multiple UI technologies** - REST APIs for diverse clients may be simpler
- **Business logic is minimal** - Simple CRUD apps may not need the framework power

## Next Steps

Ready to get started?

1. [Create a Simple Application](/gettingstarted/createsimpleapplication) - Video walkthrough
2. [Client Setup](/setup/client/) - Configure your Blazor client
3. [Server Setup](/setup/server/) - Configure your ASP.NET backend
4. [Person Example](/example/person/) - Complete working example with source code

For deeper understanding of the concepts:

- [Entity Overview](/overview/entity/) - How entities work
- [Rule Overview](/overview/rule/) - Business rules and validation
- [Factory Overview](/overview/factory/) - Lifecycle and persistence
- [Authorization Overview](/overview/authorization/) - Securing operations
