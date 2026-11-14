# Stack Companion: Cai / .NET / HotChocolate GraphQL

> **Purpose:** Gotchas, patterns, and anti-patterns specific to the Cai framework on top of HotChocolate + EF Core. Apply this alongside Pass 1 of the Engineering Review Playbook. The core five passes apply unchanged — this file only adds what is easy to miss on this specific stack.

**Applies to:** Projects using the Cai framework with HotChocolate GraphQL and EF Core on .NET.

---

## Mutations

### `CallResult` vs `CallResult<T>`

- `CallResult` — mutation succeeds or fails, no return data.
- `CallResult<T>` — mutation returns a typed result on success.
- The record attribute must match: `[Mutation<CallResult>]` or `[Mutation<CallResult<T>>]`.
- Using `CallResult` when the caller expects a payload silently returns null.

### `CallResult.NotPermitted()` vs `CallResult.Error(string)`

- `NotPermitted()` is for auth failures (403 semantics). Never substitute `CallResult.Error("Not authorized")`.
- `Error(string)` is for business-rule violations, not-found, or bad input (400/404 semantics).
- Blurring the two makes it harder to distinguish auth bugs from logic bugs in logs.

### Handler method signature

```csharp
public static async Task<CallResult> Handle(
    InputRecord input, DataContext db, AuthContext authContext, CancellationToken cancellationToken)
```

Cai resolves parameters by type. Any required service can be injected. The handler class must be a `partial class` with the matching attribute.

---

## Queries

### HotChocolate attribute order

Attributes on `IQueryable` query methods must appear in this exact order:

```csharp
[UseOffsetPaging(IncludeTotalCount = true, DefaultPageSize = 20)]
[UseProjection]
[UseFiltering]
[UseSorting]
```

Wrong order silently breaks projection or filtering — HotChocolate applies middleware in declaration order and each layer wraps the one below it.

### `[UseProjection]` vs explicit `.Include()`

- On query methods returning `IQueryable<T>` with `[UseProjection]`: HotChocolate translates the client's field selection to a SQL projection. `.Include()` calls in these methods are redundant and ignored during projection.
- In handler methods (mutations or non-projection queries): navigation properties are NOT auto-projected. Use explicit `.Include()` / `.ThenInclude()` to load what you need.
- Mixing both causes confusion: if `[UseProjection]` is active, adding an `.Include()` does nothing and signals a misunderstanding of how the query executes.

### `IQueryable` method bodies are not async

Query methods that return `IQueryable<T>` are not `async`. EF defers execution until materialization — synchronous calls inside these methods are fine. The check to do: synchronous DB calls inside **handler** (mutation) methods, where deferred execution doesn't apply.

### `AsNoTracking()` for reads

- Read-only queries should use `.AsNoTracking()` to skip EF change tracking.
- Handler mutations that load an entity and then update it must NOT use `.AsNoTracking()` — EF needs to track the entity to generate the UPDATE.

---

## EF Core patterns

### Always use async in handlers

In mutation handlers, always await the async variants:

- `FirstOrDefaultAsync` not `FirstOrDefault`
- `AnyAsync` not `Any` (when called outside an IQueryable expression)
- `SaveChangesAsync` not `SaveChanges`
- `ToListAsync` not `ToList` (when materializing)

A synchronous call in an async handler blocks the thread pool and can cause deadlocks under load.

### Input trimming before duplicate checks

If an entity is saved with `.Trim()` on a string field, the duplicate-existence check must also use the trimmed value. Using the raw input lets whitespace variants bypass uniqueness guards.

```csharp
// correct
var trimmed = input.name.Trim();
var exists = await db.regions.AnyAsync(x => x.name == trimmed, cancellationToken);
if (exists) return CallResult.Error("...");
entity = new() { name = trimmed };
```

### Null-forgiving `!` in LINQ expressions

Patterns like `x.benefit!.benefit!.auditAction` are common in LINQ `Where` clauses when navigation properties are always expected to be loaded via projection. The `!` suppresses nullable warnings. Before adding one, verify the navigation property is genuinely always non-null for the records being queried.

---

## Authorization

### Two-layer guard pattern

Every query and mutation starts with:

```csharp
if (!authContext.isLoggedIn) return ...;
if (!authContext.HasPermission("PermissionName")) return ...;
```

- Queries return `Array.Empty<T>().AsQueryable()` for both checks.
- Mutations return `CallResult.NotPermitted()` for the permission check (not for `isLoggedIn`, which is a different concern).
- `isLoggedIn` is always first. Checking permission before login check can throw.

### Permissions.cs is source of truth

All permission strings must be declared in `ManagementPermissions.all` in `Permissions.cs` (SharedModels). Adding a permission check in a handler without adding it to `Permissions.cs` means it can never be assigned to a role — the feature is effectively locked for all users. When adding a new permission string, always do both in the same change.

---

## Audit trail

### `AuditCreate` / `AuditUpdate` / `AuditDelete`

Always call the appropriate helper after the corresponding operation and after `SaveChangesAsync`. Skipping it means the action has no audit record and support cannot reconstruct what happened.

### Feed entries and status changes

When a mutation changes status, record the **actual** previous status as the feed's `oldValue`, not a hardcoded string. If a guard accepts multiple valid prior statuses (e.g. Approve accepts `Submitted` or `InReview`), hardcoding assumes only one was possible and produces wrong feed history.

```csharp
// correct
var prevStatus = entity.status;
entity.status = NewStatus;
await db.SaveChangesAsync(cancellationToken);
// use prevStatus.ToString() as oldValue in the feed entry
```

---

## State machines

### Dead-end state check

For any feature with a status enum, draw the full transition graph before reviewing the handlers. Confirm:

- Every status can be exited via at least one valid mutation (no dead-end states).
- Every mutation's status guard matches the actual incoming statuses, not an assumed linear flow.
- If a `Review` or `InReview` step exists, verify that both the approval path AND the rejection/cancellation path accept that status.

### Guard pattern for multiple valid prior statuses

```csharp
// prefer this form when multiple statuses are valid
if (entity.status is not (StatusA or StatusB))
    return CallResult.Error("...");
```

---

## Common anti-patterns

- Using `CallResult.Error("Not authorized")` instead of `CallResult.NotPermitted()`.
- Hardcoding the expected prior status as the feed `oldValue` when the guard accepts multiple statuses.
- Adding a permission check without adding the string to `Permissions.cs`.
- Using `.Include()` on a query method with `[UseProjection]` and expecting it to work.
- Not trimming input before a string duplicate check.
- Forgetting `AuditCreate` / `AuditUpdate` / `AuditDelete` after a save.
- Synchronous `FirstOrDefault()` / `SaveChanges()` inside async handler methods.
