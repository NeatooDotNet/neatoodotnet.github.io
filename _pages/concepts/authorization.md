---
title: "Authorization Model"
layout: single
permalink: /concepts/authorization/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

Neatoo implements a "check before execute" authorization philosophy that integrates seamlessly with the factory pattern. Rather than scattering authorization checks throughout your code, you define authorization classes that the generated factories call automatically. This ensures consistent, testable authorization that works identically on client and server.

## The Authorization Philosophy

### Traditional Approaches: Scattered and Fragile

In traditional applications, authorization often ends up scattered throughout the codebase:

```csharp
// Controller - checks permission
public async Task<IActionResult> CreatePerson(PersonDto dto)
{
    if (!User.IsInRole("Admin"))
        return Forbid();

    // Create logic...
}

// Service layer - another check
public async Task<Person> UpdatePerson(int id, PersonDto dto)
{
    if (!_authService.CanUpdatePerson(id))
        throw new UnauthorizedException();

    // Update logic...
}

// UI - yet another check
@if (user.HasRole("Admin"))
{
    <button>Create New</button>
}
```

This approach has serious problems:

- **Duplication**: Same checks in multiple places
- **Inconsistency**: Different checks might use different logic
- **Forgotten checks**: Easy to miss authorization in new code
- **Hard to test**: Authorization is interleaved with business logic
- **Client-server mismatch**: UI might show buttons the user cannot actually click

### Neatoo's Approach: Centralized and Automatic

Neatoo centralizes authorization in dedicated classes that integrate with the factory system:

```csharp
// Define authorization rules once
public class PersonAuth : IPersonAuth
{
    private readonly IUser _user;

    public PersonAuth(IUser user) => _user = user;

    [Authorize(AuthorizeOperation.Create)]
    public bool HasCreate() => _user.Role >= Role.Admin;

    [Authorize(AuthorizeOperation.Read)]
    public bool HasRead() => _user.Role >= Role.User;

    [Authorize(AuthorizeOperation.Write)]
    public bool HasWrite() => _user.Role >= Role.Editor;
}

// Link to entity
[Factory]
[Authorize<IPersonAuth>]
internal partial class Person : EntityBase<Person>, IPerson
{
    // Factory methods are automatically authorized
}
```

The generated factory automatically:
1. Resolves authorization class from DI
2. Calls appropriate authorization methods before each operation
3. Returns `null` or throws based on authorization result
4. Provides `CanXYZ()` methods for UI binding

## Defining Authorization Classes

An authorization class contains methods that determine whether operations are permitted.

### Interface Definition

Define an interface with methods decorated with `[Authorize]`:

```csharp
public interface IPersonAuth
{
    [Authorize(AuthorizeOperation.Read | AuthorizeOperation.Write)]
    bool HasAccess();

    [Authorize(AuthorizeOperation.Create)]
    bool HasCreate();

    [Authorize(AuthorizeOperation.Read)]
    bool HasRead();

    [Authorize(AuthorizeOperation.Write)]
    bool HasWrite();

    [Authorize(AuthorizeOperation.Execute)]
    bool HasDelete();
}
```

The `[Authorize]` attribute specifies which operations trigger each method.

### Implementation Class

Implement the interface with your authorization logic:

```csharp
public class PersonAuth : IPersonAuth
{
    private readonly IUser _user;

    public PersonAuth(IUser user)
    {
        _user = user;
    }

    public bool HasAccess()
    {
        // Any authenticated user can access
        return _user.Role > Role.None;
    }

    public bool HasCreate()
    {
        // Only admins can create
        return _user.Role >= Role.Admin;
    }

    public bool HasRead()
    {
        // Users and above can read
        return _user.Role >= Role.User;
    }

    public bool HasWrite()
    {
        // Editors and above can modify
        return _user.Role >= Role.Editor;
    }

    public bool HasDelete()
    {
        // Only admins can delete
        return _user.Role >= Role.Admin;
    }
}
```

### Registering Authorization Services

Register your authorization implementation in DI:

```csharp
// In Program.cs
builder.Services.AddScoped<IPersonAuth, PersonAuth>();

// The IUser service should be populated from your authentication system
builder.Services.AddScoped<IUser, User>();
```

## The Authorize Attribute on Entities

Link your authorization class to an entity using the `[Authorize<T>]` attribute:

```csharp
[Factory]
[Authorize<IPersonAuth>]
internal partial class Person : EntityBase<Person>, IPerson
{
    // All factory operations are now authorized
    // via IPersonAuth methods

    [Create]
    public void Create() { /* ... */ }

    [Remote]
    [Fetch]
    public async Task<bool> Fetch([Service] IDbContext db) { /* ... */ }

    [Remote]
    [Insert]
    public async Task Insert([Service] IDbContext db) { /* ... */ }

    [Remote]
    [Update]
    public async Task Update([Service] IDbContext db) { /* ... */ }

    [Remote]
    [Delete]
    public async Task Delete([Service] IDbContext db) { /* ... */ }
}
```

### How Authorization Connects to Factory Operations

The generated factory calls authorization methods based on the operation type:

| Factory Operation | AuthorizeOperation Components | Methods Called |
|-------------------|------------------------------|----------------|
| `Create()` | `Read \| Create` | `HasAccess()`, `HasCreate()` |
| `Fetch()` | `Read` | `HasAccess()`, `HasRead()` |
| `Save()` (Insert) | `Read \| Create \| Write` | `HasAccess()`, `HasCreate()`, `HasWrite()` |
| `Save()` (Update) | `Read \| Write` | `HasAccess()`, `HasWrite()` |
| `Save()` (Delete) | `Read \| Execute` | `HasAccess()`, `HasDelete()` |

If any called method returns `false`, the operation is denied.

## Generated CanXYZ() Methods

For each factory operation, Neatoo generates a corresponding `Can` method:

```csharp
public interface IPersonFactory
{
    // Operations
    IPerson? Create();
    Task<IPerson?> Fetch(Guid id);
    Task<IPerson?> Save(IPerson target);
    Task<Authorized<IPerson>> TrySave(IPerson target);

    // Authorization checks (generated when [Authorize<T>] is applied)
    Authorized CanCreate();
    Authorized CanFetch();
    Authorized CanInsert();
    Authorized CanUpdate();
    Authorized CanDelete();
    Authorized CanSave();  // Routes based on entity state
}
```

### Using CanXYZ() in Code

Check authorization before attempting operations:

```csharp
var factory = serviceProvider.GetRequiredService<IPersonFactory>();

// Check if user can create
if (factory.CanCreate().IsAuthorized)
{
    var person = factory.Create();
    // ...
}
else
{
    ShowMessage("You don't have permission to create persons.");
}

// Get authorization message
var canDelete = factory.CanDelete();
if (!canDelete.IsAuthorized)
{
    Console.WriteLine(canDelete.Message);  // Explains why not authorized
}
```

### The Authorized Return Type

The `Authorized` struct provides authorization status and an optional message:

```csharp
public readonly struct Authorized
{
    public bool IsAuthorized { get; }
    public string? Message { get; }

    // Factory methods
    public static Authorized Yes => new(true);
    public static Authorized No(string message) => new(false, message);
}
```

Authorization methods can return `bool` (simple) or `Authorized` (with message):

```csharp
// Simple boolean
[Authorize(AuthorizeOperation.Create)]
public bool HasCreate() => _user.IsAdmin;

// With message
[Authorize(AuthorizeOperation.Create)]
public Authorized HasCreate()
{
    if (_user.IsAdmin)
        return Authorized.Yes;

    return Authorized.No("Only administrators can create new records.");
}
```

## UI Authorization Patterns

Neatoo's authorization model enables consistent UI experiences.

### Showing/Hiding Based on Permissions

```razor
@inject IPersonFactory PersonFactory

@if (PersonFactory.CanCreate().IsAuthorized)
{
    <button @onclick="CreateNew">Create New Person</button>
}

@if (PersonFactory.CanDelete().IsAuthorized)
{
    <button @onclick="DeleteSelected" class="danger">Delete</button>
}
```

### Disabling with Explanation

```razor
@{
    var canSave = PersonFactory.CanSave();
}

<button @onclick="Save"
        disabled="@(!canSave.IsAuthorized)"
        title="@(canSave.IsAuthorized ? "" : canSave.Message)">
    Save
</button>
```

### Authorization-Aware Edit Pages

```razor
@page "/person/{Id:guid}"
@inject IPersonFactory PersonFactory

@if (_person == null)
{
    <p>Loading...</p>
}
else if (!_canEdit.IsAuthorized)
{
    <div class="alert alert-warning">
        @_canEdit.Message
    </div>
    <!-- Show read-only view -->
    <PersonReadOnlyView Person="_person" />
}
else
{
    <!-- Show editable form -->
    <PersonEditForm Person="_person" OnSave="HandleSave" />
}

@code {
    [Parameter] public Guid Id { get; set; }

    private IPerson? _person;
    private Authorized _canEdit;

    protected override async Task OnInitializedAsync()
    {
        _canEdit = PersonFactory.CanUpdate();
        _person = await PersonFactory.Fetch(Id);
    }
}
```

### Combined Savability Check

Remember that `IsSavable` combines authorization with entity state:

```razor
@* Comprehensive save button *@
<button @onclick="Save"
        disabled="@(!_person.IsSavable || !PersonFactory.CanSave().IsAuthorized)">
    @if (_person.IsBusy)
    {
        <span>Validating...</span>
    }
    else if (!_person.IsModified)
    {
        <span>No Changes</span>
    }
    else if (!_person.IsValid)
    {
        <span>Fix Validation Errors</span>
    }
    else if (!PersonFactory.CanSave().IsAuthorized)
    {
        <span>Not Authorized</span>
    }
    else
    {
        <span>Save</span>
    }
</button>
```

## The Check-Before-Execute Guarantee

Neatoo enforces authorization at the factory level, providing several guarantees:

### Server-Side Enforcement

Even if a malicious client bypasses UI checks, the server enforces authorization:

```csharp
// Client attempts unauthorized operation
var person = factory.Create();
await factory.Save(person);  // Server rejects if not authorized
```

### Consistent Client and Server

The same authorization logic runs on both tiers:

```csharp
// Client checks (instant feedback)
if (!factory.CanCreate().IsAuthorized)
{
    ShowError("Not authorized");
    return;
}

// Server also checks (authoritative)
// The factory.Create() call is authorized on server too
```

### Atomic Authorization

All required authorization methods are checked before any operation begins:

```csharp
// For Insert: HasAccess(), HasCreate(), and HasWrite() all checked
// If ANY fails, Insert never executes
// No partial operations with incomplete authorization
```

## Contrast with Traditional Authorization

### Traditional: Scattered Checks

```csharp
// Must remember to check everywhere
public class PersonController : Controller
{
    [HttpPost]
    public async Task<IActionResult> Create(PersonDto dto)
    {
        // Authorization check #1
        if (!_authService.CanCreatePerson())
            return Forbid();

        var person = _mapper.Map<Person>(dto);
        await _repository.Add(person);
        return Ok();
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, PersonDto dto)
    {
        // Authorization check #2 - different pattern
        if (!User.HasClaim("CanEditPerson", "true"))
            return Forbid();

        // ...
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        // Authorization check #3 - forgot this one!
        // Security vulnerability
        await _repository.Delete(id);
        return Ok();
    }
}
```

### Neatoo: Centralized and Automatic

```csharp
// Authorization defined once
[Factory]
[Authorize<IPersonAuth>]
internal partial class Person : EntityBase<Person>, IPerson
{
    // All operations automatically authorized
    // Cannot forget - factory enforces it
}

// Usage is clean
var person = factory.Create();      // Authorized automatically
await factory.Save(person);         // Authorized automatically
```

## Best Practices

### Use Role-Based Authorization

Structure authorization around roles for maintainability:

```csharp
public class PersonAuth : IPersonAuth
{
    public bool HasCreate() => _user.Role >= Role.Admin;
    public bool HasRead() => _user.Role >= Role.User;
    public bool HasWrite() => _user.Role >= Role.Editor;
}
```

### Provide Meaningful Messages

Return `Authorized` with explanations for better UX:

```csharp
public Authorized HasDelete()
{
    if (_user.Role < Role.Admin)
        return Authorized.No("Only administrators can delete records.");

    return Authorized.Yes;
}
```

### Register Authorization Services as Scoped

Authorization services should be scoped to the request:

```csharp
builder.Services.AddScoped<IPersonAuth, PersonAuth>();
builder.Services.AddScoped<IUser, User>();
```

### Always Check CanXYZ() Before Showing UI Actions

Prevent confusing UX where buttons are visible but operations fail:

```razor
@if (Factory.CanDelete().IsAuthorized)
{
    <button @onclick="Delete">Delete</button>
}
```

## Related Topics

- [Authorization System Reference](/reference/authorization/) - Complete API reference
- [Factory Operations Reference](/reference/factory-operations/) - Factory method details
- [Dependency Injection Patterns](/reference/dependency-injection/) - Service registration
- [Server Setup](/setup/server/) - User context configuration
