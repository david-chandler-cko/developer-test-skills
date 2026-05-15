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
4. **Prefer fakes over mocks.** A fake is a dumb in-memory implementation of a real interface (e.g., a `ConcurrentDictionary`-backed store that satisfies `IOrderRepository`). You assert on its state, not on which methods got called. Mocks that verify call patterns are brittle and leak implementation details; use them only when there's no observable state to check.

## When project precedent contradicts the methodology

Some existing test projects will violate this skill — usually because they predate it. **An existing wrong pattern is not licence to repeat it.** Apply the methodology in new code regardless of what neighbours do. If the cleanup is small (e.g., replacing a `Fake*ApiClient` with an HTTP interceptor registration), pull it into the same change. If large, flag the deviation in the PR and track a follow-up ticket — don't silently match the wrong shape.

Common drift shapes:

- A fake implementing a typed HTTP/gRPC client interface (`Fake*ApiClient : I*ApiClient` whose real implementation wraps `HttpClient` or a generated gRPC stub).
- A `Mock<I*Service>` where the interface is an in-process collaborator.
- A test project that re-implements the SUT's DI composition root.
- A shared singleton fake with no discriminator and no `[Collection]` serialisation.
- Tests that construct the handler class directly (`new EntryPoint()`, `new Function()`) instead of driving the production entry point.

## Decision tree: real vs. fake vs. intercept

```
Is this dependency in-process?
├── Yes → Use it for real. Do not mock. Do not replace.
└── No (crosses network)
    ├── HTTP?     → Intercept with httpclient-interception (or language equivalent)
    ├── gRPC?     → gRPC interceptor
    ├── AWS SDK?  → Fake the client interface (IAmazonS3, IAmazonSQS, ...)
    ├── DB client? → Fake the repository interface; OR use Testcontainers for a real DB
    ├── Feature flags? → Fake the feature-flag client
    └── Auth/token issuer? → Fake issuer + register a signing key the service trusts
```

See [references/decision-rules.md](references/decision-rules.md) for expansion, including "when is Testcontainers the right answer".

## What to test, what not to test

- **Test entrypoints**: HTTP endpoint → response + side effects. Lambda handler → response + side effects. Message consumer → side effects. Generic entrypoint (ports-and-adapters) → command/query → side effects.
- **Don't test internal classes in isolation.** If they matter, they're reached via an entrypoint. If they're not reached, they're dead code.
- **Assert on observable state**, not on internal calls. Query the fakes (`_testScope.FakeEventPublisher.GetPublishedRecords()`), the response, the HTTP interceptor's recorded requests, emitted logs. Don't assert that method X was invoked.
- **Don't assert on fake call counts.** `fake.CallCount.Should().Be(1)` couples the test to the handler's implementation (caching, batching, retries are all free to change the count without changing behaviour). Assert on the response or the data the fake stored — those are the contract. See `references/pitfalls.md` "Asserting on fake call counts".

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

## References

- [references/pitfalls.md](references/pitfalls.md) — symptoms and fixes for the common ways outside-in tests go wrong (singleton pollution, undisposed interceptors, discriminator gaps, `PreserveExecutionContext`, warmup tasks, etc.).
- [references/glossary.md](references/glossary.md) — short definitions: TestScope, fixture, fake, mock, stub, interceptor, discriminator, port/adapter.
- [examples/example-patterns.md](examples/example-patterns.md) — canonical shapes the methodology collapses to (Lambda, API variants, ports-and-adapters) and when to reach for each.

This skill does not generate code. For .NET, invoke the `dotnet-developer-tests` skill. For other languages, adapt the principles here by hand.
