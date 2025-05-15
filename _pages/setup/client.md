---
title: Client Setup
layout: single
permalink: /setup/client/
---

To get started with Neatoo, follow these steps:

1. Install the [Neatoo NuGet package](https://www.nuget.org/packages/Neatoo) in your client project

2. Register the Neatoo services
	
    ```csharp

    builder.Services.AddNeatooServices(NeatooFactory.Remote, typeof(IPersonModel).Assembly);
    builder.Services.AddKeyedScoped(Neatoo.RemoteFactory.RemoteFactoryServices.HttpClientKey, (sp, key) => {
            return new HttpClient { BaseAddress = new Uri("http://localhost/") };
    });
    ```
    - NeatooFactory.Remote signifies to serialize Remote calls to the ASP.NET backend (server)
    - Set BaseAddress to the actual URL of the ASP.NET backend for the application