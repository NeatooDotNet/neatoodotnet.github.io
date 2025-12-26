---
title: "Properties and Meta-Properties"
layout: single
permalink: /concepts/properties/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

Neatoo's property system provides far more than simple value storage. Through Roslyn Source Generators, partial properties become sophisticated managed properties with automatic change tracking, validation state, busy indicators, and seamless UI binding. This page explains how the property system works and documents every meta-property available at both entity and property levels.

## Why Partial Properties?

In traditional .NET development, you implement `INotifyPropertyChanged` by writing verbose property setters:

```csharp
// Traditional approach - repetitive and error-prone
private string? _firstName;
public string? FirstName
{
    get => _firstName;
    set
    {
        if (_firstName != value)
        {
            _firstName = value;
            OnPropertyChanged(nameof(FirstName));
            OnPropertyChanged(nameof(IsModified));
            // Don't forget validation...
            // Don't forget busy state tracking...
            // Don't forget to trigger rules...
        }
    }
}
```

Multiply this by dozens of properties across multiple entities, and you have a maintenance burden and a breeding ground for bugs. Neatoo eliminates this entirely.

### The Partial Property Solution

Neatoo entities declare properties using the `partial` keyword:

```csharp
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    [Required]
    [DisplayName("First Name")]
    public partial string? FirstName { get; set; }

    public partial string? LastName { get; set; }

    public partial string? Email { get; set; }
}
```

The Roslyn Source Generator creates the actual implementation:

```csharp
// Generated code (simplified for illustration)
public partial string? FirstName
{
    get => Getter<string?>();
    set => Setter(value);
}
```

The generated `Getter<T>()` and `Setter(value)` methods handle:

- **Value storage** in the PropertyManager
- **Change detection** comparing old and new values
- **PropertyChanged events** for UI binding
- **Modification tracking** for persistence
- **Rule triggering** when values change
- **Busy state management** for async operations
- **Validation execution** from data annotations

You write simple property declarations; the framework provides enterprise-grade infrastructure.

## What the Source Generator Creates

For each partial property on a Neatoo class, the source generator produces:

### Property Implementation

```csharp
public partial string? FirstName
{
    get => Getter<string?>();
    set => Setter(value);
}
```

### Property Wrapper Object

Internally, each property is backed by an `IEntityProperty` (or `IValidateProperty` or `IProperty` depending on the base class) that tracks:

- Current value
- Original value (for modification detection)
- Validation messages
- Busy state (async operations in progress)
- Display name for UI

### Accessor via Indexer

You can access the property wrapper directly:

```csharp
// Access property wrapper by name
IEntityProperty prop = person[nameof(person.FirstName)];

// Read meta-information
bool modified = prop.IsModified;
bool valid = prop.IsValid;
bool busy = prop.IsBusy;
```

## Entity-Level Meta-Properties

Entity-level meta-properties provide aggregate state information about the entire entity and its children. These properties automatically update as the entity graph changes and raise `PropertyChanged` events for UI binding.

### IsValid

```csharp
public virtual bool IsValid { get; }
```

Returns `true` when this entity **and all child entities** pass validation. This cascades through the entire aggregate hierarchy.

```csharp
var order = await orderFactory.Create();
order.CustomerName = "John";        // order.IsSelfValid might be true

var line = order.Lines.AddLine();   // New line has required fields
// order.Lines[0].IsValid == false  (empty required fields)
// order.IsValid == false           (child is invalid)
```

**Use case:** Bind save buttons to `IsValid` to prevent invalid saves:

```razor
<button disabled="@(!order.IsValid)">Save</button>
```

### IsSelfValid

```csharp
public virtual bool IsSelfValid { get; }
```

Returns `true` when **this entity's own properties** pass validation, ignoring child entities.

```csharp
// Parent passes its own rules, but child fails
order.IsSelfValid == true
order.Lines[0].IsSelfValid == false
order.IsValid == false  // Aggregate is still invalid
```

**Use case:** Show validation indicators specific to each entity level:

```razor
<div class="@(order.IsSelfValid ? "" : "has-errors")">
    <!-- Order header fields -->
</div>
```

### IsBusy

```csharp
public virtual bool IsBusy { get; }
```

Returns `true` when async operations are executing on this entity **or any child entity**. This includes:

- Async validation rules running
- Async property setters in progress
- Child entities with pending operations

```csharp
person.Email = "test@example.com";  // Triggers async uniqueness check
// person.IsBusy == true

await person.WaitForTasks();
// person.IsBusy == false
```

**Use case:** Show loading indicators and prevent actions during async operations:

```razor
<button disabled="@(person.IsBusy)" @onclick="Save">
    @(person.IsBusy ? "Validating..." : "Save")
</button>
```

### IsSelfBusy

```csharp
public virtual bool IsSelfBusy { get; }
```

Returns `true` when **this specific entity** has async operations in progress, ignoring children.

**Use case:** Show per-entity loading state in aggregate editors.

### IsNew

```csharp
public virtual bool IsNew { get; protected set; }
```

Returns `true` when the entity was created via a `[Create]` factory method and has not yet been persisted.

```csharp
var person = personFactory.Create();
// person.IsNew == true

await personFactory.Save(person);   // Calls [Insert]
// person.IsNew == false
```

The framework uses `IsNew` to route `Save()` calls:

| IsNew | IsDeleted | Operation Called |
|-------|-----------|------------------|
| `true` | `false` | `[Insert]` |
| `false` | `false` | `[Update]` |
| any | `true` | `[Delete]` |

### IsModified

```csharp
public virtual bool IsModified { get; }
```

Returns `true` when this entity **or any child entity** has been modified. Specifically, `IsModified` is `true` when:

- Any property value has changed since load/save
- `IsNew` is `true`
- `IsDeleted` is `true`
- Any child entity is modified
- `MarkModified()` was called

```csharp
var person = await personFactory.Fetch(id);
// person.IsModified == false

person.FirstName = "Updated";
// person.IsModified == true

// Or modify a child
person.Phones[0].PhoneNumber = "555-1234";
// person.Phones[0].IsModified == true
// person.IsModified == true (child modified)
```

**Use case:** Track unsaved changes and warn users before navigation:

```csharp
if (person.IsModified)
{
    var confirm = await Dialog.Confirm("You have unsaved changes. Discard?");
    if (!confirm) return;
}
```

### IsSelfModified

```csharp
public virtual bool IsSelfModified { get; }
```

Returns `true` when **this specific entity** has been modified, ignoring child entities. True when:

- Direct properties have changed
- `IsDeleted` is `true`
- `MarkModified()` was called

```csharp
// Only the child changed
order.Lines[0].Quantity = 5;

order.Lines[0].IsSelfModified == true
order.IsSelfModified == false   // Order itself unchanged
order.IsModified == true        // But aggregate is modified
```

**Use case:** Generate efficient UPDATE statements that only include changed entities.

### IsDeleted

```csharp
public virtual bool IsDeleted { get; protected set; }
```

Returns `true` when `Delete()` has been called on the entity.

```csharp
person.Delete();
// person.IsDeleted == true
// person.IsModified == true (deletion is a modification)

await personFactory.Save(person);  // Calls [Delete]
```

Calling `UnDelete()` reverses the deletion mark:

```csharp
person.Delete();
// User cancels
person.UnDelete();
// person.IsDeleted == false
```

### IsSavable

```csharp
public virtual bool IsSavable => IsModified && IsValid && !IsBusy && !IsChild;
```

Returns `true` when the entity can be saved. All four conditions must be met:

| Condition | Required Value | Reason |
|-----------|----------------|--------|
| `IsModified` | `true` | Nothing to persist if unchanged |
| `IsValid` | `true` | Cannot persist invalid data |
| `IsBusy` | `false` | Async rules must complete |
| `IsChild` | `false` | Children save through parent |

```csharp
// Enable save button only when appropriate
<button disabled="@(!person.IsSavable)" @onclick="Save">Save</button>
```

**Common reasons for `IsSavable == false`:**

```csharp
if (!person.IsSavable)
{
    if (!person.IsModified) Console.WriteLine("No changes to save");
    if (!person.IsValid) Console.WriteLine("Validation errors exist");
    if (person.IsBusy) Console.WriteLine("Waiting for async rules");
    if (person.IsChild) Console.WriteLine("Save the parent instead");
}
```

### IsChild

```csharp
public virtual bool IsChild { get; protected set; }
```

Returns `true` when this entity is part of a parent aggregate and cannot be saved independently.

An entity becomes a child when:
- It is added to an `EntityListBase` collection
- It is assigned to another entity's property
- `MarkAsChild()` is called explicitly

```csharp
var order = await orderFactory.Create();
var line = await order.Lines.AddLine();

// line.IsChild == true
// line.Parent == order

await line.Save();  // Throws! Cannot save child independently
await order.Save(); // Correct - saves entire aggregate
```

### ModifiedProperties

```csharp
public virtual IEnumerable<string> ModifiedProperties { get; }
```

Returns the names of properties that have changed since the entity was loaded or last saved.

```csharp
person.FirstName = "John";
person.Email = "john@example.com";

foreach (var prop in person.ModifiedProperties)
{
    Console.WriteLine($"Changed: {prop}");
}
// Output:
// Changed: FirstName
// Changed: Email
```

**Use cases:**
- Audit logging
- Building optimized UPDATE statements
- Debugging change tracking issues

### PropertyMessages

```csharp
public IReadOnlyCollection<IPropertyMessage> PropertyMessages { get; }
```

Returns all validation messages for this entity. Each message contains:
- `PropertyName` - The property that failed validation
- `Message` - Human-readable error text
- `Severity` - Error, Warning, or Information

```csharp
foreach (var msg in person.PropertyMessages)
{
    Console.WriteLine($"{msg.PropertyName}: {msg.Message}");
}
// Output:
// FirstName: First Name is required
// Email: Invalid email format
```

## Property-Level Meta-Properties

Beyond entity-level aggregates, each individual property has its own meta-properties accessible through the property wrapper.

### Accessing Property Wrappers

Access property wrappers using the indexer:

```csharp
IEntityProperty firstNameProp = person[nameof(person.FirstName)];
```

Or via PropertyManager:

```csharp
IEntityProperty prop = person.PropertyManager["FirstName"];
```

### Property.IsBusy

```csharp
bool IsBusy { get; }
```

Returns `true` when an async operation is executing for this specific property.

```csharp
person.Email = "test@example.com";  // Triggers async validation

var emailProp = person[nameof(person.Email)];
// emailProp.IsBusy == true (while rule executes)

await person.WaitForTasks();
// emailProp.IsBusy == false
```

### Property.IsValid

```csharp
bool IsValid { get; }
```

Returns `true` when this specific property passes all validation rules.

```csharp
var emailProp = person[nameof(person.Email)];

if (!emailProp.IsValid)
{
    // Show error styling for this field
}
```

**Use in Blazor components:**

```razor
<input @bind="person.Email"
       class="@(person[nameof(person.Email)].IsValid ? "" : "is-invalid")" />
```

### Property.IsModified

```csharp
bool IsModified { get; }  // Available on IEntityProperty
```

Returns `true` when this property's value has changed since load/save.

```csharp
person.FirstName = "John";

var firstNameProp = person[nameof(person.FirstName)];
// firstNameProp.IsModified == true

var lastNameProp = person[nameof(person.LastName)];
// lastNameProp.IsModified == false (unchanged)
```

### Property.PropertyMessages

```csharp
IEnumerable<IPropertyMessage> PropertyMessages { get; }  // Available on IValidateProperty
```

Returns validation messages specific to this property.

```csharp
var emailProp = person[nameof(person.Email)];

foreach (var msg in emailProp.PropertyMessages)
{
    Console.WriteLine(msg.Message);
}
```

## Meta-Property Propagation

Meta-properties automatically propagate through the aggregate hierarchy. Understanding this propagation is key to building reactive UIs.

### Propagation Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    Order (Root)                         │
│  IsValid = IsSelfValid && Lines.IsValid                 │
│  IsModified = IsSelfModified || Lines.IsModified        │
│  IsBusy = IsSelfBusy || Lines.IsBusy                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │              OrderLineList                        │  │
│  │  IsValid = All(line => line.IsValid)              │  │
│  │  IsModified = Any(line => line.IsModified)        │  │
│  ├───────────────────────────────────────────────────┤  │
│  │                                                   │  │
│  │  ┌─────────────┐  ┌─────────────┐                 │  │
│  │  │ OrderLine 1 │  │ OrderLine 2 │                 │  │
│  │  │ IsValid     │  │ IsValid     │                 │  │
│  │  │ IsModified  │  │ IsModified  │                 │  │
│  │  └─────────────┘  └─────────────┘                 │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘

Changes propagate UP:
  OrderLine.IsModified changes → OrderLineList re-evaluates
    → Order.IsModified updates → PropertyChanged fires
```

### How Propagation Works

1. A property value changes on a child entity
2. The child raises `PropertyChanged` and `NeatooPropertyChanged` events
3. The parent's `ChildNeatooPropertyChanged` handler receives the event
4. The parent re-evaluates its own meta-properties
5. If meta-properties changed, the parent raises `PropertyChanged`
6. This continues up to the aggregate root

### Reacting to Child Changes

Override `ChildNeatooPropertyChanged` to react to changes anywhere in the aggregate:

```csharp
protected override Task ChildNeatooPropertyChanged(
    NeatooPropertyChangedEventArgs eventArgs)
{
    // Recalculate when any line item changes
    if (eventArgs.PropertyName == nameof(IOrderLine.Quantity) ||
        eventArgs.PropertyName == nameof(IOrderLine.UnitPrice))
    {
        RecalculateTotal();
    }

    return base.ChildNeatooPropertyChanged(eventArgs);
}
```

## SetValue vs LoadValue

The property system provides two distinct ways to assign values, each with different behaviors.

### SetValue (Default Setter)

When you assign a property normally, `SetValue` is used internally:

```csharp
person.FirstName = "John";  // Uses SetValue
```

**SetValue behavior:**
- Triggers validation rules
- Marks the property as modified
- Raises `PropertyChanged` events
- Cascades to dependent rules
- Updates `IsModified` on entity

Use the normal property setter for:
- User input from UI
- Programmatic changes that should validate
- Any change that represents a "modification"

### LoadValue

Use `LoadValue` when loading data that should not trigger validation or modification tracking:

```csharp
// Access property wrapper and load without triggering rules
person[nameof(person.FirstName)].LoadValue("John");
```

**LoadValue behavior:**
- Does NOT trigger validation rules
- Does NOT mark property as modified
- Does NOT raise PropertyChanged (until complete)
- Sets the value as the "original" baseline

Use `LoadValue` for:
- Loading from database in `[Fetch]` methods
- Setting values within rules (to avoid cascading)
- Initializing defaults that should not count as changes

### MapFrom Uses LoadValue

The generated `MapFrom()` method uses `LoadValue` internally:

```csharp
[Fetch]
public async Task<bool> Fetch([Service] IDbContext db)
{
    var entity = await db.Persons.FindAsync(Id);
    if (entity == null) return false;

    MapFrom(entity);  // Uses LoadValue - no rules triggered
    return true;
}
```

After `MapFrom()`:
- `IsModified == false` (no changes detected)
- `IsNew == false` (entity was fetched)
- Rules have not executed (call `RunRules()` if needed)

### Practical Example

```csharp
// Scenario: Load person, then apply a default
[Fetch]
public async Task<bool> Fetch([Service] IDbContext db)
{
    var entity = await db.Persons.FindAsync(Id);
    MapFrom(entity);  // LoadValue - clean state

    // This would incorrectly mark the entity as modified:
    // if (string.IsNullOrEmpty(Email)) Email = "default@example.com";

    // Correct approach - use LoadValue for defaults:
    if (string.IsNullOrEmpty(Email))
    {
        this[nameof(Email)].LoadValue("default@example.com");
    }

    return true;
}
```

## PauseAllActions for Bulk Updates

When making multiple changes, you can defer events and rules using `PauseAllActions()`:

```csharp
using (person.PauseAllActions())
{
    // These changes do NOT trigger individual events or rules
    person.FirstName = "John";
    person.LastName = "Doe";
    person.Email = "john@example.com";
}
// Events fire and rules execute after dispose
```

This is more efficient than triggering rules three times and is useful for:
- Bulk data import
- Resetting multiple fields
- Complex initialization sequences

## Best Practices

### Declare Properties Correctly

Always use `partial` for entity properties:

```csharp
// Correct
public partial string? FirstName { get; set; }

// Incorrect - source generator cannot process this
public string? FirstName { get; set; }
```

### Use Data Annotations

Neatoo respects standard data annotations:

```csharp
[Required(ErrorMessage = "First name is required")]
[StringLength(50, ErrorMessage = "First name too long")]
[DisplayName("First Name")]
public partial string? FirstName { get; set; }
```

### Check IsBusy Before Validation Checks

Always wait for async operations before checking validity:

```csharp
person.Email = "test@example.com";  // May trigger async rule

// Wrong - might check before rule completes
if (person.IsValid) Save();

// Correct
await person.WaitForTasks();
if (person.IsValid) Save();
```

### Bind UI to Meta-Properties

Take advantage of automatic UI updates:

```razor
<button disabled="@(!entity.IsSavable)">Save</button>
<span class="@(entity.IsBusy ? "loading" : "")">
    @(entity.IsBusy ? "Validating..." : "")
</span>
```

## Related Topics

- [EntityBase Reference](/reference/entity-base/) - Complete entity API
- [Aggregates and Entity Graphs](/concepts/aggregates/) - Parent-child relationships
- [Rules Engine Reference](/reference/rules/) - Validation and transformation rules
- [Data Mapping Reference](/reference/data-mapping/) - MapFrom, MapTo, MapModifiedTo
