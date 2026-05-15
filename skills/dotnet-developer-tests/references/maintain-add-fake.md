# Maintain: add a fake

## When to reach for a fake vs. interceptor

- **HTTP / gRPC call**: use an interceptor (see [maintain-add-http-interception.md](maintain-add-http-interception.md)). Do NOT wrap the HTTP client in a fake.
- **Repository / Store / SDK client** (AWS, database, feature flags, token issuer): use a fake.

## Checklist

1. **Find the real interface(s).** In `src/`, grep for `I<Name>Repository`, `I<Name>Store`, `I<Name>Client`, `I<Name>Publisher`, or the SDK interface (`IAmazonSQS`, etc.). The service should already depend on the interface — if it depends on a concrete AWS SDK type directly, wrap it in a thin interface first.

   **Check for CQRS-style pairs.** If you find `IQuery<T>` and `IStore<T>` (or `IRead<T>` / `IWrite<T>`, or `IGet<T>` / `ISave<T>`) for the same underlying entity, **treat them as one fake implementing both interfaces** — see [Pair CQRS interfaces](#pair-cqrs-interfaces) below before continuing.

2. **Determine the project's isolation strategy.** Inspect an existing fake in `Infrastructure/Fakes/` (or `Infrastructure/` for smaller projects). Two signatures to look for:
   - **Per-test-instance** (strategy 1): plain `Dictionary<>`/`List<>`, seed/query methods take no discriminator.
   - **Discriminator-keyed** (strategy 2): `ConcurrentDictionary<TDiscriminator, ...>`, query methods take the discriminator.

3. **Create `Fake<Name>.cs`** matching that strategy.

### Per-test-instance fake

```csharp
public sealed class FakeThingRepository : IThingRepository
{
    private readonly Dictionary<string, Thing> _things = new();

    public void AddThing(Thing thing) => _things[thing.Id] = thing;

    public Task<Thing?> GetByIdAsync(string id, CancellationToken ct)
    {
        _things.TryGetValue(id, out var thing);
        return Task.FromResult(thing);
    }

    public IReadOnlyList<Thing> GetAll() => _things.Values.ToList();
}
```

### Discriminator-keyed fake

```csharp
public sealed class FakeThingPublisher : IThingPublisher
{
    private readonly ConcurrentDictionary<string, List<Thing>> _byCorrelationId = new();

    public Task PublishAsync(Thing thing, CancellationToken ct)
    {
        // The service must propagate the correlation ID — read it from wherever it lives
        var cid = thing.CorrelationId;
        var list = _byCorrelationId.GetOrAdd(cid, _ => new List<Thing>());
        lock (list) { list.Add(thing); }
        return Task.CompletedTask;
    }

    public IReadOnlyList<Thing> GetPublished(string correlationId) =>
        _byCorrelationId.TryGetValue(correlationId, out var list)
            ? list.ToArray()
            : Array.Empty<Thing>();
}
```

4. **Register it.**

   - **Lambda (per-test-instance)**: add a property on `TestScope`, wire it in `ModifyServices`:
     ```csharp
     public FakeThingRepository FakeThingRepository { get; } = new();
     // in ModifyServices:
     serviceCollection.AddSingleton<IThingRepository>(FakeThingRepository);
     ```
   - **API-WAF**: in `ApiFixture.ConfigureTestServices` (or subclass override):
     ```csharp
     services.RemoveAll<IThingRepository>();
     services.AddSingleton<IThingRepository, FakeThingRepository>();
     ```
   - **API-Xunit.DI**: in `Startup.ConfigureServices`:
     ```csharp
     services.AddSingleton<FakeThingRepository>();
     services.AddSingleton<IThingRepository>(sp => sp.GetRequiredService<FakeThingRepository>());
     ```
   - **API-Alba**: in `ApiFixture.InitializeAsync`'s `builder.ConfigureServices` block (same pattern as Xunit.DI).

5. **Expose access from `TestScope`**. Add a property or method so scenarios can seed and query:
   ```csharp
   public FakeThingRepository FakeThingRepository { get; }
   // in constructor, resolved from the host services
   FakeThingRepository = services.GetRequiredService<FakeThingRepository>();
   ```

6. **Sanity-check.**
   - Every interface method is implemented.
   - Query methods return copies (`.ToArray()`, `.ToList()`) — never expose the live collection.
   - No `if`/`else` business logic beyond "seed + read".
   - For strategy 2: the fake reads the discriminator from the same place the service writes it (request header → context item → propagated value).

## Pair CQRS interfaces

When production splits a single underlying repository into a read interface and a write interface — `IQueryOrders` / `IStoreOrders`, `IReadCustomer` / `IWriteCustomer`, `IGet<T>` / `ISave<T>` — **the fake should implement both interfaces and be registered against both, as a single shared instance**. The in-memory store is the source of truth; both interfaces are views over it.

```csharp
public sealed class FakeOrderRepository : IQueryOrders, IStoreOrders
{
    private readonly Dictionary<string, Order> _orders = new();

    public Task SaveAsync(Order order, CancellationToken ct)
    {
        _orders[order.Id] = order;
        return Task.CompletedTask;
    }

    public Task<Order?> GetByIdAsync(string id, CancellationToken ct)
    {
        _orders.TryGetValue(id, out var o);
        return Task.FromResult(o);
    }
}
```

Register **the same instance** against both interfaces — never `new FakeOrderRepository()` twice:

```csharp
// Lambda (per-test-instance) — in TestScope:
public FakeOrderRepository OrderRepository { get; } = new();

// in ModifyServices:
services.AddSingleton<IQueryOrders>(OrderRepository);
services.AddSingleton<IStoreOrders>(OrderRepository);
```

```csharp
// WAF / Xunit.DI / Alba — in ConfigureTestServices (per-instance pattern):
services.RemoveAll<IQueryOrders>();
services.RemoveAll<IStoreOrders>();
services.AddSingleton<FakeOrderRepository>();
services.AddSingleton<IQueryOrders>(sp => sp.GetRequiredService<FakeOrderRepository>());
services.AddSingleton<IStoreOrders>(sp => sp.GetRequiredService<FakeOrderRepository>());
```

**Why this matters.** Two fakes for one underlying store splits state across two halves, so a scenario that seeds via `IStore` and asserts via `IQuery` silently asserts on the wrong store. The write/read round-trip the handler actually performs goes unverified.

**When to skip the pair.** If the two interfaces back genuinely different stores in production (e.g., read replica vs. write DB, confirmed by different concrete classes against different connection strings), they're separate dependencies — two fakes is correct.

## Gotchas

- **Service depends on a concrete AWS SDK type** (`AmazonSQSClient`, not `IAmazonSQS`): introduce the interface first. Fakes can't substitute concrete types.
- **Fake needs to generate IDs** (new entity identifiers): seed the IDs from the test, or inject a fake `IIdFactory` / `IClock` the test controls.
- **Fake "forgets" data between calls** in strategy 2: you're probably keying by a value the service doesn't propagate. Put a `Debug.WriteLine` in the fake's write and read paths with the discriminator value and compare.
- **You only faked one half of a CQRS pair** (`FakeQueryOrders : IQueryOrders` with no matching `IStoreOrders` fake): a single fake should implement both interfaces. See [Pair CQRS interfaces](#pair-cqrs-interfaces).
