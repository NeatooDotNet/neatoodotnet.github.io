---
title: "Authorization System Reference"
layout: single
permalink: /reference/authorization/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

This reference provides complete documentation for the Neatoo authorization system, including attributes, method signatures, and integration patterns. For conceptual understanding, see [Authorization Model](/concepts/authorization/).

## AuthorizeFactoryOperation Enum

The `AuthorizeFactoryOperation` enum defines the operations that can be authorized:

```csharp
[Flags]
public enum AuthorizeFactoryOperation
{
    Create  = 1,    // Creating new instances
    Fetch   = 2,    // Reading/fetching instances
    Insert  = 4,    // Inserting new records
    Update  = 8,    // Modifying existing records
    Delete  = 16,   // Removing records
    Read    = 32,   // General read access (bitwise combination)
    Write   = 64,   // General write access (bitwise combination)
    Execute = 128   // Executing special operations (bitwise combination)
}
```

These values are bitwise flags used to define the `FactoryOperation` enum, allowing combinations of operations to be specified.

### Usage with [AuthorizeFactory] Attribute

The enum is used with the `[AuthorizeFactory]` attribute on authorization methods:

```csharp
public interface IPersonAuth
{
    // Single operation
    [AuthorizeFactory(AuthorizeFactoryOperation.Create)]
    bool CanCreate();

    // Multiple operations (bitwise OR)
    [AuthorizeFactory(AuthorizeFactoryOperation.Fetch)]
    bool CanFetch();
}
```

### Semantic Meaning

| Operation | Meaning | Typical Use |
|-----------|---------|-------------|
| `Create` | Permission to create new instances | `[Create]` factory methods |
| `Fetch` | Permission to fetch/retrieve data | `[Fetch]` factory methods |
| `Insert` | Permission to insert new records | `[Insert]` factory methods |
| `Update` | Permission to modify existing records | `[Update]` factory methods |
| `Delete` | Permission to delete records | `[Delete]` factory methods |
| `Read` | General read permission | Broad read access |
| `Write` | General write permission | Broad write access |
| `Execute` | Permission for special operations | Custom operations |

## How Authorization Works

When a factory operation is invoked, Neatoo determines which authorization methods to call based on the operation type:

### Authorization Flow

1. **Client-side check**: `Can*` methods (e.g., `CanCreate()`, `CanFetch()`) enable UI decisions
2. **Server-side enforcement**: Server always checks authorization before execution

### Practical Example

Given this authorization interface:

```csharp
public interface IPersonAuth
{
    [AuthorizeFactory(AuthorizeFactoryOperation.Create)]
    bool CanCreate();

    [AuthorizeFactory(AuthorizeFactoryOperation.Fetch)]
    bool CanFetch();

    [AuthorizeFactory(AuthorizeFactoryOperation.Update)]
    bool CanUpdate();

    [AuthorizeFactory(AuthorizeFactoryOperation.Delete)]
    bool CanDelete();
}
```

The factory operations call these methods:

| Operation | Authorization Method Called |
|-----------|----------------------------|
| `factory.Create()` | `CanCreate()` |
| `factory.Fetch()` | `CanFetch()` |
| `factory.Save()` (Insert) | `CanCreate()` |
| `factory.Save()` (Update) | `CanUpdate()` |
| `factory.Save()` (Delete) | `CanDelete()` |

## Authorization Method Signatures

Authorization methods can use several signatures:

### Boolean Return

Simple boolean return for basic authorization:

```csharp
[AuthorizeFactory(AuthorizeFactoryOperation.Create)]
public bool CanCreate()
{
    return _user.Role >= Role.Admin;
}
```

### Authorized Return Type

Return `Authorized` struct for richer feedback:

```csharp
[AuthorizeFactory(AuthorizeFactoryOperation.Create)]
public Authorized CanCreate()
{
    if (_user.Role >= Role.Admin)
        return Authorized.Yes;

    return Authorized.No("Only administrators can create records.");
}
```

### The Authorized Struct

```csharp
public readonly struct Authorized
{
    public bool IsAuthorized { get; }
    public string? Message { get; }

    // Factory methods
    public static Authorized Yes => new(true, null);
    public static Authorized No(string? message = null) => new(false, message);

    // Implicit conversion from bool
    public static implicit operator Authorized(bool value) => new(value, null);
    public static implicit operator bool(Authorized auth) => auth.IsAuthorized;
}
```

### Generic Authorized Struct

For operations that return values:

```csharp
public readonly struct Authorized<T>
{
    public bool IsAuthorized { get; }
    public string? Message { get; }
    public T? Value { get; }

    public static Authorized<T> Yes(T value) => new(true, null, value);
    public static Authorized<T> No(string? message = null) => new(false, message, default);
}
```

Used by `TrySave()`:

```csharp
Task<Authorized<IPerson>> TrySave(IPerson target);

// Usage
var result = await factory.TrySave(person);
if (result.IsAuthorized)
{
    var savedPerson = result.Value;
}
else
{
    var errorMessage = result.Message;
}
```

## Complete Authorization Interface Example

```csharp
public interface IPersonAuth
{
    /// <summary>
    /// Permission to create new persons.
    /// </summary>
    [AuthorizeFactory(AuthorizeFactoryOperation.Create)]
    Authorized CanCreate();

    /// <summary>
    /// Permission to read/fetch persons.
    /// </summary>
    [AuthorizeFactory(AuthorizeFactoryOperation.Fetch)]
    Authorized CanFetch();

    /// <summary>
    /// Permission to update existing persons.
    /// </summary>
    [AuthorizeFactory(AuthorizeFactoryOperation.Update)]
    Authorized CanUpdate();

    /// <summary>
    /// Permission to delete persons.
    /// </summary>
    [AuthorizeFactory(AuthorizeFactoryOperation.Delete)]
    Authorized CanDelete();
}
```

## Complete Authorization Implementation Example

```csharp
public class PersonAuth : IPersonAuth
{
    private readonly IUser _user;

    public PersonAuth(IUser user)
    {
        _user = user ?? throw new ArgumentNullException(nameof(user));
    }

    public Authorized CanCreate()
    {
        if (_user.Role < Role.Admin)
            return Authorized.No("Only administrators can create new persons.");

        return Authorized.Yes;
    }

    public Authorized CanFetch()
    {
        if (_user.Role < Role.User)
            return Authorized.No("You don't have permission to view person records.");

        return Authorized.Yes;
    }

    public Authorized CanUpdate()
    {
        if (_user.Role < Role.Editor)
            return Authorized.No("You don't have permission to edit person records.");

        return Authorized.Yes;
    }

    public Authorized CanDelete()
    {
        if (_user.Role < Role.Admin)
            return Authorized.No("Only administrators can delete persons.");

        return Authorized.Yes;
    }
}
```

## Resolving Authorization from DI

### Registration

Register authorization services in your DI container:

```csharp
// Program.cs

// Register the authorization implementation
builder.Services.AddScoped<IPersonAuth, PersonAuth>();

// Register the user context service
builder.Services.AddScoped<IUser, User>();
```

### Generated Factory Resolution

The generated factory resolves authorization automatically:

```csharp
// Generated code (simplified)
public class PersonFactory : IPersonFactory
{
    private readonly IServiceProvider _serviceProvider;

    public IPerson? Create()
    {
        // Resolve authorization service
        var auth = _serviceProvider.GetRequiredService<IPersonAuth>();

        // Check all applicable authorization methods
        var result = auth.HasAccess();
        if (!result.IsAuthorized)
            return null;

        result = auth.HasCreate();
        if (!result.IsAuthorized)
            return null;

        // Proceed with creation
        return CreateInternal();
    }
}
```

### User Context Population

Populate user context from your authentication system:

```csharp
// Server-side middleware
app.Use(async (context, next) =>
{
    var user = context.RequestServices.GetRequiredService<IUser>();

    if (context.User.Identity?.IsAuthenticated == true)
    {
        user.UserId = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        user.Role = Enum.Parse<Role>(
            context.User.FindFirst(ClaimTypes.Role)?.Value ?? "None");
    }

    await next();
});
```

## ASP.NET Core IAuthorizationService Integration

You can integrate with ASP.NET Core's built-in authorization:

### Define Policies

```csharp
// Program.cs
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanCreatePerson", policy =>
        policy.RequireRole("Admin"));

    options.AddPolicy("CanEditPerson", policy =>
        policy.RequireRole("Admin", "Editor"));

    options.AddPolicy("CanViewPerson", policy =>
        policy.RequireAuthenticatedUser());
});
```

### Use in Authorization Class

```csharp
public class PersonAuth : IPersonAuth
{
    private readonly IAuthorizationService _authService;
    private readonly IHttpContextAccessor _httpContextAccessor;

    public PersonAuth(
        IAuthorizationService authService,
        IHttpContextAccessor httpContextAccessor)
    {
        _authService = authService;
        _httpContextAccessor = httpContextAccessor;
    }

    private ClaimsPrincipal User =>
        _httpContextAccessor.HttpContext?.User ?? new ClaimsPrincipal();

    public async Task<Authorized> HasCreateAsync()
    {
        var result = await _authService.AuthorizeAsync(User, "CanCreatePerson");

        return result.Succeeded
            ? Authorized.Yes
            : Authorized.No("Policy 'CanCreatePerson' not satisfied.");
    }

    // Sync wrapper for interface compatibility
    [Authorize(AuthorizeOperation.Create)]
    public Authorized HasCreate()
    {
        return HasCreateAsync().GetAwaiter().GetResult();
    }
}
```

### Registration with HttpContext

```csharp
builder.Services.AddHttpContextAccessor();
builder.Services.AddScoped<IPersonAuth, PersonAuth>();
```

## Generated Factory Methods

When `[AuthorizeFactory<T>]` is applied, the factory includes authorization checks and CanXYZ methods:

```csharp
public interface IPersonFactory
{
    // Standard operations
    IPerson? Create();
    Task<IPerson?> Fetch(Guid id);
    Task<IPerson?> Save(IPerson target);
    Task<Authorized<IPerson>> TrySave(IPerson target);

    // Authorization check methods
    Authorized CanCreate();
    Authorized CanFetch();
    Authorized CanInsert();
    Authorized CanUpdate();
    Authorized CanDelete();
    Authorized CanSave();
}
```

### CanSave() Routing

`CanSave()` returns authorization based on entity state:

```csharp
public Authorized CanSave()
{
    // Routes based on entity state:
    // - IsDeleted? Check delete authorization
    // - IsNew? Check insert authorization
    // - Otherwise? Check update authorization
}
```

## Unit Testing Authorization

### Testing Authorization Classes Directly

```csharp
[TestClass]
public class PersonAuthTests
{
    [TestMethod]
    public void HasCreate_WhenAdmin_ReturnsAuthorized()
    {
        // Arrange
        var user = new User { Role = Role.Admin };
        var auth = new PersonAuth(user);

        // Act
        var result = auth.HasCreate();

        // Assert
        Assert.IsTrue(result.IsAuthorized);
    }

    [TestMethod]
    public void HasCreate_WhenUser_ReturnsUnauthorized()
    {
        // Arrange
        var user = new User { Role = Role.User };
        var auth = new PersonAuth(user);

        // Act
        var result = auth.HasCreate();

        // Assert
        Assert.IsFalse(result.IsAuthorized);
        Assert.AreEqual("Only administrators can create new persons.", result.Message);
    }

    [TestMethod]
    public void HasDelete_WhenEditor_ReturnsUnauthorized()
    {
        // Arrange
        var user = new User { Role = Role.Editor };
        var auth = new PersonAuth(user);

        // Act
        var result = auth.HasDelete();

        // Assert
        Assert.IsFalse(result.IsAuthorized);
    }
}
```

### Integration Testing with Factory

```csharp
[TestClass]
public class PersonFactoryAuthorizationTests : IntegrationTestBase
{
    [TestMethod]
    public void Create_WhenUnauthorized_ReturnsNull()
    {
        // Arrange
        SetCurrentUser(Role.User);  // Not admin
        var factory = GetService<IPersonFactory>();

        // Act
        var person = factory.Create();

        // Assert
        Assert.IsNull(person);
    }

    [TestMethod]
    public void Create_WhenAuthorized_ReturnsPerson()
    {
        // Arrange
        SetCurrentUser(Role.Admin);
        var factory = GetService<IPersonFactory>();

        // Act
        var person = factory.Create();

        // Assert
        Assert.IsNotNull(person);
    }

    [TestMethod]
    public void CanCreate_WhenUnauthorized_ReturnsFalseWithMessage()
    {
        // Arrange
        SetCurrentUser(Role.User);
        var factory = GetService<IPersonFactory>();

        // Act
        var result = factory.CanCreate();

        // Assert
        Assert.IsFalse(result.IsAuthorized);
        Assert.IsNotNull(result.Message);
    }

    [TestMethod]
    public async Task Save_WhenUnauthorized_ThrowsOrReturnsNull()
    {
        // Arrange
        SetCurrentUser(Role.Admin);  // Can create
        var factory = GetService<IPersonFactory>();
        var person = factory.Create();
        person.FirstName = "Test";

        SetCurrentUser(Role.User);  // Cannot save

        // Act & Assert
        var result = await factory.TrySave(person);
        Assert.IsFalse(result.IsAuthorized);
    }

    private void SetCurrentUser(Role role)
    {
        var user = GetService<IUser>();
        user.Role = role;
    }
}
```

### Testing with Mock Authorization

```csharp
[TestClass]
public class PersonAuthMockTests
{
    [TestMethod]
    public void HasAccess_WithNullUser_ReturnsFalse()
    {
        // Arrange
        var mockUser = new Mock<IUser>();
        mockUser.Setup(u => u.Role).Returns(Role.None);

        var auth = new PersonAuth(mockUser.Object);

        // Act
        var result = auth.HasAccess();

        // Assert
        Assert.IsFalse(result.IsAuthorized);
    }
}
```

## Authorization Patterns

### Hierarchical Roles

```csharp
public enum Role
{
    None = 0,
    Guest = 1,
    User = 2,
    Editor = 3,
    Admin = 4,
    SuperAdmin = 5
}

public class PersonAuth : IPersonAuth
{
    public Authorized CanFetch() =>
        _user.Role >= Role.Guest ? Authorized.Yes
            : Authorized.No("Guests and above can view.");

    public Authorized CanUpdate() =>
        _user.Role >= Role.Editor ? Authorized.Yes
            : Authorized.No("Editors and above can modify.");

    public Authorized CanCreate() =>
        _user.Role >= Role.Admin ? Authorized.Yes
            : Authorized.No("Admins and above can create.");
}
```

### Resource-Based Authorization

```csharp
public class PersonAuth : IPersonAuth
{
    private readonly IUser _user;
    private readonly IPerson? _person;  // Target resource

    public PersonAuth(IUser user, IContextAccessor<IPerson> context)
    {
        _user = user;
        _person = context.Current;
    }

    public Authorized CanUpdate()
    {
        // Admins can update anyone
        if (_user.Role >= Role.Admin)
            return Authorized.Yes;

        // Users can update their own record
        if (_person?.OwnerId == _user.UserId)
            return Authorized.Yes;

        return Authorized.No("You can only edit your own profile.");
    }
}
```

### Feature Flag Integration

```csharp
public class PersonAuth : IPersonAuth
{
    private readonly IUser _user;
    private readonly IFeatureManager _features;

    public PersonAuth(IUser user, IFeatureManager features)
    {
        _user = user;
        _features = features;
    }

    public Authorized CanCreate()
    {
        if (!_features.IsEnabled("PersonCreation"))
            return Authorized.No("Person creation is currently disabled.");

        if (_user.Role < Role.Admin)
            return Authorized.No("Only administrators can create persons.");

        return Authorized.Yes;
    }
}
```

## Related Topics

- [Authorization Model](/concepts/authorization/) - Conceptual overview
- [Factory Operations Reference](/reference/factory-operations/) - Factory method details
- [Dependency Injection Patterns](/reference/dependency-injection/) - Service registration
- [Server Setup](/setup/server/) - User context configuration
