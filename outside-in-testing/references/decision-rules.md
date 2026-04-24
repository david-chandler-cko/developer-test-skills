# Decision rules

## Should I fake, intercept, or run it real?

| The dependency… | Choose | Why |
|---|---|---|
| Runs in the same process (validators, domain services, mappers) | **Real** | Mocking hides real wiring bugs. |
| Talks HTTP to an external service | **Intercept** (httpclient-interception) | Exercises the real outbound HTTP path; test controls the response. |
| Talks gRPC to an external service | **Intercept** (gRPC interceptor) | Same reason, same level. |
| Uses an AWS SDK client (SNS, SQS, DynamoDB, S3, Kinesis) | **Fake** the client interface | AWS SDK types are hard to intercept cleanly. The service should depend on a thin interface (`ISnsPublisher`), which is trivial to fake. |
| Uses a database directly | **Fake repository** OR **Testcontainers** | Fake is faster; Testcontainers catches SQL/schema bugs. Use Testcontainers when the DB behaviour (transactions, queries, migrations) is part of what you want to test. |
| Reads feature flags (LaunchDarkly, etc.) | **Fake client** | Return deterministic values; seed per test. |
| Validates JWTs | **Fake issuer + signing key** | Register the fake issuer's key in the test host's JWT config; mint real tokens in tests. |

## Fake vs. Testcontainers for DBs

Prefer **fake** when:
- The repository is a thin CRUD wrapper.
- Tests don't care about transactions, concurrent writes, or query planning.
- Startup cost matters.

Prefer **Testcontainers** when:
- You're testing migrations, transactional behaviour, or complex queries.
- The codebase has several layers of DB interaction you want exercised.
- The service is DB-heavy and a fake would duplicate too much logic.

Both approaches are valid for the same underlying dependency — different cost/coverage trade-offs.

## Isolation strategies

When multiple tests run in parallel and touch the same code, their state must not mix. Three strategies, in order of preference:

### 1. Per-test fake instances (preferred)

Each `TestScope` creates its own fakes in its constructor.

```csharp
public sealed class TestScope : IDisposable
{
    public FakeThingStore FakeThingStore { get; } = new();
    // Register in the host DI when invoking the entrypoint — one DI container per test.
}
```

No discriminator needed. Best for:
- Lambdas (the test server is cheap, so a fresh DI graph per test is fine).
- Small APIs where you can rebuild the host per test.

Downsides:
- Expensive hosts (AlbaHost, full TestServer) shouldn't be rebuilt every test.

### 2. Shared singleton fakes keyed by a discriminator

Fakes registered once as singletons. They partition state by a per-test value the service propagates through the call chain.

```csharp
public sealed class FakeSnsPublisher : ISnsPublisher
{
    private readonly ConcurrentDictionary<string, List<SnsMessage>> _byCorrelationId = new();

    public Task PublishAsync(SnsMessage msg, CancellationToken ct)
    {
        var list = _byCorrelationId.GetOrAdd(msg.CorrelationId, _ => new List<SnsMessage>());
        lock (list) { list.Add(msg); }
        return Task.CompletedTask;
    }

    public IReadOnlyList<SnsMessage> GetMessages(string correlationId) =>
        _byCorrelationId.TryGetValue(correlationId, out var list)
            ? list.ToArray()
            : Array.Empty<SnsMessage>();
}
```

Required when:
- The host is shared across tests (one AlbaHost for the assembly, one TestServer with `IClassFixture`).
- Fakes are composed into the real DI graph, so re-registering per-test is awkward.

The discriminator can be anything, but:
- It must be unique per test (`Guid.NewGuid()` is fine).
- It must be propagated by the service into every call the fake intercepts. If the service reads a correlation ID from HTTP headers, the test must set that header. If the service uses `Activity.Current.Id`, the fake must read that.
- `ThrowOnMissingRegistration` and similar fail-fast flags help catch gaps.

### 3. Shared singleton fakes + `[Collection]` serialisation (fallback)

```csharp
[CollectionDefinition(ConfigurationCollection.Name)]
public class ConfigurationCollection : ICollectionFixture<ConfigurationApiFixture> { public const string Name = "config"; }

[Collection(ConfigurationCollection.Name)]
public class WhenRetrievingSomething { /* ... */ }
```

All tests in the same collection run serially, so shared-state fakes don't collide. Use only when:
- The host cost forces shared fakes AND
- No discriminator is available (no correlation ID flows through, or the fake gets called outside the request context).

Slows the suite down. Not scalable. Keep the collection scope small.

## Shared vs. per-test fixture

| You have… | Fixture |
|---|---|
| An AlbaHost, `WebApplicationFactory<Program>`, or Testcontainers DB | **Shared** (`IClassFixture<T>` or collection fixture) — building it every test is too slow. |
| An AWS Lambda test server | **Per-test** — it starts in milliseconds. |
| A generic ports-and-adapters setup with no transport | **Per-test** — it's just DI. |

Shared fixture ≠ shared state. Even with a shared fixture, each test should get its own `TestScope` and either strategy 1 (instance fakes injected into a per-test child scope) or strategy 2 (discriminator-keyed singletons). Strategy 3 only if 1 and 2 don't fit.

## Auth in tests

- **OAuth/JWT**: spin up a fake issuer (in-memory `OpenIdConnectConfiguration` with a known signing key). Tests mint tokens with whatever claims the scenario needs. No real IDP calls. A typical `FakeTokenIssuer` exposes a static `Issuer`, a `SecurityKey`, and a `CreateToken(claims)` helper.
- **API keys**: generate random GUIDs at fixture init; configure the app to accept them; pass them in request headers via a scenario-builder extension (`WithApiKey`).
- **Bypass entirely**: valid for tests that don't exercise auth, but make it explicit — register a test authentication handler that accepts everything, don't just remove the middleware.

## Ports-and-adapters entrypoint

If the service has:
- A thin HTTP/Lambda adapter that deserialises the request, calls an application handler, and serialises the response.
- An application handler (`ICommandHandler<Create>`, `IRequestHandler<...>`) that contains the actual behaviour.

You have two entrypoint choices:

- **Drive the transport** (HTTP endpoint / Lambda handler). Exercises the serialisation + middleware. Preferred when the adapter has meaningful logic (auth, validation, error mapping).
- **Drive the port** (call `commandHandler.Handle(command, ct)` directly). Faster; skips adapter concerns. Preferred when the adapter is generated or trivial.

Fakes and isolation work identically either way. The only thing that changes is how the test kicks off the scenario.
