---
title: "Entity"
layout: single
permalink: /overview/entity/
---

Neatoo [EntityBase](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Neatoo/EntityBase.cs) is the main component of Neatoo. 
It provides the bindable properties, triggers rules on property change and bindable meta properties.
It also defines the Factory Methods as the Data Mapper to the persistance layer.

This is a brief overview of [EntityBase](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Neatoo/EntityBase.cs) using the [Person](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/Person.cs) entity. More in-depth pages are in the works for each feature.

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

### Property Meta Properties

- Each property is a [class](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Neatoo/IEntityProperty.cs) with bindable meta properties IsBusy, IsValid and IsModified.
- These property classes are accessed thru EntityBase.PropertyManager or the EntityBase indexer (ex "MyEntity[nameof(FirstName)]")

## Bindable Meta Properties

### IsValid

If a Rule returns a message for a property the Entity and it's Parent Entities are InValid=true.

### IsBusy

Async Rules are executing

### IsNew

A Factory Create was used to instantiate the Entity. On Save the Factory Update will be called.

### IsModified

The Entity or a Child Entity has a modified property or IsNew=true.

### IsDelete

The Entity has been marked for Deletion. On Save the Factory Delete will be called.

### IsSelfXYZ

Each Meta Property has a corresponding IsSelf. This excludes Child Enties.

## Property Message

- Each Entity property has a list of messages
- If a property has a message the property is InValid=false, the Entity is IsValid=false and Parent Entities are IsValid=false
- EntityBase.PropertyMessages is the full list of property messages

## Parent Property

If an EntityBase ("child") is assigned to the property of another EntityBase ("parent") then child.Parent will be automatically set i.e. child.Parent = parent. Only the Aggregate Root should have Parent = null.

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

- These are [Data Mapper methods](https://martinfowler.com/eaaCatalog/dataMapper.html) for interacting with the persistance layer
- See [Factory](/overview/factory/)

## Events

Events propagate up thru the Aggregate. An Entity can see all of it's Child Entity events using EntityBase by overidding ChildNeatooPropertyChanged.

The breadcrumbs of the properties is available in the event args [NeatooPropertyChangedEventArgs](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Neatoo/NeatooPropertyChangedEventArgs.cs)

## Entity Lists

A corresponding [EntityList](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Neatoo/EntityListBase.cs) provides a list of Entities while supporting all of the EntityBase Aggregate Entity features.

- When an item is removed from an [EntityList](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Neatoo/EntityListBase.cs) it is marked IsDeleted=true and added to EntityList.DeletedList. Entities within EntityList.DeletedList need to be handled in the Insert/Update/Delete Factory Methods. Ex [PersonPhoneList](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/PersonPhoneList.cs).Update.
- The value of the Parent property of EntityBases within the list will be the parent of the list not the list itself.
  - Ex. PersonPhone.Parent = Person. NOT PersonPhone.Parent = PersonPhoneList