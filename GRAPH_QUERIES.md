# Graph traversals with LINQ and `IQueryable`

This guide builds on the [general `IQueryable` guide](./IQUERYABLE.md).

The LINQ provider can translate typed graph paths to [SurrealQL arrow syntax](https://surrealdb.com/docs/learn/data-models/graph/graph-traversal).
Graph traversal methods are expression markers: use them inside a query created by `Select<T>()`; calling them on materialized records throws `NotSupportedException`.

## Model nodes and relations

Map node and relation tables with `TableAttribute`. A relation inherits from `RelationRecord`, whose `In` and `Out` properties contain the raw endpoint IDs.

Optional typed endpoint properties make the one-type traversal syntax available. Mark the record on the left of `RELATE` with `SurrealIn` and the record on the right with `SurrealOut`:

```csharp
using System.ComponentModel.DataAnnotations.Schema;
using System.Text.Json.Serialization;
using Dahomey.Cbor.Attributes;
using SurrealDb.Net;
using SurrealDb.Net.Attributes;
using SurrealDb.Net.Models;

[Table("user")]
public sealed class User : Record
{
    [Column("name")]
    public string Name { get; set; } = string.Empty;
}

[Table("product")]
public sealed class Product : Record
{
    [Column("name")]
    public string Name { get; set; } = string.Empty;

    [Column("price")]
    public float Price { get; set; }
}

[Table("purchased")]
public sealed class Purchased : RelationRecord
{
    // RELATE user:...->purchased->product:...
    [SurrealIn]
    [JsonIgnore]
    [CborIgnore]
    public User User { get; } = default!;

    [SurrealOut]
    [JsonIgnore]
    [CborIgnore]
    public Product Product { get; } = default!;

    [Column("quantity")]
    public int Quantity { get; set; }
}
```

The typed endpoint properties are query-navigation markers, not additional stored fields. Ignoring them during JSON and CBOR serialization keeps writes limited to the inherited `in` and `out` record IDs.

Each relation type used by a one-type traversal must have exactly one public `SurrealIn` property and one public `SurrealOut` property, and both property types must implement `IRecord`.

## Traverse to typed nodes

Use the two-type overloads when the result should be a sequence of nodes. `TEdge` selects the relation table and `TNode` selects the endpoint table:

```csharp
var query = client
    .Select<User>()
    .Select(user =>
        user.Out<Purchased, Product>()
            .Where(product => product.Price > 100)
            .Select(product => product.Name)
    );

var productNamesByUser = await query.ToListAsync();
```

This produces a path equivalent to:

```surql
SELECT VALUE $this->purchased->product[WHERE price > 100].name FROM user;
```

The direction is relative to the current node:

- `Out<TEdge, TNode>()` translates to `->edge->node`.
- `In<TEdge, TNode>()` translates to `<-edge<-node`.
- `Both<TEdge, TNode>()` translates to `<->edge<->node`.

The explicit `Out<TEdge, TNode>()` and `In<TEdge, TNode>()` overloads can be used without typed endpoint properties. In that case, the generic types are the source of the relation and endpoint table names, so make sure they match the SurrealDB relation schema and direction.

## Work with edge and node fields together

For a typed node traversal, use a `GraphStep<TEdge, TNode>` lambda when a filter or projection needs both the relation record and the reached node:

```csharp
var query = client
    .Select<User>()
    .SelectMany(user =>
        user.Out<Purchased, Product>()
            .Where(step => step.Edge.Quantity > 1 && step.Node.Price > 100)
            .Select(step => new
            {
                step.Node.Name,
                step.Edge.Quantity,
                Sales = step.Edge.Quantity * step.Node.Price,
            })
    );
```

Edge predicates are placed on the relation table, while node-only predicates are placed on the endpoint array.

## Use the attributed edge shorthand

With `SurrealIn` and `SurrealOut` endpoint properties, the one-type form exposes the relation record:

```csharp
var query = client.Select<User>().Select(user =>
    user.Out<Purchased>()
        .Where(purchase => purchase.Quantity > 1 && purchase.Product.Price > 100)
        .Select(purchase => purchase.Product.Name)
);
```

The endpoint property selects the node reached by the current direction. Other properties select fields stored on the edge.

Unlike the two-type form, `Out<TEdge>()`, `In<TEdge>()`, and `Both<TEdge>()` are edge traversals and must be projected with `Select` before they can be materialized. This is a SurrealDB-specific convenience API; despite the similar names, it does not have the same return semantics as Gremlin's vertex-returning `out()`, `in()`, and `both()` steps.

## Chain graph paths

Traversal methods can be chained to build multi-hop paths:

```csharp
var query = client.Select<Product>().Select(product =>
    product.In<Purchased, User>()
        .Out<Purchased, Product>()
        .Select(relatedProduct => relatedProduct.Name)
        .Distinct()
);
```

Equivalent path:

```surql
SELECT VALUE array::flatten(array::distinct(
    $this<-purchased<-user->purchased->product.name
)) FROM product;
```

The LINQ provider automatically adds flattening where chained graph lookups introduce nested arrays.

## Traverse symmetric relations with `Both`

`Both` is intended for symmetric relations whose `in` and `out` endpoints have the same node type:

```csharp
[Table("friends")]
public sealed class Friends : RelationRecord
{
    [SurrealIn, JsonIgnore, CborIgnore]
    public User FirstUser { get; } = default!;

    [SurrealOut, JsonIgnore, CborIgnore]
    public User SecondUser { get; } = default!;
}

var query = client.Select<User>().Select(user =>
    user.Both<Friends, User>()
        .Select(friend => friend.Name)
        .Distinct()
);
```

`Both<Friends, User>()` maps directly to SurrealQL's `<->friends<->user`. SurrealQL returns both endpoints of every matching edge, so the current source node is included when it is one of those endpoints. This differs from Gremlin's `both()` neighbor semantics. If the source must be excluded, use an explicit SurrealQL projection with `array::complement`, or remove it after materialization.

Both endpoint properties are required for `Both`, and they must use the same CLR type. Use `Both<TEdge>()` only to filter or project fields on the symmetric relation record itself.

Avoid `GraphStep<TEdge, TNode>` edge-and-node lambdas with `Both` for now. A bidirectional edge has no single fixed `in` or `out` property that always represents the other endpoint; project nodes and edge fields separately, or use an explicit SurrealQL query when both are required together.

## Supported LINQ operations

Typed node traversals support common sequence operations such as:

- `Where`, `Select`, and `SelectMany`
- `Any`, `Count`, and `Distinct`
- `OrderBy`, `OrderByDescending`, `ThenBy`, and `ThenByDescending`
- `Skip`, `Take`, and `FirstOrDefault`
- `Sum`, `Average`, `Min`, and `Max` after projecting a numeric node or edge field

Use `ToQueryString()` while developing to inspect the generated SurrealQL:

```csharp
string surql = query.ToQueryString();
```

Graph operations that cannot be translated by the provider fail during query translation instead of running on the client.
