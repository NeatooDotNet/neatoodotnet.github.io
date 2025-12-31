---
title: "Migration Guide: From Anemic Models"
layout: single
permalink: /guides/migration-anemic/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

Migrating from an anemic domain model to Neatoo can transform your codebase from scattered procedural logic into a cohesive, behavior-rich domain model. This guide helps you recognize anemic model symptoms, map existing patterns to Neatoo equivalents, and execute a step-by-step migration that maintains backward compatibility.

## Recognizing Anemic Domain Model Symptoms

An "anemic domain model" is a pattern where domain objects contain only data (properties) with no behavior (methods, validation, business rules). The term was coined by Martin Fowler to describe what he called an "anti-pattern" because it violates the fundamental object-oriented principle of combining data and behavior.

### Symptom 1: Entities with Only Getters and Setters

Your domain objects look like this:

```csharp
// Anemic: Entity is just a data container
public class Person
{
    public Guid Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public DateTime DateOfBirth { get; set; }
    public bool IsActive { get; set; }
}
```

The entity has no methods, no validation, no business logic. It is a "dumb" data transfer object wearing an entity costume.

**With Neatoo**, entities encapsulate behavior:

```csharp
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    public Person(IEntityBaseServices<Person> services,
                  IUniqueEmailRule uniqueEmailRule) : base(services)
    {
        RuleManager.AddRule(uniqueEmailRule);

        // Computed property
        RuleManager.AddAction(
            (Person p) => p.FullName = $"{p.FirstName} {p.LastName}",
            p => p.FirstName, p => p.LastName);

        // Age calculation
        RuleManager.AddAction(
            (Person p) => p.Age = CalculateAge(p.DateOfBirth),
            p => p.DateOfBirth);
    }

    [Required(ErrorMessage = "First name is required")]
    public partial string? FirstName { get; set; }

    [Required(ErrorMessage = "Last name is required")]
    public partial string? LastName { get; set; }

    // Computed - not persisted
    public partial string? FullName { get; set; }

    // Computed - not persisted
    public partial int Age { get; set; }

    private static int CalculateAge(DateTime? dob)
    {
        if (dob == null) return 0;
        var today = DateTime.Today;
        var age = today.Year - dob.Value.Year;
        if (dob.Value.Date > today.AddYears(-age)) age--;
        return age;
    }
}
```

### Symptom 2: Business Logic in Service Classes

In anemic architectures, all behavior lives in services:

```csharp
// Anemic: Service contains all the logic
public class PersonService
{
    private readonly IPersonRepository _repository;
    private readonly IEmailValidator _emailValidator;

    public async Task<ValidationResult> ValidatePerson(Person person)
    {
        var errors = new List<string>();

        if (string.IsNullOrEmpty(person.FirstName))
            errors.Add("First name is required");

        if (string.IsNullOrEmpty(person.LastName))
            errors.Add("Last name is required");

        if (!_emailValidator.IsValid(person.Email))
            errors.Add("Invalid email format");

        if (await _repository.EmailExistsAsync(person.Email, person.Id))
            errors.Add("Email already in use");

        return new ValidationResult(errors);
    }

    public string GetFullName(Person person)
    {
        return $"{person.FirstName} {person.LastName}";
    }

    public int CalculateAge(Person person)
    {
        // Age calculation logic here
    }

    public async Task SavePerson(Person person)
    {
        var result = await ValidatePerson(person);
        if (!result.IsValid)
            throw new ValidationException(result.Errors);

        await _repository.SaveAsync(person);
    }
}
```

**Problems with this approach:**

1. Looking at `Person`, you cannot tell what rules govern it
2. Different callers might use different services (or forget validation)
3. Logic is scattered across multiple service classes
4. Testing requires instantiating the entire service dependency graph
5. Client and server validation are completely separate

**With Neatoo**, rules live on the entity:

```csharp
// Rules are first-class citizens
public class UniqueEmailRule : AsyncRuleBase<Person>
{
    private readonly IEmailService _emailService;

    public UniqueEmailRule(IEmailService emailService)
        : base(p => p.Email)
    {
        _emailService = emailService;
    }

    protected override async Task<IRuleMessages> Execute(
        Person target, CancellationToken? token = null)
    {
        if (string.IsNullOrEmpty(target.Email))
            return None;

        var exists = await _emailService.EmailExistsAsync(
            target.Email, target.Id, token ?? CancellationToken.None);

        return exists
            ? (nameof(target.Email), "Email already in use").AsRuleMessages()
            : None;
    }
}
```

### Symptom 3: Validation Scattered Across Layers

Anemic architectures often duplicate validation:

```csharp
// Controller layer validation
[HttpPost]
public async Task<IActionResult> CreatePerson(PersonDto dto)
{
    if (string.IsNullOrEmpty(dto.FirstName))
        ModelState.AddModelError("FirstName", "Required");

    if (!ModelState.IsValid)
        return BadRequest(ModelState);

    // ... more validation
}

// Service layer validation
public class PersonService
{
    public ValidationResult Validate(Person person)
    {
        // Duplicate validation logic here
    }
}

// Client-side validation (JavaScript/Blazor)
function validatePerson(person) {
    if (!person.firstName) {
        errors.push("First name is required");
    }
    // More duplicate validation
}
```

**With Neatoo**, validation is defined once:

```csharp
[Required(ErrorMessage = "First name is required")]
public partial string? FirstName { get; set; }
```

This validation runs:
- On the Blazor client for immediate feedback
- On the server for authoritative enforcement
- With no duplication

### Symptom 4: DTOs That Mirror Entities

Anemic architectures often have separate classes for each layer:

```csharp
// Database entity
public class PersonEntity
{
    public Guid Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}

// Domain entity (often identical)
public class Person
{
    public Guid Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}

// API DTO
public class PersonDto
{
    public Guid? Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
}

// View model
public class PersonViewModel
{
    public Guid? Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string FullName => $"{FirstName} {LastName}";
}
```

Plus mapping code between each layer:

```csharp
public static PersonDto ToDto(Person person) { ... }
public static Person ToDomain(PersonDto dto) { ... }
public static PersonEntity ToEntity(Person person) { ... }
public static Person ToDomain(PersonEntity entity) { ... }
```

**With Neatoo**, one entity serves all purposes:

```csharp
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    // This IS your domain model, API contract, and UI binding target
    public partial Guid? Id { get; set; }
    public partial string? FirstName { get; set; }
    public partial string? LastName { get; set; }
}
```

## Mapping Existing Patterns to Neatoo

### Service Methods to Entity Factory Methods

**Before (Anemic):**

```csharp
public class PersonService
{
    public async Task<Person> CreatePerson()
    {
        var person = new Person
        {
            Id = Guid.NewGuid(),
            CreatedDate = DateTime.UtcNow,
            IsActive = true
        };
        return person;
    }

    public async Task<Person> GetPerson(Guid id)
    {
        var entity = await _dbContext.Persons.FindAsync(id);
        return MapToDomin(entity);
    }

    public async Task SavePerson(Person person)
    {
        var validation = await ValidatePerson(person);
        if (!validation.IsValid)
            throw new ValidationException(validation.Errors);

        if (person.Id == Guid.Empty)
            await InsertPerson(person);
        else
            await UpdatePerson(person);
    }
}
```

**After (Neatoo):**

```csharp
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    [Create]
    public void Create()
    {
        CreatedDate = DateTime.UtcNow;
        IsActive = true;
    }

    [Remote]
    [Fetch]
    public async Task<bool> Fetch([Service] IPersonDbContext db)
    {
        var entity = await db.Persons.FindAsync(Id);
        if (entity == null) return false;

        LoadProperty(nameof(Id), entity.Id);
        LoadProperty(nameof(FirstName), entity.FirstName);
        LoadProperty(nameof(LastName), entity.LastName);
        LoadProperty(nameof(CreatedDate), entity.CreatedDate);
        LoadProperty(nameof(IsActive), entity.IsActive);
        return true;
    }

    [Remote]
    [Insert]
    public async Task Insert([Service] IPersonDbContext db)
    {
        Id = Guid.NewGuid();

        var entity = new PersonEntity
        {
            Id = Id.Value,
            FirstName = FirstName,
            LastName = LastName,
            CreatedDate = CreatedDate,
            IsActive = IsActive
        };
        db.Persons.Add(entity);
        await db.SaveChangesAsync();
    }

    [Remote]
    [Update]
    public async Task Update([Service] IPersonDbContext db)
    {
        var entity = await db.Persons.FindAsync(Id);

        if (this[nameof(FirstName)].IsModified)
            entity.FirstName = FirstName;
        if (this[nameof(LastName)].IsModified)
            entity.LastName = LastName;
        if (this[nameof(IsActive)].IsModified)
            entity.IsActive = IsActive;

        await db.SaveChangesAsync();
    }
}

// Usage is similar but simpler
var person = personFactory.Create();
person.FirstName = "John";
// Validation happens automatically via rules
await personFactory.Save(person);
```

### Validation Services to Rules

**Before (Anemic):**

```csharp
public class PersonValidationService
{
    private readonly IPersonRepository _repository;

    public async Task<ValidationResult> Validate(Person person)
    {
        var errors = new List<ValidationError>();

        // Required fields
        if (string.IsNullOrEmpty(person.FirstName))
            errors.Add(new ValidationError("FirstName", "Required"));

        if (string.IsNullOrEmpty(person.LastName))
            errors.Add(new ValidationError("LastName", "Required"));

        // Format validation
        if (!string.IsNullOrEmpty(person.Email) && !IsValidEmail(person.Email))
            errors.Add(new ValidationError("Email", "Invalid format"));

        // Business rule: unique email
        if (!string.IsNullOrEmpty(person.Email))
        {
            var exists = await _repository.EmailExistsAsync(
                person.Email, person.Id);
            if (exists)
                errors.Add(new ValidationError("Email", "Already in use"));
        }

        // Cross-property validation
        if (person.EndDate < person.StartDate)
            errors.Add(new ValidationError("EndDate", "Must be after start"));

        return new ValidationResult(errors);
    }
}
```

**After (Neatoo):**

```csharp
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    public Person(IEntityBaseServices<Person> services,
                  IUniqueEmailRule uniqueEmailRule) : base(services)
    {
        // Inject and add business rules
        RuleManager.AddRule(uniqueEmailRule);

        // Format validation (inline)
        RuleManager.AddValidation(
            nameof(Email),
            (Person p) => string.IsNullOrEmpty(p.Email) || IsValidEmail(p.Email)
                ? RuleMessage.None
                : RuleMessage.Error("Invalid email format"));

        // Cross-property validation (inline)
        RuleManager.AddValidation(
            (Person p) => p.EndDate >= p.StartDate
                ? RuleMessages.None
                : (nameof(p.EndDate), "Must be after start date").AsRuleMessages(),
            p => p.StartDate, p => p.EndDate);
    }

    // Required field validation via attributes
    [Required(ErrorMessage = "First name is required")]
    public partial string? FirstName { get; set; }

    [Required(ErrorMessage = "Last name is required")]
    public partial string? LastName { get; set; }

    public partial string? Email { get; set; }
    public partial DateTime? StartDate { get; set; }
    public partial DateTime? EndDate { get; set; }
}

// Async business rule as a separate class
public class UniqueEmailRule : AsyncRuleBase<Person>, IUniqueEmailRule
{
    private readonly IEmailService _emailService;

    public UniqueEmailRule(IEmailService emailService)
        : base(p => p.Email)
    {
        _emailService = emailService;
    }

    protected override async Task<IRuleMessages> Execute(
        Person target, CancellationToken? token = null)
    {
        if (string.IsNullOrEmpty(target.Email))
            return None;

        var exists = await _emailService.EmailExistsAsync(
            target.Email, target.Id);

        return exists
            ? (nameof(target.Email), "Email already in use").AsRuleMessages()
            : None;
    }
}
```

### DTOs Eliminated

**Before (Anemic):**

```csharp
// Controller uses DTOs
[HttpPost]
public async Task<ActionResult<PersonDto>> Create(CreatePersonDto dto)
{
    var person = _mapper.Map<Person>(dto);
    await _service.SavePerson(person);
    return Ok(_mapper.Map<PersonDto>(person));
}

// Blazor uses ViewModels
@code {
    private PersonViewModel _viewModel = new();

    private async Task Save()
    {
        var dto = MapToDto(_viewModel);
        var result = await Http.PostAsJsonAsync("api/persons", dto);
        // ...
    }
}
```

**After (Neatoo):**

```csharp
// No controller needed - single endpoint handles all entities
// Blazor uses the entity directly
@code {
    private IPerson _person;

    protected override async Task OnInitializedAsync()
    {
        _person = await PersonFactory.Fetch(Id);
    }

    private async Task Save()
    {
        await _person.WaitForTasks();
        if (_person.IsSavable)
        {
            await PersonFactory.Save(_person);
        }
    }
}
```

### Repository Pattern to Factory Methods

**Before (Anemic):**

```csharp
public interface IPersonRepository
{
    Task<Person> GetByIdAsync(Guid id);
    Task<IEnumerable<Person>> GetAllAsync();
    Task AddAsync(Person person);
    Task UpdateAsync(Person person);
    Task DeleteAsync(Guid id);
}

public class PersonRepository : IPersonRepository
{
    private readonly AppDbContext _context;

    public async Task<Person> GetByIdAsync(Guid id)
    {
        var entity = await _context.Persons.FindAsync(id);
        return MapToDomain(entity);
    }

    public async Task AddAsync(Person person)
    {
        var entity = MapToEntity(person);
        _context.Persons.Add(entity);
        await _context.SaveChangesAsync();
    }
    // ... more methods
}
```

**After (Neatoo):**

```csharp
// Factory operations are defined on the entity itself
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    [Remote]
    [Fetch]
    public async Task<bool> Fetch([Service] IPersonDbContext db)
    {
        var entity = await db.Persons.FindAsync(Id);
        if (entity == null) return false;

        LoadProperty(nameof(Id), entity.Id);
        LoadProperty(nameof(FirstName), entity.FirstName);
        LoadProperty(nameof(LastName), entity.LastName);
        return true;
    }

    [Remote]
    [Insert]
    public async Task Insert([Service] IPersonDbContext db)
    {
        Id = Guid.NewGuid();

        var entity = new PersonEntity
        {
            Id = Id.Value,
            FirstName = FirstName,
            LastName = LastName
        };
        db.Persons.Add(entity);
        await db.SaveChangesAsync();
    }

    [Remote]
    [Update]
    public async Task Update([Service] IPersonDbContext db)
    {
        var entity = await db.Persons.FindAsync(Id);

        if (this[nameof(FirstName)].IsModified)
            entity.FirstName = FirstName;
        if (this[nameof(LastName)].IsModified)
            entity.LastName = LastName;

        await db.SaveChangesAsync();
    }

    [Remote]
    [Delete]
    public async Task DeleteMethod([Service] IPersonDbContext db)
    {
        var entity = await db.Persons.FindAsync(Id);
        if (entity != null)
        {
            db.Persons.Remove(entity);
            await db.SaveChangesAsync();
        }
    }
}

// Usage via generated factory
var person = await personFactory.Fetch(id);    // GetById
await personFactory.Save(person);               // Add or Update
person.Delete();
await personFactory.Save(person);               // Delete
```

## Step-by-Step Migration Process

### Step 1: Assess Your Current Architecture

Before migrating, document:

1. **Entity inventory**: List all domain entities
2. **Service inventory**: List all service classes and their methods
3. **Validation locations**: Where is validation currently performed?
4. **API contracts**: What DTOs exist and how are they used?
5. **UI bindings**: How does the UI interact with data?

### Step 2: Set Up Neatoo Infrastructure

Install the required NuGet packages:

```bash
# Server project
dotnet add package Neatoo
dotnet add package Neatoo.RemoteFactory

# Client project (Blazor WASM)
dotnet add package Neatoo
dotnet add package Neatoo.RemoteFactory
dotnet add package Neatoo.Blazor.MudNeatoo  # Optional, for MudBlazor
```

Configure services:

```csharp
// Server Program.cs
builder.Services.AddNeatooServices(NeatooFactory.Server);
app.MapNeatoo();

// Client Program.cs
builder.Services.AddNeatooServices(NeatooFactory.Remote);
```

### Step 3: Create a Parallel Entity

Start with one entity. Create the Neatoo version alongside the existing one:

```csharp
// Existing anemic entity (keep for now)
public class Person { ... }

// New Neatoo entity
public partial interface IPerson : IEntityBase
{
    Guid? Id { get; set; }
    string? FirstName { get; set; }
    string? LastName { get; set; }
    string? Email { get; set; }
}

[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    public Person(IEntityBaseServices<Person> services) : base(services)
    {
    }

    public partial Guid? Id { get; set; }
    public partial string? FirstName { get; set; }
    public partial string? LastName { get; set; }
    public partial string? Email { get; set; }
}
```

### Step 4: Migrate Validation Rules

Move validation from services to the entity:

```csharp
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    public Person(IEntityBaseServices<Person> services,
                  IUniqueEmailRule uniqueEmailRule) : base(services)
    {
        // Add injected rules
        RuleManager.AddRule(uniqueEmailRule);

        // Migrate inline validations
        RuleManager.AddValidation(
            nameof(Email),
            (Person p) => string.IsNullOrEmpty(p.Email) ||
                          p.Email.Contains("@")
                ? RuleMessage.None
                : RuleMessage.Error("Invalid email format"));
    }

    [Required(ErrorMessage = "First name is required")]
    public partial string? FirstName { get; set; }

    [Required(ErrorMessage = "Last name is required")]
    public partial string? LastName { get; set; }

    public partial string? Email { get; set; }
}
```

### Step 5: Implement Factory Operations

Add CRUD operations to the entity:

```csharp
[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    [Create]
    public void Create()
    {
        CreatedDate = DateTime.UtcNow;
    }

    [Remote]
    [Fetch]
    public async Task<bool> Fetch([Service] IPersonDbContext db)
    {
        var entity = await db.Persons.FindAsync(Id);
        if (entity == null) return false;

        LoadProperty(nameof(Id), entity.Id);
        LoadProperty(nameof(FirstName), entity.FirstName);
        LoadProperty(nameof(LastName), entity.LastName);
        LoadProperty(nameof(Email), entity.Email);
        LoadProperty(nameof(CreatedDate), entity.CreatedDate);
        return true;
    }

    [Remote]
    [Insert]
    public async Task Insert([Service] IPersonDbContext db)
    {
        Id = Guid.NewGuid();

        var entity = new PersonEntity
        {
            Id = Id.Value,
            FirstName = FirstName,
            LastName = LastName,
            Email = Email,
            CreatedDate = CreatedDate
        };
        db.Persons.Add(entity);
        await db.SaveChangesAsync();
    }

    [Remote]
    [Update]
    public async Task Update([Service] IPersonDbContext db)
    {
        var entity = await db.Persons.FindAsync(Id);

        if (this[nameof(FirstName)].IsModified)
            entity.FirstName = FirstName;
        if (this[nameof(LastName)].IsModified)
            entity.LastName = LastName;
        if (this[nameof(Email)].IsModified)
            entity.Email = Email;

        await db.SaveChangesAsync();
    }

    [Remote]
    [Delete]
    public async Task DeleteMethod([Service] IPersonDbContext db)
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

### Step 6: Update UI to Use Neatoo Entity

Replace the old binding with Neatoo:

```razor
@* Before *@
@code {
    private PersonViewModel _viewModel = new();
    private List<string> _errors = new();

    private async Task Save()
    {
        _errors = await _validationService.Validate(_viewModel);
        if (_errors.Any()) return;

        await _personService.SavePerson(MapToPerson(_viewModel));
    }
}

@* After *@
@inject IPersonFactory PersonFactory

@code {
    private IPerson _person;

    protected override async Task OnInitializedAsync()
    {
        _person = PersonFactory.Create();
    }

    private async Task Save()
    {
        await _person.WaitForTasks();
        if (_person.IsSavable)
        {
            await PersonFactory.Save(_person);
        }
    }
}
```

### Step 7: Remove Legacy Code

Once the Neatoo version is working:

1. Remove the old anemic entity class
2. Remove the validation service
3. Remove the repository (if using Neatoo for persistence)
4. Remove DTOs and mapping code
5. Remove old API endpoints (Neatoo uses single `/api/neatoo` endpoint)

### Step 8: Repeat for Remaining Entities

Migrate entities in dependency order:
1. Standalone entities first
2. Then entities with child collections
3. Finally, complex aggregates

## Maintaining Backward Compatibility

### Coexistence Period

During migration, both architectures can coexist:

```csharp
// Old service still works
public class PersonService
{
    public async Task<OldPerson> GetPerson(Guid id) { ... }
}

// New Neatoo factory also works
public interface INewPersonFactory
{
    Task<INewPerson> Fetch(Guid id);
}
```

### API Versioning

If you have external API consumers:

```csharp
// Keep legacy API endpoints
[Route("api/v1/persons")]
public class LegacyPersonsController
{
    [HttpGet("{id}")]
    public async Task<PersonDto> Get(Guid id)
    {
        // Use Neatoo internally, return legacy DTO
        var person = await _personFactory.Fetch(id);
        return MapToLegacyDto(person);
    }
}

// New API uses Neatoo endpoint
// POST /api/neatoo - handled automatically
```

### Gradual Rollout

Use feature flags to control which code path executes:

```csharp
public class HybridPersonService
{
    private readonly IFeatureFlags _flags;
    private readonly IPersonFactory _neatooFactory;
    private readonly IOldPersonService _oldService;

    public async Task<IPerson> GetPerson(Guid id)
    {
        if (_flags.UseNeatooEntities)
        {
            return await _neatooFactory.Fetch(id);
        }
        else
        {
            var old = await _oldService.GetPerson(id);
            return AdaptToInterface(old);
        }
    }
}
```

## Before and After Code Comparison

### Complete Example: Person Entity

**Before (Anemic Architecture):**

```csharp
// Entity - just data
public class Person
{
    public Guid Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public List<Phone> Phones { get; set; } = new();
}

// DTO for API
public class PersonDto
{
    public Guid? Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public List<PhoneDto> Phones { get; set; } = new();
}

// Validation service
public class PersonValidationService
{
    private readonly IPersonRepository _repository;

    public async Task<ValidationResult> Validate(Person person)
    {
        var errors = new List<string>();

        if (string.IsNullOrEmpty(person.FirstName))
            errors.Add("First name is required");
        if (string.IsNullOrEmpty(person.LastName))
            errors.Add("Last name is required");
        if (!string.IsNullOrEmpty(person.Email) &&
            await _repository.EmailExistsAsync(person.Email, person.Id))
            errors.Add("Email already in use");

        return new ValidationResult(errors);
    }
}

// Repository
public class PersonRepository : IPersonRepository
{
    private readonly AppDbContext _context;

    public async Task<Person> GetByIdAsync(Guid id)
    {
        var entity = await _context.Persons
            .Include(p => p.Phones)
            .FirstOrDefaultAsync(p => p.Id == id);
        return MapToDomain(entity);
    }

    public async Task SaveAsync(Person person)
    {
        if (person.Id == Guid.Empty)
        {
            person.Id = Guid.NewGuid();
            var entity = MapToEntity(person);
            _context.Persons.Add(entity);
        }
        else
        {
            var entity = await _context.Persons.FindAsync(person.Id);
            UpdateEntity(entity, person);
        }
        await _context.SaveChangesAsync();
    }

    // Many more mapping methods...
}

// Service orchestrates everything
public class PersonService
{
    private readonly IPersonRepository _repository;
    private readonly PersonValidationService _validationService;

    public async Task<Person> CreatePerson()
    {
        return new Person();
    }

    public async Task<Person> GetPerson(Guid id)
    {
        return await _repository.GetByIdAsync(id);
    }

    public async Task SavePerson(Person person)
    {
        var result = await _validationService.Validate(person);
        if (!result.IsValid)
            throw new ValidationException(result.Errors);

        await _repository.SaveAsync(person);
    }
}

// API Controller
[ApiController]
[Route("api/[controller]")]
public class PersonsController : ControllerBase
{
    private readonly PersonService _service;
    private readonly IMapper _mapper;

    [HttpGet("{id}")]
    public async Task<ActionResult<PersonDto>> Get(Guid id)
    {
        var person = await _service.GetPerson(id);
        if (person == null) return NotFound();
        return _mapper.Map<PersonDto>(person);
    }

    [HttpPost]
    public async Task<ActionResult<PersonDto>> Create(PersonDto dto)
    {
        var person = _mapper.Map<Person>(dto);
        await _service.SavePerson(person);
        return CreatedAtAction(nameof(Get), new { id = person.Id },
            _mapper.Map<PersonDto>(person));
    }
}

// Blazor Component
@page "/person/edit/{Id:guid}"
@inject HttpClient Http
@inject PersonValidationService ValidationService

<EditForm Model="_dto">
    <input @bind="_dto.FirstName" />
    <input @bind="_dto.LastName" />
    <input @bind="_dto.Email" />

    @foreach (var error in _errors)
    {
        <div class="error">@error</div>
    }

    <button @onclick="Save">Save</button>
</EditForm>

@code {
    private PersonDto _dto = new();
    private List<string> _errors = new();

    protected override async Task OnInitializedAsync()
    {
        _dto = await Http.GetFromJsonAsync<PersonDto>($"api/persons/{Id}");
    }

    private async Task Save()
    {
        // Client-side validation (duplicated from server)
        _errors.Clear();
        if (string.IsNullOrEmpty(_dto.FirstName))
            _errors.Add("First name is required");
        if (string.IsNullOrEmpty(_dto.LastName))
            _errors.Add("Last name is required");

        if (_errors.Any()) return;

        var response = await Http.PostAsJsonAsync("api/persons", _dto);
        if (!response.IsSuccessStatusCode)
        {
            var serverErrors = await response.Content.ReadFromJsonAsync<List<string>>();
            _errors.AddRange(serverErrors);
        }
    }
}
```

**After (Neatoo):**

```csharp
// Domain Entity with behavior
public partial interface IPerson : IEntityBase
{
    Guid? Id { get; set; }
    string? FirstName { get; set; }
    string? LastName { get; set; }
    string? Email { get; set; }
    IPersonPhoneList? Phones { get; set; }
}

[Factory]
internal partial class Person : EntityBase<Person>, IPerson
{
    private readonly IPersonPhoneListFactory _phoneListFactory;

    public Person(IEntityBaseServices<Person> services,
                  IPersonPhoneListFactory phoneListFactory,
                  IUniqueEmailRule uniqueEmailRule) : base(services)
    {
        _phoneListFactory = phoneListFactory;
        RuleManager.AddRule(uniqueEmailRule);
    }

    public partial Guid? Id { get; set; }

    [Required(ErrorMessage = "First name is required")]
    public partial string? FirstName { get; set; }

    [Required(ErrorMessage = "Last name is required")]
    public partial string? LastName { get; set; }

    public partial string? Email { get; set; }

    // Child collection
    public partial IPersonPhoneList? Phones { get; set; }

    [Create]
    public void Create()
    {
        Phones = _phoneListFactory.Create();
    }

    [Remote]
    [Fetch]
    public async Task<bool> Fetch([Service] IPersonDbContext db)
    {
        var entity = await db.Persons
            .Include(p => p.Phones)
            .FirstOrDefaultAsync(p => p.Id == Id);

        if (entity == null) return false;

        // Load properties without triggering rules
        LoadProperty(nameof(Id), entity.Id);
        LoadProperty(nameof(FirstName), entity.FirstName);
        LoadProperty(nameof(LastName), entity.LastName);
        LoadProperty(nameof(Email), entity.Email);

        Phones = await _phoneListFactory.Fetch(entity.Phones);
        return true;
    }

    [Remote]
    [Insert]
    public async Task Insert([Service] IPersonDbContext db)
    {
        Id = Guid.NewGuid();

        var entity = new PersonEntity
        {
            Id = Id.Value,
            FirstName = FirstName,
            LastName = LastName,
            Email = Email
        };
        db.Persons.Add(entity);
        await _phoneListFactory.Save(Phones, Id.Value);
        await db.SaveChangesAsync();
    }

    [Remote]
    [Update]
    public async Task Update([Service] IPersonDbContext db)
    {
        var entity = await db.Persons.FindAsync(Id);

        // Update only modified properties
        if (this[nameof(FirstName)].IsModified)
            entity.FirstName = FirstName;
        if (this[nameof(LastName)].IsModified)
            entity.LastName = LastName;
        if (this[nameof(Email)].IsModified)
            entity.Email = Email;

        await _phoneListFactory.Save(Phones, Id.Value);
        await db.SaveChangesAsync();
    }

    [Remote]
    [Delete]
    public async Task DeleteMethod([Service] IPersonDbContext db)
    {
        var entity = await db.Persons.FindAsync(Id);
        if (entity != null)
        {
            db.Persons.Remove(entity);
            await db.SaveChangesAsync();
        }
    }
}

// Blazor Component - much simpler
@page "/person/edit/{Id:guid}"
@inject IPersonFactory PersonFactory

<EditForm Model="_person">
    <MudNeatooTextField For="() => _person.FirstName" Label="First Name" />
    <MudNeatooTextField For="() => _person.LastName" Label="Last Name" />
    <MudNeatooTextField For="() => _person.Email" Label="Email" />

    <NeatooValidationSummary Entity="_person" />

    <button @onclick="Save" disabled="@(!_person.IsSavable)">Save</button>
</EditForm>

@code {
    [Parameter] public Guid Id { get; set; }
    private IPerson _person;

    protected override async Task OnInitializedAsync()
    {
        _person = await PersonFactory.Fetch(Id);
    }

    private async Task Save()
    {
        await _person.WaitForTasks();
        if (_person.IsSavable)
        {
            await PersonFactory.Save(_person);
        }
    }
}
```

## Addressing Common Concerns

### "This is a big change for our team"

Start small:
1. Migrate one entity as a proof of concept
2. Let the team experience the benefits firsthand
3. Create internal documentation and patterns
4. Gradually expand to more entities

### "We have extensive API contracts"

Neatoo can coexist with REST APIs:
- Keep legacy endpoints for external consumers
- Use Neatoo internally for Blazor applications
- Gradually migrate as contracts allow

### "What about testing?"

Neatoo improves testability:
- Rules are unit-testable classes
- Entities can be instantiated with mock services
- No service layer to mock for validation testing

```csharp
[Fact]
public async Task UniqueEmailRule_WhenEmailExists_ReturnsError()
{
    var mockService = new Mock<IEmailService>();
    mockService.Setup(s => s.EmailExistsAsync("taken@example.com", It.IsAny<Guid?>()))
        .ReturnsAsync(true);

    var rule = new UniqueEmailRule(mockService.Object);
    var person = new MockPerson { Email = "taken@example.com" };

    var result = await rule.Execute(person);

    Assert.Contains(result, m => m.Message.Contains("already in use"));
}
```

### "What if Neatoo does not support our use case?"

Neatoo is extensible:
- Custom rules for complex logic
- Factory methods can call any service
- Authorization rules for any security model
- Standard .NET patterns work alongside Neatoo

## Related Topics

- [Introduction to Neatoo](/introduction/) - Framework overview
- [DDD Concepts in Neatoo](/concepts/ddd-overview/) - How Neatoo implements DDD
- [Rules Philosophy](/concepts/rules-philosophy/) - Business rules as first-class citizens
- [Factory Pattern](/concepts/factories-overview/) - Understanding factories
- [EntityBase Reference](/reference/entity-base/) - Complete entity API
