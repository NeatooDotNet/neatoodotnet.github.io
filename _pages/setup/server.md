---
title: Server Setup
layout: single
permalink: /setup/server/
---

To add Neatoo to ASP.NET, follow these steps:

1. Install the [Neatoo NuGet package](https://www.nuget.org/packages/Neatoo) in your client and server projects.
2. Register the Neatoo services

    ```csharp
    builder.Services.AddNeatooServices(NeatooFactory.Server, typeof(IPerson).Assembly);
    ```
    - NeatooFactory.Server signifies to execute all Factory Methods locally
    - Includes all libraries with Neatoo Entity classes in the list of assemblies

3. Add a single controller to your ASP.NET server application

    ```csharp
    app.MapPost("/api/neatoo", (HttpContext httpContext, RemoteRequestDto request) =>
    {
        var handleRemoteDelegateRequest = httpContext.RequestServices.GetRequiredService<HandleRemoteDelegateRequest>();
        return handleRemoteDelegateRequest(request);
    });
    ```

    - All Neatoo client to server calls are made to this endpoint however many entities and factories there are.
  
