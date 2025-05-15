---
title: "Factory"
layout: single
permalink: /overview/factory/
---

Neatoo [RemoteFactory](https://github.com/NeatooDotNet/RemoteFactory) features a 3-Tier Architecture where the Aggregate Entity Graph is automatically serialized between the client and the remote backend (i.e. "Server"). Authorization is implemented in the Factories.

All instantiations of a Neatoo Entity must be done using a Factory.

## Example

Here is an [example](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.FactoryGenerator/DomainModel.PersonFactory.g.cs) Factory Service definition

``` csharp
    public interface IPersonFactory
    {
        IPerson? Create();
        Task<IPerson?> Fetch();
        Task<IPerson?> Save(IPerson target);
        Task<Authorized<IPerson>> TrySave(IPerson target);
        Authorized CanCreate();
        Authorized CanFetch();
        Authorized CanInsert();
        Authorized CanUpdate();
        Authorized CanDelete();
        Authorized CanSave();
    }
```

## Roslyn Source Generator

For each Entity a Factory is live generated using a [Roslyn Source Generator](https://github.com/NeatooDotNet/RemoteFactory/blob/main/src/RemoteFactory.FactoryGenerator/FactoryGenerator.cs). As changes are made to the Entity they are reflected live in the corresponding Factory. 

## Name

The generated factory is matched to the Entity name with a 'Factory' postfix. If there is an interface matched by name (ex Entity -> IEntity) the factory returns the interface. For example, the factory for [Person](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Person.cs) which implements IPerson is [IPersonFactory](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.FactoryGenerator/DomainModel.PersonFactory.g.cs).

## Factory Methods

Factory Methods are the methods within an Entity that perform the Data Mapping to the persistance layer. They are decorated with a Factory Operation Attribute (see below).

``` csharp
    [Create]
    public void Create([Service] IPersonPhoneList PersonPhoneList) {}

    [Remote]
    [Fetch]
    public async Task<bool> Fetch([Service] IPersonDbContext personContext,
                                    [Service] IPersonPhoneListFactory PersonPhoneListFactory) {}

    [Remote]
    [Insert]
    public async Task<PersonEntity?> Insert([Service] IPersonDbContext personContext,
                                    [Service] IPersonPhoneListFactory PersonPhoneListFactory) {}

    [Remote]
    [Update]
    public async Task<PersonEntity?> Update([Service] IPersonDbContext personContext,
                                    [Service] IPersonPhoneListFactory PersonPhoneListFactory) {}

    [Remote]
    [Delete]
    public async Task Delete([Service] IPersonDbContext personContext) {}

```

- Each Entity defines its Data Mapping. Parent Entities should use a Child Entities Factory to instantiate and persist the Child Entity.
  - As is done in the above code with IPersonPhoneListFactory

## Parameters

All Factory Methods can take parameters.
When a parameter is added to an Entity's Factory Method the live source generation automatically updates the Factory.
The Save operations (Insert, Update, Delete) require the Entity as a parameter.

### Dependency Injection

For Factory Methods Neatoo uses method injection. DI services can be injected by decorating a parameter with the [Service] attribute. All non-service parameters must proceed [Service] parameters.

## Factory Operations
### Create
Instantiate a new Entity. IsNew=true

### Fetch
Instantiate an existing Entity.

Return false to indicate that the Entity could not be instantiated and return null to the caller. I.E. "Not Found"

### Insert
Call on Factory.Save on Entities with IsNew=True

### Update
Call on Factory.Save on Entities with IsNew=False

### Delete
Call on Factory.Save on Entities with IsDeleted=true

## Save

The Factory Operations Insert, Update and Delete are consolidated to a Save method in the Factory. Which Factory Method is called is dependant on the Meta properties of the Entity.

## Remote

Factory Methods that must be called on the server are decorated with the [Remote] attribute.

## Serialization

Neatoo RemoteFactory uses System.Text.Json to do serialization. Serialization of the Entities is enhanced by Neatoo Remote Factory custom serializers. Serialization of Interfaces is supported even within lists. 

Neatoo [implements a reference resolver](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/preserve-references) so that instantiations are not duplicated.

### Dependency Injection

If registered, Neatoo custom serializers resolve new instances from the DI container.

### Value Objects

For Value Objects Neatoo RemoteFactory supports non-EntityBase classes. Simply decorate the class with a [Factory] attribute and define Factory Methods. Ex [ConstructorCreateObject](https://github.com/NeatooDotNet/RemoteFactory/blob/main/src/Tests/FactoryGeneratorTests/Factory/ConstructorCreateTests.cs). 

