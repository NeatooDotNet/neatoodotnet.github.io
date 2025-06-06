---
title: Person Example
layout: single
permalink: /example/person/
---

[Github](https://github.com/NeatooDotNet/Neatoo/tree/main/src/Examples/Person)

Person is a Blazor Standalone Application. For now, it is located in the main Neatoo solution. It is an example of modifying an DDD Aggregate composed of a parent entity with a child list. It shows how to do CRUD, validation, authorization.

Some complexity is included including validation that [involves a trip to the database](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/UniqueNameRule.cs) and [unique values in the list](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/UniquePhoneNumberRule.cs).

## Code

* [Entities](https://github.com/NeatooDotNet/Neatoo/tree/main/src/Examples/Person/Person.DomainModel)
  - [Person](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Person.cs) and [PersonPhone](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/PersonPhone.cs)
* Validation
  - [UniqueNameRule](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/UniqueNameRule.cs), [UniquePhoneNumberRule](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/UniquePhoneNumberRule.cs) and [UniquePhoneTypeRule](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/UniquePhoneTypeRule.cs)
* [Entity 3-Tier Factories](https://github.com/NeatooDotNet/Neatoo/tree/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.FactoryGenerator)
  - [PersonFactory](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.FactoryGenerator/DomainModel.PersonFactory.g.cs), [PersonPhoneFactory](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.FactoryGenerator/DomainModel.PersonPhoneFactory.g.cs), [PersonPhoneListFactory](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.FactoryGenerator/DomainModel.PersonPhoneFactory.g.cs) and [UniqueNameRuleFactory](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.FactoryGenerator/DomainModel.UniqueNameFactory.g.cs)
  - These are all live source generation
* Bindable Meta-properties
  - Provided by [EntityBase](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Neatoo/EntityBase.cs) also see [IMetaProperties](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Neatoo/IMetaProperties.cs)
* Authorization
  - [PersonAuth](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/PersonAuth.cs)


## Animation

This is an animation of the application in action:

<img src="https://raw.githubusercontent.com/NeatooDotNet/Neatoo/main/NeatooPersonRules.gif" width="60%" />

## Videos

{% include video id="FeS1umd3O1Y" provider="youtube" %}
