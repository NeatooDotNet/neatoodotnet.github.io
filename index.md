---
layout: home
author_profile: false
sidebar:
    nav: "central"
---

[![Logo](https://raw.githubusercontent.com/NeatooDotNet/Neatoo/main/Logo_411.png 'Logo')](http://neatoo.net)

Neatoo is a DDD Aggregate Framework for .NET C# to create bindable and serializable Aggregate Entity Graphs for Blazor and WPF.
With Neatoo you define the business logic once and reuse it in both the UI and the Application Service. Allowing you to do more with less code while maintaining consistency. 

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

Neatoo is for applications with complex business logic. 
A robust user experience is delivered by the bindable properties, property messages and authorization check methods.
The user knows exactly what they can do, what they need to do to save and that the save will succeed!

Neatoo uniquely uses Roslyn Source Generators to create a readable and performant Factory specific to each Entity. 
The 3-Tier Factories transfers the Entity for you. Authorization is implemented with the Factory.
Before Source Generators frameworks had to rely on reflection.
The end result wasn't known until execution.
Now the end result is code you can see and step thru!

[Discord](https://discord.gg/M3dVuZkG)
[Nuget](https://www.nuget.org/packages/Neatoo)

## Neatoo Lifecycle

This is a common lifecycle for a Neatoo Aggregate Entity Graph to and from the database using a 3-Tier infrastructure.

![Lifecycle](https://raw.githubusercontent.com/NeatooDotNet/Neatoo/main/AggregateLifecycle_960.gif)

## Video

This video highlights the validation, factory and authorization of Neatoo.

{% include video id="hvk9FcdYydQ" provider="youtube" %}