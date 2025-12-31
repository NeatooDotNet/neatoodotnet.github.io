---
title: "EntityBase Reference"
layout: single
permalink: /reference/entity-base/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

`EntityBase<T>` is the foundation class for entities in Neatoo. It provides change tracking, validation, persistence awareness, and UI binding capabilities that make building DDD applications straightforward.

## Class Hierarchy

Neatoo's base classes form a hierarchy where each level adds specific capabilities:

```
Base<T>
  └── ValidateBase<T>
        └── EntityBase<T>
```

| Class | Purpose |
|-------|---------|
| `Base<T>` | Property management, parent-child relationships, `INotifyPropertyChanged`, async task tracking |
| `ValidateBase<T>` | Rules engine, validation messages, `IsValid`, `IsBusy` |
| `EntityBase<T>` | Persistence tracking (`IsNew`, `IsModified`, `IsDeleted`), `Save()`, factory integration |

Choose your base class based on what you need:

- **`EntityBase<T>`** - Full entities with persistence tracking (most common)
- **`ValidateBase<T>`** - Validation without persistence (wizard steps, DTOs with rules)
- **`Base<T>`** - Value objects, simple bindable objects

## Class Declaration

Entities must be declared as `partial` classes inheriting from `EntityBase<T>`:

```csharp
internal partial class Person : EntityBase<Person>, IPerson
{
    // ...
}
```

The `partial` keyword is required because the Roslyn Source Generator creates additional partial class files containing:
- Property backing field implementations
- Factory-related infrastructure

## Constructor Pattern

Entity constructors must accept `IEntityBaseServices<T>` and pass it to the base class:

```csharp
public Person(IEntityBaseServices<Person> services,
              IUniqueNameRule uniqueNameRule) : base(services)
{
    RuleManager.AddRule(uniqueNameRule);
}
```

The constructor is where you:
- Inject services via dependency injection
- Add business rules to the `RuleManager`
- Perform any one-time initialization

`IEntityBaseServices<T>` provides framework services to the entity. You never interact with it directly; just pass it to `base()`.

## Partial Properties

Entity properties are declared as `partial` properties:

```csharp
[DisplayName("First Name*")]
[Required(ErrorMessage = "First Name is required")]
public partial string? FirstName { get; set; }

[DisplayName("Last Name*")]
[Required(ErrorMessage = "Last Name is required")]
public partial string? LastName { get; set; }

public partial string? Email { get; set; }
```

The source generator creates full property implementations that:
- Store values in the `PropertyManager`
- Raise `PropertyChanged` events
- Trigger rules when values change
- Track modification state

### Supported Attributes

Properties support standard data annotation attributes:

- `[Required]` - Value must not be null/empty
- `[StringLength]` - Min/max string length
- `[Range]` - Numeric range validation
- `[DisplayName]` - UI-friendly name

## Meta-Properties

Meta-properties provide real-time status information about the entity. They are bindable, meaning UI components automatically update when state changes.

### IsNew

```csharp
public virtual bool IsNew { get; protected set; }
```

`true` when the entity was created via a `[Create]` factory method and has not yet been persisted. The factory's `Save()` method routes to `[Insert]` when `IsNew` is `true`.

```csharp
var factory = serviceProvider.GetRequiredService<IPersonFactory>();
var person = factory.Create();  // person.IsNew == true

await factory.Save(person);     // Calls [Insert], then person.IsNew == false
```

### IsModified

```csharp
public virtual bool IsModified =>
    PropertyManager.IsModified || IsDeleted || IsNew || IsSelfModified;
```

`true` when this entity OR any child entity has been modified. This cascades up the aggregate hierarchy. Use this to determine if any changes need to be saved.

### IsSelfModified

```csharp
public virtual bool IsSelfModified =>
    PropertyManager.IsSelfModified || IsDeleted || IsMarkedModified;
```

`true` when this specific entity has been modified, ignoring child entities. Useful when you need to know if the entity itself changed versus its children.

### IsMarkedModified

```csharp
public virtual bool IsMarkedModified { get; protected set; }
```

`true` when `MarkModified()` has been called explicitly. This is separate from property-change-based modification tracking.

### IsDeleted

```csharp
public virtual bool IsDeleted { get; protected set; }
```

`true` when `Delete()` has been called. The factory's `Save()` method routes to `[Delete]` when `IsDeleted` is `true`.

```csharp
person.Delete();              // person.IsDeleted == true
await factory.Save(person);   // Calls [Delete]
```

### IsSavable

```csharp
public virtual bool IsSavable => IsModified && IsValid && !IsBusy && !IsChild;
```

`true` when the entity can be saved. All conditions must be met:
- `IsModified` - There are changes to persist
- `IsValid` - All validation rules pass
- `!IsBusy` - No async rules are currently executing
- `!IsChild` - This is a root entity (children save through their parent)

Bind save buttons to `IsSavable`:

```razor
<button @onclick="Save" disabled="@(!person.IsSavable)">Save</button>
```

### IsChild

```csharp
public virtual bool IsChild { get; protected set; }
```

`true` when this entity is a child within an aggregate. Child entities cannot be saved directly; they are persisted when the aggregate root is saved.

When an entity is assigned to another entity's property or added to an `EntityListBase`, `MarkAsChild()` is called automatically.

### IsValid

```csharp
public virtual bool IsValid { get; }  // Inherited from ValidateBase<T>
```

`true` when this entity and all child entities pass validation (no rule messages). Cascades up the aggregate hierarchy.

### IsSelfValid

```csharp
public virtual bool IsSelfValid { get; }  // Inherited from ValidateBase<T>
```

`true` when this specific entity passes validation, ignoring child entities.

### IsBusy

```csharp
public virtual bool IsBusy { get; }  // Inherited from Base<T>
```

`true` when async rules or property setters are executing. The UI can show loading indicators based on this property.

### IsSelfBusy

```csharp
public virtual bool IsSelfBusy { get; }  // Inherited from Base<T>
```

`true` when this specific entity has async operations in progress, ignoring children.

## ModifiedProperties

```csharp
public virtual IEnumerable<string> ModifiedProperties => PropertyManager.ModifiedProperties;
```

Returns the names of properties that have been modified since the entity was loaded or saved. Useful for:
- Debugging change tracking
- Building update statements that only include changed columns
- Audit logging

```csharp
foreach (var propName in person.ModifiedProperties)
{
    Console.WriteLine($"Changed: {propName}");
}
```

## PropertyMessages

```csharp
public IReadOnlyCollection<IPropertyMessage> PropertyMessages { get; }
    // Inherited from ValidateBase<T>
```

All validation messages for this entity. Each message includes:
- Property name
- Message text
- Severity (Error, Warning, Information)

```csharp
foreach (var message in person.PropertyMessages)
{
    Console.WriteLine($"{message.PropertyName}: {message.Message}");
}
```

## Parent Property

```csharp
public IBase? Parent { get; }  // Inherited from Base<T>
```

Reference to the parent entity in the aggregate hierarchy. Set automatically when:
- An entity is assigned to another entity's property
- An entity is added to an `EntityListBase`

Only the aggregate root should have `Parent == null`.

```csharp
// PersonPhone.Parent will be set to the Person instance
person.PersonPhoneList.Add(newPhone);
// newPhone.Parent == person (not the list!)
```

Note: For items in an `EntityListBase`, `Parent` points to the list's parent, not the list itself.

## RuleManager

```csharp
protected IRuleManager RuleManager { get; }  // Inherited from ValidateBase<T>
```

Manages business rules attached to the entity. Use in the constructor to add rules:

```csharp
public Person(IEntityBaseServices<Person> services,
              IUniqueNameRule uniqueNameRule,
              IFullNameRule fullNameRule) : base(services)
{
    RuleManager.AddRule(uniqueNameRule);
    RuleManager.AddRule(fullNameRule);
}
```

### Adding Rules

Rules are added via `AddRule()`:

```csharp
RuleManager.AddRule(injectedRule);
```

For inline validation, use the fluent API:

```csharp
// Simple validation
RuleManager.AddValidation(
    nameof(Email),
    (Person p) => p.Email?.Contains("@") == true
        ? RuleMessage.None
        : RuleMessage.Error("Invalid email format"));

// Async validation
RuleManager.AddValidationAsync(
    nameof(Username),
    async (Person p, CancellationToken ct) =>
    {
        var exists = await CheckUsernameExists(p.Username, ct);
        return exists
            ? RuleMessage.Error("Username already taken")
            : RuleMessage.None;
    });

// Action (transformation without message)
RuleManager.AddAction(
    nameof(FirstName),
    (Person p) => p.FullName = $"{p.FirstName} {p.LastName}");
```

## Methods

### Save()

```csharp
public virtual async Task<IEntityBase> Save()
```

Persists the entity through its factory. Routes to `[Insert]`, `[Update]`, or `[Delete]` based on entity state.

Throws `SaveOperationException` if:
- `IsChild` is `true` (children save through parent)
- `IsValid` is `false` (validation failed)
- Not modified (nothing to save)
- `IsBusy` is `true` (async operations pending)
- No factory method available

```csharp
try
{
    var saved = await person.Save();
}
catch (SaveOperationException ex)
{
    // Handle save failure
    Console.WriteLine(ex.Reason);
}
```

### Delete()

```csharp
public void Delete()
```

Marks the entity for deletion. Sets `IsDeleted = true`. The actual deletion occurs when `Save()` is called.

```csharp
person.Delete();              // Mark for deletion
await person.Save();          // Execute [Delete] factory method
```

### UnDelete()

```csharp
public void UnDelete()
```

Reverses a `Delete()` call. Sets `IsDeleted = false`. Useful when the user cancels a delete operation.

```csharp
person.Delete();
// User cancels
person.UnDelete();
```

### MarkModified()

```csharp
protected virtual void MarkModified()
```

Explicitly marks the entity as modified, even if no properties changed. Useful when external factors require a save.

### MarkAsChild()

```csharp
protected virtual void MarkAsChild()
```

Called automatically when an entity is assigned as a child. You rarely need to call this directly.

### MarkUnmodified()

```csharp
protected virtual void MarkUnmodified()
```

Resets modification tracking. Called automatically after successful save operations.

### RunRules()

```csharp
public Task RunRules(string propertyName, CancellationToken? token = null)
public Task RunRules(RunRulesFlag flags, CancellationToken? token = null)
```

Manually executes rules. Normally rules run automatically when properties change. Manual execution is useful for:
- Initial validation before display
- Re-running rules after external data changes

```csharp
await person.RunRules(RunRulesFlag.All);
```

### WaitForTasks()

```csharp
public Task WaitForTasks()  // Inherited from Base<T>
```

Waits for all pending async operations (rules, property setters) to complete. Useful before checking validation state:

```csharp
person.Email = "test@example.com";  // May trigger async rule
await person.WaitForTasks();        // Wait for validation to complete
if (person.IsValid) { ... }
```

### PauseAllActions()

```csharp
public virtual IDisposable PauseAllActions()
```

Temporarily suppresses property change events and rule execution. Useful for bulk updates:

```csharp
using (person.PauseAllActions())
{
    // These won't trigger individual property changes or rules
    person.FirstName = "John";
    person.LastName = "Doe";
    person.Email = "john@example.com";
}
// Events fire and rules run after dispose
```

Returns an `IDisposable` for use with `using` statement.

## Events

### PropertyChanged

```csharp
public event PropertyChangedEventHandler? PropertyChanged;
    // Inherited from Base<T> via INotifyPropertyChanged
```

Standard .NET property change notification. Fires when any property or meta-property changes.

### NeatooPropertyChanged

```csharp
public event NeatooPropertyChangedEventHandler? NeatooPropertyChanged;
```

Enhanced property change event with additional context. The `NeatooPropertyChangedEventArgs` includes:
- Property name
- Source entity
- Parent chain (breadcrumbs)

Events propagate up the aggregate hierarchy. Override `ChildNeatooPropertyChanged` to react to child changes:

```csharp
protected override Task ChildNeatooPropertyChanged(NeatooPropertyChangedEventArgs eventArgs)
{
    // React to any child property change
    return base.ChildNeatooPropertyChanged(eventArgs);
}
```

## PropertyManager

```csharp
protected IEntityPropertyManager PropertyManager { get; }
```

Provides access to individual property objects and their metadata.

### Accessing Property Objects

Access properties by name using the indexer:

```csharp
IEntityProperty firstNameProp = person[nameof(person.FirstName)];
```

Or through `PropertyManager`:

```csharp
IEntityProperty prop = person.PropertyManager["FirstName"];
```

### Property-Level Meta-Properties

Each property is a class with its own meta-properties:

```csharp
IEntityProperty prop = person[nameof(person.FirstName)];

bool isModified = prop.IsModified;  // Has this property changed?
bool isValid = prop.IsValid;        // Does this property pass validation?
bool isBusy = prop.IsBusy;          // Is an async rule running for this property?
```

This enables fine-grained UI binding:

```razor
<input @bind="person.FirstName"
       class="@(person[nameof(person.FirstName)].IsValid ? "" : "error")" />
```

## Complete Example

Here is a complete entity definition using `EntityBase<T>`:

```csharp
internal partial class Person : EntityBase<Person>, IPerson
{
    public Person(IEntityBaseServices<Person> services,
                  IUniqueNameRule uniqueNameRule) : base(services)
    {
        RuleManager.AddRule(uniqueNameRule);

        // Inline validation
        RuleManager.AddValidation(
            nameof(Email),
            (Person p) => string.IsNullOrEmpty(p.Email) || p.Email.Contains("@")
                ? RuleMessage.None
                : RuleMessage.Error("Invalid email format"));
    }

    // Identity
    public partial Guid? Id { get; set; }

    // Properties with validation
    [Required(ErrorMessage = "First Name is required")]
    public partial string? FirstName { get; set; }

    [Required(ErrorMessage = "Last Name is required")]
    public partial string? LastName { get; set; }

    public partial string? Email { get; set; }
    public partial string? Notes { get; set; }

    // Child collection
    public partial IPersonPhoneList? PersonPhoneList { get; set; }

    // Factory methods
    [Create]
    public void Create([Service] IPersonPhoneListFactory phoneListFactory)
    {
        PersonPhoneList = phoneListFactory.Create();
    }

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

        // Load properties without triggering rules
        LoadProperty(nameof(Id), entity.Id);
        LoadProperty(nameof(FirstName), entity.FirstName);
        LoadProperty(nameof(LastName), entity.LastName);
        LoadProperty(nameof(Email), entity.Email);
        LoadProperty(nameof(Notes), entity.Notes);

        PersonPhoneList = await phoneListFactory.Fetch(entity.Phones);
        return true;
    }

    [Remote]
    [Insert]
    public async Task Insert([Service] IPersonDbContext dbContext)
    {
        Id = Guid.NewGuid();

        var entity = new PersonEntity
        {
            Id = Id.Value,
            FirstName = FirstName,
            LastName = LastName,
            Email = Email,
            Notes = Notes
        };

        dbContext.Persons.Add(entity);
        await dbContext.SaveChangesAsync();
    }

    [Remote]
    [Update]
    public async Task Update([Service] IPersonDbContext dbContext)
    {
        var entity = await dbContext.Persons.FindAsync(Id);

        // Update only modified properties
        if (this[nameof(FirstName)].IsModified)
            entity.FirstName = FirstName;
        if (this[nameof(LastName)].IsModified)
            entity.LastName = LastName;
        if (this[nameof(Email)].IsModified)
            entity.Email = Email;
        if (this[nameof(Notes)].IsModified)
            entity.Notes = Notes;

        await dbContext.SaveChangesAsync();
    }

    [Remote]
    [Delete]
    public async Task Delete([Service] IPersonDbContext dbContext)
    {
        var entity = await dbContext.Persons.FindAsync(Id);
        if (entity != null)
        {
            dbContext.Persons.Remove(entity);
            await dbContext.SaveChangesAsync();
        }
    }
}
```

## Related Topics

- [Entity Overview](/overview/entity/) - Conceptual introduction
- [Rule Overview](/overview/rule/) - Business rules and validation
- [Factory Overview](/overview/factory/) - Entity lifecycle and persistence
- [Person Example](/example/person/) - Complete working example
