# Maintain: add a new scenario

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
        published.Should().HaveCount(1);
    }

    public void Dispose() => _testScope.Dispose();
}
```

## Pattern

1. **Arrange**: seed fakes, configure HTTP interceptor overrides, build the request.
2. **Act**: invoke the entrypoint via the TestScope (`_testScope.Scenario(...)`, `_testScope.InvokeFunction(...)`, `apiClient.DoX(...)`).
3. **Assert**: check the response, query the fakes, check the recorded HTTP requests.

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

## Checklist before committing

- [ ] The scenario exercises the full entrypoint, not an internal class.
- [ ] Fakes are queried for side effects (publishes, writes) rather than mocked with `Verify` calls.
- [ ] If the project uses strategy 2 (shared singletons), the correlation ID is set on requests and the scope uses the same value.
- [ ] No `Thread.Sleep` / `Task.Delay` in assertions — if the code is async, await the deterministic completion point.
- [ ] No static shared data between scenarios - use Builders/AutoFixture for example.
- [ ] When building fake data examples, prefer to generate the values using a library like Bogus or AutoFixture over hardcoding values.
- [ ] `IDisposable` / `IAsyncLifetime` is wired so `TestScope.Dispose()` is called.
- [ ] New outbound dependencies introduced by the code change have HTTP interceptor registrations (see [maintain-add-http-interception.md](maintain-add-http-interception.md)) or a fake (see [maintain-add-fake.md](maintain-add-fake.md)).
