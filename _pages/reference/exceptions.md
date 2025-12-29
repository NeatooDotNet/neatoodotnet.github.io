---
title: "Exception Handling Reference"
layout: single
permalink: /reference/exceptions/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

Neatoo provides a structured exception hierarchy for handling errors that occur during entity operations. Understanding these exceptions helps you build robust error handling and provide meaningful feedback to users.

## Exception Hierarchy

All Neatoo exceptions inherit from `NeatooException`:

```
NeatooException
├── SaveOperationException
├── ChildObjectBusyException
├── TypeNotRegisteredException
├── RuleNotAddedException
├── PropertyReadOnlyException
└── PropertyMissingException
```

## NeatooException

The base exception class for all Neatoo-specific errors.

```csharp
public class NeatooException : Exception
{
    public NeatooException(string message) : base(message) { }
    public NeatooException(string message, Exception innerException)
        : base(message, innerException) { }
}
```

### When It's Thrown

`NeatooException` is the base class; you'll typically catch its subclasses for specific error handling. Catching `NeatooException` handles all framework-specific errors.

```csharp
try
{
    await factory.Save(entity);
}
catch (NeatooException ex)
{
    // Handle any Neatoo framework error
    logger.LogError(ex, "Neatoo operation failed");
}
```

## SaveOperationException

The most common exception you'll encounter. Thrown when `Save()` cannot proceed due to entity state issues.

```csharp
public class SaveOperationException : NeatooException
{
    public SaveOperationReason Reason { get; }

    public SaveOperationException(SaveOperationReason reason, string message)
        : base(message)
    {
        Reason = reason;
    }
}
```

### SaveOperationReason Enum

The `Reason` property indicates why the save failed:

| Reason | Description | Common Cause |
|--------|-------------|--------------|
| `IsChildObject` | Entity is a child in an aggregate | Trying to save a child directly instead of through the root |
| `IsInvalid` | Entity failed validation | Validation rules returned errors |
| `NotModified` | Entity has no changes | Calling save on an unmodified entity |
| `IsBusy` | Async operations pending | Saving while validation rules are still running |
| `NoFactoryMethod` | No appropriate factory method found | Missing `[Insert]`, `[Update]`, or `[Delete]` attribute |

### Handling SaveOperationException

```csharp
try
{
    await factory.Save(person);
}
catch (SaveOperationException ex) when (ex.Reason == SaveOperationReason.IsInvalid)
{
    // Show validation errors to user
    foreach (var msg in person.PropertyMessages)
    {
        Console.WriteLine($"{msg.PropertyName}: {msg.Message}");
    }
}
catch (SaveOperationException ex) when (ex.Reason == SaveOperationReason.IsBusy)
{
    // Wait for pending operations and retry
    await person.WaitForTasks();
    if (person.IsSavable)
    {
        await factory.Save(person);
    }
}
catch (SaveOperationException ex) when (ex.Reason == SaveOperationReason.IsChildObject)
{
    // Save through parent instead
    var parent = person.Parent as IOrder;
    if (parent != null)
    {
        await orderFactory.Save(parent);
    }
}
catch (SaveOperationException ex)
{
    // Handle other save failures
    logger.LogError("Save failed: {Reason} - {Message}", ex.Reason, ex.Message);
}
```

### Proactive Prevention

Rather than catching `SaveOperationException`, check `IsSavable` before attempting to save:

```csharp
// Preferred approach - check before save
await person.WaitForTasks();

if (!person.IsSavable)
{
    if (!person.IsModified)
        ShowMessage("No changes to save.");
    else if (!person.IsValid)
        ShowValidationErrors(person.PropertyMessages);
    else if (person.IsBusy)
        ShowMessage("Please wait for validation to complete.");
    else if (person.IsChild)
        ShowMessage("Save through the parent entity.");
    return;
}

await factory.Save(person);
```

## ChildObjectBusyException

Thrown when attempting to modify a child collection while async operations are in progress on child entities.

```csharp
public class ChildObjectBusyException : NeatooException
{
    public bool IsAddOperation { get; }

    public ChildObjectBusyException(bool isAddOperation)
        : base(isAddOperation
            ? "Cannot add child while async operations are in progress"
            : "Cannot remove child while async operations are in progress")
    {
        IsAddOperation = isAddOperation;
    }
}
```

### When It's Thrown

- Adding to an `EntityListBase` while a child has `IsBusy == true`
- Removing from an `EntityListBase` while a child has `IsBusy == true`

### Handling ChildObjectBusyException

```csharp
try
{
    order.Lines.Add(newLine);
}
catch (ChildObjectBusyException ex)
{
    if (ex.IsAddOperation)
    {
        // Wait for existing items to complete validation
        foreach (var line in order.Lines)
        {
            await line.WaitForTasks();
        }
        // Retry add
        order.Lines.Add(newLine);
    }
}
```

### Prevention

Wait for tasks before collection modifications:

```csharp
// Wait for all children to finish async operations
foreach (var line in order.Lines)
{
    await line.WaitForTasks();
}

// Now safe to modify
order.Lines.Add(newLine);
order.Lines.Remove(existingLine);
```

## TypeNotRegisteredException

Thrown when a required type is not registered in the dependency injection container.

```csharp
public class TypeNotRegisteredException : NeatooException
{
    public Type MissingType { get; }

    public TypeNotRegisteredException(Type type)
        : base($"Type '{type.FullName}' is not registered in the service container")
    {
        MissingType = type;
    }
}
```

### Common Causes

1. **Missing `AddNeatooServices()` call:**
```csharp
// Forgot this in Program.cs
builder.Services.AddNeatooServices(NeatooFactory.Server, typeof(IPerson).Assembly);
```

2. **Assembly not included:**
```csharp
// Entity is in a different assembly that wasn't registered
builder.Services.AddNeatooServices(
    NeatooFactory.Server,
    typeof(IPerson).Assembly,
    typeof(IOrder).Assembly  // Don't forget additional assemblies
);
```

3. **Missing service registration:**
```csharp
// Custom service not registered
builder.Services.AddScoped<IMyCustomService, MyCustomService>();
```

### Handling

This is typically a configuration error. Fix the registration rather than catching the exception:

```csharp
// In Program.cs - ensure all assemblies are registered
builder.Services.AddNeatooServices(
    NeatooFactory.Server,
    typeof(IPerson).Assembly,
    typeof(IOrder).Assembly,
    typeof(IInventory).Assembly
);

// Register custom services
builder.Services.AddScoped<IEmailService, EmailService>();
builder.Services.AddScoped<IUniqueNameRule, UniqueNameRule>();
```

## RuleNotAddedException

Thrown when attempting to execute a rule that was never added to the `RuleManager`.

```csharp
public class RuleNotAddedException : NeatooException
{
    public Type RuleType { get; }

    public RuleNotAddedException(Type ruleType)
        : base($"Rule '{ruleType.Name}' was not added to RuleManager")
    {
        RuleType = ruleType;
    }
}
```

### Common Cause

Forgetting to add an injected rule to the `RuleManager`:

```csharp
public Person(
    IEntityBaseServices<Person> services,
    IUniqueNameRule uniqueNameRule) : base(services)
{
    // Forgot this line!
    // RuleManager.AddRule(uniqueNameRule);
}
```

### Prevention

Always add injected rules in the constructor:

```csharp
public Person(
    IEntityBaseServices<Person> services,
    IUniqueNameRule uniqueNameRule,
    IEmailValidationRule emailRule) : base(services)
{
    RuleManager.AddRule(uniqueNameRule);
    RuleManager.AddRule(emailRule);
}
```

## PropertyReadOnlyException

Thrown when attempting to set a property that is marked as read-only.

```csharp
public class PropertyReadOnlyException : NeatooException
{
    public string PropertyName { get; }

    public PropertyReadOnlyException(string propertyName)
        : base($"Property '{propertyName}' is read-only")
    {
        PropertyName = propertyName;
    }
}
```

### When It's Thrown

When setting a property that has been marked read-only through the property system.

## PropertyMissingException

Thrown when attempting to access a property that doesn't exist on the entity.

```csharp
public class PropertyMissingException : NeatooException
{
    public string PropertyName { get; }
    public Type EntityType { get; }

    public PropertyMissingException(string propertyName, Type entityType)
        : base($"Property '{propertyName}' not found on type '{entityType.Name}'")
    {
        PropertyName = propertyName;
        EntityType = entityType;
    }
}
```

### Common Cause

Typo in property name when using string-based property access:

```csharp
// Wrong - typo in property name
var prop = person["FistName"];  // Should be "FirstName"

// Correct
var prop = person[nameof(person.FirstName)];
```

## Exception Handling Best Practices

### 1. Prefer Proactive Validation

Check entity state before operations rather than relying on exceptions:

```csharp
// Good - proactive checking
public async Task SavePerson(IPerson person)
{
    await person.WaitForTasks();

    if (!person.IsSavable)
    {
        HandleNotSavable(person);
        return;
    }

    await factory.Save(person);
}

// Less ideal - reactive exception handling
public async Task SavePerson(IPerson person)
{
    try
    {
        await factory.Save(person);
    }
    catch (SaveOperationException ex)
    {
        HandleSaveFailure(ex);
    }
}
```

### 2. Use When Filters for Specific Handling

Use C# exception filters for targeted handling:

```csharp
try
{
    await factory.Save(person);
}
catch (SaveOperationException ex) when (ex.Reason == SaveOperationReason.IsInvalid)
{
    // Handle validation errors specifically
}
catch (SaveOperationException ex) when (ex.Reason == SaveOperationReason.IsBusy)
{
    // Handle busy state specifically
}
```

### 3. Log with Context

Include relevant entity state when logging exceptions:

```csharp
catch (SaveOperationException ex)
{
    logger.LogError(ex,
        "Save failed for {EntityType}. Reason: {Reason}, IsValid: {IsValid}, IsBusy: {IsBusy}, IsModified: {IsModified}",
        entity.GetType().Name,
        ex.Reason,
        entity.IsValid,
        entity.IsBusy,
        entity.IsModified);
}
```

### 4. Blazor Component Pattern

Standard pattern for save operations in Blazor:

```csharp
private async Task Save()
{
    try
    {
        IsSaving = true;

        // Wait for async validation
        await _person.WaitForTasks();

        if (!_person.IsSavable)
        {
            if (!_person.IsValid)
            {
                Snackbar.Add("Please fix validation errors", Severity.Warning);
            }
            return;
        }

        await _factory.Save(_person);
        Snackbar.Add("Saved successfully", Severity.Success);
        NavigationManager.NavigateTo("/persons");
    }
    catch (SaveOperationException ex)
    {
        Snackbar.Add($"Save failed: {ex.Message}", Severity.Error);
    }
    catch (HttpRequestException ex)
    {
        Snackbar.Add("Connection error. Please try again.", Severity.Error);
    }
    finally
    {
        IsSaving = false;
    }
}
```

### 5. TrySave for Authorization-Aware Operations

Use `TrySave()` when authorization might prevent the operation:

```csharp
var result = await factory.TrySave(person);

if (result.IsAuthorized)
{
    var savedPerson = result.Value;
    ShowSuccess("Person saved");
}
else
{
    ShowError(result.Message ?? "You don't have permission to save");
}
```

## Diagnostic Helper

Use this helper method to diagnose entity state issues:

```csharp
public static void DiagnoseEntityState(IEntityBase entity)
{
    Console.WriteLine($"Entity: {entity.GetType().Name}");
    Console.WriteLine($"  IsNew: {entity.IsNew}");
    Console.WriteLine($"  IsModified: {entity.IsModified}");
    Console.WriteLine($"  IsSelfModified: {entity.IsSelfModified}");
    Console.WriteLine($"  IsValid: {entity.IsValid}");
    Console.WriteLine($"  IsSelfValid: {entity.IsSelfValid}");
    Console.WriteLine($"  IsBusy: {entity.IsBusy}");
    Console.WriteLine($"  IsSelfBusy: {entity.IsSelfBusy}");
    Console.WriteLine($"  IsChild: {entity.IsChild}");
    Console.WriteLine($"  IsDeleted: {entity.IsDeleted}");
    Console.WriteLine($"  IsSavable: {entity.IsSavable}");

    if (!entity.IsValid)
    {
        Console.WriteLine("  Validation Errors:");
        foreach (var msg in entity.PropertyMessages)
        {
            Console.WriteLine($"    {msg.PropertyName}: {msg.Message}");
        }
    }
}
```

## Related Topics

- [Troubleshooting Guide](/guides/troubleshooting/) - Common issues and solutions
- [EntityBase Reference](/reference/entity-base/) - Entity state properties
- [Rules Engine Reference](/reference/rules/) - Validation rule errors
- [Factory Operations Reference](/reference/factory-operations/) - Save operation details
