---
title: "Rule"
layout: single
permalink: /overview/rule/
---

This is a brief overview of a Neatoo Rule to be used with an Entity using [UniqueNameRule](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Examples/Person/Person.DomainModel/UniqueNameRule.cs) as an example. 

- If any Rules assign a message to a property the Entity.IsValid is set to False
  - If an Entity has any child Entities that have IsValid=False the Parent Entity has IsValid=False as well
- Rules are cascading. Changes to the target within the Execute method will trigger the rules on the modified properties
- A property may have multiple messages

## Class Declaration

``` csharp
internal class UniqueNameRule : AsyncRuleBase<IPerson>, IUniqueNameRule
```
- Inherit from AsyncRuleBase<T> for Async Task Rules and RuleBase<T> otherwise
- T is the target of the rule. In this case the target T is IPerson
- For shared rules use a common interface as the type T.

## Constructor

``` csharp
    private UniqueName.IsUniqueName isUniqueName;

    public UniqueNameRule(UniqueName.IsUniqueName isUniqueName) : base()
    {
        this.isUniqueName = isUniqueName;
        AddTriggerProperties(p => p.FirstName, p => p.LastName);
    }
```

- Dependency Injection is fully supported
- AddTriggerProperties - Expressions to specify which Entity property changed will trigger the rule. 
  - In this case changes to IPerson.FirstName and IPerson.LastName will trigger Rule.Execute

## Execute

``` csharp
    protected override async Task<IRuleMessages> Execute(IPerson t, CancellationToken? token = null)
```

- On Rule Execute the target instance is sent in
- Properties on the target can be changed
  - Rules are cascading. Changing properties on the Target will trigger any assigned Rules

## Rule Messages

``` csharp
                return None;
                ...
                return (new[]
                {
                    (nameof(t.FirstName), "First and Last name combination is not unique"),
                    (nameof(t.LastName), "First and Last name combination is not unique")
                }).AsRuleMessages();

```

- Return None to specify no validation errors
- Return [RuleMessages](https://github.com/NeatooDotNet/Neatoo/blob/main/src/Neatoo/Rules/RuleMessage.cs) to specify validation errors
  - RuleMessage is a (property, message) pair
  - There are a number of overrides as in this where Tuple<string, string>[] is being transformed to RuleMessages
- If an Entity or any of its Child Entities have a broken 
- A property may have multiple validation messages