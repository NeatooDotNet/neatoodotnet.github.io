---
title: "Server Setup"
layout: single
permalink: /setup/server/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

This guide covers setting up the ASP.NET Core backend for Neatoo applications. The server handles remote factory operations, persistence, and serves as the authoritative tier for validation and authorization.

## NuGet Packages

Install these packages in your ASP.NET Core server project:

| Package | Purpose |
|---------|---------|
| [Neatoo](https://www.nuget.org/packages/Neatoo) | Core framework: EntityBase, ValidateBase, Base, Rules |
| [Neatoo.RemoteFactory](https://www.nuget.org/packages/Neatoo.RemoteFactory) | 3-tier factory infrastructure and remote request handling |

```bash
dotnet add package Neatoo
dotnet add package Neatoo.RemoteFactory
```

Your server project should also reference your shared domain model library (containing your Neatoo entities) and any data access libraries (e.g., EF Core DbContext).

## Service Registration

Register Neatoo services in your `Program.cs` using `AddNeatooServices()`:

```csharp
builder.Services.AddNeatooServices(NeatooFactory.Server, typeof(IPerson).Assembly);
```

The parameters:

- **`NeatooFactory.Server`** - Configures factories to execute all operations locally on the server. This is required for handling remote requests from clients.
- **Assembly parameter(s)** - Pass all assemblies containing your Neatoo entities. The framework scans these to register generated factories and serializers.

## The Neatoo Endpoint

Neatoo uses a single POST endpoint to handle all remote factory operations. Add this endpoint to your `Program.cs`:

```csharp
app.MapPost("/api/neatoo", (HttpContext httpContext, RemoteRequestDto request) =>
{
    var handleRemoteDelegateRequest = httpContext.RequestServices
        .GetRequiredService<HandleRemoteDelegateRequest>();
    return handleRemoteDelegateRequest(request);
});
```

This single endpoint:
- Receives serialized entities and operation metadata from the client
- Deserializes using Neatoo's custom serializers
- Routes to the appropriate factory method (Create, Fetch, Insert, Update, Delete)
- Returns the result serialized back to the client

No matter how many entities and factories you have, everything flows through this one endpoint.

## EF Core Integration

Register your EF Core DbContext as you normally would:

```csharp
// Register your DbContext
builder.Services.AddDbContext<PersonDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Register the DbContext interface used by your entities
builder.Services.AddScoped<IPersonDbContext>(sp => sp.GetRequiredService<PersonDbContext>());
```

Your factory methods inject the DbContext via the `[Service]` attribute:

```csharp
[Remote]
[Fetch]
public async Task<bool> Fetch(
    [Service] IPersonDbContext dbContext,
    [Service] IPersonPhoneListFactory phoneListFactory)
{
    var personEntity = await dbContext.Persons.FindAsync(Id);
    if (personEntity == null) return false;

    MapFrom(personEntity);
    return true;
}
```

## CORS Configuration

For Blazor WebAssembly clients hosted on a different origin, configure CORS:

```csharp
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins("https://localhost:5002")  // Your Blazor client URL
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});
```

Apply CORS in the middleware pipeline:

```csharp
app.UseCors();
```

For development, you might use a more permissive policy:

```csharp
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});
```

## Complete Program.cs Example

Here is a complete `Program.cs` for an ASP.NET Core server:

```csharp
using Microsoft.EntityFrameworkCore;
using Neatoo;
using Neatoo.RemoteFactory;
using MyApp.DomainModel;
using MyApp.Ef;

var builder = WebApplication.CreateBuilder(args);

// Configure CORS for Blazor WASM client
builder.Services.AddCors(options =>
{
    options.AddDefaultPolicy(policy =>
    {
        policy.WithOrigins(builder.Configuration["ClientUrl"] ?? "https://localhost:5002")
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

// Register Neatoo services with Server factory mode
// Include all assemblies containing your Neatoo entities
builder.Services.AddNeatooServices(
    NeatooFactory.Server,
    typeof(IPerson).Assembly,
    typeof(IOrder).Assembly  // Add more assemblies as needed
);

// Register EF Core DbContext
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Register DbContext interfaces for injection into factory methods
builder.Services.AddScoped<IPersonDbContext>(sp => sp.GetRequiredService<AppDbContext>());

// Register additional application services
builder.Services.AddScoped<IEmailService, EmailService>();

var app = builder.Build();

// Apply CORS middleware
app.UseCors();

// Add authentication/authorization if needed
app.UseAuthentication();
app.UseAuthorization();

// The single Neatoo endpoint handles all remote factory operations
app.MapPost("/api/neatoo", (HttpContext httpContext, RemoteRequestDto request) =>
{
    var handleRemoteDelegateRequest = httpContext.RequestServices
        .GetRequiredService<HandleRemoteDelegateRequest>();
    return handleRemoteDelegateRequest(request);
});

app.Run();
```

## Authentication and Authorization

### Integrating ASP.NET Core Identity

For authenticated endpoints, add authentication middleware before the Neatoo endpoint:

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

### Passing User Context to Neatoo

To make user information available to your authorization classes, register a user service and populate it from the request:

```csharp
// Define a user interface for your domain
public interface IUser
{
    string? UserId { get; set; }
    string? Role { get; set; }
}

public class User : IUser
{
    public string? UserId { get; set; }
    public string? Role { get; set; }
}

// Register the user service
builder.Services.AddScoped<IUser, User>();

// Populate user from request context with middleware
app.Use(async (context, next) =>
{
    var user = context.RequestServices.GetRequiredService<IUser>();

    // Populate from claims, headers, or other source
    if (context.User.Identity?.IsAuthenticated == true)
    {
        user.UserId = context.User.FindFirst("sub")?.Value;
        user.Role = context.User.FindFirst("role")?.Value;
    }

    await next();
});
```

Your authorization classes can then inject `IUser`:

```csharp
public class PersonAuth : IAuthorization
{
    private readonly IUser _user;

    public PersonAuth(IUser user)
    {
        _user = user;
    }

    [Authorize(AuthorizeOperation.Create)]
    public Authorized CanCreate()
    {
        return _user.Role == "Admin"
            ? Authorized.Yes
            : Authorized.No("Only administrators can create persons");
    }
}
```

## Convention-Based Registration

Neatoo provides `RegisterMatchingName()` for convention-based service registration. This automatically registers services where the implementation name matches the interface name (minus the "I" prefix):

```csharp
builder.Services.RegisterMatchingName(
    typeof(IPerson).Assembly,
    ServiceLifetime.Scoped
);
```

This would register:
- `IPersonRule` -> `PersonRule`
- `IEmailService` -> `EmailService`
- etc.

## Middleware Pipeline Order

The order of middleware matters. Here is a recommended order:

```csharp
var app = builder.Build();

// 1. Exception handling (development vs production)
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/error");
}

// 2. HTTPS redirection
app.UseHttpsRedirection();

// 3. CORS (must be before endpoints)
app.UseCors();

// 4. Authentication and Authorization
app.UseAuthentication();
app.UseAuthorization();

// 5. Custom middleware (user context, etc.)
app.Use(async (context, next) =>
{
    // Populate user context
    await next();
});

// 6. Endpoints
app.MapPost("/api/neatoo", (HttpContext httpContext, RemoteRequestDto request) =>
{
    var handleRemoteDelegateRequest = httpContext.RequestServices
        .GetRequiredService<HandleRemoteDelegateRequest>();
    return handleRemoteDelegateRequest(request);
});

app.Run();
```

## Troubleshooting

### "No factory found for type X"

Ensure:
1. The assembly containing the entity is passed to `AddNeatooServices()`
2. The server uses `NeatooFactory.Server` (not `Remote`)
3. The entity class is `partial` and inherits from `EntityBase<T>`

### Serialization errors

If deserialization fails:
1. Verify client and server reference the same domain model assembly version
2. Check that all entity properties are serializable
3. Ensure custom types have appropriate serializers registered

### DbContext not found

If factory methods fail to resolve DbContext:
1. Verify DbContext is registered with `AddDbContext<T>()`
2. Register any interface aliases: `AddScoped<IMyDbContext>(sp => sp.GetRequiredService<MyDbContext>())`

### CORS errors

If the browser blocks requests:
1. Verify CORS is configured in services: `builder.Services.AddCors(...)`
2. Apply CORS middleware: `app.UseCors()`
3. Ensure the client origin is in the allowed origins list
4. Check that CORS middleware is before the endpoint mapping

## Next Steps

- [Client Setup](/setup/client/) - Configure the Blazor client
- [Factory Overview](/overview/factory/) - Learn about factory operations
- [Authorization Overview](/overview/authorization/) - Implement access control
