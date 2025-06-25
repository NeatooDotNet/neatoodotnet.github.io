---
layout: home
author_profile: false
sidebar:
    nav: "central"
---

[![Logo](https://raw.githubusercontent.com/NeatooDotNet/Neatoo/main/Logo_411.png 'Logo')](http://neatoo.net)

Neatoo is a DDD framework for .NET Blazor and WPF applications. Entities are transferred, linked, tracked and validated by Neatoo allowing developers to focus on their business behavior. Using data-binding for logical separation the Entities are used on the client as well as the server providing a rich user experience with no duplication of logic. The rules engine supports complex business logic, validation, data-binding and unit testability.

**With Neatoo you can build a DDD application with less effort than ever before!**

[Discord](https://discord.gg/M3dVuZkG)
[Nuget](https://www.nuget.org/packages/Neatoo)

# Features

* Entities
    * Root/Child object linking
    * Child PropertyChanged events propagate up the Object Graph
    * Rules Engine
    * Bindable Meta Properties
        * Validation: IsValid, IsBusy
        * Change Tracking: IsNew, IsModified, IsDeleted
    * Data-Mapper methods that are automatically called by the Factory
    * Each Entity Property has their own Bindable-Meta properties
        * Ex. Ability to know if an individual Entity Property has been modified
* Rules
    * Separate Rule classes to transform and validate your Entities
    * Limitless complexity
    * Execution triggered on Entity Property Changed
    * Full cascading support
    * Full Async Support
    * Unit Testable
* Factory
    * Neatoo moves the Entity’s state between Client and Server
    * Authorization
        * Custom Authorization and Asp authorization supported
    * Data-Mapping 
        * Correct Entity Data-Mapping method called based on Entitie’s change tracking meta properties
    * Powered by Roslyn Source Generators
        * Factory methods automatically change with the Data-Mapper definitions
* Dependency Injection
    * Neatoo is Dependency Injection centric
    * Supports Constructor Injection
    * Data-Mapper Method Injection Pattern
        * Disposable server-side only services are injected specifically where they are needed
    * Fully Extensible
        * None of the framework components are static or sealed. All are resolved as an service interface. Thus, the framework can be extended by providing a different implementation for a service interface.
* Unit Testing
    * Fully decoupled unit testing supported via constructor injection and full interface support
    * Entities can reference other Entities interfaces supporting de-coupling
    * Factories are interfaces and can implement Entity interfaces
    * Rules are injectable and can be linked to the Entity interface

# When to use Neatoo

* You are new to DDD.
* Your client is Blazor or WPF.
* You have complex business behavior
* Currently your business logic is spread out between your UI, controllers/view models and database code.
* Currently you look at the ORM to understand the Domain 
    * This can be described a number of ways like using the ORM within your controllers or the endpoints mimic CRUD operations
    * If DTOs are used for 3-Tier it may be an Anemic Domain Model. 
    * This is a trap many applications have fallen into.
* The Anemica Domain Model or Object–relational impedance mismatch principles are unfamiliar
* You have an Anemic Domain Model 
    * These only work in the simplest of cases. Article
* You like CSLA

# When NOT to use Neatoo

* Business behavior is not the major concern of the application
* You need a high transaction throughput system
* Many users will be editing the same data
* The application has many UI technologies and/or API endpoints

# Pros
* Build an DDD application very fast
* Go well beyond simple attribute Validation with complex Rules that are Unit Testable
* UI awareness of all validation and authorization makes a rich user experience easier than ever
* No duplication of the Business Logic or Authorization between the Client and Server
* Clear architecture and pattern for Aggregate, Entities and Rules
* Clear pattern for Unit Testing everything
* Clear and consistent implementation for Authorization
* Clearly defined Data-Mapper implementation

# Cons
* Rigid architecture
* Coupling to the Neatoo Base classes
* Not an Event based architecture

# Why is Neatoo New?

Neatoo uses established DDD principles and implements them with two game changing technologies: Blazor WebAssembly and Roslyn Live Source Generation.

## Blazor and WebAssembly

WebAssembly means having our .NET libraries with their defined Entities and their behavior *in the browser*.
This was not possible with a Javascript SPA application with a C# backend.
Now there's no longer the need to define individual endpoints for each behavior.
The client already has the Entities with their behavior.
All that needs to be transferred is the Entity data.

## Roslyn Source Generators

Neatoo uniquely uses Roslyn Source Generators to create a readable and performant Factory *specific* to each Entity. 
The 3-Tier Factories transfers the Entity Data for you. 
Authorization is implemented within the Factory.
Before Source Generators frameworks had to rely on reflection.
The end result wasn't known until execution.
Now many runtime errors are compile errors and there is code you can see and step thru!


## Neatoo Lifecycle

This is a common lifecycle for a Neatoo Aggregate Entity Graph to and from the database using a 3-Tier infrastructure.

![Lifecycle](https://raw.githubusercontent.com/NeatooDotNet/Neatoo/main/AggregateLifecycle_960.gif)

## Video

This video highlights the validation, factory and authorization of Neatoo.

{% include video id="hvk9FcdYydQ" provider="youtube" %}
