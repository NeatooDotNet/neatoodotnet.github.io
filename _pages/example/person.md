---
title: Person Example
layout: single
permalink: /example/person/
---

[Github](https://github.com/NeatooDotNet/Neatoo/tree/main/src/Examples/Person)

Person is a Blazor Standalone Application. For now, it is located in the main Neatoo solution. It is an example of modifying an DDD Aggregate composed of a parent entity with a child list. It shows how to do CRUD, validation, authorization.

Some complexity is included including validation that [involves a trip to the database](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/UniqueNameRule.cs) and [unique values in the list](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/UniquePhoneNumberRule.cs).

## Code

* Entities
  - [Person](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/PersonModel.cs) and [PersonPhoneModel](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/PersonPhoneModel.cs)
* Validation
  - [UniqueNameRule](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/UniqueNameRule.cs), [UniquePhoneNumberRule](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/UniquePhoneNumberRule.cs) and [UniquePhoneTypeRule](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/UniquePhoneTypeRule.cs)
* Bindable Meta-properties
  - Provided by [EntityBase](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Neatoo/EntityBase.cs) also see [IMetaProperties](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Neatoo/IMetaProperties.cs)
* Authorization
  - [PersonModelAuth](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/PersonModelAuth.cs)
* 3-Tier Factory
  - A Roslyn Source Generated Factory to call [Create], [Fetch], [Insert], [Update] and [Delete] methods in [Person](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/PersonModel.cs) based on meta-state. 
  - [PersonModelFactory](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.FactoryGenerator/Person.DomainModel.PersonModelFactory.g.cs), [PersonModelPhoneFactory](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.FactoryGenerator/Person.DomainModel.PersonPhoneModelFactory.g.cs), [PersonModelPhoneListFactory](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.FactoryGenerator/Person.DomainModel.PersonPhoneModelFactory.g.cs) and [UniqueNameRuleFactory](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.FactoryGenerator/Person.DomainModel.UniqueNameFactory.g.cs)

## Animation

This is an animation of the application in action:

<img src="https://raw.githubusercontent.com/NeatooDotNet/Neatoo/main/NeatooPersonRules.gif" width="60%" />

## Videos

{% include video id="FeS1umd3O1Y" provider="youtube" %}
