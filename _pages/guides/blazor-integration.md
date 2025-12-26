---
title: "Blazor UI Integration"
layout: single
permalink: /guides/blazor-integration/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

Neatoo entities are designed to integrate seamlessly with Blazor applications. Every Neatoo entity implements `INotifyPropertyChanged`, meaning UI components automatically update when entity properties change. This page covers the patterns and components for building reactive Blazor UIs with Neatoo.

## Automatic UI Binding

All Neatoo entities inherit from `Base<T>`, which implements `INotifyPropertyChanged`. When you bind a Blazor component to an entity property, changes propagate automatically:

```razor
@page "/person/edit/{Id:guid}"
@inject IPersonFactory PersonFactory

<h1>Edit Person</h1>

<EditForm Model="@_person">
    <div class="form-group">
        <label>First Name</label>
        <InputText @bind-Value="_person.FirstName" class="form-control" />
    </div>

    <div class="form-group">
        <label>Last Name</label>
        <InputText @bind-Value="_person.LastName" class="form-control" />
    </div>

    <button type="button" @onclick="Save" disabled="@(!_person.IsSavable)">
        Save
    </button>
</EditForm>

@code {
    [Parameter] public Guid Id { get; set; }

    private IPerson _person = default!;

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

**Why This Works:**

- `_person.FirstName` changes trigger `PropertyChanged`
- Blazor detects the event and re-renders
- Meta-properties like `IsSavable` also raise `PropertyChanged`, keeping the save button state synchronized

## MudNeatoo Component Library

The `Neatoo.Blazor.MudNeatoo` package provides MudBlazor-integrated components that simplify entity binding and validation display. These components wrap MudBlazor inputs with automatic Neatoo property integration.

### Installation

```bash
dotnet add package Neatoo.Blazor.MudNeatoo
```

### MudNeatooTextField

Text input with automatic validation display and busy state handling:

```razor
@using Neatoo.Blazor.MudNeatoo

<MudNeatooTextField For="() => _person.FirstName"
                    Label="First Name"
                    Required="true" />

<MudNeatooTextField For="() => _person.Email"
                    Label="Email"
                    InputType="InputType.Email" />
```

**Features:**

- Automatically reads and writes property values
- Displays validation messages from `PropertyMessages`
- Shows busy indicator when the property has async rules running
- Applies error styling when property is invalid
- Uses `DisplayName` attribute for label if `Label` not specified

### MudNeatooNumericField

Numeric input with type safety:

```razor
<MudNeatooNumericField For="() => _orderLine.Quantity"
                       Label="Quantity"
                       Min="1"
                       Max="1000" />

<MudNeatooNumericField For="() => _orderLine.UnitPrice"
                       Label="Unit Price"
                       Format="C2"
                       Adornment="Adornment.Start"
                       AdornmentIcon="@Icons.Material.Filled.AttachMoney" />
```

### MudNeatooDatePicker

Date input with validation:

```razor
<MudNeatooDatePicker For="() => _order.OrderDate"
                     Label="Order Date"
                     Required="true" />

<MudNeatooDatePicker For="() => _order.ShipDate"
                     Label="Ship Date"
                     MinDate="@_order.OrderDate" />
```

### MudNeatooSelect

Dropdown selection with enum or collection support:

```razor
@* Enum binding *@
<MudNeatooSelect For="() => _phone.PhoneType"
                 Label="Phone Type">
    @foreach (PhoneType type in Enum.GetValues<PhoneType>())
    {
        <MudSelectItem Value="@type">@type</MudSelectItem>
    }
</MudNeatooSelect>

@* Collection binding *@
<MudNeatooSelect For="() => _order.CustomerId"
                 Label="Customer"
                 T="Guid?">
    @foreach (var customer in _customers)
    {
        <MudSelectItem Value="@customer.Id">@customer.Name</MudSelectItem>
    }
</MudNeatooSelect>
```

### MudNeatooCheckBox

Boolean input:

```razor
<MudNeatooCheckBox For="() => _order.IsRush"
                   Label="Rush Order" />

<MudNeatooCheckBox For="() => _person.IsActive"
                   Label="Active"
                   Color="Color.Primary" />
```

### NeatooValidationSummary

Displays all validation messages for an entity:

```razor
<NeatooValidationSummary Entity="_person" />

@* With custom styling *@
<NeatooValidationSummary Entity="_person"
                         Class="validation-errors"
                         ShowPropertyNames="true" />
```

**Output Example:**

```html
<div class="validation-errors">
    <ul>
        <li>First Name: First Name is required</li>
        <li>Email: Invalid email format</li>
    </ul>
</div>
```

## Manual Binding Patterns

For scenarios where MudNeatoo components are not suitable, you can bind to Neatoo entities manually.

### Accessing Property Values

Standard property binding works directly:

```razor
<input @bind="_person.FirstName" />
<input @bind="_person.LastName" />
```

### Using SetValue for Changes

For more control over property changes, access the property wrapper:

```razor
<input value="@_person.FirstName"
       @onchange="@(e => SetFirstName(e.Value?.ToString()))" />

@code {
    private void SetFirstName(string? value)
    {
        // Access the property wrapper for additional control
        var prop = _person[nameof(_person.FirstName)];
        prop.SetValue(value);
    }
}
```

### Displaying PropertyMessages

Show validation errors for specific properties:

```razor
<div class="form-group">
    <label>Email</label>
    <input @bind="_person.Email"
           class="@(GetInputClass(nameof(_person.Email)))" />

    @foreach (var msg in GetMessagesFor(nameof(_person.Email)))
    {
        <span class="text-danger">@msg.Message</span>
    }
</div>

@code {
    private string GetInputClass(string propertyName)
    {
        return _person[propertyName].IsValid ? "form-control" : "form-control is-invalid";
    }

    private IEnumerable<IPropertyMessage> GetMessagesFor(string propertyName)
    {
        return _person.PropertyMessages
            .Where(m => m.PropertyName == propertyName);
    }
}
```

### Property-Level Meta-Properties

Access detailed state for each property:

```razor
@{
    var emailProp = _person[nameof(_person.Email)];
}

<div class="form-group @(emailProp.IsValid ? "" : "has-error")">
    <label>Email</label>
    <input @bind="_person.Email"
           disabled="@emailProp.IsBusy" />

    @if (emailProp.IsBusy)
    {
        <span class="spinner">Validating...</span>
    }

    @if (!emailProp.IsValid)
    {
        @foreach (var msg in emailProp.PropertyMessages)
        {
            <span class="text-danger">@msg.Message</span>
        }
    }

    @if (emailProp.IsModified)
    {
        <span class="badge">Modified</span>
    }
</div>
```

## Validation Display Patterns

### Inline Validation

Show validation messages immediately next to the field:

```razor
<div class="form-group">
    <MudNeatooTextField For="() => _person.Email" Label="Email" />
    @* MudNeatoo components show validation inline automatically *@
</div>
```

Or manually:

```razor
<div class="form-group">
    <label>Email</label>
    <input @bind="_person.Email" />
    <ValidationMessage For="@(() => _person.Email)" />
</div>

@code {
    // Custom ValidationMessage component for Neatoo
    public class ValidationMessage : ComponentBase
    {
        [Parameter] public Expression<Func<object>> For { get; set; }
        // Implementation reads from entity PropertyMessages
    }
}
```

### Summary Validation

Show all errors in one location:

```razor
<EditForm Model="@_person">
    @* Form fields here *@

    <div class="validation-summary">
        @if (!_person.IsValid)
        {
            <h4>Please correct the following errors:</h4>
            <ul>
                @foreach (var msg in _person.PropertyMessages)
                {
                    <li><strong>@msg.PropertyName:</strong> @msg.Message</li>
                }
            </ul>
        }
    </div>

    <button disabled="@(!_person.IsSavable)">Save</button>
</EditForm>
```

### Combined Approach

Use inline validation for immediate feedback and summary for save-time overview:

```razor
<EditForm Model="@_order">
    @* Inline validation on each field *@
    <MudNeatooTextField For="() => _order.CustomerName" Label="Customer" />
    <MudNeatooTextField For="() => _order.ShippingAddress" Label="Shipping Address" />

    @* Summary shown only when attempting to save *@
    @if (_showValidationSummary && !_order.IsValid)
    {
        <NeatooValidationSummary Entity="_order" />
    }

    <button @onclick="TrySave">Save</button>
</EditForm>

@code {
    private bool _showValidationSummary;

    private async Task TrySave()
    {
        _showValidationSummary = true;
        await _order.WaitForTasks();

        if (_order.IsSavable)
        {
            await OrderFactory.Save(_order);
            _showValidationSummary = false;
        }
    }
}
```

## Busy State Handling

Neatoo tracks async operations through `IsBusy` at both entity and property levels. Use these to provide loading feedback.

### Entity-Level IsBusy

Shows when any async operation is running on the entity or its children:

```razor
<div class="@(_person.IsBusy ? "loading" : "")">
    <EditForm Model="@_person">
        @* Form fields *@

        <button disabled="@(_person.IsBusy || !_person.IsSavable)">
            @if (_person.IsBusy)
            {
                <span class="spinner-border spinner-border-sm"></span>
                <span>Validating...</span>
            }
            else
            {
                <span>Save</span>
            }
        </button>
    </EditForm>
</div>

@if (_person.IsBusy)
{
    <div class="overlay">
        <MudProgressCircular Indeterminate="true" />
    </div>
}
```

### Property-Level IsBusy

For fine-grained loading indicators on specific fields:

```razor
<div class="form-group">
    <label>Email</label>
    <div class="input-group">
        <input @bind="_person.Email"
               disabled="@_person[nameof(_person.Email)].IsBusy" />

        @if (_person[nameof(_person.Email)].IsBusy)
        {
            <div class="input-group-append">
                <span class="spinner-border spinner-border-sm"></span>
            </div>
        }
    </div>
    <small class="text-muted">
        @(_person[nameof(_person.Email)].IsBusy
            ? "Checking availability..."
            : "")
    </small>
</div>
```

### Preventing Premature Actions

Always wait for async operations before checking validity:

```razor
@code {
    private async Task Save()
    {
        // WRONG: Might check before async rules complete
        // if (_person.IsSavable) await PersonFactory.Save(_person);

        // CORRECT: Wait for all async operations
        await _person.WaitForTasks();

        if (_person.IsSavable)
        {
            await PersonFactory.Save(_person);
            NavigationManager.NavigateTo("/persons");
        }
    }
}
```

## The Save Button Pattern

A robust save button implementation should:

1. Disable when entity is not savable
2. Wait for async rules to complete
3. Show appropriate loading states
4. Handle both success and failure

### Basic Save Button

```razor
<button @onclick="Save"
        disabled="@(!_person.IsSavable)"
        class="btn btn-primary">
    Save
</button>
```

### Complete Save Button Implementation

```razor
<button @onclick="HandleSave"
        disabled="@(!CanSave)"
        class="btn btn-primary">
    @if (_isSaving)
    {
        <span class="spinner-border spinner-border-sm"></span>
        <span>Saving...</span>
    }
    else if (_person.IsBusy)
    {
        <span>Validating...</span>
    }
    else
    {
        <span>Save</span>
    }
</button>

@if (!_person.IsModified && !_person.IsNew)
{
    <span class="text-muted ms-2">No changes</span>
}

@code {
    private bool _isSaving;

    private bool CanSave => !_isSaving && _person.IsSavable;

    private async Task HandleSave()
    {
        // Wait for any pending validations
        await _person.WaitForTasks();

        // Re-check after waiting
        if (!_person.IsSavable)
        {
            StateHasChanged();
            return;
        }

        try
        {
            _isSaving = true;
            StateHasChanged();

            await PersonFactory.Save(_person);

            // Navigate on success
            NavigationManager.NavigateTo("/persons");
        }
        catch (Exception ex)
        {
            // Handle save failure
            Snackbar.Add($"Save failed: {ex.Message}", Severity.Error);
        }
        finally
        {
            _isSaving = false;
        }
    }
}
```

### Why WaitForTasks() Before Save

When a property changes, async validation rules may execute. Without `WaitForTasks()`, you might:

1. Check `IsSavable` before rules complete
2. Get `true` because `IsBusy` is still `true` (makes `IsSavable` false anyway)
3. Or worse, start saving while validation is incomplete

```csharp
// The IsSavable property already checks IsBusy:
public virtual bool IsSavable => IsModified && IsValid && !IsBusy && !IsChild;
```

Using `WaitForTasks()` ensures all rules have completed and `IsValid` reflects the true validation state.

## Collection Binding with Tables

Entity lists (`EntityListBase<T>`) bind naturally to Blazor table components.

### Basic Table Binding

```razor
<table class="table">
    <thead>
        <tr>
            <th>Product</th>
            <th>Quantity</th>
            <th>Unit Price</th>
            <th>Total</th>
            <th></th>
        </tr>
    </thead>
    <tbody>
        @foreach (var line in _order.Lines)
        {
            <tr class="@(line.IsValid ? "" : "table-danger")">
                <td>
                    <input @bind="line.ProductName" />
                </td>
                <td>
                    <input type="number" @bind="line.Quantity" />
                </td>
                <td>
                    <input type="number" @bind="line.UnitPrice" step="0.01" />
                </td>
                <td>@line.Total.ToString("C")</td>
                <td>
                    <button @onclick="() => RemoveLine(line)"
                            class="btn btn-danger btn-sm">
                        Remove
                    </button>
                </td>
            </tr>
        }
    </tbody>
</table>

<button @onclick="AddLine" class="btn btn-secondary">Add Line</button>

@code {
    private async Task AddLine()
    {
        var line = await OrderLineFactory.Create();
        _order.Lines.Add(line);
    }

    private void RemoveLine(IOrderLine line)
    {
        _order.Lines.Remove(line);
        // If line was persisted, it moves to DeletedList
        // If line was new, it's simply removed
    }
}
```

### MudBlazor DataGrid Binding

```razor
<MudDataGrid Items="@_order.Lines"
             Bordered="true"
             Dense="true"
             EditMode="DataGridEditMode.Cell">
    <Columns>
        <PropertyColumn Property="x => x.ProductName" Title="Product">
            <EditTemplate>
                <MudNeatooTextField For="() => context.Item.ProductName" />
            </EditTemplate>
        </PropertyColumn>

        <PropertyColumn Property="x => x.Quantity" Title="Qty">
            <EditTemplate>
                <MudNeatooNumericField For="() => context.Item.Quantity" />
            </EditTemplate>
        </PropertyColumn>

        <PropertyColumn Property="x => x.UnitPrice" Title="Price" Format="C2" />

        <PropertyColumn Property="x => x.Total" Title="Total" Format="C2" />

        <TemplateColumn>
            <CellTemplate>
                <MudIconButton Icon="@Icons.Material.Filled.Delete"
                               OnClick="() => RemoveLine(context.Item)"
                               Color="Color.Error" />
            </CellTemplate>
        </TemplateColumn>
    </Columns>
</MudDataGrid>
```

### Showing Row-Level Validation

```razor
<MudDataGrid Items="@_order.Lines">
    <Columns>
        @* ... columns ... *@

        <TemplateColumn Title="Status">
            <CellTemplate>
                @if (!context.Item.IsValid)
                {
                    <MudTooltip>
                        <ChildContent>
                            <MudIcon Icon="@Icons.Material.Filled.Error"
                                     Color="Color.Error" />
                        </ChildContent>
                        <TooltipContent>
                            <ul>
                                @foreach (var msg in context.Item.PropertyMessages)
                                {
                                    <li>@msg.Message</li>
                                }
                            </ul>
                        </TooltipContent>
                    </MudTooltip>
                }
                else
                {
                    <MudIcon Icon="@Icons.Material.Filled.CheckCircle"
                             Color="Color.Success" />
                }
            </CellTemplate>
        </TemplateColumn>
    </Columns>
</MudDataGrid>
```

## Authorization-Aware UI Patterns

Use factory authorization methods to conditionally render UI elements.

### Hiding Unauthorized Actions

```razor
@if (PersonFactory.CanCreate().IsAuthorized)
{
    <MudButton OnClick="CreateNew" Color="Color.Primary">
        Create New Person
    </MudButton>
}

@if (PersonFactory.CanDelete().IsAuthorized)
{
    <MudButton OnClick="Delete" Color="Color.Error">
        Delete
    </MudButton>
}
```

### Showing Authorization Messages

```razor
@{
    var canUpdate = PersonFactory.CanUpdate();
}

@if (!canUpdate.IsAuthorized)
{
    <MudAlert Severity="Severity.Warning">
        @canUpdate.Message
    </MudAlert>

    @* Show read-only view *@
    <PersonReadOnlyView Person="_person" />
}
else
{
    @* Show editable form *@
    <PersonEditForm Person="_person" />
}
```

### Complete Authorization Pattern

```razor
@page "/person/edit/{Id:guid}"
@inject IPersonFactory PersonFactory

@if (_loading)
{
    <MudProgressCircular Indeterminate="true" />
}
else if (_person == null)
{
    <MudAlert Severity="Severity.Error">Person not found</MudAlert>
}
else
{
    <MudPaper Class="pa-4">
        <MudForm>
            @if (_canEdit)
            {
                <MudNeatooTextField For="() => _person.FirstName" Label="First Name" />
                <MudNeatooTextField For="() => _person.LastName" Label="Last Name" />
                <MudNeatooTextField For="() => _person.Email" Label="Email" />
            }
            else
            {
                <MudText>First Name: @_person.FirstName</MudText>
                <MudText>Last Name: @_person.LastName</MudText>
                <MudText>Email: @_person.Email</MudText>
            }
        </MudForm>

        <MudDivider Class="my-4" />

        <MudStack Row="true" Justify="Justify.FlexEnd">
            @if (_canEdit && PersonFactory.CanSave().IsAuthorized)
            {
                <MudButton OnClick="Save"
                           Disabled="@(!_person.IsSavable)"
                           Color="Color.Primary"
                           Variant="Variant.Filled">
                    Save
                </MudButton>
            }

            @if (PersonFactory.CanDelete().IsAuthorized && !_person.IsNew)
            {
                <MudButton OnClick="Delete"
                           Color="Color.Error"
                           Variant="Variant.Outlined">
                    Delete
                </MudButton>
            }

            <MudButton OnClick="Cancel" Variant="Variant.Text">
                Cancel
            </MudButton>
        </MudStack>
    </MudPaper>
}

@code {
    [Parameter] public Guid Id { get; set; }

    private IPerson? _person;
    private bool _loading = true;
    private bool _canEdit;

    protected override async Task OnInitializedAsync()
    {
        _person = await PersonFactory.Fetch(Id);
        _canEdit = PersonFactory.CanUpdate().IsAuthorized;
        _loading = false;
    }

    private async Task Save()
    {
        await _person!.WaitForTasks();
        if (_person.IsSavable)
        {
            await PersonFactory.Save(_person);
            NavigationManager.NavigateTo("/persons");
        }
    }

    private async Task Delete()
    {
        var confirm = await DialogService.ShowMessageBox(
            "Confirm Delete",
            "Are you sure you want to delete this person?",
            yesText: "Delete", cancelText: "Cancel");

        if (confirm == true)
        {
            _person!.Delete();
            await PersonFactory.Save(_person);
            NavigationManager.NavigateTo("/persons");
        }
    }

    private void Cancel()
    {
        NavigationManager.NavigateTo("/persons");
    }
}
```

## Best Practices

### Always Wait for Tasks Before Save

```csharp
private async Task Save()
{
    await _entity.WaitForTasks();  // Never skip this
    if (_entity.IsSavable)
    {
        await Factory.Save(_entity);
    }
}
```

### Check IsSavable, Not Just IsValid

`IsSavable` includes all necessary checks:

```csharp
// IsSavable checks: IsModified && IsValid && !IsBusy && !IsChild
// Always use IsSavable for save button state
<button disabled="@(!_person.IsSavable)">Save</button>
```

### Handle Unsaved Changes on Navigation

```razor
@inject NavigationManager NavigationManager

@code {
    protected override void OnInitialized()
    {
        NavigationManager.RegisterLocationChangingHandler(OnLocationChanging);
    }

    private async ValueTask OnLocationChanging(LocationChangingContext context)
    {
        if (_person?.IsModified == true)
        {
            var confirmed = await DialogService.ShowMessageBox(
                "Unsaved Changes",
                "You have unsaved changes. Leave anyway?",
                yesText: "Leave", cancelText: "Stay");

            if (confirmed != true)
            {
                context.PreventNavigation();
            }
        }
    }
}
```

### Dispose Entity Subscriptions

If you subscribe to entity events manually, dispose them:

```razor
@implements IDisposable

@code {
    private IPerson _person;

    protected override async Task OnInitializedAsync()
    {
        _person = await PersonFactory.Fetch(Id);
        _person.PropertyChanged += OnPropertyChanged;
    }

    private void OnPropertyChanged(object? sender, PropertyChangedEventArgs e)
    {
        InvokeAsync(StateHasChanged);
    }

    public void Dispose()
    {
        if (_person != null)
        {
            _person.PropertyChanged -= OnPropertyChanged;
        }
    }
}
```

## Related Topics

- [EntityBase Reference](/reference/entity-base/) - Complete entity API
- [Properties and Meta-Properties](/concepts/properties/) - Property system details
- [Client Setup](/setup/client/) - Configuring Blazor clients
- [Authorization Concept](/concepts/authorization/) - Authorization model
- [Person Example](/example/person/) - Complete working example
