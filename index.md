---
layout: home
author_profile: false
sidebar:
    nav: "central"
---

[![Logo](https://raw.githubusercontent.com/NeatooDotNet/Neatoo/main/Logo_411.png 'Logo')](http://neatoo.net)

Neatoo is a DDD Aggregate Framework for .NET C# to create bindable and serializable Aggregate Entity Graphs for Blazor and WPF.
With Neatoo you define the business logic once and reuse it in both the UI and the Application Service. This gives you less code with more consistancy. 
Neatoo uniquely uses Roslyn Source Generators to create readable and performant code specific to the Entities. It creates 3-Tier Factories that transfers the Entities for you. 

**Write complex Aggregate Entity Graphs with no DTOs and only one controller!**

Features:
* Validation
* Authorization
* Bindable Properties
* Bindable Meta-Properties
* 3-Tier Factory
* Dependency Injection
* Async/Await
* Live Source Generation


[Discord](https://discord.gg/M3dVuZkG)
[Nuget](https://www.nuget.org/packages/Neatoo)

## Neatoo Lifecycle

This is a common lifecycle for a Neatoo Aggregate Entity Graph to and from the database using a 3-Tier infrastructure.

![Lifecycle](https://raw.githubusercontent.com/NeatooDotNet/Neatoo/main/AggregateLifecycle_960.gif)

## Video

This video highlights the validation, factory and authorization of Neatoo.

{% include video id="hvk9FcdYydQ" provider="youtube" %}