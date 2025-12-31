# Documentation Sync TODO

**Date:** 2025-12-30
**Last Synced Commits:**
- Neatoo: `a16fb5b` (Dec 29, 2025)
- RemoteFactory: `cb0db17` (Dec 29, 2025)

**New Commits to Review:**
- Neatoo: 6 commits up to `4c83d45` (Dec 31, 2025) - version 9.21.0
- RemoteFactory: 3 commits up to `9e62dda` (Dec 30, 2025)

---

## Documentation Changes Required

### 1. Mapper Methods Breaking Change (HIGH PRIORITY)

**Source:** `docs/mapper-methods.md` in Neatoo repo, commit `013a51e`

**Change:**
- `MapFrom` and `MapTo` are now **manually implemented** (no longer source-generated)
- Only `MapModifiedTo` remains **source-generated** by Neatoo

**Site pages to update:**
- `_pages/reference/data-mapping.md`
- Any examples showing MapFrom/MapTo as generated

---

### 2. Save() Reassignment Pattern (HIGH PRIORITY)

**Source:** `docs/todos/save-reassignment-pattern.md` in Neatoo repo (marked completed 2025-12-29)

**Key Point:** `Save()` returns a new deserialized instance - must be captured:
```csharp
person = await person.Save();  // CORRECT
await person.Save();            // WRONG - stale object
```

**Consequences of forgetting:**
- Database-generated IDs remain empty
- UI shows old values
- IsModified still true
- Navigation to /{id} routes fail

**Site pages to check/update:**
- `_pages/reference/factory-operations.md`
- `_pages/guides/blazor-integration.md`
- `_pages/guides/troubleshooting.md`

---

### 3. AsyncTasks Design / WaitForTasks Pattern (MEDIUM PRIORITY)

**Source:** `docs/todos/async-tasks-design-rationale.md` in Neatoo repo

**Key Points:**
- C# property setters cannot be async (language limitation)
- Neatoo uses "fire-and-forget with rendezvous" pattern
- Must call `await entity.WaitForTasks()` before checking validity
- `IsBusy` property indicates async operations in progress

**Pattern:**
```csharp
person.FirstName = "John";   // Triggers async rules
person.Email = "a@b.com";    // More async rules
await person.WaitForTasks(); // Wait for all to complete
Assert.IsTrue(person.IsValid);
```

**Site pages to check/update:**
- `_pages/concepts/rules-overview.md`
- `_pages/reference/rules-engine.md`
- `_pages/guides/database-dependent-validation.md`

---

### 4. ListBase Parent Behavior (MEDIUM PRIORITY)

**Source:** `docs/todos/listbase-parent-behavior.md` in Neatoo repo

**Key Point:** When items are added to `ListBase<T>`, their `Parent` is set to the **list's parent** (grandparent), not the list itself.

```
Person (aggregate root)
└── PersonPhoneList (child list)
    └── PersonPhone → Parent = Person (NOT PersonPhoneList)
```

**Correct access pattern:**
```csharp
public IPerson? ParentPerson => this.Parent as IPerson;  // Correct
```

**Common mistake:**
```csharp
public IPerson? ParentPerson => this.Parent?.Parent as IPerson;  // WRONG
```

**Site pages to check/update:**
- `_pages/reference/entity-list-base.md`
- `_pages/concepts/aggregates.md`

---

## Verification Results (2025-12-30)

### 1. Mapper Methods Breaking Change
**Status: ⚠️ NEEDS UPDATE**

| File | Current State | Issue |
|------|---------------|-------|
| `data-mapping.md` | ✅ Uses explicit `LoadProperty()` and property assignments | OK - no MapFrom/MapTo references |
| `factory-operations.md` | ❌ Shows MapFrom/MapTo as `partial` (source-generated) | **OUTDATED** - Lines 690-692 show all three as partial |

**Required Changes:**
- Update `factory-operations.md` examples to show:
  - `MapFrom` as manually implemented (not partial)
  - `MapTo` as manually implemented (not partial)
  - `MapModifiedTo` remains source-generated (partial)
- Update any documentation stating all three are source-generated

---

### 2. Save() Reassignment Pattern
**Status: ✅ ALREADY FULLY COVERED**

| File | Coverage |
|------|----------|
| `factory-operations.md` | Lines 442-478: "Critical: Always Reassign After Save()" ✅ |
| `troubleshooting.md` | Lines 172-222: "Stale Data After Save" with mailing analogy ✅ |
| `blazor-integration.md` | Lines 566-636: Blazor-specific guidance ✅ |

**No action needed** - Documentation is current and comprehensive.

---

### 3. AsyncTasks / WaitForTasks Pattern
**Status: ✅ BASIC COVERAGE EXISTS - OPTIONAL ENHANCEMENT**

| File | Coverage |
|------|----------|
| `troubleshooting.md` | Lines 124-142: "IsBusy is True and Blocking Save" ✅ |
| `blazor-integration.md` | Lines 447-463: "Preventing Premature Actions" ✅ |
| `blazor-integration.md` | Lines 551-564: "Why WaitForTasks() Before Save" ✅ |

**What's missing (optional):**
- Deep explanation of C# property setters can't be async
- "Fire-and-forget with rendezvous" pattern explanation
- Technical rationale from `async-tasks-design-rationale.md`

**Recommendation:** Consider adding an advanced topic page for developers who want to understand the internals. Not critical for basic usage.

---

### 4. ListBase Parent Behavior
**Status: ✅ ALREADY FULLY COVERED**

| File | Coverage |
|------|----------|
| `entity-list-base.md` | Lines 118-152: "Parent Assignment" section explains items get list's parent ✅ |
| `aggregates.md` | Lines 124-136: Same concept explained ✅ |

**No action needed** - Documentation correctly explains that list items' Parent points to the aggregate root, not the list.

---

## Action Items

### Required (Breaking Change) - ✅ COMPLETED
- [x] Update `_pages/reference/factory-operations.md`:
  - [x] Changed `public partial void MapFrom(...)` to implemented method with LoadProperty calls
  - [x] Changed `public partial void MapTo(...)` to implemented method with property assignments
  - [x] Kept `public partial void MapModifiedTo(...)` as partial (still source-generated)
  - [x] Added explanatory comments about which methods are generated

### Optional (Enhancement)
- [ ] Consider adding async validation deep-dive documentation explaining:
  - C# property setter async limitation
  - Fire-and-forget with rendezvous pattern
  - AsyncTasks implementation rationale

---

## Completed Updates (2025-12-30)

CLAUDE.md commit tracking tables updated:
| Repository | Last Synced Commit | Date |
|------------|-------------------|------|
| Neatoo | `4c83d45` | 2025-12-30 |
| RemoteFactory | `9e62dda` | 2025-12-30 |
