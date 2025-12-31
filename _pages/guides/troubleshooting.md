---
title: "Troubleshooting and Common Pitfalls"
layout: single
permalink: /guides/troubleshooting/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

This guide provides solutions to common problems encountered when developing with Neatoo. Each issue is presented in Problem/Solution format for quick scanning and resolution.

## Entity Saving Issues

### My Entity Will Not Save

When `IsSavable` returns `false` or `Save()` throws an exception, work through this checklist:

**Problem:** `IsSavable` is `false` and you cannot determine why.

**Solution:** Check each condition that `IsSavable` evaluates:

```csharp
// IsSavable requires all of these to be true
public virtual bool IsSavable => IsModified && IsValid && !IsBusy && !IsChild;
```

Use this diagnostic code:

```csharp
void DiagnoseNotSavable(IEntityBase entity)
{
    Console.WriteLine($"IsSavable: {entity.IsSavable}");
    Console.WriteLine($"  IsModified: {entity.IsModified}");
    Console.WriteLine($"  IsValid: {entity.IsValid}");
    Console.WriteLine($"  IsBusy: {entity.IsBusy}");
    Console.WriteLine($"  IsChild: {entity.IsChild}");

    if (!entity.IsValid)
    {
        Console.WriteLine("  Validation errors:");
        foreach (var msg in entity.PropertyMessages)
        {
            Console.WriteLine($"    {msg.PropertyName}: {msg.Message}");
        }
    }
}
```

### IsModified is False When I Expected True

**Problem:** You changed property values but `IsModified` remains `false`.

**Solution 1:** Check if you used `LoadProperty` instead of the property setter:

```csharp
// Wrong - LoadProperty does not set modification flag
LoadProperty(nameof(FirstName), "John");

// Correct - setter triggers modification tracking
FirstName = "John";
```

**Solution 2:** Check if `PauseAllActions` was active when changes were made:

```csharp
using (entity.PauseAllActions())
{
    entity.FirstName = "John";  // Changes tracked, but...
}
// After dispose, modification state is preserved
```

**Solution 3:** Check if the entity was fetched and not actually modified:

```csharp
var person = await personFactory.Fetch(id);
// At this point, IsModified == false
// You need to actually change something
person.FirstName = person.FirstName;  // Same value = no modification
person.FirstName = "Different Value"; // Now IsModified == true
```

### IsValid is False But I Do Not See Errors

**Problem:** `IsValid` is `false` but you cannot find the validation errors.

**Solution 1:** Check child entities in the aggregate:

```csharp
// IsValid cascades through the entire aggregate
if (!order.IsValid)
{
    // Check order's own messages
    Console.WriteLine("Order messages:");
    foreach (var msg in order.PropertyMessages)
        Console.WriteLine($"  {msg.PropertyName}: {msg.Message}");

    // Check children
    foreach (var line in order.Lines)
    {
        if (!line.IsValid)
        {
            Console.WriteLine($"Line {line.Id} messages:");
            foreach (var msg in line.PropertyMessages)
                Console.WriteLine($"  {msg.PropertyName}: {msg.Message}");
        }
    }
}
```

**Solution 2:** Use `IsSelfValid` to check just the entity itself:

```csharp
// IsSelfValid ignores children
if (entity.IsSelfValid)
{
    Console.WriteLine("Entity itself is valid, problem is in children");
}
```

### IsBusy is True and Blocking Save

**Problem:** `IsBusy` is `true` and prevents saving.

**Solution:** Wait for async operations to complete:

```csharp
// Always wait before checking validity or saving
await entity.WaitForTasks();

if (entity.IsSavable)
{
    await factory.Save(entity);
}
```

**Common causes of IsBusy:**
- Async validation rules are running
- Property setters triggered async operations
- Parent entity has pending operations

### IsChild is True for Root Entity

**Problem:** Your aggregate root has `IsChild == true`, blocking save.

**Solution 1:** Check if the entity was added to a collection unexpectedly:

```csharp
// Adding to a list marks as child
someList.Add(entity);  // entity.IsChild == true
```

**Solution 2:** Check if the entity was assigned to another entity's property:

```csharp
// Assignment to a property marks as child
parentEntity.Child = entity;  // entity.IsChild == true
```

**Solution 3:** Verify you are saving the root, not a child:

```csharp
// Wrong - saving a child
var line = order.Lines[0];
await lineFactory.Save(line);  // Throws - line.IsChild == true

// Correct - save through the root
await orderFactory.Save(order);  // Saves entire aggregate
```

### Stale Data After Save / UI Not Updating

**Problem:** After calling `Save()`:
- Database-generated ID is still empty/zero
- UI shows old values
- `IsModified` is still `true` when it should be `false`
- Navigation to `/{id}` routes fail with empty ID

**Solution:** You forgot to reassign the return value from `Save()`:

```csharp
// WRONG
await personFactory.Save(person);
// person is now stale - it's the PRE-save instance

// CORRECT
person = await personFactory.Save(person);
// person is now the POST-save instance with updated state
```

**Why This Happens:**

`Save()` uses the Remote Factory pattern which serializes your object to the server and deserializes a NEW instance back. The original object in memory is unchanged - it's a completely different object from what the server returns.

Think of it like mailing a document:
1. You write a document (your aggregate)
2. You mail it (serialize to server)
3. Someone adds information and mails it back (server persistence + serialize back)
4. You receive a NEW document (deserialized instance)
5. Your original draft is still on your desk unchanged (original object)

**The Fix:**

Always capture the return value:

```csharp
// In a Blazor component
this._person = await PersonFactory.Save(this._person);

// In a service/handler
var savedPerson = await personFactory.Save(person);
return savedPerson;

// Chained operations
person = await personFactory.Save(person);
var id = person.Id;  // Now has the database-generated ID
```

See also:
- [Factory Operations - Critical: Always Reassign After Save](/reference/factory-operations/#critical-always-reassign-after-save)
- [Blazor Integration - Reassign After Save](/guides/blazor-integration/#critical-reassign-after-save-in-blazor-components)

## Validation Issues

### Validation Not Running

**Problem:** Rules are defined but validation messages never appear.

**Solution 1:** Verify the rule was added to the RuleManager:

```csharp
public Person(IEntityBaseServices<Person> services,
              IUniqueEmailRule uniqueEmailRule) : base(services)
{
    // Did you forget this line?
    RuleManager.AddRule(uniqueEmailRule);
}
```

**Solution 2:** Check trigger properties match the property names:

```csharp
// Wrong - property name mismatch
public class EmailRule : RuleBase<Person>
{
    public EmailRule() : base(p => p.email) { }  // lowercase 'e'
}

// Correct
public class EmailRule : RuleBase<Person>
{
    public EmailRule() : base(p => p.Email) { }  // matches property
}
```

**Solution 3:** Verify the rule is registered in DI:

```csharp
// Make sure rules are registered
services.AddScoped<IUniqueEmailRule, UniqueEmailRule>();
```

**Solution 4:** Check if `LoadProperty` was used in Fetch (skips rules):

```csharp
// LoadProperty does not trigger rules
LoadProperty(nameof(Email), entity.Email);

// If you need rules to run after fetch:
await RunRules(RunRulesFlag.All);
```

### Rule Triggers on Wrong Properties

**Problem:** A rule fires when unrelated properties change.

**Solution:** Verify trigger properties are correctly specified:

```csharp
public class TotalRule : RuleBase<Order>
{
    // Triggers when Subtotal OR TaxRate changes
    public TotalRule() : base(o => o.Subtotal, o => o.TaxRate) { }

    protected override IRuleMessages Execute(Order target)
    {
        target.Total = target.Subtotal * (1 + target.TaxRate);
        return None;
    }
}
```

### Async Rule Never Completes

**Problem:** An async rule appears to hang or never finish.

**Solution 1:** Check for deadlocks with `.Result` or `.Wait()`:

```csharp
// Wrong - can deadlock
protected override Task<IRuleMessages> Execute(Person target, CancellationToken? token)
{
    var result = _service.CheckAsync(target.Email).Result;  // Deadlock!
    return Task.FromResult(result ? None : ErrorMessages());
}

// Correct - properly async
protected override async Task<IRuleMessages> Execute(
    Person target, CancellationToken? token)
{
    var result = await _service.CheckAsync(target.Email);
    return result ? None : ErrorMessages();
}
```

**Solution 2:** Ensure CancellationToken is being used:

```csharp
protected override async Task<IRuleMessages> Execute(
    Person target, CancellationToken? token = null)
{
    var ct = token ?? CancellationToken.None;

    // Use the token in async calls
    var result = await _service.CheckAsync(target.Email, ct);
    return result ? None : ErrorMessages();
}
```

**Solution 3:** Check if the service is properly registered:

```csharp
// If the service is missing, the rule constructor may fail silently
public class UniqueEmailRule : AsyncRuleBase<Person>
{
    private readonly IEmailService _emailService;

    public UniqueEmailRule(IEmailService emailService)  // null if not registered!
        : base(p => p.Email)
    {
        _emailService = emailService ?? throw new ArgumentNullException(nameof(emailService));
    }
}
```

### Validation Messages Not Clearing

**Problem:** Validation error stays visible after fixing the value.

**Solution:** Return `None` when validation passes:

```csharp
protected override IRuleMessages Execute(Person target)
{
    if (string.IsNullOrEmpty(target.Email))
        return None;  // Empty is OK, Required handles that

    if (!target.Email.Contains("@"))
        return (nameof(target.Email), "Invalid format").AsRuleMessages();

    // Important! Return None to clear previous messages
    return None;
}
```

## Property Issues

### Property Not Being Tracked

**Problem:** Changes to a property do not trigger rules or modification tracking.

**Solution:** Ensure the property has the `partial` keyword:

```csharp
// Wrong - not tracked
public string? FirstName { get; set; }

// Correct - source generator creates tracking implementation
public partial string? FirstName { get; set; }
```

### Property Changes Not Raising Events

**Problem:** `PropertyChanged` events are not firing for a property.

**Solution 1:** Verify the `partial` keyword (same as above)

**Solution 2:** Check if `PauseAllActions` is active:

```csharp
using (entity.PauseAllActions())
{
    entity.FirstName = "John";  // No events during pause
}
// Events fire after dispose
```

**Solution 3:** Verify you are not in a `LoadProperty` call:

```csharp
// LoadProperty does not raise PropertyChanged
LoadProperty(nameof(FirstName), "John");  // No event

// Setter raises PropertyChanged
FirstName = "John";  // Event raised
```

### Generated Property Code Not Updating

**Problem:** After changing property definitions, the generated code seems stale.

**Solution:**
1. Clean the solution: `dotnet clean`
2. Rebuild: `dotnet build`
3. In Visual Studio, close and reopen the solution
4. Check for source generator errors in the Error List window

## Serialization Issues

### Circular Reference Error During Serialization

**Problem:** Getting "A possible object cycle was detected" error.

**Solution:** Neatoo handles circular references automatically through the framework serializer. If you encounter this:

1. Verify you are using the Neatoo endpoint (`/api/neatoo`) not custom serialization
2. Check for manually serialized properties that contain parent references

```csharp
// Problem: Manual serialization includes Parent
var json = JsonSerializer.Serialize(entity);  // Circular reference!

// Solution: Use Neatoo's factory for transfer
await factory.Save(entity);  // Handled automatically
```

### Type Information Lost After Serialization

**Problem:** Deserialized entity has wrong type or missing properties.

**Solution:** Ensure interfaces are properly defined and registered:

```csharp
// Entity interface must inherit from framework interfaces
public partial interface IPerson : IEntityBase  // Important!
{
    Guid? Id { get; set; }
    string? FirstName { get; set; }
}

// Entity must implement the interface
[Factory]
internal partial class Person : EntityBase<Person>, IPerson  // Both required
{
    // ...
}
```

### Properties Missing After Remote Call

**Problem:** Some properties are null or default after a remote operation.

**Solution 1:** Ensure the property is `partial`:

```csharp
// Non-partial properties are not serialized
public string? FullName { get; set; }  // Won't transfer!

// Partial properties are serialized
public partial string? FullName { get; set; }  // Will transfer
```

**Solution 2:** Check if the property type is serializable:

```csharp
// Non-serializable types cause issues
public partial CustomService SomeService { get; set; }  // Service can't serialize!
```

## Authorization Issues

### Authorization Check Always Fails

**Problem:** `CanCreate()` or other authorization methods always return unauthorized.

**Solution 1:** Verify the authorization class is registered:

```csharp
// Register in DI
services.AddScoped<IPersonAuthorization, PersonAuthorization>();
```

**Solution 2:** Check the authorization attribute on the entity:

```csharp
[Authorize<IPersonAuthorization>]  // Must reference the interface
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    // ...
}
```

**Solution 3:** Verify the authorization method signature:

```csharp
public class PersonAuthorization : IPersonAuthorization
{
    // Method must match expected signature
    public Authorized CanCreate()  // Not Task<Authorized>!
    {
        return Authorized.Yes();
    }
}
```

### Authorization Checked But Operation Still Fails

**Problem:** `CanCreate()` returns authorized but `Create()` still fails.

**Solution:** Authorization and operation are separate. Check for other issues:

```csharp
// Authorization passed, but maybe:
// - Factory method is missing [Create] attribute
// - Service injection failed
// - Exception in the Create method itself

[Create]  // Don't forget this!
public void Create([Service] IChildFactory childFactory)
{
    // Check for null services
    if (childFactory == null)
        throw new InvalidOperationException("ChildFactory not injected");
}
```

### Different Authorization for Different Operations

**Problem:** Need different authorization rules for Create vs. Update vs. Delete.

**Solution:** Use `AuthorizeOperation` to specify which operation:

```csharp
public class PersonAuthorization : IPersonAuthorization
{
    [Authorize(AuthorizeOperation.Create)]
    public Authorized CanCreate()
    {
        // Only admins can create
        return _user.IsAdmin ? Authorized.Yes() : Authorized.No("Admin required");
    }

    [Authorize(AuthorizeOperation.Update)]
    public Authorized CanUpdate()
    {
        // Anyone authenticated can update
        return _user.IsAuthenticated ? Authorized.Yes() : Authorized.No("Login required");
    }
}
```

## Parent-Child Relationship Issues

### Child Entity Not Linked to Parent

**Problem:** `child.Parent` is null when it should reference the parent.

**Solution 1:** Add children through the list's `Add()` method:

```csharp
// Wrong - direct instantiation
var phone = new PersonPhone();  // phone.Parent == null

// Correct - through factory and add
var phone = await phoneFactory.Create();
person.PersonPhoneList.Add(phone);  // phone.Parent == person
```

**Solution 2:** Verify the list is assigned to the parent:

```csharp
// The list must be a property of the parent
public partial IPersonPhoneList? PersonPhoneList { get; set; }

[Create]
public void Create([Service] IPersonPhoneListFactory listFactory)
{
    PersonPhoneList = listFactory.Create();
    // Now items added will have Parent == this
}
```

### Parent is the List Instead of the Entity

**Problem:** `child.Parent` references the list, not the parent entity.

**Solution:** This is actually not how Neatoo works. Items in `EntityListBase` have their `Parent` set to the list's parent:

```csharp
// person.PersonPhoneList contains phones
// phone.Parent == person (not the list)

// If Parent seems to be the list, check:
// 1. The list itself has a parent set
// 2. The list is assigned to an entity property
```

### DeletedList Not Being Processed in Update

**Problem:** Removed items are not deleted from the database.

**Solution:** Iterate `DeletedList` in your `[Update]` method:

```csharp
[Remote]
[Update]
public async Task Update(Guid parentId, [Service] IDbContext db)
{
    // Don't forget the DeletedList!
    foreach (var deleted in DeletedList.Cast<IPersonPhone>())
    {
        var entity = await db.PersonPhones.FindAsync(deleted.Id);
        if (entity != null)
        {
            db.PersonPhones.Remove(entity);
        }
    }

    // Then process remaining items
    foreach (var item in this)
    {
        if (item.IsNew)
        {
            // Insert logic
        }
        else if (item.IsModified)
        {
            // Update logic
        }
    }

    await db.SaveChangesAsync();
}
```

### Removed Item Reappears After Save

**Problem:** An item removed from a list reappears after saving.

**Solution:** Check if you are re-fetching after save:

```csharp
// The DeletedList is cleared after save
// If you re-fetch, the item might come back from the database

// Make sure Update actually deletes:
foreach (var deleted in DeletedList.Cast<IPersonPhone>())
{
    var entity = await db.PersonPhones.FindAsync(deleted.Id);
    if (entity != null)
    {
        db.PersonPhones.Remove(entity);  // Actually remove!
    }
}
await db.SaveChangesAsync();  // Commit the deletion
```

## Rule Execution Issues

### Rules Execute in Wrong Order

**Problem:** Transformation rules run after validation rules that depend on them.

**Solution:** Rules execute based on trigger properties, not addition order. If you need specific ordering:

```csharp
public Person(IEntityBaseServices<Person> services) : base(services)
{
    // Transformation first
    RuleManager.AddAction(
        (Person p) => p.NormalizedEmail = p.Email?.ToLower(),
        p => p.Email);

    // Validation that depends on transformed value
    RuleManager.AddValidation(
        nameof(NormalizedEmail),  // Trigger on normalized, not Email
        (Person p) => ValidateNormalizedEmail(p.NormalizedEmail));
}
```

### Rule Triggers Infinite Loop

**Problem:** Application hangs because rules keep triggering each other.

**Solution:** Neatoo prevents a rule from triggering itself, but cross-rule loops can occur:

```csharp
// Rule A sets PropertyX
// Rule B triggers on PropertyX, sets PropertyY
// Rule A triggers on PropertyY, sets PropertyX
// Loop!

// Solution: Be careful about what triggers each rule
public class RuleA : RuleBase<Order>
{
    // Only trigger on user-input properties
    public RuleA() : base(o => o.Quantity) { }

    protected override IRuleMessages Execute(Order target)
    {
        // Setting Total triggers RuleB
        target.Total = target.Quantity * target.Price;
        return None;
    }
}

public class RuleB : RuleBase<Order>
{
    // Trigger on Total, but don't set anything RuleA triggers on
    public RuleB() : base(o => o.Total) { }

    protected override IRuleMessages Execute(Order target)
    {
        target.TotalWithTax = target.Total * 1.1m;  // Safe - RuleA doesn't trigger on this
        return None;
    }
}
```

## Performance Issues

### Large Aggregates Cause Memory Issues

**Problem:** Aggregates with many child items consume excessive memory.

**Solution 1:** Consider aggregate design - should this be one aggregate?

```csharp
// Maybe Orders with 10,000 lines shouldn't be one aggregate
// Consider: Order -> OrderLineRef (just IDs) + separate OrderLine aggregates
```

**Solution 2:** Use pagination for display, full aggregate only for editing:

```csharp
// For list display, use a read model
var orderSummaries = await db.Orders
    .Select(o => new OrderSummaryDto { Id = o.Id, Total = o.Total })
    .ToListAsync();

// Only load full aggregate when editing
var order = await orderFactory.Fetch(orderId);
```

**Solution 3:** Lazy load child collections:

```csharp
[Fetch]
public async Task<bool> Fetch(
    bool includeLines,  // Parameter to control loading
    [Service] IDbContext db,
    [Service] IOrderLineListFactory lineFactory)
{
    var query = db.Orders.AsQueryable();

    if (includeLines)
        query = query.Include(o => o.Lines);

    var entity = await query.FirstOrDefaultAsync(o => o.Id == Id);
    if (entity == null) return false;

    // Load properties
    LoadProperty(nameof(Id), entity.Id);
    LoadProperty(nameof(OrderDate), entity.OrderDate);
    // ... other properties

    if (includeLines)
        Lines = await lineFactory.Fetch(entity.Lines);

    return true;
}
```

### Slow Rule Execution

**Problem:** Many rules make the UI feel sluggish.

**Solution 1:** Use async rules for expensive operations:

```csharp
// Slow sync rule blocks UI
public class SlowRule : RuleBase<Person>
{
    protected override IRuleMessages Execute(Person target)
    {
        Thread.Sleep(1000);  // Bad!
        return None;
    }
}

// Async rule keeps UI responsive
public class BetterRule : AsyncRuleBase<Person>
{
    protected override async Task<IRuleMessages> Execute(
        Person target, CancellationToken? token)
    {
        await Task.Delay(1000);  // UI stays responsive
        return None;
    }
}
```

**Solution 2:** Debounce rapid property changes:

```csharp
// In Blazor component
private Timer _debounceTimer;

private void OnEmailChanged(string value)
{
    _debounceTimer?.Dispose();
    _debounceTimer = new Timer(_ =>
    {
        InvokeAsync(() =>
        {
            _person.Email = value;  // Only triggers after pause
            StateHasChanged();
        });
    }, null, 300, Timeout.Infinite);
}
```

## Debugging Generated Code

### Finding Generated Code

**Problem:** You need to see what the source generator created.

**Solution:** Enable source generator output:

1. Add to your `.csproj`:
```xml
<PropertyGroup>
    <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
    <CompilerGeneratedFilesOutputPath>$(BaseIntermediateOutputPath)\GeneratedFiles</CompilerGeneratedFilesOutputPath>
</PropertyGroup>
```

2. Build the project

3. Find generated files in:
   - `obj/Debug/net8.0/GeneratedFiles/Neatoo.SourceGenerators/`

### Stepping Through Generated Code

**Problem:** You want to debug generated factory code.

**Solution:**
1. Enable generated file output (see above)
2. In Visual Studio, enable "Just My Code" = Off
3. Set breakpoints in the generated `.cs` files
4. The debugger will stop in generated code

### Source Generator Errors

**Problem:** Build fails with source generator errors.

**Solution:** Check for these common issues:

1. **Missing `partial` keyword:**
```csharp
// Wrong
public class Person : EntityBase<Person> { }

// Correct
public partial class Person : EntityBase<Person> { }
```

2. **Incorrect interface inheritance:**
```csharp
// Wrong - missing IEntityBase
public partial interface IPerson
{
    Guid? Id { get; set; }
}

// Correct
public partial interface IPerson : IEntityBase
{
    Guid? Id { get; set; }
}
```

## Common Development Mistakes

### Forgetting to Await Async Operations

**Problem:** Code runs but produces unexpected results.

**Solution:** Always await async operations:

```csharp
// Wrong - fire and forget
personFactory.Save(person);  // Returns immediately!
NavigateTo("/persons");  // May happen before save completes

// Correct
await personFactory.Save(person);
NavigateTo("/persons");
```

### Using new Instead of Factory

**Problem:** Entity missing framework features or services.

**Solution:** Always use the factory:

```csharp
// Wrong - bypasses DI and initialization
var person = new Person();

// Correct - proper initialization
var person = personFactory.Create();
```

### Modifying Entity During Save

**Problem:** Changes made during save are not persisted.

**Solution:** Make all changes before calling save:

```csharp
// Wrong - change after save starts
var saveTask = factory.Save(person);
person.LastModified = DateTime.Now;  // May not be saved!
await saveTask;

// Correct - change before save
person.LastModified = DateTime.Now;
await factory.Save(person);
```

### Not Handling Null After Fetch

**Problem:** `NullReferenceException` when entity not found.

**Solution:** Always check for null:

```csharp
// Wrong
var person = await personFactory.Fetch(id);
person.FirstName = "John";  // NullReferenceException if not found!

// Correct
var person = await personFactory.Fetch(id);
if (person == null)
{
    // Handle not found
    return NotFound();
}
person.FirstName = "John";
```

## Quick Reference: IsSavable Checklist

When an entity will not save, check these conditions:

| Condition | Value Needed | How to Check | Common Fix |
|-----------|--------------|--------------|------------|
| `IsModified` | `true` | `entity.IsModified` | Change a property value |
| `IsValid` | `true` | `entity.IsValid` | Fix validation errors |
| `IsBusy` | `false` | `entity.IsBusy` | `await entity.WaitForTasks()` |
| `IsChild` | `false` | `entity.IsChild` | Save through parent |

## Related Topics

- [EntityBase Reference](/reference/entity-base/) - Complete entity API
- [Rules Engine Reference](/reference/rules/) - Rule system details
- [Factory Operations Reference](/reference/factory-operations/) - Factory patterns
- [EntityListBase Reference](/reference/entity-list-base/) - Collection handling
- [Data Mapping Reference](/reference/data-mapping/) - Property mapping patterns
- [Authorization System Reference](/reference/authorization/) - Auth troubleshooting
