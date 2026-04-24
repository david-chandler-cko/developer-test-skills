# Scaffold recipe: API with plain WebApplicationFactory

Matches Pattern C in the methodology skill's `examples/example-patterns.md` (`WebApplicationFactory<Program>`, plain `HttpClient` from tests, Testcontainers where real DBs are needed, collection fixtures when strategy 3 is required).

Recommended default for new API services. Smallest surface area, fewest moving parts.

## Assumptions

- The API uses the minimal hosting model: `Program.cs` builds a `WebApplication`, i.e., there's a `public partial class Program { }` (or equivalent) that `WebApplicationFactory<Program>` can reference.
- Tests call the API via plain `HttpClient` from `WebApplicationFactory.CreateClient()`. If the service already exposes a Refit (or similar typed) interface, reuse it via `RefitSettings.HttpMessageHandlerFactory = server.CreateHandler` and uncomment the Refit `<PackageReference>` lines in the csproj template.
- Default isolation is **strategy 1** (per-test fake instances): each test instantiates its own `ApiFixture`, which builds its own `WebApplicationFactory<Program>`. The templates ship with this shape. Strategy 2 (discriminator-keyed singletons) and strategy 3 (`[Collection]`) are wired below.

## Version-matching rule

`Microsoft.AspNetCore.Mvc.Testing` MUST match the target framework's ASP.NET version:

| `<TargetFramework>` | `Microsoft.AspNetCore.Mvc.Testing` Version |
|---|---|
| `net10.0` | `10.0.0` (or latest 10.0.x) |
| `net9.0` | `9.0.x` |
| `net8.0` | `8.0.x` |

A mismatch produces silent ref-pack failures at runtime. The template uses `{{AspNetCoreTestingVersion}}` — substitute the matching version when copying.

## Files to create

Under `test/{{ServiceName}}.Tests/`:

```
{{ServiceName}}.Tests.csproj
xunit.runner.json
appsettings.DeveloperTests.json       (if config overrides are needed)
Infrastructure/
  ApiFixture.cs                       (base class; virtual ConfigureServices/ConfigureTestServices)
  TestScope.cs                        (optional thin wrapper over the fixture's services)
  Fakes/
    FakeWhatever.cs
Scenarios/
  When<Scenario>.cs
```

## Steps

1. **Add `Program.cs` visibility in the API project**:
   ```csharp
   // At the end of Program.cs in src/
   public partial class Program;
   ```
   Required so `WebApplicationFactory<Program>` can find it.

2. **Copy `templates/api-waf/ApiFixture.cs.template`** to `Infrastructure/ApiFixture.cs`. Subclass it per feature area if you need different fake wiring per group of tests (`ConfigurationApiFixture : ApiFixture`). Override `ConfigureServices` and `ConfigureTestServices` to register fakes / replace real services.

3. **Identify outbound dependencies** and create fakes in `Infrastructure/Fakes/` using `templates/api-waf/FakeRepository.cs.template`.

4. **Wire fakes** in `ApiFixture.ConfigureTestServices` or a subclass:
   ```csharp
   services.RemoveAll<IRealRepository>();
   services.AddSingleton<IRealRepository, FakeRealRepository>();
   ```
   Registration lifetime depends on isolation strategy — see below.

5. **Copy `templates/api-waf/TestProject.csproj.template`** and add a `ProjectReference` to the API source project.

6. **Copy `templates/api-waf/appsettings.DeveloperTests.json.template`** and tune for the service. `.csproj` must copy it to output (`<CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>`).

7. **Create a first scenario** from `templates/api-waf/WhenScenario.cs.template`.

## Isolation strategy wiring

### Strategy 1 — per-test fake instances (template default)
Each test instantiates `new ApiFixture()` in its own `InitializeAsync`. The fixture's constructor builds the `WebApplicationFactory`; `DisposeAsync` tears it down. Fakes are fields on the fixture and live exactly one test long. This is what `WhenScenario.cs.template` ships with.

### Strategy 2 — shared singletons keyed by discriminator
Promote `ApiFixture` to a class- or assembly-level fixture (`IClassFixture<ApiFixture>` or `ICollectionFixture<ApiFixture>`). Register fakes as `AddSingleton`. Each test generates a `Guid.NewGuid()` correlation ID; every request carries it in a header; middleware places it on `HttpContext.Items["correlation-id"]` or `Activity.Current`; fakes partition state by it.

### Strategy 3 — `[Collection]` serialisation
Wrap the fixture with a collection definition:
```csharp
[CollectionDefinition(SharedApi.Name)]
public class SharedApi : ICollectionFixture<ApiFixture> { public const string Name = "shared-api"; }

[Collection(SharedApi.Name)]
public class WhenSomething : IAsyncLifetime { ... }
```
All tests in the collection run serially.

## Testcontainers (real-DB pattern)

If you want a real dependency instead of a fake — e.g., a real DynamoDB via Testcontainers:

```csharp
protected override void ConfigureServices(IServiceCollection services)
{
    var container = new DynamoDbBuilder().Build();
    container.StartAsync().GetAwaiter().GetResult();
    services.AddSingleton<IAmazonDynamoDB>(new AmazonDynamoDBClient(
        new AmazonDynamoDBConfig { ServiceURL = container.GetConnectionString() }));
}
```
Remember to `StopAsync` and `DisposeAsync` in the fixture's `DisposeAsync`.

## Gotchas

- `WebApplicationFactory<Program>` calls the entire `Program.cs` pipeline, including anything that connects on startup (health checks, warmup tasks). Disable those in `ConfigureTestServices`.
- `services.RemoveAll<T>()` only works on `IServiceCollection` — if the original registration uses keyed services or a factory, you might need to `Replace` instead.
- Refit's `RefitSettings.HttpMessageHandlerFactory = server.CreateHandler` is how you wire it to the TestServer.
