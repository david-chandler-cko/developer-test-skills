# Glossary

**Entrypoint.** The outermost code path the test drives: HTTP endpoint (`POST /payments`), Lambda handler (`FunctionHandler`), message consumer (`HandleAsync(SQSEvent)`), or generic port (`ICommandHandler<T>.Handle`).

**TestScope.** Per-test object that holds the state and fakes for one scenario. Constructed by a fixture or factory. Usually `IDisposable`. Provides methods to invoke the entrypoint and query fake state.

**TestContext.** Factory that composes defaults + per-test overrides and produces a `TestScope`. Used in the Xunit.DependencyInjection variant. Not all projects have one — simpler projects construct the TestScope directly in the fixture.

**Fixture.** xUnit construct that lives longer than a single test (class fixture, collection fixture, or assembly fixture). Used for expensive setup that can be shared (hosts, test servers, containers).

**Fake.** A dumb in-memory implementation of a real interface. Implements every method, usually with `ConcurrentDictionary<>` backing and `Task.FromResult`. Exposes test-only methods to seed (`Add*`) and query (`Get*`). No business logic.

**Mock.** A test double that records calls and lets you assert on call patterns (e.g., Moq's `Verify(x => x.Send(It.IsAny<T>()), Times.Once)`). Avoid — asserts on implementation, not behaviour.

**Stub.** A test double that returns canned answers. Fakes subsume stubs (a fake can be used as a stub by seeding + querying).

**Interceptor.** Intercepts a call at the transport level rather than replacing the client interface. `JustEat.HttpClientInterception` intercepts HTTP; gRPC interceptors do the same for gRPC. Used when the service talks to an external HTTP/gRPC API.

**Correlation ID.** A per-request identifier the service propagates through its internal calls (logs, outbound requests, events). In tests with shared-singleton fakes, the fakes use the correlation ID as the state discriminator so parallel tests don't collide.

**Discriminator.** More general term for the value a shared-singleton fake uses to partition state. Usually a correlation ID. Can be any unique, per-test, service-propagated value — tenant ID, request ID, etc.

**Port / adapter.** Hexagonal/ports-and-adapters architecture. A *port* is the application-level interface (e.g., `ICommandHandler<CreatePayment>`); an *adapter* is the transport-specific wiring (HTTP controller, Lambda handler). Tests can target the port directly when the adapter is a thin shell.

**In-memory test server.** ASP.NET `TestServer` (via `WebApplicationFactory`) or `MartinCostello.Testing.AwsLambdaTestServer`. Runs the service in-process; the test client calls via `HttpMessageHandler` without opening a real socket.

**Scenario.** A single test case, usually named `When<Situation>` with one or more `[Fact]`/`[Test]` methods. One scenario class per situation, multiple facts per class when they share setup.

**Sensor / observer.** A fake (or interceptor subscription) whose purpose is to record side effects you'll assert on afterwards: `_testScope.FakeKafkaPublisher.GetPublishedRecords()`, `_testScope.GetRequestMessages()`.
