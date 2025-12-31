---
title: "Data Mapping Reference"
layout: single
permalink: /reference/data-mapping/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

Data mapping in Neatoo transfers property values between your domain entities and persistence entities. This is done through explicit property assignments in your factory methods, giving you full control over how data flows between your rich domain objects and flat database entities used by EF Core.

## The Data Mapping Challenge

Without a clear pattern, mapping between domain and persistence entities requires scattered, error-prone code:

```csharp
// Manual mapping - repetitive and easy to forget properties
public void LoadFromDatabase(PersonEntity entity)
{
    _id = entity.Id;
    _firstName = entity.FirstName;
    _lastName = entity.LastName;
    _email = entity.Email;
    // Easy to forget: _phone = entity.Phone;
    // No tracking: did rules run?
}

public void SaveToDatabase(PersonEntity entity)
{
    entity.Id = _id;
    entity.FirstName = _firstName;
    entity.LastName = _lastName;
    entity.Email = _email;
    // Copy all 20 properties even if only one changed?
}
```

Neatoo solves this by providing:

1. **`LoadProperty()`** - For loading data without triggering rules or modification tracking
2. **Property indexer with `IsModified`** - For efficient updates that only write changed properties
3. **A clear pattern** - Explicit property assignments in `[Fetch]`, `[Insert]`, and `[Update]` methods

## Loading Data in Fetch Operations

The `[Fetch]` method loads data from persistence into your domain entity. Use `LoadProperty()` to set values without triggering rules or modification flags.

### The LoadProperty() Method

```csharp
[Remote]
[Fetch]
public async Task<bool> Fetch([Service] IPersonDbContext dbContext)
{
    var entity = await dbContext.Persons.FindAsync(Id);
    if (entity == null) return false;

    // Load properties without triggering rules
    LoadProperty(nameof(Id), entity.Id);
    LoadProperty(nameof(FirstName), entity.FirstName);
    LoadProperty(nameof(LastName), entity.LastName);
    LoadProperty(nameof(Email), entity.Email);
    LoadProperty(nameof(Phone), entity.Phone);

    return true;
}
```

### Why LoadProperty?

`LoadProperty()` differs from property setters in important ways:

| Aspect | Property Setter | LoadProperty() |
|--------|-----------------|----------------|
| PropertyChanged events | Yes | No |
| Rules execution | Yes | No |
| IsModified flag | Set to true | Not changed |
| Use case | User edits | Database loads |

Using `LoadProperty()` ensures:

- **No Rules Execute**: Data is loaded as-is from the database
- **No Modification Tracking**: `IsModified` stays `false` after fetch
- **No PropertyChanged Events**: UI updates happen after method completes
- **Consistent State**: Entity reflects exactly what is in the database

### Complete Fetch Example

```csharp
[Remote]
[Fetch]
public async Task<bool> Fetch(
    [Service] IPersonDbContext db,
    [Service] IPersonPhoneListFactory phoneListFactory)
{
    var entity = await db.Persons
        .AsNoTracking()          // Optimization - no EF tracking needed
        .Include(p => p.Phones)
        .FirstOrDefaultAsync(p => p.Id == Id);

    if (entity == null) return false;

    // Load all scalar properties
    LoadProperty(nameof(Id), entity.Id);
    LoadProperty(nameof(FirstName), entity.FirstName);
    LoadProperty(nameof(LastName), entity.LastName);
    LoadProperty(nameof(Email), entity.Email);
    LoadProperty(nameof(Phone), entity.Phone);
    LoadProperty(nameof(CreatedDate), entity.CreatedDate);
    LoadProperty(nameof(ModifiedDate), entity.ModifiedDate);
    LoadProperty(nameof(RowVersion), entity.RowVersion);

    // Child collection loaded by its factory
    PersonPhoneList = await phoneListFactory.Fetch(entity.Phones);

    return true;
}
```

## Saving Data in Insert Operations

The `[Insert]` method persists a new entity. Copy all property values to the persistence entity using direct assignment.

### Insert Pattern

```csharp
[Remote]
[Insert]
public async Task Insert([Service] IPersonDbContext db)
{
    Id = Guid.NewGuid();

    var entity = new PersonEntity();

    // Copy ALL properties to new persistence entity
    entity.Id = Id.Value;
    entity.FirstName = FirstName;
    entity.LastName = LastName;
    entity.Email = Email;
    entity.Phone = Phone;
    entity.CreatedDate = DateTime.UtcNow;

    db.Persons.Add(entity);
    await db.SaveChangesAsync();
}
```

### Why Copy Everything on Insert?

For new entities, every property value needs to be persisted. There is no previous state to compare against, so you copy all properties unconditionally.

### Complete Insert Example with Children

```csharp
[Remote]
[Insert]
public async Task Insert(
    [Service] IPersonDbContext db,
    [Service] IPersonPhoneListFactory phoneListFactory)
{
    Id = Guid.NewGuid();
    CreatedDate = DateTime.UtcNow;

    var entity = new PersonEntity
    {
        Id = Id.Value,
        FirstName = FirstName,
        LastName = LastName,
        Email = Email,
        Phone = Phone,
        CreatedDate = CreatedDate
    };

    db.Persons.Add(entity);

    // Save children after parent has ID
    await phoneListFactory.Save(PersonPhoneList, Id.Value);

    await db.SaveChangesAsync();
}
```

## Updating Data in Update Operations

The `[Update]` method persists changes to an existing entity. For efficiency, copy only modified properties.

### Checking Property Modification

Access individual property state through the entity indexer:

```csharp
// Check if a specific property was modified
if (this[nameof(Email)].IsModified)
{
    entity.Email = Email;
}
```

### Update Pattern

```csharp
[Remote]
[Update]
public async Task Update([Service] IPersonDbContext db)
{
    var entity = await db.Persons.FindAsync(Id);

    if (entity == null)
        throw new InvalidOperationException($"Person {Id} not found");

    // Copy only MODIFIED properties
    if (this[nameof(FirstName)].IsModified)
        entity.FirstName = FirstName;
    if (this[nameof(LastName)].IsModified)
        entity.LastName = LastName;
    if (this[nameof(Email)].IsModified)
        entity.Email = Email;
    if (this[nameof(Phone)].IsModified)
        entity.Phone = Phone;

    entity.ModifiedDate = DateTime.UtcNow;

    await db.SaveChangesAsync();
}
```

### Why Only Modified Properties?

Copying only modified properties provides:

1. **Efficient Updates**: EF Core generates smaller UPDATE statements
2. **Fewer Conflicts**: Less chance of overwriting concurrent changes
3. **Audit Clarity**: Easier to see exactly what changed
4. **Performance**: Less data transferred, fewer columns updated

### Using ModifiedProperties Collection

You can iterate all modified properties:

```csharp
// List all modified properties
foreach (var propName in ModifiedProperties)
{
    Console.WriteLine($"Changed: {propName}");
}
```

### Complete Update Example with Children

```csharp
[Remote]
[Update]
public async Task Update(
    [Service] IPersonDbContext db,
    [Service] IPersonPhoneListFactory phoneListFactory)
{
    var entity = await db.Persons.FindAsync(Id);

    if (entity == null)
        throw new InvalidOperationException($"Person {Id} not found");

    // Update only modified scalar properties
    if (this[nameof(FirstName)].IsModified)
        entity.FirstName = FirstName;
    if (this[nameof(LastName)].IsModified)
        entity.LastName = LastName;
    if (this[nameof(Email)].IsModified)
        entity.Email = Email;
    if (this[nameof(Phone)].IsModified)
        entity.Phone = Phone;

    entity.ModifiedDate = DateTime.UtcNow;

    // Save children (handles inserts, updates, deletes)
    await phoneListFactory.Save(PersonPhoneList, Id.Value);

    await db.SaveChangesAsync();
}
```

## Property Mapping Considerations

### Handling Type Conversions

When domain and persistence types differ, handle conversions explicitly:

```csharp
// Domain entity has enum
public partial PersonStatus Status { get; set; }

// Persistence entity has string
// In Fetch:
LoadProperty(nameof(Status), Enum.Parse<PersonStatus>(entity.Status));

// In Insert/Update:
entity.Status = Status.ToString();
```

### Handling Nullable Properties

```csharp
// Fetch - handle null values
LoadProperty(nameof(MiddleName), entity.MiddleName);  // null is fine

// Update - check modification
if (this[nameof(MiddleName)].IsModified)
    entity.MiddleName = MiddleName;  // Can set to null
```

### Computed Properties

Properties that are calculated from other properties do not need to be loaded or saved:

```csharp
// Computed property - set by a rule
public partial string? FullName { get; set; }

// Constructor sets up the rule
public Person(IEntityBaseServices<Person> services) : base(services)
{
    RuleManager.AddAction(
        (Person p) => p.FullName = $"{p.FirstName} {p.LastName}",
        p => p.FirstName, p => p.LastName);
}

// In Fetch - don't load FullName, let the rule calculate it
// After loading FirstName and LastName, run rules if needed:
// await RunRules(RunRulesFlag.All);
```

## Mapping Child Entities

Child entities are handled through their own factories, not by property assignment.

### Loading Child Collections

```csharp
[Remote]
[Fetch]
public async Task<bool> Fetch(
    [Service] IDbContext db,
    [Service] IPersonPhoneListFactory phoneListFactory)
{
    var entity = await db.Persons
        .Include(p => p.Phones)
        .FirstOrDefaultAsync(p => p.Id == Id);

    if (entity == null) return false;

    // Load parent properties
    LoadProperty(nameof(Id), entity.Id);
    LoadProperty(nameof(FirstName), entity.FirstName);
    // ... other properties

    // Child collection loaded by its factory
    PersonPhoneList = await phoneListFactory.Fetch(entity.Phones);

    return true;
}
```

### Child Collection Factory Pattern

```csharp
[Factory]
internal partial class PersonPhoneList
    : EntityListBase<IPersonPhone>, IPersonPhoneList
{
    [Fetch]
    public async Task Fetch(
        IEnumerable<PersonPhoneEntity> entities,
        [Service] IPersonPhoneFactory phoneFactory)
    {
        foreach (var entity in entities)
        {
            var phone = await phoneFactory.Fetch(entity);
            if (phone != null)
            {
                Add(phone);
            }
        }
    }
}
```

## EF Core Integration Patterns

### Repository-Style DbContext

Define a clean interface for your DbContext:

```csharp
public interface IPersonDbContext
{
    DbSet<PersonEntity> Persons { get; }
    DbSet<PersonPhoneEntity> PersonPhones { get; }
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

public class PersonDbContext : DbContext, IPersonDbContext
{
    public DbSet<PersonEntity> Persons { get; set; }
    public DbSet<PersonPhoneEntity> PersonPhones { get; set; }
}
```

### Handling Concurrency

For optimistic concurrency, track row versions:

```csharp
// Domain entity
public partial byte[]? RowVersion { get; set; }

// Persistence entity
public class PersonEntity
{
    [Timestamp]
    public byte[]? RowVersion { get; set; }
}

// In Update
[Remote]
[Update]
public async Task Update([Service] IPersonDbContext db)
{
    var entity = await db.Persons.FindAsync(Id);

    // Set the original row version for concurrency check
    db.Entry(entity).Property(e => e.RowVersion).OriginalValue = RowVersion;

    // Update modified properties
    if (this[nameof(FirstName)].IsModified)
        entity.FirstName = FirstName;
    // ... other properties

    try
    {
        await db.SaveChangesAsync();
        // Update our row version with the new value
        LoadProperty(nameof(RowVersion), entity.RowVersion);
    }
    catch (DbUpdateConcurrencyException)
    {
        throw new ConcurrencyException("Record was modified by another user");
    }
}
```

## Complete Example

Here is a complete entity with all data mapping patterns:

```csharp
// Persistence Entity (EF Core)
public class PersonEntity
{
    public Guid Id { get; set; }
    public string? FirstName { get; set; }
    public string? LastName { get; set; }
    public string? Email { get; set; }
    public DateTime CreatedDate { get; set; }
    public DateTime? ModifiedDate { get; set; }
    public byte[]? RowVersion { get; set; }

    public ICollection<PersonPhoneEntity> Phones { get; set; }
        = new List<PersonPhoneEntity>();
}

// Domain Entity (Neatoo)
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    private readonly IPersonPhoneListFactory _phoneListFactory;

    public Person(
        IEntityBaseServices<Person> services,
        IPersonPhoneListFactory phoneListFactory) : base(services)
    {
        _phoneListFactory = phoneListFactory;

        RuleManager.AddAction(
            (Person p) => p.FullName = $"{p.FirstName} {p.LastName}",
            p => p.FirstName, p => p.LastName);
    }

    // Mapped properties
    public partial Guid? Id { get; set; }
    public partial string? FirstName { get; set; }
    public partial string? LastName { get; set; }
    public partial string? Email { get; set; }
    public partial DateTime CreatedDate { get; set; }
    public partial DateTime? ModifiedDate { get; set; }
    public partial byte[]? RowVersion { get; set; }

    // Computed - not persisted
    public partial string? FullName { get; set; }

    // Child collection
    public partial IPersonPhoneList? PersonPhoneList { get; set; }

    [Create]
    public void Create()
    {
        PersonPhoneList = _phoneListFactory.Create();
        CreatedDate = DateTime.UtcNow;
    }

    [Remote]
    [Fetch]
    public async Task<bool> Fetch([Service] IPersonDbContext db)
    {
        var entity = await db.Persons
            .Include(p => p.Phones)
            .FirstOrDefaultAsync(p => p.Id == Id);

        if (entity == null) return false;

        // Load all properties
        LoadProperty(nameof(Id), entity.Id);
        LoadProperty(nameof(FirstName), entity.FirstName);
        LoadProperty(nameof(LastName), entity.LastName);
        LoadProperty(nameof(Email), entity.Email);
        LoadProperty(nameof(CreatedDate), entity.CreatedDate);
        LoadProperty(nameof(ModifiedDate), entity.ModifiedDate);
        LoadProperty(nameof(RowVersion), entity.RowVersion);

        // Load child collection
        PersonPhoneList = await _phoneListFactory.Fetch(entity.Phones);
        return true;
    }

    [Remote]
    [Insert]
    public async Task Insert([Service] IPersonDbContext db)
    {
        Id = Guid.NewGuid();
        CreatedDate = DateTime.UtcNow;

        var entity = new PersonEntity
        {
            Id = Id.Value,
            FirstName = FirstName,
            LastName = LastName,
            Email = Email,
            CreatedDate = CreatedDate
        };

        db.Persons.Add(entity);
        await _phoneListFactory.Save(PersonPhoneList, Id.Value);
        await db.SaveChangesAsync();

        LoadProperty(nameof(RowVersion), entity.RowVersion);
    }

    [Remote]
    [Update]
    public async Task Update([Service] IPersonDbContext db)
    {
        ModifiedDate = DateTime.UtcNow;

        var entity = await db.Persons.FindAsync(Id);
        db.Entry(entity).Property(e => e.RowVersion).OriginalValue = RowVersion;

        // Update only modified properties
        if (this[nameof(FirstName)].IsModified)
            entity.FirstName = FirstName;
        if (this[nameof(LastName)].IsModified)
            entity.LastName = LastName;
        if (this[nameof(Email)].IsModified)
            entity.Email = Email;

        entity.ModifiedDate = ModifiedDate;

        await _phoneListFactory.Save(PersonPhoneList, Id.Value);
        await db.SaveChangesAsync();

        LoadProperty(nameof(RowVersion), entity.RowVersion);
    }

    [Remote]
    [Delete]
    public async Task Delete([Service] IPersonDbContext db)
    {
        var entity = await db.Persons.FindAsync(Id);
        if (entity != null)
        {
            db.Persons.Remove(entity);
            await db.SaveChangesAsync();
        }
    }
}
```

## Best Practices

### Keep Fetch Methods Efficient

- Use `AsNoTracking()` when you do not need EF change tracking
- Use `Include()` to load related entities in one query
- Load only the properties you need

### Keep Insert/Update Methods Clean

- Separate persistence logic from business logic
- Handle child entities through their factories
- Use transactions for complex operations

### Handle Errors Gracefully

- Check for null entities in Update/Delete
- Handle concurrency exceptions
- Provide meaningful error messages

## Troubleshooting

### Property Not Being Loaded

If a property is not being loaded:

1. **Check LoadProperty call**: Ensure you are calling `LoadProperty()` for that property
2. **Check property names**: Names must match exactly (use `nameof()`)
3. **Check for nulls**: Handle null values from the database appropriately

### Rules Running During Fetch

If rules unexpectedly run during fetch:

1. **Verify LoadProperty is used**: Not property setters
2. **Check for property assignments**: Any direct assignment triggers rules

```csharp
// Wrong - triggers rules
this.Email = entity.Email;

// Correct - no rules triggered
LoadProperty(nameof(Email), entity.Email);
```

### Modified Properties Not Updating Database

If changes are not being persisted:

1. **Check IsModified**: Verify the property is marked as modified
2. **Check the Update method**: Ensure you are checking the right property names
3. **Verify SaveChangesAsync**: Ensure it is being called

## Related Topics

- [Factory Pattern Concept](/concepts/factories-overview/) - Understanding factories
- [Factory Operations Reference](/reference/factory-operations/) - Complete attribute reference
- [EntityBase Reference](/reference/entity-base/) - Entity properties and modification tracking
- [EntityListBase Reference](/reference/entity-list-base/) - Collection mapping patterns
