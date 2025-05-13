---
title: "Factory"
layout: single
permalink: /overview/factory/
---

Neatoo [RemoteFactory](https://github.com/NeatooDotNet/RemoteFactory) features a 3-Tier Architecture where the Aggregate Entity Graph is automatically serialized between the client and the remote backend (i.e. "Server").

All instantiations of a Neatoo Entity must be done using a Factory.

## Roslyn Source Generator

For each Entity a Factory is live generated using a [Roslyn Source Generator](https://github.com/NeatooDotNet/RemoteFactory/blob/main/src/RemoteFactory.FactoryGenerator/FactoryGenerator.cs). As changes are made to the Entity they are reflected live in the corresponding Factory. 

## Name

The generated factory is matched to the Entity name with a 'Factory' postfix. If there is an interface matched by name (ex Entity -> IEntity) the factory returns the interface. For example, the factory for [Person](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/PersonModel.cs) which implements IPerson is [IPersonFactory](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.FactoryGenerator/DomainModel.PersonFactory.g.cs).

## Factory Methods

Factory Methods are the methods within an Entity that perform the Data Mapping to the persistance layer. They are decorated with a Factory Operation Attribute (see below).

Each Entity defines its Data Mapping. Parent Entities should use a Child Entities Factory to instantiate and persist the Child Entity.

## Parameters

All Factory Methods can take parameters. The Save operations (Insert, Update, Delete) require the Entity as a parameter.

### Dependency Injection

For Factory Methods Neatoo uses method injection. DI services can be injected by decorating a parameter with the [Service] attribute. These parameters must be after non-service parameters.

## Factory Operations
### Create
Instantiate a new Entity. 
The true Meta properties are IsNew and IsModified.

### Fetch
Instantiate an existing Entity.
Return false to indicate that the Entity could not be instantiated and return null to the caller. I.E. "Not Found"

### Insert
Persist an Entity with IsNew=True

### Update
Persist an Entity with IsNew=False

### Delete
Persist an Entity with IsDeleted=True

## Save

The Factory Operations Insert, Update and Delete are consolidated to a Save method in the Factory. Which operation is called is dependant on the Meta properties of the Entity.

## Remote

Factory Methods that must be called on the server are decorated with the [Remote] attribute.

## Serialization

Neatoo RemoteFactory uses System.Text.Json to do serialization. Serialization of the Entities is enhanced by Neatoo Remote Factory custom serializers. Serialization of Interfaces is supported even within lists. 

Neatoo [implements a reference resolver](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/preserve-references) so that instantiations are not duplicated.

### Dependency Injection

If registered, Neatoo custom serializers resolve new instances from the DI container.

### Value Objects

For Value Objects Neatoo RemoteFactory supports non-EntityBase classes. Simply decorate the class with a [Factory] attribute and define Factory Methods. Ex [ConstructorCreateObject](https://github.com/NeatooDotNet/RemoteFactory/blob/main/src/Tests/FactoryGeneratorTests/Factory/ConstructorCreateTests.cs). 

## Entity Lists

When an item is removed from an [EntityList](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Neatoo/EntityListBase.cs) it is marked IsDeleted=true and added to EntityList.DeletedList. Entities within EntityList.DeletedList need to be handled in the Insert/Update/Delete Factory Methods. Ex [PersonPhoneList](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/PersonPhoneModelList.cs).Update.