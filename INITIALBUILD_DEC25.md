# Neatoo Documentation Build Plan - December 2025

This document tracks the comprehensive documentation build for the Neatoo framework.

## Progress Overview

| Phase | Status | Pages Complete |
|-------|--------|----------------|
| Phase 1: Foundation | **Complete** | 4/4 |
| Phase 2: Core Concepts | **Complete** | 5/5 |
| Phase 3: Factory System | **Complete** | 4/4 |
| Phase 4: Supporting Features | **Complete** | 4/4 |
| Phase 5: Integration & Examples | **Complete** | 4/4 |
| Phase 6: Advanced | **Complete** | 2/2 |
| **Total** | **Complete** | **23/23** |

---

## Current Documentation State

### Existing Pages
- `index.md` - Feature list, pros/cons, lifecycle diagram
- `_pages/overview/entity.md` - Brief entity overview
- `_pages/overview/rule.md` - Brief rule overview
- `_pages/overview/factory.md` - Brief factory overview
- `_pages/overview/authorization.md` - Brief authorization overview
- `_pages/setup/client.md` - Minimal client setup
- `_pages/setup/server.md` - Minimal server setup
- `_pages/gettingstarted/createsimpleapplication.md` - Video-based tutorial
- `_pages/example/person.md` - Person example with code links

### Identified Gaps
- No DDD conceptual introduction for newcomers
- No detailed API reference for EntityBase, ValidateBase, Base classes
- No EntityListBase documentation
- No property system deep-dive (meta-properties, PropertyManager)
- No data mapping comprehensive guide (MapFrom, MapTo, MapModifiedTo)
- No Blazor UI integration guide
- No troubleshooting or common pitfalls section
- No migration/comparison guides

---

## Phase 1: Foundation (Critical Path)

### 1.1 Introduction to Neatoo
- **File**: `_pages/introduction.md`
- **Permalink**: `/introduction/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - What is Neatoo? (framework purpose and positioning)
  - The problem with traditional n-tier architectures (anemic domain models, duplicated logic)
  - How Neatoo enables true DDD without the complexity
  - The "define once, use everywhere" philosophy
  - Overview of core components (Entities, Rules, Factories)
  - When to use Neatoo vs. alternatives
- **Key Points**:
  - Eliminates need for separate DTOs and mapping layers
  - Business logic lives in one place, executes on both client and server
  - Roslyn Source Generators create compile-time safety with no runtime reflection

### 1.2 Client Setup (Expand Existing)
- **File**: `_pages/setup/client.md`
- **Permalink**: `/setup/client/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - NuGet packages required (`Neatoo`, `Neatoo.RemoteFactory`, `Neatoo.Blazor.MudNeatoo`)
  - `AddNeatooServices()` registration with `NeatooFactory.Remote`
  - HttpClient configuration with keyed registration
  - Blazor WebAssembly vs Server considerations
  - Shared library project patterns
  - Authentication token forwarding
- **Code Examples**: Complete Program.cs for Blazor WASM client

### 1.3 Server Setup (Expand Existing)
- **File**: `_pages/setup/server.md`
- **Permalink**: `/setup/server/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - NuGet packages required
  - `AddNeatooServices()` with `NeatooFactory.Server`
  - The Neatoo endpoint (`/api/neatoo` route)
  - EF Core DbContext registration
  - Authentication/authorization middleware
  - CORS configuration for Blazor WASM
- **Code Examples**: Complete Program.cs for ASP.NET Core

### 1.4 EntityBase Complete Reference
- **File**: `_pages/reference/entity-base.md`
- **Permalink**: `/reference/entity-base/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - Class inheritance: `Base<T>` -> `ValidateBase<T>` -> `EntityBase<T>`
  - Required constructor pattern (`IEntityBaseServices<T>`)
  - Defining partial properties
  - All meta-properties: `IsNew`, `IsModified`, `IsSelfModified`, `IsDeleted`, `IsSavable`, `IsChild`, `IsValid`, `IsSelfValid`, `IsBusy`, `IsSelfBusy`
  - `ModifiedProperties` collection
  - `PropertyMessages` and validation state
  - The `Parent` property and automatic assignment
  - The `RuleManager` and adding rules
  - Methods: `Save()`, `Delete()`, `UnDelete()`, `MarkModified()`, `MarkAsChild()`, `RunRules()`, `WaitForTasks()`
  - Events: `PropertyChanged`, `NeatooPropertyChanged`
  - The `PropertyManager` and accessing property objects
  - `PauseAllActions()` for bulk updates
- **Dependencies**: Properties Concept, Aggregates Concept

---

## Phase 2: Core Concepts

### 2.1 DDD Concepts in Neatoo
- **File**: `_pages/concepts/ddd-overview.md`
- **Permalink**: `/concepts/ddd-overview/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - What is Domain-Driven Design? (brief introduction)
  - The Ubiquitous Language principle
  - Tactical patterns: Aggregates, Aggregate Roots, Entities vs. Value Objects, Business Rules
  - How Neatoo maps to these patterns:
    - `EntityBase<T>` = Entity with identity
    - `ValidateBase<T>` = Entity without persistence tracking
    - `Base<T>` = Value Objects
    - `RuleBase<T>` / `AsyncRuleBase<T>` = Business Rules
    - Factory = Aggregate creation and lifecycle
- **Analogies**:
  - Aggregate: "Order form with LineItems - submit the whole form"
  - Entity vs Value Object: "Driver's license (identity) vs $20 bill (interchangeable)"
- **Dependencies**: Introduction (1.1)

### 2.2 Aggregates and Entity Graphs
- **File**: `_pages/concepts/aggregates.md`
- **Permalink**: `/concepts/aggregates/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - What is an Aggregate? (DDD definition with analogy)
  - The Aggregate Root pattern
  - Parent-child relationships: automatic `Parent` assignment, `IsChild`, PropertyChanged propagation
  - The class hierarchy
  - Why "you cannot nest an Entity under a Value Object"
  - How modification tracking cascades through the graph
- **Includes**: Mermaid diagram of Order aggregate
- **Dependencies**: DDD Concepts (2.1)

### 2.3 EntityListBase Complete Reference
- **File**: `_pages/reference/entity-list-base.md`
- **Permalink**: `/reference/entity-list-base/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - Class hierarchy: `ListBase<I>` -> `ValidateListBase<I>` -> `EntityListBase<I>`
  - Defining collection interfaces
  - Adding items: automatic `MarkAsChild()`, `SetParent()`
  - Removing items: new items removed, existing items to `DeletedList`
  - The `DeletedList` property
  - Collection-level meta-properties
  - Factory operations for collections
  - Handling inserts, updates, deletes in `[Update]`
  - Cross-item validation
- **Dependencies**: EntityBase Reference (1.4), Aggregates Concept (2.2)

### 2.4 Rules Philosophy
- **File**: `_pages/concepts/rules-philosophy.md`
- **Permalink**: `/concepts/rules-philosophy/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - The problem with anemic domain models
  - Business rules as first-class citizens
  - Trigger-based execution
  - Cascading rules behavior
  - Sync vs async rules
  - Validation messages vs transformation rules
  - Integration with UI binding
- **Analogy**: "Spreadsheet formulas that auto-recalculate"
- **Dependencies**: DDD Concepts (2.1)

### 2.5 Rules Engine Complete Reference
- **File**: `_pages/reference/rules.md`
- **Permalink**: `/reference/rules/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - Rule base classes: `RuleBase<T>`, `AsyncRuleBase<T>`
  - Defining trigger properties
  - The `Execute()` method pattern
  - Returning rule messages: `None`, single, multiple, fluent syntax
  - Cascading rules
  - Injecting dependencies
  - Adding rules via `RuleManager.AddRule()`
  - Fluent inline rules: `AddValidation()`, `AddValidationAsync()`, `AddAction()`, `AddActionAsync()`
  - Data annotation attributes
  - Running rules manually
  - The `IsBusy` state
  - Unit testing rules
- **Dependencies**: Rules Philosophy (2.4)

---

## Phase 3: Factory System

### 3.1 Factory Pattern Concept
- **File**: `_pages/concepts/factories-overview.md`
- **Permalink**: `/concepts/factories-overview/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - The Factory pattern in DDD
  - Why entities should not be instantiated with `new`
  - How Neatoo generates factories from entity definitions
  - Lifecycle operations: Create, Fetch, Save (Insert/Update/Delete)
  - Method injection with `[Service]` attribute
  - The `[Remote]` attribute and 3-tier architecture
  - Viewing and debugging generated code
- **Includes**: Sequence diagram of factory lifecycle
- **Dependencies**: DDD Concepts (2.1), Aggregates (2.2)

### 3.2 Factory Operations Reference
- **File**: `_pages/reference/factory-operations.md`
- **Permalink**: `/reference/factory-operations/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - Factory attributes: `[Factory]`, `[Create]`, `[Fetch]`, `[Insert]`, `[Update]`, `[Delete]`, `[Remote]`
  - Method parameters: regular vs `[Service]`
  - Return values from factory methods
  - Generated factory interface: `Create()`, `Fetch()`, `Save()`, `TrySave()`, `CanXYZ()`
  - How `Save()` routes to Insert/Update/Delete
  - Handling child entities
  - Viewing generated factory code
- **Dependencies**: Factory Concept (3.1)

### 3.3 Data Mapping Reference
- **File**: `_pages/reference/data-mapping.md`
- **Permalink**: `/reference/data-mapping/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - Declaring partial mapper methods
  - `MapFrom(entity)`: Fetch operations, no rules triggered, uses `LoadProperty`
  - `MapTo(entity)`: Insert operations, copies ALL properties
  - `MapModifiedTo(entity)`: Update operations, only modified properties
  - Property matching rules
  - The `[MapperIgnore]` attribute
  - Mapping nested objects and collections
  - EF Core integration patterns
- **Dependencies**: Factory Operations (3.2)

### 3.4 Client-Server Architecture
- **File**: `_pages/concepts/client-server.md`
- **Permalink**: `/concepts/client-server/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - Traditional 3-tier challenges
  - Neatoo's approach: shared domain model
  - How `[Remote]` operations work
  - The single `/api/neatoo` endpoint
  - What gets serialized
  - Rules execute on both client and server
- **Includes**: Reference to existing lifecycle diagram
- **Dependencies**: Factory Concept (3.1)

---

## Phase 4: Supporting Features

### 4.1 Properties and Meta-Properties
- **File**: `_pages/concepts/properties.md`
- **Permalink**: `/concepts/properties/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - Why partial properties?
  - What the source generator creates
  - Entity-level meta-properties (all with explanations)
  - Property-level meta-properties
  - How meta-properties propagate through aggregates
  - `SetValue` vs `LoadValue`
- **Includes**: Visual diagram of meta-property relationships
- **Dependencies**: Aggregates (2.2)

### 4.2 Authorization Concept
- **File**: `_pages/concepts/authorization.md`
- **Permalink**: `/concepts/authorization/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - The authorization philosophy (check before execute)
  - Defining authorization classes
  - `[Authorize<T>]` and `[Authorize(AuthorizeOperation.XYZ)]` attributes
  - The generated `CanXYZ()` methods
  - UI authorization patterns
- **Dependencies**: Factory Concept (3.1)

### 4.3 Authorization System Reference
- **File**: `_pages/reference/authorization.md`
- **Permalink**: `/reference/authorization/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - `AuthorizeOperation` enum values
  - `FactoryOperation` enum and bitwise mapping
  - Authorization method signatures
  - The `Authorized` return type
  - Resolving from DI
  - ASP.NET Core integration
  - Testing authorization
- **Dependencies**: Authorization Concept (4.2)

### 4.4 Dependency Injection Patterns
- **File**: `_pages/reference/dependency-injection.md`
- **Permalink**: `/reference/dependency-injection/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - Constructor injection for entities
  - Method injection with `[Service]`
  - DI registration: `AddNeatooServices()`, `NeatooFactory.Server` vs `NeatooFactory.Remote`
  - Custom serializer DI resolution
  - Extending framework services
  - Testing with mock services
- **Dependencies**: Client-Server (3.4)

---

## Phase 5: Integration & Examples

### 5.1 Blazor UI Integration
- **File**: `_pages/guides/blazor-integration.md`
- **Permalink**: `/guides/blazor-integration/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - Entities implement `INotifyPropertyChanged`
  - MudNeatoo component library
  - Manual binding patterns
  - Validation display patterns
  - Busy state handling
  - Save button pattern
  - Collection binding
  - Authorization-aware UI
- **Dependencies**: EntityBase Reference (1.4)

### 5.2 Complete Example: Order Aggregate
- **File**: `_pages/examples/order-aggregate.md`
- **Permalink**: `/examples/order-aggregate/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - Full walkthrough: Order with LineItems
  - All entity classes, rules, factory methods
  - EF Core entities and DbContext
  - Blazor edit page
  - Unit tests
- **Dependencies**: All reference pages

### 5.3 ValidateBase Reference
- **File**: `_pages/reference/validate-base.md`
- **Permalink**: `/reference/validate-base/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - When to use vs EntityBase
  - Use cases: wizard steps, validation-only objects
  - Available meta-properties
  - Integration with UI binding
- **Dependencies**: EntityBase Reference (1.4)

### 5.4 Base<T> and Value Objects Reference
- **File**: `_pages/reference/base-value-objects.md`
- **Permalink**: `/reference/base-value-objects/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - DDD Value Object pattern
  - Using `Base<T>`
  - The `[Factory]` attribute on non-EntityBase classes
  - Immutability patterns
  - Why value objects cannot contain entities
- **Dependencies**: DDD Concepts (2.1)

---

## Phase 6: Advanced

### 6.1 Migration Guide: From Anemic Models
- **File**: `_pages/guides/migration-anemic.md`
- **Permalink**: `/guides/migration-anemic/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - Identifying anemic domain model symptoms
  - Mapping existing patterns to Neatoo
  - Step-by-step migration process
  - Maintaining backward compatibility
  - Before/after code comparisons
  - Team adoption guidance
- **Dependencies**: Introduction (1.1)

### 6.2 Troubleshooting and Common Pitfalls
- **File**: `_pages/guides/troubleshooting.md`
- **Permalink**: `/guides/troubleshooting/`
- **Status**: Complete (2025-12-26)
- **Content**:
  - "My entity won't save" checklist (IsSavable diagnosis)
  - "Validation not running" issues
  - Property tracking issues
  - Serialization errors
  - Authorization failures
  - Parent-child relationship issues
  - DeletedList not processed
  - Rule execution order and async issues
  - Debugging generated code
  - Performance considerations
- **Format**: Problem/Solution pairs
- **Dependencies**: All reference pages

---

## Proposed Navigation Structure

Update `_data/navigation.yml`:

```yaml
central:
  - title: "Getting Started"
    children:
      - title: "Introduction"
        url: /introduction/
      - title: "Quick Start"
        url: /gettingstarted/createsimpleapplication/
      - title: "Client Setup"
        url: /setup/client/
      - title: "Server Setup"
        url: /setup/server/

  - title: "Concepts"
    children:
      - title: "DDD Overview"
        url: /concepts/ddd-overview/
      - title: "Aggregates & Entity Graphs"
        url: /concepts/aggregates/
      - title: "Rules Philosophy"
        url: /concepts/rules-philosophy/
      - title: "Factory Pattern"
        url: /concepts/factories-overview/
      - title: "Client-Server Architecture"
        url: /concepts/client-server/
      - title: "Properties & Meta-Properties"
        url: /concepts/properties/
      - title: "Authorization Model"
        url: /concepts/authorization/

  - title: "Reference"
    children:
      - title: "EntityBase<T>"
        url: /reference/entity-base/
      - title: "ValidateBase<T>"
        url: /reference/validate-base/
      - title: "Base<T> & Value Objects"
        url: /reference/base-value-objects/
      - title: "EntityListBase<I>"
        url: /reference/entity-list-base/
      - title: "Rules Engine"
        url: /reference/rules/
      - title: "Factory Operations"
        url: /reference/factory-operations/
      - title: "Data Mapping"
        url: /reference/data-mapping/
      - title: "Authorization System"
        url: /reference/authorization/
      - title: "Dependency Injection"
        url: /reference/dependency-injection/

  - title: "Guides"
    children:
      - title: "Blazor Integration"
        url: /guides/blazor-integration/
      - title: "Migration from Anemic Models"
        url: /guides/migration-anemic/
      - title: "Troubleshooting"
        url: /guides/troubleshooting/

  - title: "Examples"
    children:
      - title: "Person"
        url: /example/person/
      - title: "Order Aggregate"
        url: /examples/order-aggregate/
```

---

## References

- **Main Neatoo Repository**: https://github.com/NeatooDotNet/Neatoo
- **Existing Technical Docs**: https://github.com/NeatooDotNet/Neatoo/tree/main/docs
- **Person Example**: https://github.com/NeatooDotNet/Neatoo/tree/main/src/Examples/Person

---

## Change Log

| Date | Change |
|------|--------|
| 2025-12-26 | Initial plan created |
| 2025-12-26 | Phase 1 complete: Introduction, Client Setup, Server Setup, EntityBase Reference |
| 2025-12-26 | Phase 2 complete: DDD Concepts, Aggregates, EntityListBase, Rules Philosophy, Rules Engine |
| 2025-12-26 | Phase 3 complete: Factory Pattern, Factory Operations, Data Mapping, Client-Server Architecture |
| 2025-12-26 | Phase 4 complete: Properties and Meta-Properties, Authorization Concept, Authorization System Reference, Dependency Injection Patterns |
| 2025-12-26 | Phase 5 complete: Blazor UI Integration, Order Aggregate Example, ValidateBase Reference, Base<T> and Value Objects Reference |
| 2025-12-26 | Phase 6 complete: Migration Guide from Anemic Models, Troubleshooting and Common Pitfalls. Documentation build complete! |
