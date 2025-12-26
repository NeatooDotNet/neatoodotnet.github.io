---
title: "EntityListBase<I> Reference"
layout: single
permalink: /reference/entity-list-base/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

`EntityListBase<I>` manages collections of child entities within an aggregate. It provides automatic child tracking, deleted item management, validation aggregation, and integration with the factory system for persistence operations.

## Class Hierarchy

Neatoo's collection classes form a hierarchy where each level adds capabilities:

```
ListBase<I>
  └── ValidateListBase<I>
        └── EntityListBase<I>
```

| Class | Purpose |
|-------|---------|
| `ListBase<I>` | Observable collection, parent-child relationships, `INotifyPropertyChanged` |
| `ValidateListBase<I>` | Validation aggregation across items, `IsValid`, `PropertyMessages` |
| `EntityListBase<I>` | Persistence awareness, `DeletedList`, modification tracking, factory integration |

Choose your base class based on requirements:

- **`EntityListBase<I>`** - Collections of persistable entities (most common)
- **`ValidateListBase<I>`** - Collections needing validation without persistence
- **`ListBase<I>`** - Simple observable collections

## Defining a Collection Interface

Collection interfaces follow the same pattern as entity interfaces:

```csharp
public partial interface IPersonPhoneList : IEntityListBase<IPersonPhone>
{
    Task<IPersonPhone> AddPhoneNumber();
    void RemovePhoneNumber(IPersonPhone phone);
}
```

The interface:

- Inherits from `IEntityListBase<I>` for full functionality
- Declares domain-specific methods for adding and removing items
- Uses the item's interface type (`IPersonPhone`), not the concrete type

## Defining the Collection Class

```csharp
[Factory]
internal partial class PersonPhoneList
    : EntityListBase<IPersonPhone>, IPersonPhoneList
{
    private readonly IPersonPhoneFactory _phoneFactory;

    public PersonPhoneList(
        IEntityListBaseServices<IPersonPhone> services,
        IPersonPhoneFactory phoneFactory) : base(services)
    {
        _phoneFactory = phoneFactory;
    }

    public async Task<IPersonPhone> AddPhoneNumber()
    {
        var phone = await _phoneFactory.Create();
        Add(phone);
        return phone;
    }

    public void RemovePhoneNumber(IPersonPhone phone)
    {
        Remove(phone);
    }
}
```

Key elements:

- Class is `partial` and `internal`
- Inherits from `EntityListBase<I>` with the item interface type
- Constructor accepts `IEntityListBaseServices<I>` and passes to base
- Injects the item factory for creating new items
- Provides domain-specific methods rather than exposing raw `Add`/`Remove`

## Constructor Pattern

The constructor must accept `IEntityListBaseServices<I>`:

```csharp
public PersonPhoneList(
    IEntityListBaseServices<I> services,
    IPersonPhoneFactory phoneFactory,
    IValidationService validationService) : base(services)
{
    _phoneFactory = phoneFactory;
    _validationService = validationService;
}
```

Inject any dependencies needed for:

- Creating new items (item factory)
- Validation services
- Other domain services

## Adding Items

### The Add() Method

When you call `Add()`, the framework automatically:

1. Checks if the item was previously deleted (undeletes if so)
2. Sets the item's `Parent` to the list's parent (not the list)
3. Calls `MarkAsChild()` on the item
4. Raises `CollectionChanged` events

```csharp
public async Task<IPersonPhone> AddPhoneNumber()
{
    var phone = await _phoneFactory.Create();
    Add(phone);
    // phone.IsChild == true
    // phone.Parent == this.Parent (the Person, not the list)
    return phone;
}
```

### Parent Assignment

Items in a list have their `Parent` set to the **list's parent**, not the list itself:

```csharp
var person = await personFactory.Create();
var phone = await person.PersonPhoneList.AddPhoneNumber();

// phone.Parent == person (not person.PersonPhoneList)
```

This allows items to access their aggregate root directly:

```csharp
internal partial class PersonPhone : EntityBase<PersonPhone>, IPersonPhone
{
    public IPerson? ParentPerson => Parent as IPerson;
}
```

### Marking Existing Items as Modified

When an item already exists in the list and you add it again, the item is marked as modified. This handles scenarios like re-adding an item that was removed and re-added.

## Removing Items

### New Items vs. Existing Items

The removal behavior differs based on whether the item was persisted:

| Item State | Removal Behavior |
|------------|-----------------|
| `IsNew == true` | Removed from list completely |
| `IsNew == false` | Marked deleted, moved to `DeletedList` |

```csharp
public void RemovePhoneNumber(IPersonPhone phone)
{
    Remove(phone);

    if (phone.IsNew)
    {
        // Item was never persisted, simply removed
    }
    else
    {
        // Item moved to DeletedList for processing during Update
    }
}
```

### The DeletedList Property

```csharp
public IEnumerable<IEntityBase> DeletedList { get; }
```

Contains items that were removed from the collection but need to be deleted from the database. The list persists until `FactoryComplete()` is called after a successful save.

### Processing Deleted Items

In your `[Update]` method, iterate the `DeletedList`:

```csharp
[Remote]
[Update]
public async Task Update(Guid parentId, [Service] IPersonDbContext dbContext)
{
    // Process deletions
    foreach (var deleted in DeletedList.Cast<IPersonPhone>())
    {
        var entity = await dbContext.PersonPhones.FindAsync(deleted.Id);
        if (entity != null)
        {
            dbContext.PersonPhones.Remove(entity);
        }
    }

    // Process remaining items (inserts and updates)
    foreach (var phone in this)
    {
        await _phoneFactory.Save(phone, parentId);
    }

    await dbContext.SaveChangesAsync();
}
```

### FactoryComplete()

After a successful save operation, the framework calls `FactoryComplete()` which:

1. Clears the `DeletedList`
2. Resets modification tracking

## Collection Meta-Properties

Lists expose aggregate state information about their items.

### IsModified

```csharp
public bool IsModified { get; }
```

Returns `true` if:

- Any item in the list has `IsModified == true`
- Any item is in the `DeletedList`

```csharp
var phones = person.PersonPhoneList;
phones[0].PhoneNumber = "555-1234";
// phones.IsModified == true (item is modified)
```

### IsValid

```csharp
public bool IsValid { get; }
```

Returns `true` only if all items in the list pass validation. A single invalid item makes the entire list invalid.

```csharp
var phones = person.PersonPhoneList;
await phones.AddPhoneNumber();
// New phone has empty required fields
// phones.IsValid == false
// person.IsValid == false (child collection is invalid)
```

### PropertyMessages

```csharp
public IEnumerable<IPropertyMessage> PropertyMessages { get; }
```

Aggregates validation messages from all items in the collection:

```csharp
foreach (var message in phones.PropertyMessages)
{
    Console.WriteLine($"{message.PropertyName}: {message.Message}");
}
```

### IsBusy

```csharp
public bool IsBusy { get; }
```

Returns `true` if any item has async operations in progress.

### Other Meta-Properties

These properties exist for interface compatibility but return constant values:

| Property | Value | Reason |
|----------|-------|--------|
| `IsSelfModified` | `false` | Lists delegate to items |
| `IsMarkedModified` | `false` | Lists delegate to items |
| `IsSavable` | `false` | Lists save through parent |
| `IsNew` | `false` | Lists don't have identity |
| `IsDeleted` | `false` | Lists persist with parent |
| `IsChild` | `false` | Lists are infrastructure |

## Factory Operations

Collections use factory methods for persistence, typically decorated with `[Fetch]` and `[Update]`.

### Fetch Operation

```csharp
[Factory]
internal partial class PersonPhoneList
    : EntityListBase<IPersonPhone>, IPersonPhoneList
{
    [Fetch]
    public async Task Fetch(
        IEnumerable<PersonPhoneEntity> phoneEntities,
        [Service] IPersonPhoneFactory phoneFactory)
    {
        foreach (var entity in phoneEntities)
        {
            var phone = await phoneFactory.Fetch(entity);
            Add(phone);
        }
    }
}
```

The `Fetch` method:

1. Receives data from the parent's fetch operation
2. Creates each item through the item factory
3. Adds items to the list

### Update Operation

```csharp
[Remote]
[Update]
public async Task Update(
    Guid parentId,
    [Service] IPersonDbContext dbContext,
    [Service] IPersonPhoneFactory phoneFactory)
{
    // Handle deletions
    foreach (var deleted in DeletedList.Cast<IPersonPhone>())
    {
        var entity = await dbContext.PersonPhones.FindAsync(deleted.Id);
        if (entity != null)
        {
            dbContext.PersonPhones.Remove(entity);
        }
    }

    // Handle inserts and updates
    foreach (var phone in this)
    {
        if (phone.IsNew)
        {
            // Insert
            var entity = new PersonPhoneEntity { PersonId = parentId };
            phone.MapTo(entity);
            dbContext.PersonPhones.Add(entity);
        }
        else if (phone.IsModified)
        {
            // Update
            var entity = await dbContext.PersonPhones.FindAsync(phone.Id);
            phone.MapModifiedTo(entity);
        }
    }

    await dbContext.SaveChangesAsync();
}
```

The `Update` method handles three scenarios:

1. **Deletions**: Items in `DeletedList` are removed from the database
2. **Inserts**: Items with `IsNew == true` are added to the database
3. **Updates**: Items with `IsModified == true` have changes persisted

### Calling from Parent

The parent entity's factory methods call the list's factory:

```csharp
// In Person.cs
[Remote]
[Fetch]
public async Task<bool> Fetch(
    [Service] IPersonDbContext dbContext,
    [Service] IPersonPhoneListFactory phoneListFactory)
{
    var entity = await dbContext.Persons
        .Include(p => p.Phones)
        .FirstOrDefaultAsync(p => p.Id == Id);

    if (entity == null) return false;

    MapFrom(entity);
    PersonPhoneList = await phoneListFactory.Fetch(entity.Phones);
    return true;
}

[Remote]
[Update]
public async Task Update(
    [Service] IPersonDbContext dbContext,
    [Service] IPersonPhoneListFactory phoneListFactory)
{
    var entity = await dbContext.Persons.FindAsync(Id);
    MapModifiedTo(entity);

    // Delegate to list's Update method
    await phoneListFactory.Save(PersonPhoneList, Id);

    await dbContext.SaveChangesAsync();
}
```

## Cross-Item Validation

Collections often need to validate relationships between items.

### Handling Property Changes

Override `HandleNeatooPropertyChanged` to react when any item changes:

```csharp
protected override async Task HandleNeatooPropertyChanged(
    NeatooPropertyChangedEventArgs eventArgs)
{
    await base.HandleNeatooPropertyChanged(eventArgs);

    // Re-run uniqueness validation when phone properties change
    if (eventArgs.PropertyName == nameof(IPersonPhone.PhoneNumber) ||
        eventArgs.PropertyName == nameof(IPersonPhone.PhoneType))
    {
        // Re-validate sibling items for uniqueness
        foreach (var phone in this.Where(p => p != eventArgs.Source))
        {
            await phone.RunRules(nameof(IPersonPhone.PhoneNumber));
        }
    }
}
```

### Uniqueness Rules

Items can reference their siblings through the parent:

```csharp
public class UniquePhoneNumberRule : RuleBase<IPersonPhone>
{
    public UniquePhoneNumberRule()
        : base(p => p.PhoneNumber, p => p.PhoneType) { }

    protected override IRuleMessages Execute(IPersonPhone target)
    {
        var parent = target.ParentPerson;
        if (parent?.PersonPhoneList == null)
            return None;

        var isDuplicate = parent.PersonPhoneList
            .Where(p => p != target)
            .Any(p => p.PhoneNumber == target.PhoneNumber);

        return isDuplicate
            ? (nameof(target.PhoneNumber),
               "Phone number must be unique").AsRuleMessages()
            : None;
    }
}
```

## Complete Example: PersonPhoneList

Here is a complete implementation from the Neatoo example project:

```csharp
public partial interface IPersonPhoneList : IEntityListBase<IPersonPhone>
{
    Task<IPersonPhone> AddPhoneNumber();
    void RemovePhoneNumber(IPersonPhone personPhone);
}

[Factory]
internal partial class PersonPhoneList
    : EntityListBase<IPersonPhone>, IPersonPhoneList
{
    private readonly IPersonPhoneFactory _personPhoneFactory;

    public PersonPhoneList(
        IEntityListBaseServices<IPersonPhone> services,
        IPersonPhoneFactory personPhoneFactory) : base(services)
    {
        _personPhoneFactory = personPhoneFactory;
    }

    public async Task<IPersonPhone> AddPhoneNumber()
    {
        var personPhone = await _personPhoneFactory.Create();
        Add(personPhone);
        return personPhone;
    }

    public void RemovePhoneNumber(IPersonPhone personPhone)
    {
        Remove(personPhone);
    }

    protected override async Task HandleNeatooPropertyChanged(
        NeatooPropertyChangedEventArgs eventArgs)
    {
        await base.HandleNeatooPropertyChanged(eventArgs);

        // Trigger sibling validation for uniqueness checks
        if (eventArgs.PropertyName == nameof(IPersonPhone.PhoneType) ||
            eventArgs.PropertyName == nameof(IPersonPhone.PhoneNumber))
        {
            foreach (var sibling in this.Where(p => p != eventArgs.Source))
            {
                await sibling.RunRules(eventArgs.PropertyName);
            }
        }
    }

    [Fetch]
    public async Task Fetch(
        IEnumerable<PersonPhoneEntity> phoneEntities,
        [Service] IPersonPhoneFactory phoneFactory)
    {
        foreach (var entity in phoneEntities)
        {
            var phone = await phoneFactory.Fetch(entity);
            Add(phone);
        }
    }

    [Remote]
    [Update]
    public async Task Update(
        Guid personId,
        [Service] IPersonDbContext dbContext)
    {
        // Process deletions
        foreach (var phone in DeletedList.Cast<IPersonPhone>())
        {
            var entity = await dbContext.PersonPhones.FindAsync(phone.Id);
            if (entity != null)
            {
                dbContext.PersonPhones.Remove(entity);
            }
        }

        // Process inserts and updates
        foreach (var phone in this)
        {
            if (phone.IsNew)
            {
                phone.Id = Guid.NewGuid();
                var entity = new PersonPhoneEntity { PersonId = personId };
                phone.MapTo(entity);
                dbContext.PersonPhones.Add(entity);
            }
            else if (phone.IsModified)
            {
                var entity = await dbContext.PersonPhones.FindAsync(phone.Id);
                phone.MapModifiedTo(entity);
            }
        }

        await dbContext.SaveChangesAsync();
    }
}
```

## Common Patterns

### Domain-Specific Add Methods

Instead of exposing the generic `Add()`, provide meaningful methods:

```csharp
public async Task<IOrderLine> AddLineItem(IProduct product, int quantity)
{
    var line = await _lineFactory.Create();
    line.ProductId = product.Id;
    line.ProductName = product.Name;
    line.UnitPrice = product.Price;
    line.Quantity = quantity;
    Add(line);
    return line;
}
```

### Filtering and Querying

Lists are enumerable; use LINQ for queries:

```csharp
public decimal CalculateTotal()
{
    return this.Sum(line => line.UnitPrice * line.Quantity);
}

public IEnumerable<IOrderLine> GetDiscountedLines()
{
    return this.Where(line => line.DiscountPercent > 0);
}
```

### Maximum Item Limits

Enforce business rules on collection size:

```csharp
public async Task<IPersonPhone> AddPhoneNumber()
{
    if (Count >= 5)
        throw new InvalidOperationException("Maximum 5 phone numbers allowed");

    var phone = await _phoneFactory.Create();
    Add(phone);
    return phone;
}
```

## Troubleshooting

### Deleted Items Not Processed

If items remain after delete operations:

1. Verify `DeletedList` is processed in your `[Update]` method
2. Ensure `SaveChangesAsync()` is called
3. Check that you are iterating `DeletedList`, not the main collection

### Parent is Null

If `item.Parent` is null:

1. Verify the list itself has a parent assigned
2. Check that items are added through the list's `Add()` method
3. Ensure the list is assigned to an entity property (not created standalone)

### Validation Not Running on Siblings

If cross-item validation does not trigger:

1. Override `HandleNeatooPropertyChanged`
2. Call `RunRules()` on sibling items
3. Ensure the rule's trigger properties include the relevant fields

## Related Topics

- [EntityBase Reference](/reference/entity-base/) - Entity documentation
- [Aggregates and Entity Graphs](/concepts/aggregates/) - Parent-child patterns
- [Rules Engine Reference](/reference/rules/) - Validation rules
- [Factory Operations](/overview/factory/) - Factory patterns
