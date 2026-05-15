# Example patterns

Four canonical shapes the methodology collapses to in practice. Each is described in the abstract, with the trade-offs that make it the right pick for a given service. Use them as a decision table — not as projects to clone.

The .NET scaffolder skill (`dotnet-developer-tests`) ships templates for each pattern under `templates/<pattern>/`.

## Pattern A — Lambda, per-test instance fakes

**Shape.** One Lambda function as the entrypoint. Each test gets its own `TestScope` that owns its own `new Fake*()` fields. The DI graph is rebuilt per test because the Lambda test server is cheap to start (milliseconds). No discriminator needed.

**Libraries.** xUnit v3, a Lambda test server (e.g., `MartinCostello.Testing.AwsLambdaTestServer`), an assertion library. HTTP interception (`JustEat.HttpClientInterception`) is added when the Lambda makes outbound HTTP calls — per-test, no `AsyncLocal<>` accessor needed because each test rebuilds its own DI graph. gRPC interception works the same way at the `CallInvoker` level.

**Key files.**
- `Infrastructure/TestScope.cs` — fakes as instance properties; `InvokeFunction` orchestrates the Lambda test server.
- `Infrastructure/LambdaTestServerFixture.cs` — a thin `CreateTestScope` factory.
- `When<Scenario>.cs` — single-scenario class, `[Fact]` methods assert by querying fake state.

**Reach for this when.** The service is a single Lambda with a handful of outbound dependencies. The test server is cheap, so per-test isolation is the simplest design that works.

**Scaffolder template.** `lambda/`.

## Pattern B — Lambda with a domain-specific event DSL

**Shape.** Pattern A plus a thin builder/DSL layer on top. Events are rich enough (multiple transports, nested payloads, or many variants) that inlined builder calls in each test become repetitive.

**Key additions.** A small set of types that name the domain concepts — e.g., `FlowEvent*`, `EventBuilder`, `EventSource`, `EventDestination`, `EventInstruction`. They compose into a `Given(...).Expect(...)` style that reads like the product specification.

**Reach for this when.** You already have Pattern A and the third or fourth scenario is copy-pasting the same setup prefix. Don't scaffold the DSL up front — grow into it.

**Scaffolder template.** `lambda/` as the base; add the DSL by hand once it pays its way.

## Pattern C — API, plain `WebApplicationFactory`

**Shape.** An ASP.NET API where the test drives `WebApplicationFactory<Program>` directly — no extra host abstraction. A typed HTTP client (e.g., Refit) or plain `HttpClient` talks through the in-memory TestServer.

**Libraries.** xUnit v3, `Microsoft.AspNetCore.Mvc.Testing`, optional `Refit`, optional `Testcontainers` when a real DB is cheaper than a faithful fake.

**Fakes.** Registered in `ConfigureServices` or `ConfigureTestServices`. Isolation strategy depends on how the fixture is scoped:
- per-test-instance if the factory is rebuilt per test,
- discriminator-keyed singletons if the factory is a class/assembly fixture,
- `[Collection]` serialisation as the last resort.

**Reach for this when.** The service is small, has a limited number of outbound dependencies, and you want the smallest scaffold that works. Also the right pick when Testcontainers is justified (testing migrations, transactions, or SQL behaviour).

**Scaffolder template.** `api-waf/`.

## Pattern D — API, `Xunit.DependencyInjection` + rich `TestScope`

**Shape.** An ASP.NET API with a `Startup`-style test host (Xunit.DependencyInjection convention). Shared `TestServer` per assembly. Fakes are singletons in the host DI graph, keyed by correlation ID. A `TestContext` composes default HTTP interceptor registrations with per-scenario overrides into a per-test `TestScope`.

**Libraries.** xUnit v2 (Xunit.DependencyInjection does not yet ship a v3 release), `Microsoft.AspNetCore.TestHost`, `JustEat.HttpClientInterception`, Refit, an assertion library, optionally a logging sink that captures to `ITestOutputHelper`, optionally a fake token issuer for JWT validation.

**Key files.**
- `Startup.cs` — static `ConfigureServices` and `ConfigureHost`, both discovered by convention.
- `Infrastructure/AsyncHttpClientInterceptorOptionsAccessor.cs` — `AsyncLocal<>`-backed accessor.
- `Infrastructure/HttpClientInterceptionFilter.cs` — attaches the `InterceptingHttpMessageHandler` to every `HttpClient` pipeline.
- `Infrastructure/TestContext.cs` — factory for `TestScope`.
- `Infrastructure/TestScopeOptions.cs` — fluent per-scenario overrides.

**Reach for this when.** The service has many (>20) distinct outbound dependencies, complex auth (OAuth introspection + JWT), and assertions that need to query internal state (events, cache, logs) after the fact.

**Scaffolder template.** `api-xunit-di/`.

## Pattern E — API, Alba + shared host

**Shape.** An ASP.NET API tested through `Alba`, which provides a fluent HTTP DSL (`scenario.Post.Json(x).ToUrl(...)`). Shared `AlbaHost` per assembly. Fakes are singletons in the host DI graph, keyed by correlation ID. gRPC can be intercepted alongside HTTP.

**Libraries.** xUnit v3, `Alba`, `JustEat.HttpClientInterception`, an assertion library. Custom gRPC interceptor if needed.

**Reach for this when.** You want Alba's fluent DSL over typed clients, or the service uses gRPC and you need gRPC interception alongside HTTP.

**Scaffolder template.** `api-alba/`.

## Choosing between patterns

| Dimension | A (Lambda) | B (Lambda + DSL) | C (API-WAF) | D (API-Xunit.DI) | E (API-Alba) |
|---|---|---|---|---|---|
| Entrypoint | Lambda handler | Lambda handler | HTTP endpoint | HTTP endpoint | HTTP endpoint |
| Host cost | Cheap (ms) | Cheap (ms) | Moderate | High | High |
| Default isolation | Per-test instance (1) | Per-test instance (1) | Per-test instance (1) or discriminator (2) | Discriminator (2) | Discriminator (2) |
| HTTP / gRPC interception | When Lambda calls out | When Lambda calls out | Optional | Yes | Yes |
| Typical fake count | Few (≤5) | Few–medium | Few (≤10) | Many (>20) | Medium (10–20) |
| Auth | None / fake | None / fake | API key or JWT | JWT + OAuth introspection | API key |

## Ports-and-adapters variant

Any of the patterns above can be adapted to drive the application-level port (`ICommandHandler<T>`, `IRequestHandler<T, R>`) directly instead of the transport. Valid when the transport is a thin shell and meaningful middleware isn't being exercised. Outbound dependencies still need fakes; isolation still matters. See the .NET skill's `references/ports-and-adapters.md` for the mechanics.
