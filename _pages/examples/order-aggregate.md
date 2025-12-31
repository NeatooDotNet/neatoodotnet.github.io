---
title: "Complete Example: Order Aggregate"
layout: single
permalink: /examples/order-aggregate/
toc: true
toc_sticky: true
sidebar:
    nav: "central"
---

This page presents a complete, end-to-end implementation of an Order aggregate using Neatoo. The example demonstrates all core Neatoo concepts working together: aggregate roots, child entities, child collections, business rules, factory operations, data mapping, and Blazor UI integration.

## The Domain Model

An Order is a classic DDD aggregate example. The Order aggregate consists of:

- **Order** (Aggregate Root) - The main order with customer information and totals
- **OrderLineItemList** (Child Collection) - The list of line items
- **OrderLineItem** (Child Entity) - Individual products being ordered

```
┌─────────────────────────────────────────────────────────────┐
│                        Order                                │
│                    (Aggregate Root)                         │
├─────────────────────────────────────────────────────────────┤
│  Id: Guid                                                   │
│  OrderNumber: string                                        │
│  CustomerName: string (required)                            │
│  OrderDate: DateTime                                        │
│  ShipToAddress: string (required)                           │
│  Subtotal: decimal (calculated)                             │
│  TaxRate: decimal                                           │
│  Tax: decimal (calculated)                                  │
│  Total: decimal (calculated)                                │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              OrderLineItemList                       │    │
│  ├─────────────────────────────────────────────────────┤    │
│  │                                                     │    │
│  │  ┌─────────────────┐  ┌─────────────────┐          │    │
│  │  │ OrderLineItem   │  │ OrderLineItem   │  ...     │    │
│  │  ├─────────────────┤  ├─────────────────┤          │    │
│  │  │ ProductId       │  │ ProductId       │          │    │
│  │  │ ProductName     │  │ ProductName     │          │    │
│  │  │ Quantity        │  │ Quantity        │          │    │
│  │  │ UnitPrice       │  │ UnitPrice       │          │    │
│  │  │ LineTotal       │  │ LineTotal       │          │    │
│  │  └─────────────────┘  └─────────────────┘          │    │
│  │                                                     │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

## Entity Interfaces

Define interfaces for each entity type. Interfaces enable loose coupling and support dependency injection.

### IOrder Interface

```csharp
public interface IOrder : IEntityBase
{
    Guid? Id { get; set; }
    string? OrderNumber { get; set; }
    string? CustomerName { get; set; }
    DateTime OrderDate { get; set; }
    string? ShipToAddress { get; set; }
    decimal TaxRate { get; set; }

    // Calculated properties
    decimal Subtotal { get; set; }
    decimal Tax { get; set; }
    decimal Total { get; set; }

    // Child collection
    IOrderLineItemList? LineItems { get; set; }
}
```

### IOrderLineItemList Interface

```csharp
public interface IOrderLineItemList : IEntityListBase<IOrderLineItem>
{
}
```

### IOrderLineItem Interface

```csharp
public interface IOrderLineItem : IEntityBase
{
    Guid? Id { get; set; }
    Guid? ProductId { get; set; }
    string? ProductName { get; set; }
    int Quantity { get; set; }
    decimal UnitPrice { get; set; }

    // Calculated
    decimal LineTotal { get; set; }

    // Navigation to parent
    IOrder? ParentOrder { get; }
}
```

## Entity Implementations

### Order (Aggregate Root)

```csharp
using System.ComponentModel.DataAnnotations;
using Neatoo;

namespace OrderDomain;

[Factory]
internal partial class Order : EntityBase<Order>, IOrder
{
    public Order(
        IEntityBaseServices<Order> services,
        IOrderTotalRule orderTotalRule) : base(services)
    {
        RuleManager.AddRule(orderTotalRule);
    }

    // Identity
    public partial Guid? Id { get; set; }

    // Order information
    [DisplayName("Order Number")]
    public partial string? OrderNumber { get; set; }

    [Required(ErrorMessage = "Customer name is required")]
    [DisplayName("Customer Name")]
    public partial string? CustomerName { get; set; }

    [DisplayName("Order Date")]
    public partial DateTime OrderDate { get; set; }

    [Required(ErrorMessage = "Shipping address is required")]
    [DisplayName("Ship To Address")]
    public partial string? ShipToAddress { get; set; }

    [Range(0, 1, ErrorMessage = "Tax rate must be between 0 and 100%")]
    [DisplayName("Tax Rate")]
    public partial decimal TaxRate { get; set; }

    // Calculated totals (set by rules)
    public partial decimal Subtotal { get; set; }
    public partial decimal Tax { get; set; }
    public partial decimal Total { get; set; }

    // Child collection
    public partial IOrderLineItemList? LineItems { get; set; }

    // Factory Methods

    [Create]
    public void Create([Service] IOrderLineItemListFactory lineItemListFactory)
    {
        LineItems = lineItemListFactory.Create();
        OrderDate = DateTime.Today;
        TaxRate = 0.08m; // Default 8% tax
    }

    [Remote]
    [Fetch]
    public async Task<bool> Fetch(
        [Service] IOrderDbContext dbContext,
        [Service] IOrderLineItemListFactory lineItemListFactory)
    {
        var entity = await dbContext.Orders
            .Include(o => o.LineItems)
            .FirstOrDefaultAsync(o => o.Id == Id);

        if (entity == null) return false;

        // Load properties without triggering rules
        LoadProperty(nameof(Id), entity.Id);
        LoadProperty(nameof(OrderNumber), entity.OrderNumber);
        LoadProperty(nameof(OrderDate), entity.OrderDate);
        LoadProperty(nameof(CustomerName), entity.CustomerName);
        LoadProperty(nameof(ShippingAddress), entity.ShippingAddress);
        LoadProperty(nameof(TaxRate), entity.TaxRate);
        LoadProperty(nameof(Subtotal), entity.Subtotal);
        LoadProperty(nameof(Tax), entity.Tax);
        LoadProperty(nameof(Total), entity.Total);

        LineItems = await lineItemListFactory.Fetch(entity.LineItems);
        return true;
    }

    [Remote]
    [Fetch]
    public async Task<bool> FetchByOrderNumber(
        string orderNumber,
        [Service] IOrderDbContext dbContext,
        [Service] IOrderLineItemListFactory lineItemListFactory)
    {
        var entity = await dbContext.Orders
            .Include(o => o.LineItems)
            .FirstOrDefaultAsync(o => o.OrderNumber == orderNumber);

        if (entity == null) return false;

        // Load properties without triggering rules
        LoadProperty(nameof(Id), entity.Id);
        LoadProperty(nameof(OrderNumber), entity.OrderNumber);
        LoadProperty(nameof(OrderDate), entity.OrderDate);
        LoadProperty(nameof(CustomerName), entity.CustomerName);
        LoadProperty(nameof(ShippingAddress), entity.ShippingAddress);
        LoadProperty(nameof(TaxRate), entity.TaxRate);
        LoadProperty(nameof(Subtotal), entity.Subtotal);
        LoadProperty(nameof(Tax), entity.Tax);
        LoadProperty(nameof(Total), entity.Total);

        LineItems = await lineItemListFactory.Fetch(entity.LineItems);
        return true;
    }

    [Remote]
    [Insert]
    public async Task Insert(
        [Service] IOrderDbContext dbContext,
        [Service] IOrderLineItemListFactory lineItemListFactory)
    {
        Id = Guid.NewGuid();
        OrderNumber = await GenerateOrderNumber(dbContext);

        var entity = new OrderEntity
        {
            Id = Id.Value,
            OrderNumber = OrderNumber,
            OrderDate = OrderDate,
            CustomerName = CustomerName,
            ShippingAddress = ShippingAddress,
            TaxRate = TaxRate,
            Subtotal = Subtotal,
            Tax = Tax,
            Total = Total
        };
        dbContext.Orders.Add(entity);

        // Save child collection
        await lineItemListFactory.Save(LineItems, Id.Value);

        await dbContext.SaveChangesAsync();
    }

    [Remote]
    [Update]
    public async Task Update(
        [Service] IOrderDbContext dbContext,
        [Service] IOrderLineItemListFactory lineItemListFactory)
    {
        var entity = await dbContext.Orders.FindAsync(Id);
        if (entity == null)
            throw new InvalidOperationException($"Order {Id} not found");

        // Update only modified properties
        if (this[nameof(OrderDate)].IsModified)
            entity.OrderDate = OrderDate;
        if (this[nameof(CustomerName)].IsModified)
            entity.CustomerName = CustomerName;
        if (this[nameof(ShippingAddress)].IsModified)
            entity.ShippingAddress = ShippingAddress;
        if (this[nameof(TaxRate)].IsModified)
            entity.TaxRate = TaxRate;
        if (this[nameof(Subtotal)].IsModified)
            entity.Subtotal = Subtotal;
        if (this[nameof(Tax)].IsModified)
            entity.Tax = Tax;
        if (this[nameof(Total)].IsModified)
            entity.Total = Total;

        // Save child collection (handles inserts, updates, deletes)
        await lineItemListFactory.Save(LineItems, Id!.Value);

        await dbContext.SaveChangesAsync();
    }

    [Remote]
    [Delete]
    public async Task Delete([Service] IOrderDbContext dbContext)
    {
        var entity = await dbContext.Orders
            .Include(o => o.LineItems)
            .FirstOrDefaultAsync(o => o.Id == Id);

        if (entity != null)
        {
            // EF Core cascade delete handles line items
            dbContext.Orders.Remove(entity);
            await dbContext.SaveChangesAsync();
        }
    }

    // Helper method for generating order numbers
    private async Task<string> GenerateOrderNumber(IOrderDbContext dbContext)
    {
        var today = DateTime.Today;
        var prefix = $"ORD-{today:yyyyMMdd}-";

        var lastOrder = await dbContext.Orders
            .Where(o => o.OrderNumber!.StartsWith(prefix))
            .OrderByDescending(o => o.OrderNumber)
            .FirstOrDefaultAsync();

        var sequence = 1;
        if (lastOrder?.OrderNumber != null)
        {
            var lastSequence = lastOrder.OrderNumber.Substring(prefix.Length);
            if (int.TryParse(lastSequence, out var parsed))
            {
                sequence = parsed + 1;
            }
        }

        return $"{prefix}{sequence:D4}";
    }
}
```

### OrderLineItemList (Child Collection)

```csharp
using Neatoo;

namespace OrderDomain;

[Factory]
internal partial class OrderLineItemList
    : EntityListBase<IOrderLineItem>, IOrderLineItemList
{
    public OrderLineItemList(
        IEntityListBaseServices<IOrderLineItem> services)
        : base(services) { }

    [Create]
    public void Create()
    {
        // Empty collection ready for items
    }

    [Fetch]
    public async Task Fetch(
        IEnumerable<OrderLineItemEntity> entities,
        [Service] IOrderLineItemFactory lineItemFactory)
    {
        foreach (var entity in entities)
        {
            var lineItem = await lineItemFactory.Fetch(entity);
            if (lineItem != null)
            {
                Add(lineItem);
            }
        }
    }

    [Remote]
    [Update]
    public async Task Update(
        Guid orderId,
        [Service] IOrderDbContext dbContext)
    {
        // Process deletions first
        foreach (var deleted in DeletedList.Cast<IOrderLineItem>())
        {
            var entity = await dbContext.OrderLineItems.FindAsync(deleted.Id);
            if (entity != null)
            {
                dbContext.OrderLineItems.Remove(entity);
            }
        }

        // Process remaining items
        foreach (var lineItem in this)
        {
            if (lineItem.IsNew)
            {
                // Insert new item
                lineItem.Id = Guid.NewGuid();
                var entity = new OrderLineItemEntity
                {
                    Id = lineItem.Id.Value,
                    OrderId = orderId,
                    ProductId = lineItem.ProductId,
                    ProductName = lineItem.ProductName,
                    Quantity = lineItem.Quantity,
                    UnitPrice = lineItem.UnitPrice,
                    LineTotal = lineItem.LineTotal
                };
                dbContext.OrderLineItems.Add(entity);
            }
            else if (lineItem.IsSelfModified)
            {
                // Update existing item
                var entity = await dbContext.OrderLineItems.FindAsync(lineItem.Id);
                if (entity != null)
                {
                    if (lineItem[nameof(lineItem.ProductId)].IsModified)
                        entity.ProductId = lineItem.ProductId;
                    if (lineItem[nameof(lineItem.ProductName)].IsModified)
                        entity.ProductName = lineItem.ProductName;
                    if (lineItem[nameof(lineItem.Quantity)].IsModified)
                        entity.Quantity = lineItem.Quantity;
                    if (lineItem[nameof(lineItem.UnitPrice)].IsModified)
                        entity.UnitPrice = lineItem.UnitPrice;
                    if (lineItem[nameof(lineItem.LineTotal)].IsModified)
                        entity.LineTotal = lineItem.LineTotal;
                }
            }
        }

        await dbContext.SaveChangesAsync();
    }
}
```

### OrderLineItem (Child Entity)

```csharp
using System.ComponentModel.DataAnnotations;
using Neatoo;

namespace OrderDomain;

[Factory]
internal partial class OrderLineItem
    : EntityBase<OrderLineItem>, IOrderLineItem
{
    public OrderLineItem(
        IEntityBaseServices<OrderLineItem> services,
        ILineTotalRule lineTotalRule,
        IUniqueProductRule uniqueProductRule) : base(services)
    {
        RuleManager.AddRule(lineTotalRule);
        RuleManager.AddRule(uniqueProductRule);

        // Inline validation for quantity
        RuleManager.AddValidation(
            nameof(Quantity),
            (OrderLineItem item) => item.Quantity > 0
                ? RuleMessage.None
                : RuleMessage.Error("Quantity must be greater than zero"));
    }

    // Identity
    public partial Guid? Id { get; set; }

    // Product information
    public partial Guid? ProductId { get; set; }

    [Required(ErrorMessage = "Product name is required")]
    [DisplayName("Product")]
    public partial string? ProductName { get; set; }

    [Range(1, 10000, ErrorMessage = "Quantity must be between 1 and 10,000")]
    [DisplayName("Qty")]
    public partial int Quantity { get; set; }

    [Range(0.01, 1000000, ErrorMessage = "Unit price must be positive")]
    [DisplayName("Unit Price")]
    public partial decimal UnitPrice { get; set; }

    // Calculated
    [DisplayName("Line Total")]
    public partial decimal LineTotal { get; set; }

    // Parent navigation
    public IOrder? ParentOrder => Parent as IOrder;

    [Create]
    public void Create()
    {
        Quantity = 1;
        UnitPrice = 0m;
    }

    [Fetch]
    public void Fetch(OrderLineItemEntity entity)
    {
        // Load properties without triggering rules
        LoadProperty(nameof(Id), entity.Id);
        LoadProperty(nameof(ProductId), entity.ProductId);
        LoadProperty(nameof(ProductName), entity.ProductName);
        LoadProperty(nameof(Quantity), entity.Quantity);
        LoadProperty(nameof(UnitPrice), entity.UnitPrice);
        LoadProperty(nameof(LineTotal), entity.LineTotal);
    }
}
```

## Business Rules

### OrderTotalRule

Calculates order totals based on line items and tax rate:

```csharp
using Neatoo;

namespace OrderDomain.Rules;

public interface IOrderTotalRule : IRule<IOrder> { }

public class OrderTotalRule : RuleBase<Order>, IOrderTotalRule
{
    public OrderTotalRule()
        : base(o => o.LineItems, o => o.TaxRate) { }

    protected override IRuleMessages Execute(Order target)
    {
        // Calculate subtotal from line items
        target.Subtotal = target.LineItems?
            .Where(li => !li.IsDeleted)
            .Sum(li => li.LineTotal) ?? 0m;

        // Calculate tax
        target.Tax = target.Subtotal * target.TaxRate;

        // Calculate total
        target.Total = target.Subtotal + target.Tax;

        return None;
    }
}
```

**Note:** This rule triggers when `LineItems` property changes. To also recalculate when individual line items change, override `ChildNeatooPropertyChanged` in the Order class:

```csharp
protected override Task ChildNeatooPropertyChanged(
    NeatooPropertyChangedEventArgs eventArgs)
{
    // Recalculate when any line item total changes
    if (eventArgs.PropertyName == nameof(IOrderLineItem.LineTotal))
    {
        // Trigger the rule by raising property changed on LineItems
        RaisePropertyChanged(nameof(LineItems));
    }

    return base.ChildNeatooPropertyChanged(eventArgs);
}
```

### LineTotalRule

Calculates individual line totals:

```csharp
using Neatoo;

namespace OrderDomain.Rules;

public interface ILineTotalRule : IRule<IOrderLineItem> { }

public class LineTotalRule : RuleBase<OrderLineItem>, ILineTotalRule
{
    public LineTotalRule()
        : base(li => li.Quantity, li => li.UnitPrice) { }

    protected override IRuleMessages Execute(OrderLineItem target)
    {
        target.LineTotal = target.Quantity * target.UnitPrice;
        return None;
    }
}
```

### UniqueProductRule

Ensures no duplicate products in the same order:

```csharp
using Neatoo;

namespace OrderDomain.Rules;

public interface IUniqueProductRule : IRule<IOrderLineItem> { }

public class UniqueProductRule : RuleBase<OrderLineItem>, IUniqueProductRule
{
    public UniqueProductRule()
        : base(li => li.ProductId) { }

    protected override IRuleMessages Execute(OrderLineItem target)
    {
        // Access parent order through the aggregate
        var order = target.ParentOrder;
        if (order?.LineItems == null)
            return None;

        // Check for duplicate products (excluding this item)
        var isDuplicate = order.LineItems
            .Where(li => li != target && !li.IsDeleted)
            .Any(li => li.ProductId == target.ProductId
                       && target.ProductId != null);

        return RuleMessages.If(
            isDuplicate,
            nameof(target.ProductId),
            "This product is already in the order");
    }
}
```

## EF Core Entity Mapping

### Database Entities

```csharp
namespace OrderDomain.Data;

public class OrderEntity
{
    public Guid Id { get; set; }
    public string? OrderNumber { get; set; }
    public string? CustomerName { get; set; }
    public DateTime OrderDate { get; set; }
    public string? ShipToAddress { get; set; }
    public decimal TaxRate { get; set; }
    public decimal Subtotal { get; set; }
    public decimal Tax { get; set; }
    public decimal Total { get; set; }

    // Navigation
    public ICollection<OrderLineItemEntity> LineItems { get; set; }
        = new List<OrderLineItemEntity>();
}

public class OrderLineItemEntity
{
    public Guid Id { get; set; }
    public Guid OrderId { get; set; }
    public Guid? ProductId { get; set; }
    public string? ProductName { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal LineTotal { get; set; }

    // Navigation
    public OrderEntity? Order { get; set; }
}
```

### DbContext

```csharp
using Microsoft.EntityFrameworkCore;

namespace OrderDomain.Data;

public interface IOrderDbContext
{
    DbSet<OrderEntity> Orders { get; }
    DbSet<OrderLineItemEntity> OrderLineItems { get; }
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}

public class OrderDbContext : DbContext, IOrderDbContext
{
    public OrderDbContext(DbContextOptions<OrderDbContext> options)
        : base(options) { }

    public DbSet<OrderEntity> Orders => Set<OrderEntity>();
    public DbSet<OrderLineItemEntity> OrderLineItems => Set<OrderLineItemEntity>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<OrderEntity>(entity =>
        {
            entity.HasKey(e => e.Id);

            entity.Property(e => e.OrderNumber)
                .HasMaxLength(50);

            entity.Property(e => e.CustomerName)
                .HasMaxLength(200);

            entity.Property(e => e.ShipToAddress)
                .HasMaxLength(500);

            entity.Property(e => e.Subtotal)
                .HasPrecision(18, 2);

            entity.Property(e => e.Tax)
                .HasPrecision(18, 2);

            entity.Property(e => e.Total)
                .HasPrecision(18, 2);

            entity.Property(e => e.TaxRate)
                .HasPrecision(5, 4);

            entity.HasMany(e => e.LineItems)
                .WithOne(li => li.Order)
                .HasForeignKey(li => li.OrderId)
                .OnDelete(DeleteBehavior.Cascade);
        });

        modelBuilder.Entity<OrderLineItemEntity>(entity =>
        {
            entity.HasKey(e => e.Id);

            entity.Property(e => e.ProductName)
                .HasMaxLength(200);

            entity.Property(e => e.UnitPrice)
                .HasPrecision(18, 2);

            entity.Property(e => e.LineTotal)
                .HasPrecision(18, 2);
        });
    }
}
```

## Dependency Injection Setup

### Server Registration

```csharp
// Program.cs (Server)
using Neatoo;
using OrderDomain;
using OrderDomain.Data;
using OrderDomain.Rules;

var builder = WebApplication.CreateBuilder(args);

// Database
builder.Services.AddDbContext<OrderDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
builder.Services.AddScoped<IOrderDbContext>(sp =>
    sp.GetRequiredService<OrderDbContext>());

// Neatoo services
builder.Services.AddNeatooServices(
    NeatooFactory.Server,
    typeof(Order).Assembly);

// Rules (convention-based registration)
builder.Services.AddScoped<IOrderTotalRule, OrderTotalRule>();
builder.Services.AddScoped<ILineTotalRule, LineTotalRule>();
builder.Services.AddScoped<IUniqueProductRule, UniqueProductRule>();

var app = builder.Build();

// Neatoo endpoint
app.MapNeatoo("/api/neatoo");

app.Run();
```

### Client Registration

```csharp
// Program.cs (Blazor WASM Client)
using Neatoo;
using OrderDomain;

var builder = WebAssemblyHostBuilder.CreateDefault(args);

// Neatoo services
builder.Services.AddNeatooServices(
    NeatooFactory.Remote,
    typeof(IOrder).Assembly);

// HTTP client for Neatoo
builder.Services.AddKeyedScoped("Neatoo", (sp, key) =>
{
    var client = new HttpClient
    {
        BaseAddress = new Uri(builder.HostEnvironment.BaseAddress)
    };
    return client;
});

await builder.Build().RunAsync();
```

## Blazor UI Component

A complete order editing page:

```razor
@page "/orders/edit/{Id:guid?}"
@using OrderDomain
@using Neatoo.Blazor.MudNeatoo
@inject IOrderFactory OrderFactory
@inject IOrderLineItemFactory LineItemFactory
@inject NavigationManager NavigationManager
@inject ISnackbar Snackbar

<PageTitle>@(_isNew ? "New Order" : $"Edit Order {_order?.OrderNumber}")</PageTitle>

@if (_loading)
{
    <MudProgressCircular Indeterminate="true" />
}
else if (_order == null)
{
    <MudAlert Severity="Severity.Error">Order not found</MudAlert>
}
else
{
    <MudPaper Class="pa-4">
        <MudText Typo="Typo.h4" Class="mb-4">
            @(_isNew ? "New Order" : $"Order {_order.OrderNumber}")
        </MudText>

        @* Order Header *@
        <MudGrid>
            <MudItem xs="12" md="6">
                <MudNeatooTextField For="() => _order.CustomerName"
                                    Label="Customer Name"
                                    Required="true" />
            </MudItem>
            <MudItem xs="12" md="6">
                <MudNeatooDatePicker For="() => _order.OrderDate"
                                     Label="Order Date" />
            </MudItem>
            <MudItem xs="12">
                <MudNeatooTextField For="() => _order.ShipToAddress"
                                    Label="Shipping Address"
                                    Required="true"
                                    Lines="2" />
            </MudItem>
            <MudItem xs="12" md="4">
                <MudNeatooNumericField For="() => _order.TaxRate"
                                       Label="Tax Rate"
                                       Format="P2"
                                       Step="0.01M" />
            </MudItem>
        </MudGrid>

        <MudDivider Class="my-4" />

        @* Line Items *@
        <MudText Typo="Typo.h6" Class="mb-2">Line Items</MudText>

        <MudTable Items="@_order.LineItems"
                  Dense="true"
                  Hover="true"
                  Bordered="true">
            <HeaderContent>
                <MudTh>Product</MudTh>
                <MudTh Style="width: 120px">Quantity</MudTh>
                <MudTh Style="width: 150px">Unit Price</MudTh>
                <MudTh Style="width: 150px">Line Total</MudTh>
                <MudTh Style="width: 80px"></MudTh>
            </HeaderContent>
            <RowTemplate>
                <MudTd>
                    <MudNeatooTextField For="() => context.ProductName"
                                        Margin="Margin.Dense"
                                        Variant="Variant.Text" />
                </MudTd>
                <MudTd>
                    <MudNeatooNumericField For="() => context.Quantity"
                                           Margin="Margin.Dense"
                                           Variant="Variant.Text"
                                           Min="1" />
                </MudTd>
                <MudTd>
                    <MudNeatooNumericField For="() => context.UnitPrice"
                                           Margin="Margin.Dense"
                                           Variant="Variant.Text"
                                           Format="C2"
                                           Step="0.01M" />
                </MudTd>
                <MudTd>
                    <MudText>@context.LineTotal.ToString("C2")</MudText>
                </MudTd>
                <MudTd>
                    <MudIconButton Icon="@Icons.Material.Filled.Delete"
                                   Color="Color.Error"
                                   Size="Size.Small"
                                   OnClick="() => RemoveLineItem(context)" />
                </MudTd>
            </RowTemplate>
            <FooterContent>
                <MudTd colspan="5">
                    <MudButton StartIcon="@Icons.Material.Filled.Add"
                               Color="Color.Primary"
                               Variant="Variant.Text"
                               OnClick="AddLineItem">
                        Add Line Item
                    </MudButton>
                </MudTd>
            </FooterContent>
        </MudTable>

        @* Totals *@
        <MudGrid Class="mt-4">
            <MudItem xs="12" md="6" />
            <MudItem xs="12" md="6">
                <MudPaper Class="pa-4" Elevation="0" Outlined="true">
                    <MudStack>
                        <MudStack Row="true" Justify="Justify.SpaceBetween">
                            <MudText>Subtotal:</MudText>
                            <MudText>@_order.Subtotal.ToString("C2")</MudText>
                        </MudStack>
                        <MudStack Row="true" Justify="Justify.SpaceBetween">
                            <MudText>Tax (@_order.TaxRate.ToString("P2")):</MudText>
                            <MudText>@_order.Tax.ToString("C2")</MudText>
                        </MudStack>
                        <MudDivider />
                        <MudStack Row="true" Justify="Justify.SpaceBetween">
                            <MudText Typo="Typo.h6">Total:</MudText>
                            <MudText Typo="Typo.h6">@_order.Total.ToString("C2")</MudText>
                        </MudStack>
                    </MudStack>
                </MudPaper>
            </MudItem>
        </MudGrid>

        @* Validation Summary *@
        @if (!_order.IsValid)
        {
            <MudAlert Severity="Severity.Error" Class="mt-4">
                <MudText Typo="Typo.body2"><strong>Please correct the following errors:</strong></MudText>
                <ul class="mb-0">
                    @foreach (var msg in _order.PropertyMessages)
                    {
                        <li>@msg.PropertyName: @msg.Message</li>
                    }
                    @foreach (var lineItem in _order.LineItems)
                    {
                        @foreach (var msg in lineItem.PropertyMessages)
                        {
                            <li>Line Item - @msg.PropertyName: @msg.Message</li>
                        }
                    }
                </ul>
            </MudAlert>
        }

        <MudDivider Class="my-4" />

        @* Actions *@
        <MudStack Row="true" Justify="Justify.FlexEnd" Spacing="2">
            <MudButton Variant="Variant.Text"
                       OnClick="Cancel">
                Cancel
            </MudButton>

            @if (!_isNew && OrderFactory.CanDelete().IsAuthorized)
            {
                <MudButton Variant="Variant.Outlined"
                           Color="Color.Error"
                           OnClick="DeleteOrder">
                    Delete
                </MudButton>
            }

            <MudButton Variant="Variant.Filled"
                       Color="Color.Primary"
                       Disabled="@(!CanSave)"
                       OnClick="SaveOrder">
                @if (_saving)
                {
                    <MudProgressCircular Size="Size.Small"
                                         Indeterminate="true"
                                         Class="mr-2" />
                    <span>Saving...</span>
                }
                else if (_order.IsBusy)
                {
                    <span>Validating...</span>
                }
                else
                {
                    <span>Save</span>
                }
            </MudButton>
        </MudStack>
    </MudPaper>
}

@code {
    [Parameter] public Guid? Id { get; set; }

    private IOrder? _order;
    private bool _loading = true;
    private bool _saving;
    private bool _isNew;

    private bool CanSave => !_saving && _order?.IsSavable == true;

    protected override async Task OnInitializedAsync()
    {
        if (Id.HasValue)
        {
            _order = await OrderFactory.Fetch(Id.Value);
            _isNew = false;
        }
        else
        {
            _order = OrderFactory.Create();
            _isNew = true;
        }

        _loading = false;
    }

    private void AddLineItem()
    {
        var lineItem = LineItemFactory.Create();
        _order!.LineItems!.Add(lineItem);
    }

    private void RemoveLineItem(IOrderLineItem lineItem)
    {
        _order!.LineItems!.Remove(lineItem);
    }

    private async Task SaveOrder()
    {
        // Wait for any pending async validations
        await _order!.WaitForTasks();

        if (!_order.IsSavable)
        {
            StateHasChanged();
            return;
        }

        try
        {
            _saving = true;
            StateHasChanged();

            await OrderFactory.Save(_order);

            Snackbar.Add("Order saved successfully", Severity.Success);
            NavigationManager.NavigateTo("/orders");
        }
        catch (Exception ex)
        {
            Snackbar.Add($"Save failed: {ex.Message}", Severity.Error);
        }
        finally
        {
            _saving = false;
        }
    }

    private async Task DeleteOrder()
    {
        var confirmed = await DialogService.ShowMessageBox(
            "Confirm Delete",
            "Are you sure you want to delete this order?",
            yesText: "Delete",
            cancelText: "Cancel");

        if (confirmed == true)
        {
            _order!.Delete();
            await OrderFactory.Save(_order);
            Snackbar.Add("Order deleted", Severity.Info);
            NavigationManager.NavigateTo("/orders");
        }
    }

    private void Cancel()
    {
        NavigationManager.NavigateTo("/orders");
    }
}
```

## Key Patterns Demonstrated

### Aggregate Boundaries

All persistence operations go through the Order aggregate root. Line items cannot be saved independently:

```csharp
// Correct: Save through aggregate root
await OrderFactory.Save(order);

// Wrong: Line items are children, cannot save directly
// await LineItemFactory.Save(lineItem);  // Would throw
```

### Cascading Calculations

When line item quantities or prices change:

1. `LineTotalRule` recalculates `lineItem.LineTotal`
2. `ChildNeatooPropertyChanged` detects the change
3. `OrderTotalRule` recalculates `Subtotal`, `Tax`, and `Total`
4. UI automatically updates via `INotifyPropertyChanged`

### Validation Flow

Validation rules execute automatically:

```csharp
lineItem.Quantity = 0;  // Triggers validation rule
// lineItem.IsValid == false
// order.IsValid == false (child is invalid)
// order.IsSavable == false
```

### Collection Operations

Adding and removing line items:

```csharp
// Adding - automatic child marking
var newLine = LineItemFactory.Create();
order.LineItems.Add(newLine);
// newLine.IsChild == true
// newLine.Parent == order

// Removing new item - simply removed
order.LineItems.Remove(newLine);

// Removing existing item - moved to DeletedList
order.LineItems.Remove(existingLine);
// existingLine is now in order.LineItems.DeletedList
// Will be deleted on Save()
```

### EF Core Integration

The factory methods handle all database operations:

- `[Fetch]` loads the aggregate with related data
- `[Insert]` creates new records
- `[Update]` updates only modified properties for efficient persistence
- `[Delete]` handles cascade delete
- `[Update]` on the list processes `DeletedList` for removed items

## Testing the Aggregate

```csharp
[Fact]
public void Order_WhenLineItemAdded_CalculatesTotals()
{
    // Arrange
    var order = _orderFactory.Create();

    // Act
    var lineItem = _lineItemFactory.Create();
    lineItem.Quantity = 2;
    lineItem.UnitPrice = 10.00m;
    order.LineItems!.Add(lineItem);

    // Assert
    Assert.Equal(20.00m, lineItem.LineTotal);
    Assert.Equal(20.00m, order.Subtotal);
    Assert.Equal(1.60m, order.Tax);  // 8% default
    Assert.Equal(21.60m, order.Total);
}

[Fact]
public void Order_WhenNoCustomerName_IsNotValid()
{
    // Arrange
    var order = _orderFactory.Create();
    order.CustomerName = null;

    // Act & Assert
    Assert.False(order.IsValid);
    Assert.Contains(order.PropertyMessages,
        m => m.PropertyName == nameof(IOrder.CustomerName));
}

[Fact]
public void OrderLineItem_WhenDuplicateProduct_ShowsError()
{
    // Arrange
    var order = _orderFactory.Create();

    var line1 = _lineItemFactory.Create();
    line1.ProductId = Guid.NewGuid();
    order.LineItems!.Add(line1);

    var line2 = _lineItemFactory.Create();
    line2.ProductId = line1.ProductId;  // Same product

    // Act
    order.LineItems.Add(line2);

    // Assert
    Assert.False(line2.IsValid);
    Assert.Contains(line2.PropertyMessages,
        m => m.Message.Contains("already in the order"));
}
```

## Related Topics

- [Aggregates and Entity Graphs](/concepts/aggregates/) - Understanding aggregates
- [EntityBase Reference](/reference/entity-base/) - Entity API details
- [EntityListBase Reference](/reference/entity-list-base/) - Collection management
- [Factory Operations Reference](/reference/factory-operations/) - Factory patterns
- [Rules Engine Reference](/reference/rules/) - Business rules
- [Blazor Integration](/guides/blazor-integration/) - UI patterns
- [Data Mapping Reference](/reference/data-mapping/) - Property mapping patterns
