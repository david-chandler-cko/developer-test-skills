# Decision rules

## Should I fake, intercept, or run it real?

| The dependency… | Choose | Why |
|---|---|---|
| Runs in the same process (validators, domain services, mappers) | **Real** | Mocking hides real wiring bugs. |
| Talks HTTP to an external service | **Intercept** (httpclient-interception) | Exercises the real outbound HTTP path; test controls the response. |
| Talks gRPC to an external service | **Intercept** (gRPC interceptor) | Same reason, same level. |
| Uses an AWS SDK client (SNS, SQS, DynamoDB, S3, Kinesis) | **Fake** the client interface | AWS SDK types are hard to intercept cleanly. The service should depend on a thin interface (e.g. `IEventPublisher`), which is trivial to fake. |
| Uses a database directly | **Fake repository** OR **Testcontainers** | Fake is faster; Testcontainers catches SQL/schema bugs. Use Testcontainers when the DB behaviour (transactions, queries, migrations) is part of what you want to test. |
| Reads feature flags | **Fake client** | Return deterministic values; seed per test. |
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

Three strategies in preference order (see SKILL.md for the short version). Detail here on the non-trivial one:

**1. Per-test fake instances** — each `TestScope` constructs `new Fake*()` fields. Best for Lambdas and small APIs where the host is cheap to rebuild.

**2. Shared singleton fakes keyed by a discriminator** — required when the host is shared across tests (`AlbaHost`, class-fixture `TestServer`) and re-registering fakes per-test is awkward.

```csharp
public sealed class FakeEventPublisher : IEventPublisher
{
    private readonly ConcurrentDictionary<string, List<Event>> _byCorrelationId = new();

    public Task PublishAsync(Event evt, CancellationToken ct)
    {
        var list = _byCorrelationId.GetOrAdd(evt.CorrelationId, _ => new List<Event>());
        lock (list) { list.Add(evt); }
        return Task.CompletedTask;
    }

    public IReadOnlyList<Event> GetPublished(string correlationId) =>
        _byCorrelationId.TryGetValue(correlationId, out var list)
            ? list.ToArray()
            : Array.Empty<Event>();
}
```

The discriminator must be unique per test (`Guid.NewGuid()` is fine) and propagated by the service into every call the fake intercepts. If the service reads correlation ID from a header, the test must set that header; if the service uses `Activity.Current.Id`, the fake must read that. `ThrowOnMissingRegistration` and similar fail-fast flags help catch propagation gaps.

**3. Shared singletons + `[Collection]` serialisation** — fallback when 1 and 2 don't fit. Tests in the same collection run serially. Slows the suite; keep the collection scope small.

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

## Production configuration tables are a test-case source

If the production code under test contains a static table — a `Dictionary<>`, an `ImmutableArray<>` of rule records, a `switch` over a composite key, a list initialised at startup from config — **generate one `[Theory]` row (or one `[Fact]`) per entry that distinguishably affects behaviour**. The rule matrix is the spec; a single happy-path test covers row zero and silently ignores the rest.

**Signals that say "table-driven":**
- A `static readonly` collection of records / tuples / objects, indexed by enums or strings.
- A `switch` expression on a composite key (e.g., `(Tier, Region, ResourceType)`).
- A config section deserialised into a list of rules at startup.
- Constants named like `…Rule`, `…Limit`, `…Threshold`, `…Bucket`, `…Bracket`.

**What to generate.** Prefer `[Theory]` with `[InlineData]` when rows differ only in inputs and expected outputs are mechanically derivable. Fall back to one `[Fact]` per row when each case needs distinct setup or distinct assertions.

**Boundary rows matter.** For each numeric field (limits, thresholds), generate the at-limit and just-over-limit cases — that's where the table's behaviour actually changes.

**Row coverage ≠ code coverage.** Coverage tools say row 7's line was hit because the dispatcher hit it. They can't see that the test never asserted row 7's distinguishing output.

## Ports-and-adapters entrypoint

If the service has:
- A thin HTTP/Lambda adapter that deserialises the request, calls an application handler, and serialises the response.
- An application handler (`ICommandHandler<Create>`, `IRequestHandler<...>`) that contains the actual behaviour.

You have two entrypoint choices:

- **Drive the transport** (HTTP endpoint / Lambda handler). Exercises the serialisation + middleware. Preferred when the adapter has meaningful logic (auth, validation, error mapping).
- **Drive the port** (call `commandHandler.Handle(command, ct)` directly). Faster; skips adapter concerns. Preferred when the adapter is generated or trivial.

Fakes and isolation work identically either way. The only thing that changes is how the test kicks off the scenario.
