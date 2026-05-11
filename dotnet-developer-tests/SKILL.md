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

1. **Project type** — Lambda / API / Generic entrypoint (ports-and-adapters).
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
- Ports-and-adapters → [references/ports-and-adapters.md](references/ports-and-adapters.md)

Substitute placeholders (`{{ServiceName}}`, `{{Namespace}}`, `{{EntrypointType}}`, etc.) when copying templates. Templates are starting points, not verbatim requirements — match the project's existing conventions when a canonical file already exists.

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
- [ ] No fakes implementing typed HTTP/gRPC client interfaces (e.g., a `FakeFxRatesApiClient : IFxRatesApiClient` where the real implementation wraps `HttpClient`). HTTP → intercept; gRPC → intercept at the `CallInvoker`. See `outside-in-testing/references/decision-rules.md` rows "HTTP" and "gRPC".
- [ ] Tests invoke the same entry point production runs. No `new EntryPoint()` / `new Function()` / `new LambdaEntryPoint()`; no Program.cs body re-implemented in the test project. For Lambda specifically, see `references/scaffold-lambda.md` "Match the production entry point".
- [ ] Scenario classes are named `When<Situation>` and live under `Scenarios/` (or root for tiny Lambda projects).
- [ ] Test data uses the builder pattern and/or AutoFixture. Prefer not to hardcode complex objects inline in the test method.
- [ ] Do not share constants between tests for correlation IDs, tenant IDs, or any discriminator value — these should be generated per test or per test instance.
- [ ] When generating fake ID strings, inspect the code for any packages that they use and conform to the format (e.g., if the service uses `Ulid`, generate ULIDs in the fake; if it uses `Guid.NewGuid()`, generate GUIDs; If it uses an internal identifier package, use that).

## Default library versions

See [references/nuget-versions.md](references/nuget-versions.md) for the current recommended NuGet set and notes on xUnit v2 vs v3.

## Default assertion library

`AwesomeAssertions 9.3.0+`. If the project already uses `FluentAssertions` or `Shouldly`, match it — don't introduce two assertion libraries in one project.

## What this skill does NOT do

- It does not guess which fakes the service needs. Inspect `src/` — every outbound client/repository/feature-flag interface is a candidate.
- It does not generate mocks (`Moq.Mock<T>`). If something looks like it needs a mock, either use a fake (dumb store) or intercept at the transport level.
- It does not add business logic to fakes. Fakes seed + read; nothing else.
