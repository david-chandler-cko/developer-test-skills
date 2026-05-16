# Scaffold recipe: API with Xunit.DependencyInjection

Matches Pattern D in the methodology skill's `examples/example-patterns.md`. The most elaborate variant. Use when:

- The service has >20 outbound deps (many fakes).
- Complex auth (JWT + OAuth introspection).
- You want rich per-test query APIs on the `TestScope` (events, cache, logs).

Otherwise use `api-waf`.

## Assumptions

- Xunit.DependencyInjection 8.x.
- Shared `TestServer` per assembly (built once in `Startup.ConfigureHost`).
- Fakes are singletons registered in `Startup.ConfigureServices` (strategy 2 — correlation-ID keyed).
- HTTP interception via `JustEat.HttpClientInterception` + `AsyncLocal<>` accessor.
- `TestContext` factory composes default interceptor registrations + per-test overrides.
- Refit-based typed HTTP clients.

## Files to create

Under `test/{{ServiceName}}.Developer.Tests/`:

```
{{ServiceName}}.Developer.Tests.csproj
xunit.runner.json
appsettings.DeveloperTests.json
Startup.cs                            (Xunit.DI convention — static ConfigureServices + ConfigureHost)
GlobalSuppressions.cs                 (optional; xUnit1012 etc.)
Infrastructure/
  AsyncHttpClientInterceptorOptionsAccessor.cs
  HttpClientInterceptionFilter.cs
  TestContext.cs
  TestScope.cs
  TestScopeOptions.cs
  ClientCreationOptions.cs
  FakeWhatever.cs                     (one per dep)
Scenarios/
  <Area>/
    When<Scenario>.cs
```

## Steps

1. **`Startup.cs`** — Xunit.DependencyInjection discovers this by convention. Two static methods:
   - `ConfigureServices(IServiceCollection)` — registers every fake as a singleton and wires the matching real interface to it. Also disables health checks and warmup tasks.
   - `ConfigureHost(IHostBuilder)` — builds the TestServer, sets `PreserveExecutionContext = true`, layers appsettings files, registers the `TestContext` transient and the per-test `TestScope` factory.

   Copy `templates/api-xunit-di/Startup.cs.template` and fill in service-specific registrations.

2. **`AsyncHttpClientInterceptorOptionsAccessor.cs`** and **`HttpClientInterceptionFilter.cs`** — copy as-is from `templates/api-xunit-di/`. These wire `JustEat.HttpClientInterception` into the `HttpMessageHandlerBuilder` pipeline with AsyncLocal isolation.

3. **`TestScope.cs`** — holds references to all fakes, the HTTP request observer subscription, the log context (Serilog TestCorrelator if used), and the correlation ID. Provides query methods (`GetDynamoDbEvents`, `GetSnsMessages`, `GetRequestMessages`).

4. **`TestScopeOptions.cs`** — fluent builder for per-scenario overrides (register non-happy-path HTTP responses, override auth, seed specific test data). See `templates/api-xunit-di/TestScopeOptions.cs.template` for the canonical shape and grow it with one method per lever the service's scenarios need.

5. **`TestContext.cs`** — `CreateScope(Action<TestScopeOptions>? customize, ..., string? correlationId)` method that:
   - Generates a correlation ID (or uses supplied).
   - Instantiates a `TestScopeOptions`, applies the customizer.
   - Composes default HTTP interceptor registrations + per-test overrides into `ConfigureInterceptor`.
   - Sets `ThrowOnMissingRegistration = true`.
   - Calls the `TestScope` factory to produce the per-test scope.

6. **Fakes** — all registered as `AddSingleton` in `Startup.ConfigureServices`. They key state by correlation ID (see the methodology skill's decision-rules for the pattern).

7. **First scenario** — implements `IAsyncLifetime`; constructor takes `TestContext`; `InitializeAsync` creates scope, sets up interceptors, invokes entrypoint; `[Fact]` methods assert on `_result` and on `_testScope.Get*` queries.

## Gotchas

- **Inherit production's composition root, don't re-implement it.** In `Startup.ConfigureHost`, call `webHostBuilder.UseStartup<TProductionStartup>()` and then layer test overrides via `ConfigureTestServices(...)`. Tests should *re-register fakes against the same interfaces production registered* (last-registration-wins). A `TestStartup : ProductionStartup` subclass or a hand-rolled DI composition diverges silently the moment production adds a new dependency.
- **Strip production polling pumps.** If the production host registers `IHostedService`s that poll real queues, real timers, or external endpoints, remove them before the test host boots: `services.RemoveAll<IHostedService>()` if every hosted service is a network poller, or surgical removal by type if not. The test invokes handlers directly via DI; the production pump fighting the test's enqueue is a common cause of hang-then-timeout failures.
- **Xunit.DI discovery**: `Startup` must be a top-level class with exactly the right method signatures. Typos fail with cryptic "no suitable constructor" errors.
- **`PreserveExecutionContext = true`**: without this, the `AsyncLocal<>` accessor silently doesn't flow. Every request sees `Current == null`.
- **Health checks / warmup tasks**: remove them in `ConfigureServices` (see template).
- **Refit client factory**: register a `Func<ClientCreationOptions, I<Client>>` in `ConfigureTestServices` that calls `testServer.CreateHandler()` to route requests in-process.
- **JWT auth**: if the service validates JWTs, register a fake issuer + key:
  ```csharp
  services.Configure<JwtBearerOptions>(JwtBearerDefaults.AuthenticationScheme, opts =>
  {
      var cfg = new OpenIdConnectConfiguration { Issuer = FakeTokenIssuer.Issuer };
      cfg.SigningKeys.Add(FakeTokenIssuer.SecurityKey);
      opts.Authority = null;
      opts.Configuration = cfg;
  });
  ```
