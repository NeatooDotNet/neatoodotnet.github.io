---
title: "Database-Dependent Validation"
layout: single
permalink: /guides/database-validation/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

Some validation rules require database access: checking if an email is unique, verifying a username isn't taken, or ensuring date ranges don't overlap with existing records. This guide shows the correct pattern for implementing these validations in Neatoo.

## The Problem

When validation requires database access, developers often place the logic in `[Insert]` or `[Update]` factory methods because database services are readily available there:

```csharp
// Anti-pattern: Validation in factory method
[Remote]
[Insert]
public async Task Insert([Service] IDbContext db)
{
    // Check uniqueness in the insert method
    var exists = await db.Persons.AnyAsync(p => p.Email == Email);
    if (exists)
    {
        throw new ValidationException("Email already in use");
    }

    Id = Guid.NewGuid();
    var entity = new PersonEntity();
    MapTo(entity);
    db.Persons.Add(entity);
    await db.SaveChangesAsync();
}
```

**This approach has serious problems:**

### 1. Delayed Feedback

Users discover validation failures only after clicking "Save". They fill out an entire form, submit it, wait for the server round-trip, and then see an error. This is frustrating compared to immediate feedback while editing.

### 2. Exception-Based Error Handling

Throwing exceptions for validation failures returns HTTP 500 errors instead of proper validation responses. The UI receives an exception rather than a validation message it can display elegantly.

### 3. Bypasses the Rule System

Factory method validation circumvents Neatoo's rule infrastructure:

- No property triggers - validation only runs on save, not on property change
- No `IsBusy` integration - the UI can't show "validating..." indicators
- No `IsValid` coordination - the entity appears valid until save fails
- No automatic message clearing when the user fixes the problem

### 4. Inconsistent User Experience

The application behaves differently for database-dependent validations versus other validations. Simple rules like "email format" show immediately, but "email uniqueness" only shows after save. This inconsistency confuses users.

## The Solution: AsyncRuleBase + Commands

The correct pattern uses three components:

1. **Command** - A factory class with `[Execute]` that performs the database check
2. **AsyncRuleBase** - A rule that calls the command and returns validation messages
3. **Trigger Properties** - Property changes that initiate the validation

### Step 1: Create the Command

Commands are factory classes that provide remote access to server-side operations:

```csharp
public interface ICheckEmailExistsResult
{
    bool Exists { get; }
}

[Factory]
public class CheckEmailExistsResult : ICheckEmailExistsResult
{
    public bool Exists { get; set; }

    [Remote]
    [Execute]
    public async Task Execute(
        string email,
        Guid? excludeId,
        [Service] IDbContext db)
    {
        Exists = await db.Persons
            .Where(p => p.Email == email)
            .Where(p => p.Id != excludeId)
            .AnyAsync();
    }
}
```

Neatoo generates a factory interface with a delegate for the execute operation:

```csharp
// Generated
public interface ICheckEmailExistsResultFactory
{
    // The delegate is what you inject and call
    CheckEmailExistsDelegate CheckEmailExists { get; }
}

public delegate Task<ICheckEmailExistsResult> CheckEmailExistsDelegate(
    string email, Guid? excludeId);
```

### Step 2: Create the Async Rule

The rule calls the command delegate and returns validation messages:

```csharp
public interface IUniqueEmailRule : IRule<IPerson> { }

public class UniqueEmailRule : AsyncRuleBase<Person>, IUniqueEmailRule
{
    private readonly CheckEmailExistsDelegate _checkEmail;

    public UniqueEmailRule(ICheckEmailExistsResultFactory factory)
        : base(p => p.Email)  // Trigger when Email changes
    {
        _checkEmail = factory.CheckEmailExists;
    }

    protected override async Task<IRuleMessages> Execute(
        Person target,
        CancellationToken? token = null)
    {
        // Skip if email is empty (Required attribute handles that)
        if (string.IsNullOrEmpty(target.Email))
            return None;

        // Call the command
        var result = await _checkEmail(target.Email, target.Id);

        // Return validation message if duplicate found
        return result.Exists
            ? (nameof(target.Email), "This email is already in use").AsRuleMessages()
            : None;
    }
}
```

### Step 3: Register and Add the Rule

Register the rule in DI and add it to the entity:

```csharp
// Program.cs
builder.Services.AddScoped<IUniqueEmailRule, UniqueEmailRule>();
```

```csharp
// Person.cs
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    public Person(
        IEntityBaseServices<Person> services,
        IUniqueEmailRule uniqueEmailRule) : base(services)
    {
        RuleManager.AddRule(uniqueEmailRule);
    }

    public partial string? Email { get; set; }

    // ... other properties and factory methods
}
```

## How It Works

When the user types in the email field:

1. **Property Change** - `Email` setter is called
2. **Rule Triggers** - `UniqueEmailRule` is scheduled (triggers on `Email`)
3. **IsBusy = true** - Entity and property show busy state
4. **Remote Call** - Command executes on server, checks database
5. **Result Returns** - Rule receives `Exists` boolean
6. **Message Set** - If exists, validation message added to `Email` property
7. **IsBusy = false** - Busy state clears
8. **UI Updates** - `IsValid` and `PropertyMessages` update, UI reacts

The user sees:
- Input field shows "validating..." while the check runs
- Error message appears inline if email is taken
- Error clears automatically when they type a different email
- Save button stays disabled while validation is in progress

## Complete Example: Username Uniqueness

Here's a complete implementation for checking username uniqueness:

### The Command

```csharp
public interface ICheckUsernameResult
{
    bool IsTaken { get; }
    string? SuggestedAlternative { get; }
}

[Factory]
public class CheckUsernameResult : ICheckUsernameResult
{
    public bool IsTaken { get; set; }
    public string? SuggestedAlternative { get; set; }

    [Remote]
    [Execute]
    public async Task Execute(
        string username,
        Guid? excludeUserId,
        [Service] IDbContext db)
    {
        IsTaken = await db.Users
            .Where(u => u.Username == username)
            .Where(u => u.Id != excludeUserId)
            .AnyAsync();

        if (IsTaken)
        {
            // Suggest an alternative
            var baseUsername = username.TrimEnd('0', '1', '2', '3', '4', '5', '6', '7', '8', '9');
            for (int i = 1; i <= 99; i++)
            {
                var candidate = $"{baseUsername}{i}";
                var candidateTaken = await db.Users.AnyAsync(u => u.Username == candidate);
                if (!candidateTaken)
                {
                    SuggestedAlternative = candidate;
                    break;
                }
            }
        }
    }
}
```

### The Rule

```csharp
public interface IUniqueUsernameRule : IRule<IUser> { }

public class UniqueUsernameRule : AsyncRuleBase<User>, IUniqueUsernameRule
{
    private readonly CheckUsernameDelegate _checkUsername;

    public UniqueUsernameRule(ICheckUsernameResultFactory factory)
        : base(u => u.Username)
    {
        _checkUsername = factory.CheckUsername;
    }

    protected override async Task<IRuleMessages> Execute(
        User target,
        CancellationToken? token = null)
    {
        if (string.IsNullOrEmpty(target.Username))
            return None;

        var result = await _checkUsername(target.Username, target.Id);

        if (result.IsTaken)
        {
            var message = string.IsNullOrEmpty(result.SuggestedAlternative)
                ? "This username is already taken"
                : $"This username is taken. Try '{result.SuggestedAlternative}'?";

            return (nameof(target.Username), message).AsRuleMessages();
        }

        return None;
    }
}
```

### The Entity

```csharp
[Factory]
internal partial class User : EntityBase<User>, IUser
{
    public User(
        IEntityBaseServices<User> services,
        IUniqueUsernameRule uniqueUsernameRule) : base(services)
    {
        RuleManager.AddRule(uniqueUsernameRule);
    }

    public partial Guid? Id { get; set; }

    [Required(ErrorMessage = "Username is required")]
    [StringLength(30, MinimumLength = 3, ErrorMessage = "Username must be 3-30 characters")]
    [RegularExpression(@"^[a-zA-Z0-9_]+$", ErrorMessage = "Only letters, numbers, and underscores")]
    public partial string? Username { get; set; }

    [Required]
    [EmailAddress]
    public partial string? Email { get; set; }

    // Factory methods...
}
```

### UI Integration

```razor
<MudTextField @bind-Value="_user.Username"
              Label="Username"
              Adornment="Adornment.End"
              AdornmentIcon="@(_user[nameof(_user.Username)].IsBusy
                  ? Icons.Material.Filled.Refresh
                  : (_user[nameof(_user.Username)].IsValid
                      ? Icons.Material.Filled.Check
                      : Icons.Material.Filled.Error))"
              AdornmentColor="@(_user[nameof(_user.Username)].IsValid
                  ? Color.Success
                  : Color.Error)"
              Error="@(!_user[nameof(_user.Username)].IsValid)"
              ErrorText="@_user[nameof(_user.Username)].ValidationMessage" />

@if (_user[nameof(_user.Username)].IsBusy)
{
    <MudText Typo="Typo.caption" Color="Color.Info">
        Checking availability...
    </MudText>
}
```

## Example: Date Range Overlap Detection

For more complex validations like detecting scheduling conflicts:

### The Command

```csharp
public interface ICheckScheduleOverlapResult
{
    bool HasOverlap { get; }
    string? ConflictingEventName { get; }
    DateTime? ConflictStart { get; }
    DateTime? ConflictEnd { get; }
}

[Factory]
public class CheckScheduleOverlapResult : ICheckScheduleOverlapResult
{
    public bool HasOverlap { get; set; }
    public string? ConflictingEventName { get; set; }
    public DateTime? ConflictStart { get; set; }
    public DateTime? ConflictEnd { get; set; }

    [Remote]
    [Execute]
    public async Task Execute(
        Guid roomId,
        DateTime startTime,
        DateTime endTime,
        Guid? excludeEventId,
        [Service] IDbContext db)
    {
        var conflict = await db.Events
            .Where(e => e.RoomId == roomId)
            .Where(e => e.Id != excludeEventId)
            .Where(e => e.StartTime < endTime && e.EndTime > startTime)
            .Select(e => new { e.Name, e.StartTime, e.EndTime })
            .FirstOrDefaultAsync();

        if (conflict != null)
        {
            HasOverlap = true;
            ConflictingEventName = conflict.Name;
            ConflictStart = conflict.StartTime;
            ConflictEnd = conflict.EndTime;
        }
    }
}
```

### The Rule

```csharp
public class ScheduleOverlapRule : AsyncRuleBase<Event>, IScheduleOverlapRule
{
    private readonly CheckScheduleOverlapDelegate _checkOverlap;

    public ScheduleOverlapRule(ICheckScheduleOverlapResultFactory factory)
        : base(e => e.RoomId, e => e.StartTime, e => e.EndTime)
    {
        _checkOverlap = factory.CheckScheduleOverlap;
    }

    protected override async Task<IRuleMessages> Execute(
        Event target,
        CancellationToken? token = null)
    {
        // Need all values to check
        if (target.RoomId == null || target.StartTime == default || target.EndTime == default)
            return None;

        // Basic validation first
        if (target.EndTime <= target.StartTime)
            return (nameof(target.EndTime), "End time must be after start time").AsRuleMessages();

        var result = await _checkOverlap(
            target.RoomId.Value,
            target.StartTime,
            target.EndTime,
            target.Id);

        if (result.HasOverlap)
        {
            var message = $"Conflicts with '{result.ConflictingEventName}' " +
                         $"({result.ConflictStart:g} - {result.ConflictEnd:g})";
            return (nameof(target.StartTime), message).AsRuleMessages();
        }

        return None;
    }
}
```

## Anti-Pattern Recognition

If you see this pattern in your code, refactor to use rules:

```csharp
// Anti-pattern indicators:
[Insert]
public async Task Insert([Service] IDbContext db)
{
    // Red flag: Validation logic in factory method
    if (await db.SomeTable.AnyAsync(x => x.Field == this.Field))
    {
        throw new Exception("Validation failed");  // Red flag: Throwing for validation
    }
    // ...
}
```

The fix:

1. Extract the database check into a Command
2. Create an AsyncRuleBase that calls the command
3. Add the rule to the entity's RuleManager
4. Remove the validation logic from the factory method

## Performance Considerations

### Debouncing

For fields where users type continuously, consider debouncing:

```razor
@code {
    private Timer? _debounceTimer;

    private void OnUsernameInput(string value)
    {
        _debounceTimer?.Dispose();
        _debounceTimer = new Timer(_ =>
        {
            InvokeAsync(() =>
            {
                _user.Username = value;
                StateHasChanged();
            });
        }, null, 300, Timeout.Infinite);  // 300ms delay
    }
}
```

### Caching

For repeated checks of the same value:

```csharp
public class UniqueEmailRule : AsyncRuleBase<Person>, IUniqueEmailRule
{
    private readonly CheckEmailExistsDelegate _checkEmail;
    private string? _lastCheckedEmail;
    private bool _lastResult;

    protected override async Task<IRuleMessages> Execute(
        Person target,
        CancellationToken? token = null)
    {
        if (target.Email == _lastCheckedEmail)
        {
            return _lastResult
                ? (nameof(target.Email), "Email already in use").AsRuleMessages()
                : None;
        }

        var result = await _checkEmail(target.Email, target.Id);
        _lastCheckedEmail = target.Email;
        _lastResult = result.Exists;

        return result.Exists
            ? (nameof(target.Email), "Email already in use").AsRuleMessages()
            : None;
    }
}
```

## Summary

| Approach | Feedback Timing | User Experience | Neatoo Integration |
|----------|-----------------|-----------------|-------------------|
| Factory method validation | After save attempt | Frustrating | None |
| AsyncRuleBase + Command | During editing | Immediate | Full |

**The rule-based approach provides:**

- Immediate feedback as the user types
- Integrated busy state indicators
- Automatic message clearing when fixed
- Consistent behavior with other validations
- Testable, isolated validation logic

## Related Topics

- [Rules Engine Reference](/reference/rules/) - Complete rule system documentation
- [Factory Operations Reference](/reference/factory-operations/) - Factory method patterns
- [Troubleshooting Guide](/guides/troubleshooting/) - Common validation issues
- [Blazor Integration](/guides/blazor-integration/) - UI binding patterns
