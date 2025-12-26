---
title: "Client Setup"
layout: single
permalink: /setup/client/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

This guide covers setting up the Neatoo client for Blazor WebAssembly applications. The client connects to your ASP.NET backend and enables entities to seamlessly travel between browser and server.

## NuGet Packages

Install these packages in your Blazor WebAssembly client project:

| Package | Purpose |
|---------|---------|
| [Neatoo](https://www.nuget.org/packages/Neatoo) | Core framework: EntityBase, ValidateBase, Base, Rules |
| [Neatoo.RemoteFactory](https://www.nuget.org/packages/Neatoo.RemoteFactory) | 3-tier factory infrastructure and serialization |
| [Neatoo.Blazor.MudNeatoo](https://www.nuget.org/packages/Neatoo.Blazor.MudNeatoo) | Optional: MudBlazor components with Neatoo integration |

```bash
dotnet add package Neatoo
dotnet add package Neatoo.RemoteFactory
dotnet add package Neatoo.Blazor.MudNeatoo  # Optional, if using MudBlazor
```

## Service Registration

Register Neatoo services in your `Program.cs` using `AddNeatooServices()`:

```csharp
builder.Services.AddNeatooServices(NeatooFactory.Remote, typeof(IPerson).Assembly);
```

The parameters:

- **`NeatooFactory.Remote`** - Configures factories to serialize Remote-decorated operations to the server. Operations without `[Remote]` execute locally on the client.
- **Assembly parameter(s)** - Pass all assemblies containing your Neatoo entities. The framework scans these to register generated factories and serializers.

## HttpClient Configuration

Neatoo requires a keyed `HttpClient` for server communication:

```csharp
builder.Services.AddKeyedScoped(
    Neatoo.RemoteFactory.RemoteFactoryServices.HttpClientKey,
    (sp, key) =>
    {
        return new HttpClient
        {
            BaseAddress = new Uri("https://localhost:5001/")
        };
    });
```

Key points:

- Use `RemoteFactoryServices.HttpClientKey` - Neatoo looks up the HttpClient using this specific key
- Set `BaseAddress` to your actual ASP.NET backend URL
- The client automatically posts to `/api/neatoo` on this base address

## Complete Program.cs Example

Here is a complete `Program.cs` for a Blazor WebAssembly client:

```csharp
using Microsoft.AspNetCore.Components.Web;
using Microsoft.AspNetCore.Components.WebAssembly.Hosting;
using Neatoo;
using Neatoo.RemoteFactory;
using MyApp.DomainModel;  // Your domain model assembly

var builder = WebAssemblyHostBuilder.CreateDefault(args);

builder.RootComponents.Add<App>("#app");
builder.RootComponents.Add<HeadOutlet>("head::after");

// Register Neatoo services with Remote factory mode
// Include all assemblies containing your Neatoo entities
builder.Services.AddNeatooServices(
    NeatooFactory.Remote,
    typeof(IPerson).Assembly,
    typeof(IOrder).Assembly  // Add more assemblies as needed
);

// Configure the HttpClient for Neatoo server communication
builder.Services.AddKeyedScoped(
    RemoteFactoryServices.HttpClientKey,
    (sp, key) =>
    {
        return new HttpClient
        {
            BaseAddress = new Uri(builder.HostEnvironment.BaseAddress)
        };
    });

// Register any additional application services
builder.Services.AddScoped<IMyCustomService, MyCustomService>();

await builder.Build().RunAsync();
```

## Blazor WebAssembly vs Blazor Server

### Blazor WebAssembly (Recommended)

Blazor WebAssembly runs your .NET code directly in the browser via WebAssembly. This is the recommended configuration for Neatoo because:

- **Local Rule Execution**: Validation and transformation rules run instantly in the browser
- **Responsive UI**: No server round-trip for property changes and validation
- **Reduced Server Load**: Only `[Remote]` operations hit the server
- **Offline Capable**: Basic functionality works without connectivity

Use `NeatooFactory.Remote` to serialize only remote-decorated operations to the server.

### Blazor Server

Blazor Server runs all .NET code on the server with SignalR maintaining UI state. With Neatoo:

- **All Operations Local**: Since code runs on the server, you might use `NeatooFactory.Server` instead of `Remote`
- **No Serialization Needed**: Entities stay in server memory
- **SignalR Dependency**: UI responsiveness depends on connection quality

For Blazor Server, you can configure as either:

```csharp
// Option 1: Local execution (all operations on the server)
builder.Services.AddNeatooServices(NeatooFactory.Server, typeof(IPerson).Assembly);

// Option 2: Remote mode (if using a separate API server)
builder.Services.AddNeatooServices(NeatooFactory.Remote, typeof(IPerson).Assembly);
```

## Shared Library Pattern

A common project structure places domain entities in a shared library referenced by both client and server:

```
MyApp.sln
├── MyApp.DomainModel/          # Shared library
│   ├── Person.cs               # EntityBase<Person>
│   ├── IPerson.cs              # Interface
│   └── UniqueNameRule.cs       # Rules
├── MyApp.BlazorClient/         # Blazor WASM
│   ├── Program.cs              # AddNeatooServices(NeatooFactory.Remote, ...)
│   └── Pages/
├── MyApp.Server/               # ASP.NET Core
│   ├── Program.cs              # AddNeatooServices(NeatooFactory.Server, ...)
│   └── Data/                   # EF Core DbContext
└── MyApp.Ef/                   # Optional: EF entities
    └── PersonEntity.cs         # Database model
```

This structure ensures:
- Single source of truth for domain logic
- Same entity classes on both client and server
- Source generators create factories in the shared library

## Authentication Token Forwarding

For authenticated requests, configure the HttpClient to include authorization headers:

```csharp
builder.Services.AddKeyedScoped(
    RemoteFactoryServices.HttpClientKey,
    (sp, key) =>
    {
        var client = new HttpClient
        {
            BaseAddress = new Uri("https://localhost:5001/")
        };

        // Add authentication header if using token-based auth
        var tokenProvider = sp.GetService<IAccessTokenProvider>();
        if (tokenProvider != null)
        {
            // Configure token handling as appropriate for your auth setup
        }

        return client;
    });
```

For more sophisticated authentication scenarios, consider using `IHttpClientFactory` with a named or typed client that handles token refresh and retry logic.

## Troubleshooting

### "No factory found for type X"

Ensure:
1. The assembly containing the entity is passed to `AddNeatooServices()`
2. The entity class is `partial` and inherits from `EntityBase<T>`
3. The project has the Neatoo.RemoteFactory source generator referenced

### HttpClient not configured

If you see errors about missing HttpClient:
1. Verify you registered the keyed HttpClient with `RemoteFactoryServices.HttpClientKey`
2. Check that the registration uses `AddKeyedScoped` (not `AddScoped`)

### Serialization errors

If entities fail to serialize:
1. Ensure all entity properties are serializable types
2. Check that child entities also inherit from Neatoo base classes
3. Verify interfaces are correctly matched to implementations

## Next Steps

- [Server Setup](/setup/server/) - Configure the ASP.NET backend
- [Entity Overview](/overview/entity/) - Learn about EntityBase
- [Factory Overview](/overview/factory/) - Understand the factory pattern
