# Scaffold recipe: API with Alba

Matches Pattern E in the methodology skill's `examples/example-patterns.md`. Use when:

- You want a fluent HTTP DSL (`scenario.Post.Json(x).ToUrl(...)`) over typed Refit clients.
- The service uses gRPC and you need gRPC interception alongside HTTP.
- API key auth (not JWT) keeps auth setup simple.

## Assumptions

- `Alba 8.x`.
- Shared `AlbaHost` per assembly (via `IAsyncLifetime` fixture).
- Fakes are singletons registered in the `AlbaHost.For<Program>(builder => ...)` call (strategy 2 — correlation-ID keyed).
- HTTP interception via `JustEat.HttpClientInterception` + `AsyncLocal<>` accessor (same pattern as Xunit.DI variant).
- API-key auth via `scenario.WithApiKey(...)` extension.

## Files to create

Under `test/{{ServiceName}}.Developer.Tests/`:

```
{{ServiceName}}.Developer.Tests.csproj
xunit.runner.json
Infrastructure/
  ApiFixture.cs                       (IAsyncLifetime; AlbaHost.For<Program>)
  TestScope.cs                        (per-test; IDisposable; .Scenario() entrypoint)
  HttpClientInterceptorOptionsAccessor.cs
  HttpClientInterceptionFilter.cs
  Extensions/
    ScenarioExtensions.cs             (WithApiKey, WithCorrelationId, WithApiVersion, domain-specific HTTP verbs)
  Fakes/
    FakeWhatever.cs
Scenarios/
  <Area>/
    When<Scenario>.cs
```

## Steps

1. **`ApiFixture.cs`** — implements `IAsyncLifetime`. In `InitializeAsync`:
   ```csharp
   _system = await AlbaHost.For<Program>(builder =>
   {
       builder.ConfigureServices(services =>
       {
           // Replace real with fakes
           services.AddSingleton(_fakeThingRepository);
           services.AddSingleton<IThingRepository>(sp => sp.GetRequiredService<FakeThingRepository>());
           // ... etc
           services.AddSingleton(new HttpClientInterceptorOptionsAccessor());
           services.AddSingleton<IHttpMessageHandlerBuilderFilter, HttpClientInterceptionFilter>();
       });
       builder.ConfigureTestServices(services =>
       {
           services.AddLogging(l => l.AddXUnit());
           services.AddTransient<TestDataBuilder>();
       });
       builder.UseSetting("Authentication:ApiKey", _apiKey);
   });
   _system.Server.PreserveExecutionContext = true;
   ```

2. **`TestScope.cs`** — per-test instance. Constructor takes `ITestOutputHelper`, the API key, `IAlbaHost`, and any builders/fakes from the fixture. Sets a fresh correlation ID, configures the interceptor accessor, subscribes to `HttpRequestObserver`.

3. **`ScenarioExtensions.cs`** — extension methods that encapsulate common request shaping:
   ```csharp
   public static Scenario WithApiKey(this Scenario scenario, string apiKey) => scenario.WithRequestHeader("Authorization", apiKey);
   public static Scenario WithCorrelationId(this Scenario scenario, string id) => scenario.WithRequestHeader("X-Correlation-Id", id);
   public static Scenario SendCreatePayment(this Scenario scenario, CreatePaymentRequest req) => scenario.Post.Json(req).ToUrl("/payments");
   ```

4. **Fakes** — singletons in the `AlbaHost` DI graph. Keyed by correlation ID (strategy 2).

5. **First scenario** — `IClassFixture<ApiFixture>, IAsyncLifetime`; constructor calls `fixture.CreateTestScope(testOutputHelper)`; `InitializeAsync` or `[Fact]` body calls `_testScope.Scenario(s => s.SendX(...))`.

## gRPC interception

If the service talks gRPC, add a gRPC interceptor alongside the HTTP one. The gRPC equivalent of `HttpClientInterceptorOptions` is your own interceptor type — a small class that matches gRPC method descriptors against registered stubs and throws on unknown calls, mirroring the `ThrowOnMissingRegistration` behaviour of the HTTP interceptor.

## Gotchas

- `_system.Server.PreserveExecutionContext = true` — required for AsyncLocal-based interceptor accessor.
- Alba's `Scenario` is rebuilt per request; your extensions must chain correctly (return `Scenario` from each).
- `builder.ConfigureServices` runs against the real service DI; `builder.ConfigureTestServices` runs after and overrides. Put fakes in ConfigureServices (so the real registrations are replaced) or use `RemoveAll` + `Add` explicitly.
