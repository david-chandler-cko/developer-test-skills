---
name: outside-in-testing
description: Use when writing, reviewing, or scaffolding developer tests following the outside-in black-box methodology. Covers principles, decision rules, and pitfalls. TRIGGER on "developer tests", "outside-in", "black box test", "test this service end-to-end", "add a test scenario", "mock this dependency for tests", or when editing files under test/*Developer.Tests/ or test/**/Infrastructure/. Language-agnostic; for .NET code generation call the dotnet-developer-tests skill.
---

# Outside-in black-box developer testing

## What this is

A scenario-level test that exercises one service end-to-end, driving it through its real entrypoint (HTTP endpoint, Lambda handler, message consumer, or generic port in a ports-and-adapters design). In-process dependencies run for real. Everything that would cross the network in production is replaced by a **fake** (for outbound clients, repositories, feature flags) or **intercepted** (for HTTP, gRPC). Scenarios are isolated so the suite can run in parallel.

This is not integration testing (no real external systems), not unit testing (no isolation of internal classes), not contract testing (no mocks that verify call shape). It's the layer where you say "given these inputs at the edge, this is what the service does".

## Four non-negotiables

1. **Only the network boundary is mocked.** HTTP, gRPC, queues, DBs, feature-flag services, token issuers, AWS SDKs — these get fakes or interceptors. Anything that would run in-process in production runs for real in the test.
2. **In-process dependencies are never mocked.** No `Mock<IInternalService>`. If internal wiring breaks, your tests break — that's what you want.
3. **Every scenario is isolated from every other running in parallel.** *How* you isolate is a design choice (see [decision-rules](references/decision-rules.md) §Isolation strategies) — not a fixed rule.
4. **Prefer fakes over mocks.** A fake is a dumb in-memory implementation of a real interface (e.g., a `ConcurrentDictionary`-backed store that satisfies `IProcessorRepository`). You assert on its state, not on which methods got called. Mocks that verify call patterns are brittle and leak implementation details; use them only when there's no observable state to check.

## Decision tree: real vs. fake vs. intercept

```
Is this dependency in-process?
├── Yes → Use it for real. Do not mock. Do not replace.
└── No (crosses network)
    ├── HTTP?     → Intercept with httpclient-interception (or language equivalent)
    ├── gRPC?     → gRPC interceptor
    ├── AWS SDK?  → Fake the client interface (IAmazonS3, IAmazonSQS, ISnsPublisher, ...)
    ├── DB client? → Fake the repository interface; OR use Testcontainers for a real DB
    ├── Feature flags? → Fake the client (FakeLdClient, FakeFeatureManager, ...)
    └── Auth/token issuer? → Fake issuer + register a signing key the service trusts
```

See [references/decision-rules.md](references/decision-rules.md) for expansion, including "when is Testcontainers the right answer".

## What to test, what not to test

- **Test entrypoints**: HTTP endpoint → response + side effects. Lambda handler → response + side effects. Message consumer → side effects. Generic entrypoint (ports-and-adapters) → command/query → side effects.
- **Don't test internal classes in isolation.** If they matter, they're reached via an entrypoint. If they're not reached, they're dead code.
- **Assert on observable state**, not on internal calls. Query the fakes (`_testScope.FakeKafkaPublisher.GetPublishedRecords()`), the response, the HTTP interceptor's recorded requests, emitted logs. Don't assert that method X was invoked.

## Fixture scope: shared vs. per-test

- **Expensive to start** (AlbaHost, TestServer with full DI graph, Testcontainers database) → share at fixture scope (`IClassFixture<T>` or assembly-level), recreate test-scoped state inside each test.
- **Cheap to start** (Lambda test server, in-memory stub) → fresh per test. Simpler and more obviously correct.

## Isolation strategies (pick one)

In order of preference:

1. **Per-test fake instances.** Each TestScope creates `new Fake*()` instances; nothing is shared. No discriminator needed. **Default** for Lambdas and small APIs.
2. **Shared singleton fakes keyed by a per-test discriminator** (correlation ID, request ID, tenant ID, or any unique value the service already propagates). Required when fakes must live inside the host's DI container and the host is shared. Fakes partition their state by the discriminator.
3. **Shared singleton fakes + xUnit `[Collection]` serialisation**. Tests in the same collection run serially. Use only when 1 and 2 don't work — it slows the suite down.

If no natural discriminator exists AND fakes must be shared, fall back to 3. Never skip isolation.

Full treatment in [references/decision-rules.md](references/decision-rules.md).

## Ports-and-adapters: drive the port, not the transport

If the service uses ports-and-adapters (e.g., `ICommandHandler<T>`, `IQueryHandler<T>`), you may drive tests at the port and skip the HTTP/Lambda transport. Valid when the transport is a thin shell. Outbound deps still need fakes; isolation still matters. Details in the .NET skill's `references/ports-and-adapters.md`.

## Pitfalls (summary)

See [references/pitfalls.md](references/pitfalls.md) for symptoms and fixes. Headlines:

- Fake singleton pollution (shared-singleton strategy without a discriminator)
- HTTP interceptor subscriptions not disposed → tests bleed into each other
- Service reads correlation ID from one place, test writes it to another → fakes partition correctly but nothing matches
- Static state (global clocks, feature flags, singleton caches) survives between tests
- `TestServer.PreserveExecutionContext` not set → `AsyncLocal<>` isolation breaks silently
- `ThrowOnMissingRegistration` off → an unmocked external call silently hits the real URL in CI
- Debugger-attached infinite timeouts leak into CI
- Health checks / warmup tasks run during the test and poison fakes
- appsettings.*.json not copied to `bin/` → config looks right in source, wrong at runtime
- IAsyncLifetime ordering surprises (fixture init → test init → test → test dispose → fixture dispose)

## Vocabulary

Short definitions in [references/glossary.md](references/glossary.md): TestScope, TestContext, fixture, fake, mock, stub, interceptor, correlation ID, entrypoint, port/adapter, in-memory test server.

## Example patterns

See [examples/example-patterns.md](examples/example-patterns.md) for the canonical shapes this methodology collapses to in practice (Lambda, API variants, ports-and-adapters) and when to reach for each.

## Code generation

This skill does not generate code. For .NET, invoke the `dotnet-developer-tests` skill. For other languages, adapt the principles here by hand.
