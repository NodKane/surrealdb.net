# Querying with LINQ and `IQueryable`

The SDK exposes SurrealDB `SELECT` queries as `IQueryable<T>`. You compose a query with familiar LINQ operators, the provider translates the expression tree to SurrealQL, and an async terminal operation executes it against SurrealDB.

This is a query provider, not a change-tracking ORM. Use the regular client methods such as `Create`, `Update`, `Upsert`, and `Delete` for writes.

```text
Select<T>() -> compose LINQ -> inspect SurrealQL -> execute asynchronously
```

## Define a query model

Use `TableAttribute` to map a CLR type to a SurrealDB table and `ColumnAttribute` when a property name differs from its stored field name:

```csharp
using System.ComponentModel.DataAnnotations.Schema;
using SurrealDb.Net.Models;

[Table("post")]
public sealed class Post : Record
{
    [Column("title")]
    public string Title { get; set; } = string.Empty;

    [Column("content")]
    public string Content { get; set; } = string.Empty;

    [Column("status")]
    public string Status { get; set; } = string.Empty;

    [Column("created_at")]
    public DateTime? CreatedAt { get; set; }
}
```

`Record` supplies the `Id` property. A model does not need to inherit from `Record` for a normal table query, but graph nodes must implement `IRecord`.

When no table name is passed to `Select<T>()`, the provider uses `TableAttribute.Name`, or the CLR type name if the attribute is absent. An explicit table name overrides that convention:

```csharp
IQueryable<Post> byAttribute = client.Select<Post>();
IQueryable<Post> explicitTable = client.Select<Post>("post");
```

The `Select<T>(RecordId)` overload is different: it immediately loads one record and returns a `Task<T?>`. Use the parameterless or table-name overload to start an `IQueryable<T>` query.

## Filter with captured values

Captured local variables become SurrealDB query parameters:

```csharp
int minimumAge = 18;

var adults = client
    .Select<User>("user")
    .Where(user => user.Age >= minimumAge);
```

The predicate is translated to a form such as `Age >= $minimumAge`, and the value is sent separately. Compile-time constants can be embedded directly in the generated SurrealQL. Parameterization lets the same query shape be reused safely with different values.

Boolean expressions, comparisons, null checks, collection operations, and supported .NET methods are translated when the provider has a SurrealQL equivalent. Multiple `Where` calls are combined into the generated query.

## Project only what is needed

Use `Select` to return one field, an anonymous object, or a DTO:

```csharp
var summaries = await client
    .Select<Post>()
    .Where(post => post.Status == "PUBLISHED")
    .Select(post => new { Title = post.Title, CreatedAt = post.CreatedAt }))
    .ToListAsync(cancellationToken);
```

The provider selects only the fields required by the projection. `SelectMany` can flatten supported array or nested collection fields.

## Order, page, group, and aggregate

Common composable operators include:

- `Where`, `Select`, `SelectMany`, and `Distinct`
- `OrderBy`, `OrderByDescending`, `ThenBy`, and `ThenByDescending`
- `Skip` and `Take`
- `GroupBy` followed by supported projections and aggregations

For deterministic paging, apply an `OrderBy` before `Skip` and `Take`:

```csharp
var page = await client
    .Select<Post>()
    .OrderByDescending(post => post.CreatedAt)
    .ThenBy(post => post.Id)
    .Skip(pageIndex * pageSize)
    .Take(pageSize)
    .ToListAsync(cancellationToken);
```

Async scalar operations are available for common terminal queries:

```csharp
int draftCount = await client
    .Select<Post>()
    .CountAsync(post => post.Status == "DRAFT", cancellationToken);

DateTime? newest = await client
    .Select<Post>()
    .MaxAsync(post => post.CreatedAt, cancellationToken);
```

Supported terminal families include:

- `AnyAsync`, `AllAsync`, `ContainsAsync`, `CountAsync`, and `LongCountAsync`
- `FirstAsync`, `FirstOrDefaultAsync`, `LastAsync`, and `LastOrDefaultAsync`
- `SingleAsync`, `SingleOrDefaultAsync`, `ElementAtAsync`, and `ElementAtOrDefaultAsync`
- `SumAsync`, `AverageAsync`, `MinAsync`, and `MaxAsync`
- `ToListAsync`, `ToArrayAsync`, and `ForEachAsync`

All async terminal methods accept a `CancellationToken`.

## Cache repeated query translation

`Cached()` caches the generated query shape at its call site while still extracting current captured parameter values on each execution. It is useful for a frequently repeated, structurally identical query:

```csharp
IQueryable<Post> FindByStatus(string status) =>
    client
        .Select<Post>()
        .Where(post => post.Status == status)
        .Cached();

var drafts = await FindByStatus("DRAFT").ToListAsync(cancellationToken);
var published = await FindByStatus("PUBLISHED").ToListAsync(cancellationToken);
```

Apply `Cached()` once, after the complete query shape has been composed. It caches translation, not database result rows.

## Translation boundaries

An `IQueryable<T>` lambda is an expression tree, not arbitrary client-side C# code. Only expressions understood by the provider can be converted to SurrealQL. Unsupported methods or query shapes fail during translation rather than silently loading rows and evaluating them in memory.

Practical guidelines:

- Keep filtering, ordering, projection, and paging in the `IQueryable` pipeline so SurrealDB performs them.
- Call `ToQueryString()` while developing complex queries.
- Materialize first when an operation is intentionally client-side.
- Use `Query(...)` or `RawQuery(...)` for SurrealQL features that the LINQ provider cannot yet express.

For relation traversal with `Out`, `In`, and `Both`, continue with [Graph traversals with LINQ and `IQueryable`](./GRAPH_QUERIES.md).
