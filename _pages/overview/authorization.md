---
title: "Authorization"
layout: single
permalink: /overview/authorization/
---

All of an Entity's Factory Method calls can be Authorized. The authorization is performed by the Neatoo generated Factory before the Entity's Factory Method is called. An authorization class can be simple or complex. The linking between Factory Methods and Authorization Methods is done using Bitwise operators to be very flexible. A corresponding CanXYZ method is also defined. 

## Authorization Class Attribute

A seperate Authorization Class is defined that the Factory calls to perform the authorization. 

``` csharp
[Authorize<IPersonAuth>]
internal partial class Person : IPersonModel
```

The Authorization Class is linked to the Entity by the generic Authorize class attribute. Above IPersonModelAuth is the Authorization Class for the Person Entity.

If an interface is defined it is treated as a service and the implementation is resolved from the DI container.

## Authorize Method Attribute

``` csharp
    [Authorize(AuthorizeOperation.Create)]
    bool HasCreate();
```

Authorize Methods have an Authorize method attribute to define their [AuthorizeOperation](https://github.com/NeatooDotNet/RemoteFactory/blob/main/src/RemoteFactory/AuthorizeOperation.cs). When the [FactoryOperation](https://github.com/NeatooDotNet/RemoteFactory/blob/main/src/RemoteFactory/FactoryOperation.cs) matches the [AuthorizeOperation](https://github.com/NeatooDotNet/RemoteFactory/blob/main/src/RemoteFactory/AuthorizeOperation.cs) the Authorize method is called.

If an interface is defined on the Entity Class's Authorize attribute then the Authorize method attributes must be on the interface.

## AuthorizedOperation and FactoryOperation Bitwise

The [FactoryOperation](https://github.com/NeatooDotNet/RemoteFactory/blob/main/src/RemoteFactory/FactoryOperation.cs) enum corresponds one-to-one to the [Factory Method](https://github.com/NeatooDotNet/RemoteFactory/blob/main/src/RemoteFactory/FactoryAttributes.cs) attributes. The [FactoryOperation](https://github.com/NeatooDotNet/RemoteFactory/blob/main/src/RemoteFactory/FactoryOperation.cs) enum is a bitwise or of [AuthorizeOperation](https://github.com/NeatooDotNet/RemoteFactory/blob/main/src/RemoteFactory/AuthorizeOperation.cs) enums.

An Authorize Method is called for a Factory Method if the Factory Method attribute's FactoryOperation is a bitwise or match to the Authorize Method's AuthorizeOperation enum.

### Code Example

Authorization Class:

``` csharp
    [Authorize(AuthorizeOperation.Read | AuthorizeOperation.Write)]
    public bool HasAccess();
    [Authorize(AuthorizeOperation.Create)]
    bool HasCreate();
```

The following code will be added to the generated Entity Factory's before the call to the Entity's Create Factory method:

``` csharp
    IPersonAuth ipersonauth = ServiceProvider.GetRequiredService<IPersonAuth>();
    authorized = ipersonauth.HasAccess();
    if (!authorized.HasAccess)
    {
        return authorized;
    }

    authorized = ipersonauth.HasCreate();
    if (!authorized.HasAccess)
    {
        return authorized;
    }
```

- The Authorization Class is resolved from the DI container
- IPersonAuth.HasAccess is called because FactoryOperation.Create & (AuthorizeOperation.Read \| AuthorizationOperation.Write) != 0
  - Remember, FactoryOperation.Create = AuthorizeOperation.Read \| AuthorizeOperation.Create
- IPersonAuth.HasCreate is called because FactoryOperation.Create & (AuthorizeOperation.Create) != 0


## Can Method

For each Factory Method defined in the Factory when there is Authorization a "Can" method is created.
This is to check ahead of time if the call will be authorized.
If unauthorized Create and Fetch will simply return null objects.
If unauthorized Save will throw an exception and TrySave will not.