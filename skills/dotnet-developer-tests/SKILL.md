---
name: dotnet-developer-tests
description: Use to scaffold a new .NET test project OR add/modify tests, fakes, and HTTP interceptors in an existing one, following the outside-in black-box methodology. TRIGGER on "scaffold tests for this service", "add a Developer.Tests project", "add a When*.cs scenario", "add a fake for this repository", "set up httpclient-interception", or edits under test/*.Developer.Tests/Infrastructure/ and test/*.Developer.Tests/Scenarios/ in a .NET solution. Defers to the outside-in-testing skill for methodology rationale.
---

# .NET developer-tests scaffolder

Before generating code, read the methodology skill (`outside-in-testing`) at least once in this session. It defines the non-negotiables this skill assumes.

## Entry branch

Look at the repo:

- Is there a `test/*.Developer.Tests/` or `test/*.Tests/` project with an `Infrastructure/` folder containing a `TestScope.cs` (or equivalent)? There may be a Developer.Tests project per service, or a single shared Tests project with folders per service.
  - **Yes** → [Maintain flow](#maintain-flow).
  - **No** → [Scaffold flow](#scaffold-flow).

If unsure, grep:
```sh
find test -name 'TestScope.cs' 2>/dev/null
```

## Scaffold flow

Use `AskUserQuestion` to collect:

1. **Project type** — Lambda / API / Queue-consumer (long-running SQS/Kafka/Service-Bus host) / Generic entrypoint (ports-and-adapters).
2. **If API, host framework**:
   - **Plain WebApplicationFactory** (recommended default for new services).
   - **Xunit.DependencyInjection** — reach for this when the service has >20 fakes or very complex DI.
   - **Alba** — reach for this when you want a fluent HTTP DSL or need gRPC interception alongside HTTP.
3. **Isolation strategy**:
   - **Per-test fake instances** — default if feasible (Lambda, small APIs). If an API is stateless (e.g., GET only), then we can use shared singletons without a discriminator. Otherwise, per-test instances are simpler and more obviously correct.
   - **Shared singletons keyed by discriminator** — required if the host is shared and fakes must live inside its DI. Follow up: *what is the discriminator?* (correlation ID header, request ID, tenant ID, etc. — pick a value the service already propagates).
   - **Shared singletons + `[Collection]` serialisation** — fallback when no discriminator is available.
4. **Service name + solution path** — auto-detect from `*.sln` / `src/*/`; confirm with the user.
5. **Target framework** — match `src/*.csproj`; default `net10.0` if greenfield.

Then follow the variant-specific recipe:

- Lambda → [references/scaffold-lambda.md](references/scaffold-lambda.md) + `templates/lambda/`
- API (WAF) → [references/scaffold-api-webapplicationfactory.md](references/scaffold-api-webapplicationfactory.md) + `templates/api-waf/`
- API (Xunit.DI) → [references/scaffold-api-xunit-di.md](references/scaffold-api-xunit-di.md) + `templates/api-xunit-di/`
- API (Alba) → [references/scaffold-api-alba.md](references/scaffold-api-alba.md) + `templates/api-alba/`
- Queue-consumer (SQS/Kafka/Service Bus, hosted in IHost) → [references/scaffold-queue-consumer.md](references/scaffold-queue-consumer.md) (layers on top of Xunit.DI recipe)
- Ports-and-adapters → [references/ports-and-adapters.md](references/ports-and-adapters.md)

Substitute placeholders (`{{ServiceName}}`, `{{Namespace}}`, `{{EntrypointType}}`, etc.) when copying templates. Templates are starting points, not verbatim requirements — match the project's existing conventions when a canonical file already exists.

**Preflight before copying any template — match the project's existing baseline:**

1. **Target framework** — read `src/*.csproj` `<TargetFramework>` and any `Directory.Build.props` `<TargetFramework>`. Override the template's `net10.0` default if the project ships on an older framework. A test project that doesn't match the production framework can't even reference the production assembly.
2. **xUnit version** — if the project uses xUnit v2 (look for `xunit` ≥2.4 in any existing test project's `.csproj`, or `IClassFixture<>` / `Xunit.Sdk` v2 idioms), keep v2 — don't introduce a v3 test project alongside a v2 one. The two have incompatible runners and assembly-level attributes.
3. **Assertion library** — `Directory.Build.props` often pins one (`FluentAssertions`, `AwesomeAssertions`, `Shouldly`). Match it. Two assertion libraries in one solution is a smell and a frequent source of "which `Should()` extension?" confusion.
4. **Logging sink** — does the project route logs through `ITestOutputHelper` via `MartinCostello.Logging.XUnit` or similar? If so, the TestScope should accept and use the same.
5. **Sibling test project** — if `test/<OtherService>.Developer.Tests/` already exists in the solution, *read its `Infrastructure/TestScope.cs` and its `.csproj` first*. Other teams in the same solution have already made these choices; the new project should match unless there's a concrete reason not to. Cribbing from a sibling is faster and more consistent than re-deriving from templates.

## Maintain flow

Route by intent:

| User wants to… | Follow |
|---|---|
| Add a new test scenario | [references/maintain-add-scenario.md](references/maintain-add-scenario.md) |
| Add a fake for a new dependency | [references/maintain-add-fake.md](references/maintain-add-fake.md) |
| Register an HTTP interception for a new outbound call | [references/maintain-add-http-interception.md](references/maintain-add-http-interception.md) |
| Audit / bring a project into conformance | [Conformance checklist](#conformance-checklist) |

## Conformance checklist

Run through these when asked to review a project, or when adding to one that looks drifted.

- [ ] `xunit.runner.json` sets `"parallelizeAssembly": true` (unless the project deliberately uses `[Collection]` for serialisation).
- [ ] Fakes use one of the three isolation strategies (see methodology skill). Flag shared singletons with no discriminator and no collection serialisation.
- [ ] `TestScope` is `IDisposable` and disposes HTTP interceptor subscriptions in `Dispose()`.
- [ ] `appsettings.*.json` files are copied to output (check `.csproj` `<Content CopyToOutputDirectory="PreserveNewest" />`).
- [ ] `TestServer.PreserveExecutionContext = true` (when `AsyncLocal<>` is used for the interceptor accessor).
- [ ] `interceptorOptions.ThrowOnMissingRegistration = true` in the central interceptor setup.
- [ ] Health checks and warmup tasks are disabled in the test host.
- [ ] No `Mock<I...>` usages for in-process dependencies.
- [ ] No fakes implementing typed HTTP/gRPC client interfaces (e.g., a `FakeBillingApiClient : IBillingApiClient` where the real implementation wraps `HttpClient`). HTTP → intercept; gRPC → intercept at the `CallInvoker`. See `outside-in-testing/references/decision-rules.md` rows "HTTP" and "gRPC".
- [ ] Tests invoke the same entry point production runs. No `new EntryPoint()` / `new Function()` / `new LambdaEntryPoint()`; no Program.cs body re-implemented in the test project. For Lambda specifically, see `references/scaffold-lambda.md` "Match the production entry point".
- [ ] Scenario classes are named `When<Situation>` and live under `Scenarios/` (or root for tiny Lambda projects).
- [ ] Test data uses the builder pattern and/or AutoFixture. Prefer not to hardcode complex objects inline in the test method. Any non-trivial domain object referenced by ≥2 tests has its own builder (see `references/maintain-add-scenario.md` "Builders: variant factories, derived builders, AutoFixture defaults"). For request shapes with discrete variants, prefer static variant factories (`Builder.Variant1()`) over flag-style setters.
- [ ] Historical state is seeded via direct fake `Seed(...)` calls, not by round-tripping through a sibling handler — unless the sibling handler's write shape is itself under test. See `references/maintain-add-scenario.md` "Seeding test data: direct fake vs. round-trip".
- [ ] Do not share constants between tests for correlation IDs, tenant IDs, or any discriminator value — these should be generated per test or per test instance.
- [ ] Fake-generated ID strings match the format the service uses in production (ULID, GUID, internal identifier package). Inspect `src/` for the existing pattern before inventing one.
- [ ] CQRS interface pairs share a single fake. When production splits `IQuery<Thing>` / `IStore<Thing>` (or `IRead<Thing>` / `IWrite<Thing>`), one `Fake<Thing>Repository` implements both interfaces and is registered as a single shared instance against both. See `references/maintain-add-fake.md` "Pair CQRS interfaces".
- [ ] Tests don't assert on fake call counters (`fake.CallCount`, `fake.QueryCount`, etc.). Assert on the response and the data the fake stored — those are the contract. See `outside-in-testing/references/pitfalls.md` "Asserting on fake call counts".
- [ ] When production code under test contains a static rule table (`Dictionary<>`, immutable rule list, `switch` on a composite key), tests cover **each distinguishable row** via `[Theory]`/`[Fact]`, not just a happy-path. See `outside-in-testing/references/decision-rules.md` "Production configuration tables are a test-case source".
- [ ] Large AWS SDK interfaces (`IAmazonDynamoDB`, `IAmazonSimpleNotificationService`, etc.) aren't hand-faked end-to-end. Either fake only the methods production calls (rest throw `NotImplementedException`) or wrap the SDK behind a project-specific port in `src/`. See `references/scaffold-lambda.md` "Variant: AWS SDK clients".
- [ ] AWS SDK clients constructed via a factory/builder interface (`IFooClientBuilder.Build()`) are faked at the factory layer, not by reaching past it. See `references/scaffold-lambda.md` "AWS SDK clients hidden behind a factory or builder".
- [ ] TestScope's `IConfiguration` is built from production's `appsettings.json` with in-memory overrides for the keys assertions reference. No `appsettings.Tests.json` unless the diverging block is large. See `references/scaffold-lambda.md` "Configuration: how the TestScope wires `IConfiguration`".
- [ ] The test composition root *inherits* the production composition root (`UseStartup<TProductionStartup>()` + `ConfigureTestServices(...)` for API; production `ServiceRegistry.BuildServiceProvider(config, configureServices)` for Lambda) rather than re-implementing it. Last-registration-wins gives test fakes priority over the production registrations they shadow.
- [ ] Queue-consumer / long-running hosts have `services.RemoveAll<IHostedService>()` (or surgical poller removal) in the test composition root. Otherwise the production polling pump fights the test's direct handler dispatch. See `references/scaffold-queue-consumer.md`.

## Default library versions

See [references/nuget-versions.md](references/nuget-versions.md) for the current recommended NuGet set and notes on xUnit v2 vs v3.

## Default assertion library

`AwesomeAssertions 9.3.0+`. If the project already uses `FluentAssertions` or `Shouldly`, match it — don't introduce two assertion libraries in one project.

## What this skill does NOT do

- It does not guess which fakes the service needs. Inspect `src/` — every outbound client/repository/feature-flag interface is a candidate.
- It does not generate mocks (`Moq.Mock<T>`). If something looks like it needs a mock, either use a fake (dumb store) or intercept at the transport level.
- It does not add business logic to fakes. Fakes seed + read; nothing else.
