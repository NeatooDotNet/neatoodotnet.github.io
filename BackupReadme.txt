![Lifecycle](https://raw.githubusercontent.com/NeatooDotNet/Neatoo/main/Logo_411.png)

Neatoo is a DDD Aggregate Framework for .NET C#. It specializes in creating bindable and serializable Aggregate Entity Graphs for Blazor and WPF.
The goal is to define the business logic once and reuse it in both the UI and the Application Service. 
Neatoo uniquely uses Roslyn Source Generators to create readable and performant code specific to the Entities.

Features:
* Validation
* Authorization
* Bindable Properties
* Bindable Meta-Properties
* Factory
* Serialization
* Dependency Injection
* Async/Await
* Live Source Generation

#### With Neatoo you can create Blazor Business applications with much less code!

[Discord](https://discord.gg/M3dVuZkG)
[Nuget](https://www.nuget.org/packages/Neatoo)

## Neatoo Lifecycle

This is a common lifecycle for a Neatoo Aggregate Entity Graph to and from the database using a 3-Tier infrastructure.

![Lifecycle](https://raw.githubusercontent.com/NeatooDotNet/Neatoo/main/AggregateLifecycle_960.gif)

Neatoo does not limit you to a database. Any Service can be injected to the Factory Methods on the Application Server.

## Video

[![Introduction](https://raw.githubusercontent.com/NeatooDotNet/Neatoo/main/youtubetile.jpg)](https://youtu.be/hvk9FcdYydQ?si=ExSVX93Iebm2TBCz)

## Example

Please explore [the Person example](https://github.com/NeatooDotNet/Neatoo/tree/main/src/Examples/Person) Blazor Stand-Alone Web application. 

* Entities
  - [Person](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Person.cs) and [PersonPhone](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/PersonPhone.cs)
* Validation
  - [UniqueNameRule](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/UniqueNameRule.cs), [UniquePhoneNumberRule](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/UniquePhoneNumberRule.cs) and [UniquePhoneTypeRule](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/UniquePhoneTypeRule.cs)
* Bindable Meta-properties
  - Provided by [EntityBase](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Neatoo/EntityBase.cs) also see [IMetaProperties](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Neatoo/IMetaProperties.cs)
* Authorization
  - [PersonAuth](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/PersonAuth.cs)
* Factory
  - These Roslyn Source Generated Factories to call [Create], [Fetch], [Insert], [Update] and [Delete] methods in [Person](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Person.cs) based on it's meta-state. 
  - [PersonFactory](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.FactoryGenerator/Person.DomainModel.PersonFactory.g.cs), [PersonPhoneFactory](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.FactoryGenerator/Person.DomainModel.PersonPhoneFactory.g.cs), [PersonPhoneListFactory](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.FactoryGenerator/Person.DomainModel.PersonPhoneFactory.g.cs) and [UniqueNameRuleFactory](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.FactoryGenerator/Person.DomainModel.UniqueNameFactory.g.cs)

* Serialization
  - Serialized at the AggregateRoot level of Person as signified by the [Remote] Attribute on the Factory Methods of [Person](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Person.cs)

This is an animation of the application in action:

<img src="https://raw.githubusercontent.com/NeatooDotNet/Neatoo/main/NeatooPersonRules.gif" width=30% height=30%>

## Event-Driven Architecture

Neatoo is specialized to defining Aggregate Entity Graphs in a single Bounded Context. It optimizes complex screens by reusing the business logic and validation on the client and server. However, there's no reason to limit Neatoo Aggregates to an Application UI. The Aggregates can be hydrated with incoming requests or messages, perform validation, and react in the same way as if it has been a user interacting in the UI. 

A DDD principal that coincides with Event-Driven Architecture are Domain Events. A possible senario is for Neatoo Aggregate would raise a Domain Event in the Factory Operation (Insert/Update/Delete). Handling the Domain Event between Bounded Contexts can be done with any number of microservice architecture like NServiceBus.

A Bounded Context without a UI application may not benefit from Neatoo. There's no data-binding UI interaction so the validation does not need to be property change driven. I still strongly encourge a well-formed Domain Model with Aggregate Entity Graphs and something like Fluent Validator.

While Neatoo Aggregate Entities are distributable in that they are serializable I would caution on that approach. Sharing a library with Neatoo Aggregate Entities between applications or microservices re-introduces monolith concerns like versioning and teams waiting on teams. It may be a good POC start but a system should quickly evolve to an Event-Driven Architecture. 

Side note: Do not use the database to communicate between applications. This creates numerous difficulties including performance scaling and cross-team difficulties. Changing a database schema is difficult and the difficulty grows exponetially with dependency. A sign that this is being done is one application updates a record and another application has a background job to react to record changes. Event-Drive Architecture is the optimal way to architect those cases.

##  Please leave feedback!
Thus far, this was a fun winter project. If there is any interest to use it I will continue to work on it. So, please provide feedback!

[Please let me know](https://github.com/NeatooDotNet/Neatoo/issues):
- Does a framework like this already exists (besides [CSLA](https://cslanet.com/))?
- Would this be useful to you?
- What is useful/good?
- What is missing/bad?

Please be constructive.

#### [Discord](https://discord.gg/M3dVuZkG)

## Recommended Reading

Ideas that shaped Neatoo:
- Of course [Implementing Domain-Driven Design by Vaughn Vernon](https://www.amazon.com/Implementing-Domain-Driven-Design-Vaughn-Vernon/dp/0321834577/ref=asc_df_0321834577?mcid=2d57f9b4826b30adbc0f024ba5ffcee1&hvocijid=13025708246257480970-0321834577-&hvexpln=73&tag=hyprod-20&linkCode=df0&hvadid=721245378154&hvpos=&hvnetw=g&hvrand=13025708246257480970&hvpone=&hvptwo=&hvqmt=&hvdev=c&hvdvcmdl=&hvlocint=&hvlocphy=1019976&hvtargid=pla-2281435177658&psc=1)
- The rules engine is a replica of [CSLA by Rockford Lhotka](https://store.lhotka.net/).
- In addition to DDD why to have Domain Model (110) & Data Mapper (165) is in [Patterns of Enteprise Architecture](https://www.thriftbooks.com/w/patterns-of-enterprise-application-architecture_martin-fowler_david-rice/250298/?resultid=dcd84f2b-51ab-4e22-8e24-3c3a17de30bb#edition=3682851&idiq=4316361) - 
- A key idea every developer should understand is [Object–relational impedance mismatch](https://en.wikipedia.org/wiki/Object%E2%80%93relational_impedance_mismatch)
- Everyone warns about [Anemic Domain Models](https://martinfowler.com/bliki/AnemicDomainModel.html) [over](https://x.com/VaughnVernon/status/1234990953099751425?lang=en) and [over](https://www.youtube.com/watch?v=aLFMJ_frafg) and [over](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/microservice-domain-model#rich-domain-model-versus-anemic-domain-model) again
- [Async Programming : Patterns for Asynchronous MVVM Applications: Data Binding](https://learn.microsoft.com/en-us/archive/msdn-magazine/2014/march/async-programming-patterns-for-asynchronous-mvvm-applications-data-binding) is why each property has bindable meta-state
- [Extending Partial Methods](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-9.0/extending-partial-methods) and [Partial Properties](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-13.0/partial-properties) is a key Roslyn Source Generator concept
- The best articles I've found on [writing source generators](https://andrewlock.net/series/creating-a-source-generator/)