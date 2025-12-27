---
title: "Dependency Injection Patterns"
layout: single
permalink: /reference/dependency-injection/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

Neatoo embraces dependency injection as a core architectural principle. From entity construction to factory method execution, services flow through the DI container. This reference documents all DI patterns in Neatoo, including service registration, constructor injection, method injection, and testing strategies.

## DI-Centric Philosophy

Neatoo is designed around dependency injection from the ground up:

- **Entities receive services** through their constructors
- **Factory methods receive services** through `[Service]` parameters
- **Rules can inject dependencies** for external validation
- **Authorization classes** receive user context through DI
- **Factories themselves** are registered and resolved from DI

This architecture provides:

- **Testability**: Mock any dependency in isolation
- **Flexibility**: Swap implementations without code changes
- **Consistency**: Same injection patterns everywhere
- **Separation of concerns**: Entities focus on domain logic, not service location

## AddNeatooServices() Registration

The `AddNeatooServices()` extension method is your entry point for registering Neatoo with the DI container.

### Basic Registration

```csharp
builder.Services.AddNeatooServices(
    NeatooFactory.Server,           // Factory mode
    typeof(IPerson).Assembly        // Assembly containing entities
);
```

### Multiple Assemblies

Pass multiple assemblies if your entities span several projects:

```csharp
builder.Services.AddNeatooServices(
    NeatooFactory.Server,
    typeof(IPerson).Assembly,       // Core domain
    typeof(IOrder).Assembly,        // Orders module
    typeof(IInventory).Assembly     // Inventory module
);
```

### What Gets Registered

`AddNeatooServices()` registers:

| Service | Lifetime | Purpose |
|---------|----------|---------|
| `IEntityBaseServices<T>` | Transient | Base services for entities |
| `IValidateBaseServices<T>` | Transient | Base services for validate objects |
| `IBaseServices<T>` | Transient | Base services for value objects |
| `IRuleManager<T>` | Transient | Rule management per entity type |
| `IPropertyManager` delegates | Transient | Property wrapper factories |
| Generated factories | Varies | `IPersonFactory`, etc. |
| JSON converters | Singleton | Serialization support |
| Remote factory handlers | Scoped | Server-side request handling |

### Assembly Scanning

Neatoo scans provided assemblies to discover:

- Classes with `[Factory]` attribute
- Generated factory implementations
- Custom serializers
- Rule implementations

## NeatooFactory.Server vs NeatooFactory.Remote

The factory mode determines how operations execute:

### NeatooFactory.Server

Use on the server (ASP.NET Core) or for standalone applications:

```csharp
// Server-side: Execute all operations locally
builder.Services.AddNeatooServices(NeatooFactory.Server, typeof(IPerson).Assembly);
```

**Behavior:**
- All factory operations execute in-process
- Services resolved from local DI container
- No serialization/deserialization
- Database connections available

**Use for:**
- ASP.NET Core servers
- Console applications
- Desktop applications (WPF, WinForms)
- Background services

### NeatooFactory.Remote

Use on clients that call a remote server:

```csharp
// Client-side: Serialize [Remote] operations to server
builder.Services.AddNeatooServices(NeatooFactory.Remote, typeof(IPerson).Assembly);
```

**Behavior:**
- Operations without `[Remote]` execute locally
- Operations with `[Remote]` serialize to server via HTTP
- Entity state transferred with request
- Results deserialized back to client

**Use for:**
- Blazor WebAssembly
- .NET MAUI apps calling APIs
- Any client connecting to Neatoo server

### Mode Comparison

| Aspect | Server | Remote |
|--------|--------|--------|
| `[Create]` without `[Remote]` | Local | Local |
| `[Create]` with `[Remote]` | Local | Server call |
| `[Fetch]` with `[Remote]` | Local | Server call |
| Database access | Available | Not available |
| Entity serialization | Not needed | Automatic |

## Constructor Injection for Entities

Entity constructors receive services through DI.

### Required: IEntityBaseServices

Every `EntityBase<T>` must accept and pass `IEntityBaseServices<T>` to the base:

```csharp
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    public Person(IEntityBaseServices<Person> services) : base(services)
    {
    }
}
```

This service provides framework infrastructure. You never interact with it directly.

### Injecting Rules

Rules are injected via constructor and added to `RuleManager`:

```csharp
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    public Person(
        IEntityBaseServices<Person> services,
        IUniqueNameRule uniqueNameRule,
        IEmailValidationRule emailRule) : base(services)
    {
        RuleManager.AddRule(uniqueNameRule);
        RuleManager.AddRule(emailRule);
    }
}
```

### Injecting Child Factories

For entities with child collections, inject child factories:

```csharp
[Factory]
internal partial class Order : EntityBase<Order>, IOrder
{
    private readonly IOrderLineListFactory _lineListFactory;

    public Order(
        IEntityBaseServices<Order> services,
        IOrderLineListFactory lineListFactory) : base(services)
    {
        _lineListFactory = lineListFactory;
    }

    [Create]
    public void Create()
    {
        Lines = _lineListFactory.Create();
    }
}
```

### Injecting Application Services

Any registered service can be injected:

```csharp
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    private readonly ILogger<Person> _logger;

    public Person(
        IEntityBaseServices<Person> services,
        ILogger<Person> logger) : base(services)
    {
        _logger = logger;
    }
}
```

### Constructor Injection Limitations

Constructor-injected services are available throughout the entity's lifetime but:

- They increase entity memory footprint
- They are resolved when the entity is created, not when needed
- They cannot vary between factory operations

For operation-specific services, use method injection instead.

## Method Injection with [Service]

Factory methods can receive services via the `[Service]` attribute.

### Basic Usage

```csharp
[Remote]
[Fetch]
public async Task<bool> Fetch(
    [Service] IPersonDbContext dbContext,
    [Service] IPersonPhoneListFactory phoneListFactory)
{
    var entity = await dbContext.Persons.FindAsync(Id);
    if (entity == null) return false;

    MapFrom(entity);
    PersonPhoneList = await phoneListFactory.Fetch(entity.Phones);
    return true;
}
```

### Combining Regular and Service Parameters

Regular parameters come first, followed by `[Service]` parameters:

```csharp
[Remote]
[Fetch]
public async Task<bool> Fetch(
    Guid personId,              // Regular - exposed on factory interface
    bool includePhones,         // Regular - exposed on factory interface
    [Service] IDbContext db,    // Service - resolved from DI
    [Service] IPhoneListFactory phoneFactory)  // Service - resolved from DI
{
    // personId and includePhones come from caller
    // db and phoneFactory are injected
}
```

Generated factory interface:

```csharp
public interface IPersonFactory
{
    Task<IPerson?> Fetch(Guid personId, bool includePhones);
    // [Service] parameters are hidden from interface
}
```

### When to Use Method vs Constructor Injection

| Use Constructor Injection | Use Method Injection |
|--------------------------|----------------------|
| Service needed throughout entity lifetime | Service needed only for specific operation |
| Rules that need external services | Database contexts |
| Loggers | External API clients |
| Child factories used in multiple places | Operation-specific services |

### Multiple [Service] Parameters

You can inject multiple services:

```csharp
[Remote]
[Insert]
public async Task Insert(
    [Service] IDbContext db,
    [Service] IEmailService email,
    [Service] IAuditService audit,
    [Service] IChildFactory childFactory)
{
    Id = Guid.NewGuid();
    var entity = new PersonEntity();
    MapTo(entity);
    db.Persons.Add(entity);

    await childFactory.Save(Children, Id.Value);
    await db.SaveChangesAsync();

    await email.SendWelcomeEmail(Email);
    await audit.LogCreation("Person", Id.Value);
}
```

## Service Registration Patterns

### Standard Registration

Register services using standard .NET DI patterns:

```csharp
// Transient - new instance each time
builder.Services.AddTransient<IMyRule, MyRule>();

// Scoped - one instance per request/scope
builder.Services.AddScoped<IDbContext, AppDbContext>();
builder.Services.AddScoped<IUser, User>();

// Singleton - shared instance
builder.Services.AddSingleton<ICache, MemoryCache>();
```

### Interface to Implementation

Register interfaces with their implementations:

```csharp
// Explicit registration
builder.Services.AddScoped<IPersonAuth, PersonAuth>();
builder.Services.AddScoped<IUniqueNameRule, UniqueNameRule>();

// EF Core context with interface
builder.Services.AddDbContext<AppDbContext>();
builder.Services.AddScoped<IAppDbContext>(sp => sp.GetRequiredService<AppDbContext>());
```

### Convention-Based Registration

Neatoo provides `RegisterMatchingName()` for automatic registration:

```csharp
builder.Services.RegisterMatchingName(
    typeof(IPerson).Assembly,
    ServiceLifetime.Scoped
);
```

This registers all services where:
- Interface name starts with "I"
- Implementation name matches (minus "I" prefix)

Examples:
- `IPersonAuth` -> `PersonAuth`
- `IUniqueNameRule` -> `UniqueNameRule`
- `IEmailService` -> `EmailService`

### Factory Registration

Generated factories are registered automatically by `AddNeatooServices()`. You do not need to register `IPersonFactory`, `IOrderFactory`, etc.

## Dependency Injection for Rules

Rules can receive dependencies through their constructor:

```csharp
public class UniqueNameRule : AsyncRuleBase<IPerson>, IUniqueNameRule
{
    private readonly IPersonDbContext _dbContext;

    public UniqueNameRule(IPersonDbContext dbContext)
        : base(p => p.FirstName, p => p.LastName)
    {
        _dbContext = dbContext;
    }

    protected override async Task<IRuleMessages> ExecuteAsync(
        IPerson target,
        CancellationToken token)
    {
        var exists = await _dbContext.Persons
            .AnyAsync(p => p.FirstName == target.FirstName
                       && p.LastName == target.LastName
                       && p.Id != target.Id, token);

        return exists
            ? (nameof(target.LastName), "A person with this name already exists")
                .AsRuleMessages()
            : None;
    }
}
```

Register the rule:

```csharp
builder.Services.AddScoped<IUniqueNameRule, UniqueNameRule>();
```

Inject into entity:

```csharp
public Person(
    IEntityBaseServices<Person> services,
    IUniqueNameRule uniqueNameRule) : base(services)
{
    RuleManager.AddRule(uniqueNameRule);
}
```

## DI for Authorization Classes

Authorization classes receive services for making authorization decisions:

```csharp
public class PersonAuth : IPersonAuth
{
    private readonly IUser _user;
    private readonly IFeatureFlags _features;

    public PersonAuth(IUser user, IFeatureFlags features)
    {
        _user = user;
        _features = features;
    }

    [AuthorizeFactory(AuthorizeFactoryOperation.Create)]
    public Authorized CanCreate()
    {
        if (!_features.IsEnabled("PersonCreation"))
            return Authorized.No("Feature disabled");

        if (_user.Role < Role.Admin)
            return Authorized.No("Insufficient permissions");

        return Authorized.Yes;
    }
}
```

Register:

```csharp
builder.Services.AddScoped<IPersonAuth, PersonAuth>();
builder.Services.AddScoped<IUser, User>();
builder.Services.AddSingleton<IFeatureFlags, FeatureFlags>();
```

## Testing with Mock Services

### Unit Testing Entities

Use real Neatoo services, mock your application services:

```csharp
[TestClass]
public class PersonTests
{
    private IServiceProvider _serviceProvider;

    [TestInitialize]
    public void Setup()
    {
        var services = new ServiceCollection();

        // Register real Neatoo services
        services.AddNeatooServices(NeatooFactory.Server, typeof(IPerson).Assembly);

        // Mock application services
        var mockRule = new Mock<IUniqueNameRule>();
        mockRule.Setup(r => r.Execute(It.IsAny<IPerson>()))
                .Returns(RuleMessage.None);
        services.AddSingleton(mockRule.Object);

        _serviceProvider = services.BuildServiceProvider();
    }

    [TestMethod]
    public void Person_WithValidData_IsValid()
    {
        // Arrange
        var factory = _serviceProvider.GetRequiredService<IPersonFactory>();
        var person = factory.Create();

        // Act
        person.FirstName = "John";
        person.LastName = "Doe";

        // Assert
        Assert.IsTrue(person.IsValid);
    }
}
```

### Integration Testing with In-Memory Database

```csharp
[TestClass]
public class PersonIntegrationTests
{
    private IServiceProvider _serviceProvider;

    [TestInitialize]
    public void Setup()
    {
        var services = new ServiceCollection();

        // Register Neatoo
        services.AddNeatooServices(NeatooFactory.Server, typeof(IPerson).Assembly);

        // Use in-memory database
        services.AddDbContext<AppDbContext>(options =>
            options.UseInMemoryDatabase(Guid.NewGuid().ToString()));

        services.AddScoped<IAppDbContext>(sp =>
            sp.GetRequiredService<AppDbContext>());

        // Register real rules
        services.AddScoped<IUniqueNameRule, UniqueNameRule>();

        _serviceProvider = services.BuildServiceProvider();
    }

    [TestMethod]
    public async Task Person_Save_PersistsToDatabase()
    {
        // Arrange
        using var scope = _serviceProvider.CreateScope();
        var factory = scope.ServiceProvider.GetRequiredService<IPersonFactory>();
        var person = factory.Create();
        person.FirstName = "John";
        person.LastName = "Doe";

        // Act
        var saved = await factory.Save(person);

        // Assert
        var db = scope.ServiceProvider.GetRequiredService<IAppDbContext>();
        var entity = await db.Persons.FirstOrDefaultAsync();
        Assert.IsNotNull(entity);
        Assert.AreEqual("John", entity.FirstName);
    }
}
```

### Testing Authorization

```csharp
[TestClass]
public class PersonAuthTests
{
    [TestMethod]
    public void CanCreate_WhenAdmin_ReturnsTrue()
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
    public void CanCreate_WhenUser_ReturnsFalse()
    {
        // Arrange
        var user = new User { Role = Role.User };
        var auth = new PersonAuth(user);

        // Act
        var result = auth.HasCreate();

        // Assert
        Assert.IsFalse(result.IsAuthorized);
    }
}
```

### Testing Factory Authorization Integration

```csharp
[TestClass]
public class PersonFactoryAuthTests
{
    [TestMethod]
    public void Create_WhenUnauthorized_ReturnsNull()
    {
        // Arrange
        var services = new ServiceCollection();
        services.AddNeatooServices(NeatooFactory.Server, typeof(IPerson).Assembly);

        // Set up user as non-admin
        services.AddScoped<IUser>(_ => new User { Role = Role.User });
        services.AddScoped<IPersonAuth, PersonAuth>();

        var provider = services.BuildServiceProvider();
        var factory = provider.GetRequiredService<IPersonFactory>();

        // Act
        var person = factory.Create();

        // Assert
        Assert.IsNull(person);  // Unauthorized
    }

    [TestMethod]
    public void CanCreate_WhenUnauthorized_ReturnsFalseWithMessage()
    {
        // Arrange
        var services = new ServiceCollection();
        services.AddNeatooServices(NeatooFactory.Server, typeof(IPerson).Assembly);

        services.AddScoped<IUser>(_ => new User { Role = Role.User });
        services.AddScoped<IPersonAuth, PersonAuth>();

        var provider = services.BuildServiceProvider();
        var factory = provider.GetRequiredService<IPersonFactory>();

        // Act
        var result = factory.CanCreate();

        // Assert
        Assert.IsFalse(result.IsAuthorized);
        Assert.IsNotNull(result.Message);
    }
}
```

## Scoped Services and Request Lifetime

### Understanding Scopes

In ASP.NET Core, a scope is created per HTTP request. Services registered as `Scoped` share an instance within that request:

```csharp
// All resolve to same instance within a request
builder.Services.AddScoped<IUser, User>();

// In middleware
app.Use(async (context, next) =>
{
    var user = context.RequestServices.GetRequiredService<IUser>();
    user.UserId = context.User.FindFirst("sub")?.Value;
    // This same IUser instance is available to entities/rules/auth
    await next();
});
```

### Scope in Testing

Create scopes explicitly in tests:

```csharp
[TestMethod]
public async Task TestWithScope()
{
    using var scope = _serviceProvider.CreateScope();

    var user = scope.ServiceProvider.GetRequiredService<IUser>();
    user.Role = Role.Admin;  // Set up for this scope

    var factory = scope.ServiceProvider.GetRequiredService<IPersonFactory>();
    // Factory uses the same IUser instance
}
```

## Common DI Patterns

### Configuration-Based Registration

```csharp
// appsettings.json
{
  "Features": {
    "EnablePersonCreation": true
  }
}

// Registration
builder.Services.Configure<FeatureOptions>(
    builder.Configuration.GetSection("Features"));

builder.Services.AddScoped<IFeatureFlags>(sp =>
{
    var options = sp.GetRequiredService<IOptions<FeatureOptions>>();
    return new FeatureFlags(options.Value);
});
```

### Factory Pattern for Complex Dependencies

```csharp
// When rule needs runtime parameters
public interface IRuleFactory
{
    IUniqueNameRule CreateUniqueNameRule(Guid? excludeId);
}

public class RuleFactory : IRuleFactory
{
    private readonly IServiceProvider _sp;

    public RuleFactory(IServiceProvider sp) => _sp = sp;

    public IUniqueNameRule CreateUniqueNameRule(Guid? excludeId)
    {
        var dbContext = _sp.GetRequiredService<IDbContext>();
        return new UniqueNameRule(dbContext, excludeId);
    }
}
```

### Decorator Pattern

```csharp
// Wrap services with cross-cutting concerns
builder.Services.AddScoped<IPersonAuth, PersonAuth>();

builder.Services.Decorate<IPersonAuth>((inner, sp) =>
    new LoggingPersonAuth(inner, sp.GetRequiredService<ILogger<PersonAuth>>()));

public class LoggingPersonAuth : IPersonAuth
{
    private readonly IPersonAuth _inner;
    private readonly ILogger _logger;

    public LoggingPersonAuth(IPersonAuth inner, ILogger logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public Authorized HasCreate()
    {
        var result = _inner.HasCreate();
        _logger.LogInformation("HasCreate: {Result}", result.IsAuthorized);
        return result;
    }
    // ... other methods
}
```

## Troubleshooting DI Issues

### "Unable to resolve service for type..."

1. Verify the service is registered:
```csharp
builder.Services.AddScoped<IMyService, MyService>();
```

2. Check assembly is included in `AddNeatooServices()`:
```csharp
builder.Services.AddNeatooServices(NeatooFactory.Server, typeof(IMyEntity).Assembly);
```

3. For interface-to-implementation, ensure correct registration:
```csharp
// Wrong - registers concrete only
builder.Services.AddScoped<MyService>();

// Right - registers interface
builder.Services.AddScoped<IMyService, MyService>();
```

### "Circular dependency detected"

Avoid circular dependencies between services:

```csharp
// Problem: A depends on B, B depends on A
public class ServiceA { public ServiceA(IServiceB b) { } }
public class ServiceB { public ServiceB(IServiceA a) { } }

// Solutions:
// 1. Use Lazy<T>
public class ServiceA { public ServiceA(Lazy<IServiceB> b) { } }

// 2. Use method injection instead
public class ServiceA
{
    public void DoWork([Service] IServiceB b) { }
}

// 3. Refactor to break the cycle
```

### Service Lifetime Mismatch

Scoped services cannot be injected into Singleton services:

```csharp
// Wrong - IDbContext is scoped, but ICache is singleton
builder.Services.AddSingleton<ICache, Cache>();  // Cache takes IDbContext
builder.Services.AddScoped<IDbContext, AppDbContext>();

// Right - match lifetimes or use IServiceProvider
builder.Services.AddScoped<ICache, Cache>();
// Or
builder.Services.AddSingleton<ICache>(sp =>
    new Cache(() => sp.CreateScope().ServiceProvider.GetRequiredService<IDbContext>()));
```

## Related Topics

- [Client Setup](/setup/client/) - Client DI configuration
- [Server Setup](/setup/server/) - Server DI configuration
- [Authorization System Reference](/reference/authorization/) - Authorization DI patterns
- [Rules Engine Reference](/reference/rules/) - Rule injection patterns
