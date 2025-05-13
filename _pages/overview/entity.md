---
title: "Entity"
layout: single
permalink: /overview/entity/
---

This is a brief overview of Neatoo Entity features using the [Person](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/PersonModel.cs) entity. More in-depth pages are in the works for each feature.

## Class Declaration

``` csharp
internal partial class Person : IdEntityBase<Person>, IPerson
```

- Entities inherit from Neatoo.EntityBase<T> defining themself as T
  - [IdEntityBase](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/IdEntityBase.cs) inherits from Neatoo.EntityBase. It contains shared logic.
- Partial is required for Neatoo to generate property definitions and MapTo/MapFrom/etc methods


## Constructor

``` csharp
    public Person(IEntityBaseServices<Person> editBaseServices,
                  IUniqueNameRule uniqueNameRule) : base(editBaseServices)
    {
        RuleManager.AddRule(uniqueNameRule);
    }
```

- Depedency Injection using [Microsoft.Extension.DependencyInjection](https://www.nuget.org/packages/microsoft.extensions.dependencyinjection/) is fully supported
- The base constructor requires IEntityBaseServices<Person> for Neatoo Framework constructs
- Rules are injected and added to the RuleManager

## Properties

``` csharp
    [DisplayName("First Name*")]
    [Required(ErrorMessage = "First Name is required")]
    public partial string? FirstName { get; set; }

    [DisplayName("Last Name*")]
    [Required(ErrorMessage = "Last Name is required")]
    public partial string? LastName { get; set; }
```

- Partial properties are required as their [full definition](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.BaseGenerator/Neatoo.BaseGenerator.PartialBaseGenerator/Person.DomainModel.Person.g.cs) is a method call
- Required Attribute is supported. (More to come)

## Map Methods

``` csharp
    public partial void MapFrom(PersonEntity personEntity);
    public partial void MapTo(PersonEntity personEntity);
    public partial void MapModifiedTo(PersonEntity personEntity);
```

- Optional partial methods
- When included (and the class is partial) [Neatoo will generate mapping methods](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Generated/Neatoo.RemoteFactory.FactoryGenerator/Neatoo.RemoteFactory.FactoryGenerator.MapperGenerator/DomainModel.PersonMapper.g.cs) to map the parameter type to the Entity
- Mapping is done strickly on property name
- Mismatched types will lead to a compile error
- Use the [MapperIgnore] attribute on a property to exclude it [Example](https://github.com/NeatooDotNet/RemoteFactory/blob/main/src/Tests/FactoryGeneratorTests/Mapper/MapperIgnoreAttribute.cs)

## Factory Methods


``` csharp
    [Create]
    public void Create([Service] IPersonPhoneList personPhoneModelList) {}

    [Remote]
    [Fetch]
    public async Task<bool> Fetch([Service] IPersonDbContext personContext,
                                    [Service] IPersonPhoneListFactory personPhoneModelListFactory) {}

    [Remote]
    [Insert]
    public async Task<PersonEntity?> Insert([Service] IPersonDbContext personContext,
                                    [Service] IPersonPhoneListFactory personPhoneModelListFactory) {}

    [Remote]
    [Update]
    public async Task<PersonEntity?> Update([Service] IPersonDbContext personContext,
                                    [Service] IPersonPhoneListFactory personPhoneModelListFactory) {}

    [Remote]
    [Delete]
    public async Task Delete([Service] IPersonDbContext personContext) {}

```


- These are [Data Mapper methods](https://martinfowler.com/eaaCatalog/dataMapper.html) for interacting with the persistance layer
- See [Factory](/overview/factory/)