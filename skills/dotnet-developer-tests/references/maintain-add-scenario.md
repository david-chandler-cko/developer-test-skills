# Maintain: add a new scenario

## Read sibling scenarios first — they are the ground truth

Before writing the new `When*.cs`, read at least two existing scenarios in the same area (`Scenarios/<Area>/When*.cs`) and the project's `Infrastructure/TestScope.cs` + `TestContext.cs` (or equivalent). Mature test projects accumulate idioms that the skill cannot anticipate:

- **Constructor-injection pattern** (Xunit.DependencyInjection primary constructor vs. `IClassFixture<>` vs. `[Collection]`).
- **Endpoint-family base classes** (e.g., a `BaseReversalRequestTest` that centralises the auth-method selection, the HTTP-call helper, and any per-family setup). New scenarios in that family should inherit it rather than re-deriving the boilerplate.
- **The project's randomiser of choice** (a `Data.Random<T>()` wrapper, `Bogus`, `AutoFixture` direct). If the project uses its own thin wrapper, use that — don't introduce a second randomiser even if the skill suggests AutoFixture.
- **The project's request-decoration pattern** (see "Scope-level request decoration" below).

A 30-second skim of two sibling files saves an hour of re-deriving conventions. **The single most useful first step in the maintain flow.**

## Naming

- Class name: `When<ThingHappens>` (e.g., `WhenAnOrderIsCreatedWithoutABillingAddress`).
- File path: `Scenarios/<Area>/When<ThingHappens>.cs` for organised projects; root for tiny Lambda projects.
- Assertion methods: `Then<Assertion>()` or `Should<Behaviour>()` — pick one convention and use it consistently across the project.

## Shape

Prefer **class-per-scenario with one `InitializeAsync` + multiple `[Fact]` assertions** when the scenario has one Arrange+Act and multiple independent things to verify. Use **one `[Fact]` per file** when each test has its own Arrange+Act.

### Class-per-scenario template (shared Arrange+Act)

```csharp
public class WhenSomethingHappens : IAsyncLifetime
{
    private readonly TestContext _testContext;          // injected via DI (Xunit.DI) or fixture
    private ApiResponse<FooResponse> _response = null!;

    public WhenSomethingHappens(TestContext testContext)
    {
        _testContext = testContext;
    }

    public async Task InitializeAsync()
    {
        using var testScope = _testContext.CreateScope(options =>
        {
            options.ConfigureInterceptor((correlationId, interceptorOptions) =>
            {
                interceptorOptions.RegisterSomethingSpecific(correlationId);
            });
        });

        var request = new FooRequestBuilder()
            .WithBar("baz")
            .Build();

        var client = testScope.CreateApiClient();
        _response = await client.DoSomethingAsync(request);
    }

    [Fact]
    public void ResponseIsSuccessful() =>
        _response.ShouldBeSuccessful();

    [Fact]
    public void ResponseHasTheExpectedShape() =>
        _response.Content.Should().BeEquivalentTo(...);

    public Task DisposeAsync() => Task.CompletedTask;
}
```

### One-test-per-file template (Lambda, simpler)

```csharp
public class WhenSomethingHappens : IClassFixture<LambdaTestServerFixture>, IDisposable
{
    private readonly TestScope _testScope;

    public WhenSomethingHappens(ITestOutputHelper output, LambdaTestServerFixture fixture)
    {
        _testScope = fixture.CreateTestScope(output);
    }

    [Fact]
    public async Task ShouldPublishTheExpectedEvent()
    {
        _testScope.FakeStore.Seed(...);

        await _testScope.InvokeFunction(partitionKey: "x", sortKey: "y", changes: [...]);

        var published = _testScope.FakeKafkaPublisher.GetPublishedRecords();
        published.Should().BeEquivalentTo(new[] { new { Change1 = "ExpectedUpdatedField", Unchanged1 = "OriginalField", ... } });
    }

    public void Dispose() => _testScope.Dispose();
}
```

## Pattern

1. **Arrange**: seed fakes, configure HTTP interceptor overrides, build the request.
2. **Act**: invoke the entrypoint via the TestScope (`_testScope.Scenario(...)`, `_testScope.InvokeFunction(...)`, `apiClient.DoX(...)`, `_testScope.SendMessageAsync(queue, _ => message)`).
3. **Assert**: check the response, query the fakes, check the recorded HTTP requests.

### Queue-consumer entrypoint (SQS/Kafka/etc.)

For message-processing services, the test's "act" call usually looks like:

```csharp
var result = await testScope.SendMessageAsync(
    CommanderQueue.RedirectAuthorization,           // queue identifier (enum or string)
    _ => message);                                  // factory for the deserialised payload

result.ProcessingAction.Should().Be(ProcessingAction.Processed);  // or Retried / Deadlettered
```

Two things to look for in sibling scenarios:

- **The queue-identifier shape.** Some projects use an enum (`CommanderQueue.X`), others a constant string, others the SQS URL. The enum is what `TestScope` maps to the actual queue.
- **The acceptance-result return.** Queue consumers usually return a structured "what happened" record (`Processed` / `Retried` / `Deadlettered`). Always assert this first — it's the queue handler's contract — *before* snapshotting fake state. A test that snapshots state without checking acceptance can pass when the handler retries indefinitely and never processes.

### Assertion-order rule for event sequences

When the production code emits a sequence of events that represents a *progression* (state-machine transitions: `AcquirerAuthorized → Authorized → CaptureRequested → ...`), assert with **strict ordering**:

```csharp
events.Select(e => e.GetType().Name).Should().BeEquivalentTo(
    new[] { nameof(A), nameof(B), nameof(C) },
    options => options.WithStrictOrdering());
```

When the production code emits a *set* of side-effect events whose order is genuinely implementation-defined (a refund + an audit-log + a notification), assert with set equivalence (the `BeEquivalentTo` default — `WithoutStrictOrdering()` makes the intent explicit).

`BeEquivalentTo` is order-insensitive by default. A progression-style requirement that uses the default is silently testing the wrong thing: the assertion passes even when the events fire in the wrong order, which is the most common state-machine bug.

## Look for table-driven production code before writing a single `[Fact]`

If the handler dispatches behaviour off a static rule table (a `Dictionary<>`, an immutable list of records, a `switch` over a composite key, a deserialised config section), generate one `[Theory]` row — or one `[Fact]` — **per row that distinguishably affects behaviour**, not a single happy-path test. The matrix is the spec.

```csharp
[Theory]
[InlineData("free",  "US", "compute", 10,   false)]  // under-limit
[InlineData("free",  "US", "compute", 11,   true)]   // just-over
[InlineData("pro",   "EU", "compute", 1000, false)]
[InlineData("pro",   "*",  "storage", 100,  true)]
// ...one row per distinguishable rule
public async Task ShouldEnforceQuota(string tier, string region, string resource, int amount, bool rejected) { ... }
```

Prefer `[InlineData]` when rows differ only in inputs and the expected output is mechanically derivable; fall back to one `[Fact]` per row when each case has distinct setup or distinct assertions. For numeric fields, generate **at-limit and just-over-limit** rows. See `outside-in-testing/references/decision-rules.md` "Production configuration tables are a test-case source".

## Builders: variant factories, derived builders, AutoFixture defaults

A builder is more than a fluent setter chain. Two patterns make builders pay for themselves once the suite passes ~10 scenarios:

**1. Variant factories.** When the entity-under-test has a small enumeration of variants whose defaults travel together (e.g., a request where `category = "Internal"` implies `transport ∈ {"A", "B", "C"}` while `category = "External"` implies `transport ∈ {"X", "Y"}`), expose **static factory methods** rather than free-form setters that callers must remember to pair.

```csharp
public static FooRequestBuilder Variant1() { ... pre-set defaults consistent with variant 1 ... }
public static FooRequestBuilder Variant2() { ... pre-set defaults consistent with variant 2 ... }
```

Callers write `FooRequestBuilder.Variant1().WithAmount(100m)` rather than `new FooRequestBuilder().WithCategory("...").WithTransport("...").WithRegion("...").WithAmount(100m)`. The factory enforces the variant's invariants; tests only mention the field they care about.

**2. AutoFixture for non-distinguishing fields.** Identifiers, opaque IDs, and "must be a valid value but the test doesn't care which" fields should be `AutoFixture`-generated, not hard-coded constants. Constants shared across tests are a recipe for cross-test leakage when isolation strategy 1 isn't perfectly clean.

```csharp
private FooRequestBuilder(Fixture fixture, string variantField)
{
    _someString = fixture.Create<string>();
    _otherString = fixture.Create<string>();
    // For fields that must satisfy a regex (e.g., country code):
    var defaults = fixture.Create<Defaults>(); // record with [RegularExpression] annotations
    _sourceCountry = defaults.SourceCountry;
}

private record Defaults
{
    [RegularExpression("US|UK|NZ")] public string SourceCountry { get; init; } = null!;
}
```

`[RegularExpression]` keeps generated values inside a known-valid set without hand-coding the values. Only fields where the *value* affects the assertion need explicit setters.

Builders explicitly set what the test cares about and leaves sensible defaults for everything else. The assertions that can clearly refer back to the builders to test the observable behaviour of that setting that data; for example, if a builder defines `WithFoo("Something")`, then the assertion would assert `Result.Foo.Should().Be("Something")`. 

**One builder per non-trivial domain object referenced by ≥2 tests.** If two scenarios both seed a `RuleSet` (or whatever your domain type is called), there should be a `RuleSetBuilder` — not two inline `new RuleSet { ... }` blocks copy-pasted between scenarios. Inline construction is fine for one-off domain objects only touched by a single scenario.

## The default scope is the happy path — override only what differs

Mature test projects build a `TestContext.CreateScope(...)` factory that *already* registers a happy-path baseline: every external HTTP/gRPC dependency has a 2xx interceptor, every feature flag is in a sensible default state, every fake is empty. New scenarios only override what their case actually differs on.

When reading sibling scenarios, separate "what they explicitly configure" from "what they inherit". If three siblings all call `options.AddAuthorizedPayment(...)` and none of them registers an acquirer-authorisation interceptor, the acquirer-auth interceptor is already in the baseline — don't re-register it. Only declare interceptors for the response path your scenario specifically asserts on (e.g., a *failed* response, an unusual route).

The corollary for **TestContext design** (see [scaffold-api-* references](.) for the API variants): the default scope should be a fully successful happy-path scope. Scenarios that need failure inject the failure as an override. This makes scenarios short ("what's different about this case") and means a new external dependency in production is added in *one* place (the baseline) rather than in every existing test.

## Prefer the scope-options seeding helper over reaching into the fake

Mature test projects often expose **higher-level seeding helpers on the scope options object**, not just `scope.FakeRepo.Seed(builder.Build())`. The helpers encode domain-level prerequisites (e.g., `AddAuthorizedPayment`, `SeedExistingOrder`, `WithEnrolledUser`) and may seed multiple fakes together (the order repository *and* the payment-events fake *and* the audit-log fake) so the resulting state is internally consistent.

Use those helpers when they exist:

```csharp
using TestScope testScope = testContext.CreateScope(
    options =>
    {
        options.AddAuthorizedPayment(_paymentId, TestCardNumbers.MasterCard, amount: 100);
        // Higher-level than scope.FakePaymentsRepository.Seed(...); avoids leaking
        // implementation details (which fakes does authorisation actually touch?)
        // into every scenario.
    });
```

When designing the helper API (see [maintain-add-fake.md](maintain-add-fake.md) for fake design) prefer one helper per *domain prerequisite* over one per *underlying fake*. The test should read like the scenario's preconditions, not like the storage layer's schema.

The "direct seed vs. round-trip" rule below still applies to the *helper's implementation*: the helper should typically direct-seed the underlying fakes rather than round-tripping through a sibling handler.

## Scope-level request decoration (headers, auth, request origin)

If the scenario needs to set request-scope HTTP headers — `X-Request-Origin`, an `Idempotency-Key`, a specific JWT, a tenant header — the right place is usually a **scope-construction parameter that decorates the API client builder**, not the request body. The skill's `CreateScope(options => ...)` example only shows the options delegate; mature projects very often expose a second positional argument:

```csharp
using TestScope testScope = testContext.CreateScope(
    options =>
    {
        options.AddAuthorizedPayment(_paymentId, /* ... */);
    },
    clientBuilderDecorator: builder => builder.WithRequestOrigin("SOME_SDK"));
```

When you read sibling scenarios, look for that second-positional invocation — it's the seam for header-level setup. If it doesn't exist yet and your scenario needs header-level control, adding the decorator parameter to `CreateScope` (and threading it through to the Refit/HTTP-client construction) is a tiny refactor that pays for itself across the suite.

## Seeding test data: direct fake vs. round-trip through a sibling handler

When a scenario needs historical state in a fake (e.g., "given some past orders, validate a new one"), there are two ways to seed it. Pick deliberately:

| Option | When to use | Cost |
|---|---|---|
| **Direct seed** — `scope.FakeRepo.Seed(builder.Build())` | Default. The test asserts on the *current* handler under test; how the data got there is incidental. | Couples the test to the fake's `Seed` API, not to the sibling handler. |
| **Round-trip seed** — invoke the sibling handler (e.g., the SQS handler that writes orders) that *would* populate the fake in production | When the sibling handler's write shape is itself part of what's under test (e.g., this scenario implicitly verifies that what the SQS handler stored is the right shape for the ALB handler to read). | Couples this test to the sibling handler. If that handler breaks, this test fails for an unrelated reason. Slower. |

Default to **direct seed**. Reach for round-trip only when "is the data the other handler wrote actually readable by this one?" is the question being tested — and document that as the test's name (`PassesWhenSqsPersistedRecordsFitVolumeLimit`, not `PassesWhenHistoricalRecordsBelowLimit`). A scenario that round-trips by default is testing two handlers in one fact, which makes failures hard to attribute.

## If the inbound message is wrapped, build through the same envelope

When the production deserialiser unwraps a transport-level envelope before reaching the domain handler (e.g., a base64-encoded inner payload, a JWS `signedPayload` with three dot-separated segments, an SNS message envelope around an SQS body, an outer event wrapper around a CloudEvent), the test must build through *that envelope shape* — not directly through the deserialised inner type. Faking the inner type and skipping the envelope means the test never exercises the deserialiser, which is the most common place new event types break.

Look at the production deserialiser before writing the request builder. If you see `Convert.FromBase64String`, `JsonSerializer.Deserialize<Outer>`, a record split on `.`, or a property named `SignedPayload` / `Payload` / `Body` / `Message`, the inbound message is wrapped — build a `SqsEventBuilder` (or equivalent) that produces the encoded shape, and unit-test the builder once if the encoding is non-trivial.

The mirror rule applies on the outbound side: **if a fake stores an envelope (e.g., an SQS `Enqueue` request, an SNS `PublishRequest`, a Kafka `Message<K, V>`), unwrap it in the test before asserting on the domain payload.** A pattern like

```csharp
var envelope = testScope.GetSqsMessage(); // returns the queue-level envelope
envelope.Should().NotBeNull();
envelope!.Message.Should().BeOfType<ReversePayment>();
var domainMessage = (ReversePayment)envelope.Message;
// now assert on domainMessage fields
```

is the right shape. Asserting on the JSON body string of the envelope ties the test to serialiser formatting; asserting on the deserialised domain type ties it to the schema you actually care about.

## Assertions

**Validate Mappings** - Make sure that the output of the system under test is inspected in detail. For example, if the expectation is that the system has published a set of events, then rather than asserting purely on a list of the event names or types, also look at the shape of the data and assert on the field mappings that are relevant to the test.
Instead of:

```csharp
    _events.Select(e => e.EventType).Should().BeEquivalentTo("Requested", "Accepted", "Processed");
```

Use:

```csharp
    _events.Should().BeEquivalentTo(new[] { new { EventType = "Requested", RequestorId = "req123", Amount = 150m, ... });
```

**Combining Assertions** - Prefer using `BeEquivalentTo` with an anonymous type that compares the shape of the data over comparing fields one one at a time.

## Checklist before committing

- [ ] The scenario exercises the full entrypoint, not an internal class.
- [ ] Fakes are queried for side effects (publishes, writes) rather than mocked with `Verify` calls.
- [ ] Builders clearly define what is relevant to each test and assertions clearly relate to what has been configured by the builders. Any irrelevant data is randomised (for example, by using `AutoFixture`) and not hard-coded unless it's strictly a fixed value. 
- [ ] If the project uses strategy 2 (shared singletons), the correlation ID is set on requests and the scope uses the same value.
- [ ] No `Thread.Sleep` / `Task.Delay` in assertions — if the code is async, await the deterministic completion point.
- [ ] No static shared data between scenarios - use Builders/AutoFixture for example.
- [ ] When building fake data examples, prefer to generate the values using a library like Bogus or AutoFixture over hardcoding values.
- [ ] `IDisposable` / `IAsyncLifetime` is wired so `TestScope.Dispose()` is called.
- [ ] New outbound dependencies introduced by the code change have HTTP interceptor registrations (see [maintain-add-http-interception.md](maintain-add-http-interception.md)) or a fake (see [maintain-add-fake.md](maintain-add-fake.md)).
- [ ] `dotnet test` was actually run and the SUT logs were skimmed for silently-excluded seed data — lines like "skipped", "ignored", "no matching rule", "filtered" against a fake's stored records mean the builder's defaults don't match what the production reader filters on. A green test that excludes its own seed is testing nothing. See `outside-in-testing/references/pitfalls.md` "Silently-excluded seeded history". If the test environment cannot restore packages (private NuGet feeds, offline runner), report the restore failure explicitly rather than claiming success.
- [ ] If the scenario lives in an endpoint family (`Scenarios/<Family>/`), it inherits the family's base class (`Base<Family>RequestTest` or similar). Look for one before writing setup boilerplate.
- [ ] Inbound envelopes are built through (base64/JWS/SNS-over-SQS), and outbound envelope fakes are unwrapped before asserting on the domain payload. See "If the inbound message is wrapped" above.
