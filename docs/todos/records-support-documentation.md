# Records Support Documentation Update

**Created**: 2026-01-01
**Completed**: 2026-01-01
**Status**: Completed
**Priority**: High

## Summary

Neatoo 10.1.1 and RemoteFactory 10.1.0 introduced C# record support. This todo tracks updating all documentation to reflect this new capability.

## Source Repository Commits

### Neatoo (since `4c83d45` on 2025-12-30)

| Commit | Date | Description |
|--------|------|-------------|
| `0df02e5` | 2026-01-01 | 10.1.1 - RemoteFactory 10.1.1. Support for Records |
| `1b0695f` | 2025-12-31 | .NET 8, .NET 9 and .NET 10 support |
| `d4c4ce2` | 2025-12-31 | Race condition fix in AsyncTasksTests |

### RemoteFactory (since `9e62dda` on 2025-12-30)

| Commit | Date | Description |
|--------|------|-------------|
| `27760f8` | 2026-01-01 | 10.1.0 - Record Support |
| `9cb1ee5` | 2026-01-01 | v10.1.1 - Directory.build fix |
| `b90ba4d` | 2026-01-01 | .NET 8.0, .NET 9.0, .NET 10.0 target support |

## New Records Features (10.1.0+)

### Capabilities

1. **Type-Level `[Create]`**: Records with primary constructors can use `[Create]` on the type declaration
2. **Service Injection**: `[Service]` attribute works in positional record parameters
3. **Fetch Operations**: Static `[Fetch]` methods work with records (sync and async)
4. **Remote Operations**: Records fully support `[Remote]` with proper serialization
5. **Serialization**: Records serialize correctly through `NeatooJsonSerializer`

### Constraints

- ✅ `record` - fully supported
- ✅ `record class` - fully supported
- ❌ `record struct` - NOT supported (generates diagnostic NF0206)

## Documentation Site Updates

### High Priority

- [x] **`_pages/reference/base-value-objects.md`**
  - Add "C# Records for Value Objects" section
  - Show type-level `[Create]` pattern
  - Show records with service injection
  - Show records with `[Fetch]` operations
  - Document `record struct` constraint
  - Records are ideal for Value Objects - natural fit

- [x] **`_pages/reference/factory-operations.md`**
  - Add records support to `[Create]` section
  - Show `[Create]` on type declaration syntax
  - Show service injection in primary constructor
  - Add note about `record struct` limitation

### Medium Priority

- [x] **`_pages/concepts/factories-overview.md`**
  - Add brief mention of records as factory targets
  - Reference base-value-objects.md for full details

### Low Priority

- [ ] **`_pages/introduction.md`**
  - Consider adding records to feature list (optional)

### Required

- [x] **`CLAUDE.md`**
  - Update Neatoo commit tracking: `4c83d45` → `0df02e5`
  - Update RemoteFactory commit tracking: `9e62dda` → `27760f8`
  - Update dates to 2026-01-01

## Skill File Updates

### Required

- [x] **`~/.claude/skills/neatoo/SKILL.md`**
  - Update sync status table
  - Add records to "Essential Attributes" table note
  - Add records example to "Basic Entity Pattern" or new section

- [x] **`~/.claude/skills/neatoo/factories.md`**
  - Add "Records Support" section
  - Document type-level `[Create]`
  - Document service injection in primary constructors
  - Document static `[Fetch]` for records

### Optional

- [ ] **`~/.claude/skills/neatoo/entities.md`**
  - Mention records as option for simple value-carrying types

## Code Examples to Add

### Basic Record Value Object

```csharp
[Factory]
[Create]
public record Money(decimal Amount, string Currency = "USD");

// Generated:
public interface IMoneyFactory
{
    Money Create(decimal amount, string currency = "USD");
}
```

### Record with Service Injection

```csharp
[Factory]
[Create]
public record Address(
    string Street,
    string City,
    string State,
    string ZipCode,
    [Service] IAddressValidator validator);

// Generated (services hidden):
public interface IAddressFactory
{
    Address Create(string street, string city, string state, string zipCode);
}
```

### Record with Fetch

```csharp
[Factory]
[Create]
public record CustomerSummary(Guid Id, string? Name, string? Email)
{
    [Remote]
    [Fetch]
    public static async Task<CustomerSummary?> Fetch(
        Guid id,
        [Service] IDbContext db)
    {
        var entity = await db.Customers.FindAsync(id);
        return entity is null ? null
            : new CustomerSummary(entity.Id, entity.Name, entity.Email);
    }
}
```

### Record with Remote Operations

```csharp
[Factory]
[Create]
public record OrderLookupResult(
    Guid OrderId,
    string? CustomerName,
    decimal Total,
    OrderStatus Status)
{
    [Remote]
    [Fetch]
    public static async Task<OrderLookupResult?> Fetch(
        Guid orderId,
        [Service] IOrderRepository repo)
    {
        return await repo.GetOrderSummaryAsync(orderId);
    }
}
```

## Validation Checklist

After updates, verify:

- [x] All code examples compile (syntax check)
- [x] Consistent terminology ("records" not "record types")
- [x] Cross-references between pages work
- [x] Navigation includes any new pages
- [x] Skill files match documentation site content

## Related Source Files

- `https://github.com/NeatooDotNet/Neatoo/docs/todos/remotefactory-record-support-update.md`
- `https://github.com/NeatooDotNet/RemoteFactory/docs/todos/record-support-plan.md`

## Notes

- Records are a **perfect fit** for Value Objects in DDD
- The `base-value-objects.md` page should emphasize this
- Consider records the "modern" approach vs classes with private setters
- Keep existing class-based examples for backwards compatibility
