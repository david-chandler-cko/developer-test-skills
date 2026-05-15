# Scaffold recipe: .NET Lambda test project

Matches Pattern A in the methodology skill's `examples/example-patterns.md` (xUnit v3, an assertion library, per-test fake instances). If the Lambda makes outbound HTTP or gRPC calls, follow the [outbound HTTP / gRPC variant](#variant-outbound-http--grpc) below in addition to the base steps — Lambdas frequently do, and the methodology mandates intercepting at the transport, not faking a wrapper interface.

## Assumptions

- The Lambda exposes a **public** static `RunAsync(HttpClient? testServerClient = null, Action<IHostBuilder>? modifyHostBuilder = null, Action<IServiceCollection>? modifyServices = null, CancellationToken cancellationToken = default)` method that production also calls — the same composition root, no second wiring path. If the service uses a different hosting shape (top-level `Program.cs`, `Amazon.Lambda.Annotations`, plain `FunctionHandler` class), or if production currently has only a `Main` method, follow [Match the production entry point](#match-the-production-entry-point) before continuing — **adding this overload to production is part of the scaffold, not optional.**
- Fakes are per-test-instance (strategy 1). Each test gets its own `TestScope`, which owns its own `new Fake*()` fields. No discriminator, no shared state.
- `AwesomeAssertions` for assertions.

## Match the production entry point

**Principle.** The test must invoke the same code path Lambda runs in production. Constructing the handler class directly (`new EntryPoint()`, `new LambdaEntryPoint()`, `new Function()`) bypasses the bootstrap, the DI composition root, the serializer registration, and any startup code that runs before the first invocation. When the test path diverges from the production path you get two wirings to keep in sync — and tests will pass while production breaks (or vice versa) the moment they drift.

The fix: have **one** composition root that production and tests both call. Below, "EntryPoint" is whatever class the service uses to do the wiring.

### Canonical shape — class with public static `RunAsync` (template default)

The same `RunAsync` is called from production (via `Main`) and from the test (via `TestScope.InvokeFunction`). Every parameter is optional with a default so `Main` can call it argument-less, and tests pick exactly the seams they need.

```csharp
public class EntryPoint // or Program — whichever name your service already uses
{
    public static async Task RunAsync(
        HttpClient? testServerClient = null,
        Action<IHostBuilder>? modifyHostBuilder = null,
        Action<IServiceCollection>? modifyServices = null,
        CancellationToken cancellationToken = default)
    {
        var hostBuilder = Host.CreateDefaultBuilder()
            .ConfigureServices((ctx, services) =>
            {
                ConfigureServices(ctx, services);
                modifyServices?.Invoke(services);
            });

        modifyHostBuilder?.Invoke(hostBuilder);

        using var host = hostBuilder.Build();

        var handlerWrapper = HandlerWrapper.GetHandlerWrapper<TEvent, TResponse>(
            host.Services.GetRequiredService<MyFunctionHandler>().Handle,
            new DefaultLambdaJsonSerializer());

        var bootstrap = testServerClient is null
            ? LambdaBootstrapBuilder.Create(handlerWrapper).Build()
            : LambdaBootstrapBuilder.Create(handlerWrapper).UseHttpHandler(testServerClient).Build();

        await bootstrap.RunAsync(cancellationToken);
    }

    public static Task Main() => RunAsync();
}
```

**Why these specific parameters.**

- `testServerClient` (nullable): the `HttpClient` from `LambdaTestServer.CreateClient()`. Production passes nothing and the real Lambda runtime is used; tests pass the test server's client so the Lambda bootstrap talks to the in-memory server instead of the real Lambda Runtime API.
- `modifyHostBuilder`: lets a test swap out host-level configuration (`ConfigureAppConfiguration`, custom logging providers, environment) that the service genuinely uses. Skipped 90% of the time — but when you need it, retrofitting it later is invasive.
- `modifyServices`: the seam tests use to swap real dependencies for fakes. Mandatory.
- `cancellationToken`: lets the test server shut the function down once the response is enqueued.

Production hits `Main` (no args, defaults take over). Tests hit `RunAsync` with the test server's `HttpClient` and a `modifyServices` action that registers fakes. The shipped `TestScope.cs.template` is built around this signature. Nothing further to do.

### Anything else — top-level `Program.cs`, plain `FunctionHandler` class, `Amazon.Lambda.Annotations`

Each of these starts with production owning a bootstrap path the test can't reach: top-level statements compile to an inaccessible `Program.<Main>$()`; an older `Function.FunctionHandler` is invoked by `Amazon.Lambda.RuntimeSupport` directly; annotations-generated code exposes a static method but no `modifyServices` seam.

**Refactor to the canonical shape.** Extract the DI wiring into a `ConfigureServices`, add a `RunAsync` overload with the canonical signature, and have the existing entry point (`Main`, the annotations adapter, or `FunctionHandler`) become a thin call into it. Tests are the right forcing function for this change.

Don't extract the handler class into the test directly — that loses the `LambdaBootstrapBuilder` and serializer registration production relies on. Don't `new Function()` either — its constructor/field initialisers usually do one-off DI setup that diverges from the runtime path.

### Anti-patterns to flag in review

All of these create a second composition root that production never executes — the test passes (or fails) on a wiring that diverges from prod the moment either side drifts.

- Constructing the handler class directly (`new EntryPoint()`, `new Function()`, `new LambdaEntryPoint()`).
- `TestScope` re-registering the SUT's own types (`services.AddSingleton<IFoo, Foo>()`) instead of reusing the SUT's composition root.
- Production `Program.cs` body copy-pasted into the test project.
- `[assembly: InternalsVisibleTo("…Tests")]` on production so a `TestStartup : Startup` subclass can reach in. The fix is to make the seam public on production (`RunAsync` with optional parameters), not to widen production's surface for tests.
- A test project calling `LambdaBootstrapBuilder.Create(...).Build().RunAsync(...)` itself. Add the `RunAsync` overload to production and have the test call it.

## Files to create

Under `test/{{ServiceName}}.Tests/`:

```
{{ServiceName}}.Tests.csproj
xunit.runner.json          (optional — xunit v3 parallelises by default)
Infrastructure/
  TestScope.cs
  LambdaTestServerFixture.cs
  FakeWhatever.cs          (one per outbound dep)
When<Scenario>.cs          (one per scenario; often at project root for small Lambdas)
```

## Steps

1. **Identify outbound dependencies in `src/`**: grep for interfaces registered in the Lambda's `EntryPoint` / `Startup` that would cross the network — `I*Store`, `I*Repository`, `I*Client`, `I*Publisher`, AWS SDK interfaces (`IAmazonS3`, `IAmazonSQS`, etc.). Prefer project specific abstractions (`I*Repository` rather than `IAmazonDynamoDB`, etc., and recommend adding these abstractions if not available.)

   Then **classify each dependency by transport**, because the methodology routes them differently (see `outside-in-testing/references/decision-rules.md`):

   - **HTTP-backed typed client** (an `I*Client` that wraps `HttpClient` / Refit / a generated client over HTTP) → do **not** fake the interface. Intercept at the HTTP transport. Follow the [outbound HTTP / gRPC variant](#variant-outbound-http--grpc).
   - **gRPC-backed client** (a generated `*.*Client` from a `.proto`) → intercept at the gRPC channel. Same variant.
   - **AWS SDK / data store / publisher / feature flags** → fake the interface (next step). AWS SDK types are awkward to intercept, so faking is the right call there.

2. **For each non-transport dependency, create a fake** in `Infrastructure/`:
   - Read/write data store → use `templates/lambda/FakeRepository.cs.template`.
   - Write-only sender (publisher / sink / emitter) → use `templates/lambda/FakePublisher.cs.template`.
   Plain `Dictionary<>` / `List<>` is fine — only one test touches the instance, no concurrency concerns.

3. **Copy `templates/lambda/TestScope.cs.template`** to `Infrastructure/TestScope.cs`. Substitute:
   - `{{TestsNamespace}}` → e.g., `{{ServiceName}}.Tests`
   - `{{ServiceRootNamespace}}` → e.g., `{{ServiceName}}`
   - `{{EntrypointType}}` → the actual Lambda entrypoint class (usually `EntryPoint`)
   - `{{EventNamespace}}` → `SQSEvents` / `DynamoDBEvents` / `KinesisEvents` / `APIGatewayEvents`
   - `{{EventType}}` → `SQSEvent` / `DynamoDBEvent` / `KinesisEvent` / `APIGatewayProxyRequest`
   - `{{Thing}}` → the domain type the Lambda processes (e.g., `Order`)
   - Add one `Fake* { get; } = new();` property and one matching `AddSingleton<I*>()` registration per outbound dependency.

4. **Copy `templates/lambda/LambdaTestServerFixture.cs.template`** as-is (just substitute namespace).

5. **Copy `templates/lambda/TestProject.csproj.template`** to `{{ServiceName}}.Tests.csproj`. Add a `ProjectReference` to the Lambda's source project. Uncomment the event-package PackageReference that matches the Lambda's trigger.

6. **Create a first `When<Scenario>.cs`** from `templates/lambda/WhenScenario.cs.template`. Build the trigger event inline; pass it to `_testScope.InvokeFunction(...)`.

7. **Add the test project to the solution**: `dotnet sln add test/{{ServiceName}}.Tests/{{ServiceName}}.Tests.csproj`. Make sure to follow any conventions in the solution regarding where tests are stored (are they nested with the related project(s) or in a separate place?).

8. **Verify**: `dotnet build` then `dotnet test`.

## Variant: outbound HTTP / gRPC

Apply this on top of the base scaffold when Step 1 turned up a typed HTTP client or a gRPC client. The principle (`outside-in-testing/references/decision-rules.md`): exercise the real outbound transport and let the test control the response — don't replace the typed client with a fake. Faking an `I*Client` that wraps `HttpClient` skips the auth headers, retry policy, serialiser, and `BaseAddress` resolution that the production code actually relies on.

Because each Lambda test rebuilds its own DI graph (isolation strategy 1), the interception setup here is simpler than the API variants — no `AsyncLocal<>`, no shared accessor. The `HttpClientInterceptorOptions` is owned by the `TestScope` and bound into the per-test DI container.

### HTTP

1. **Add the package** in `{{ServiceName}}.Tests.csproj`:

   ```xml
   <PackageReference Include="JustEat.HttpClientInterception" Version="5.1.0" />
   ```

2. **Copy `templates/lambda/HttpClientInterceptionFilter.cs.template`** to `Infrastructure/HttpClientInterceptionFilter.cs`. Substitute `{{TestsNamespace}}`.

3. **Add an `HttpClientInterceptorOptions` to `TestScope`**:

   ```csharp
   public HttpClientInterceptorOptions HttpClientInterceptorOptions { get; } = new()
   {
       ThrowOnMissingRegistration = true,
   };
   ```

4. **Wire the filter in `ModifyServices`** so every `HttpClient` produced by `IHttpClientFactory` runs through the interceptor:

   ```csharp
   serviceCollection.AddSingleton<IHttpMessageHandlerBuilderFilter>(
       new HttpClientInterceptionFilter(HttpClientInterceptorOptions));
   ```

   Requires `Microsoft.Extensions.Http` (transitively pulled in by most Lambda hosts; add explicitly if not).

5. **Create grouped registration extensions** under `Infrastructure/Extensions/<TargetService>InterceptorExtensions.cs` — one file per external service. Use the same pattern as `references/maintain-add-http-interception.md` (which from now on is the canonical place to add new HTTP registrations for any project, Lambda included).

6. **In each scenario, register the responses the SUT should see** before invoking the function:

   ```csharp
   _testScope.HttpClientInterceptorOptions.RegisterSuccessfulPricingLookup(productId: "abc", price: 4.99m);
   await _testScope.InvokeFunction(triggerEvent);
   ```

`ThrowOnMissingRegistration = true` means any unmocked outbound HTTP call fails the test fast — that's the point.

### gRPC

1. **Add the gRPC client package(s)** the service uses (e.g., `Grpc.Net.ClientFactory`) plus `Grpc.Core.Testing` for the test-side helpers.

2. **Don't fake the generated client.** Instead, register a test `CallInvoker` (or a `GrpcChannel` whose handler short-circuits) for the typed client in `ModifyServices`:

   ```csharp
   public FakeGrpcCallInvoker PricingInvoker { get; } = new();

   // in ModifyServices:
   serviceCollection.AddSingleton<Pricing.PricingClient>(
       _ => new Pricing.PricingClient(PricingInvoker));
   ```

   `FakeGrpcCallInvoker` is a thin `CallInvoker` subclass under your control — it records requests and returns canned `AsyncUnaryCall<T>` / `AsyncServerStreamingCall<T>` / etc. responses. Tests seed it the same way HTTP tests register interceptor responses.

3. **If the service consumes the gRPC client through an `IGrpcClientFactory`**, replace its registration so it hands back the test-wired client. The methodology rule still holds: intercept at the transport (the `CallInvoker`), not by faking the typed wrapper interface above it.

The base `outside-in-testing` skill calls out gRPC interception alongside HTTP — see decision-rules.md row "gRPC".

## Variant: event DSL (Pattern B)

When the service handles events with enough structure to justify an abstraction, add a domain-specific event DSL on top of the base scaffold. This is a grow-into-it concern; don't scaffold it up front.

Typical shape:
- `FlowEventSource.cs` — where the event comes from (SQS queue, DynamoDB stream).
- `FlowEventData.cs` — the domain payload.
- `FlowEventDestination.cs` — where it ends up.
- `FlowEventInstruction.cs` — a seeded "given X, expect Y" unit composed into test runs.
- `FlowMessage.cs` — envelope wrapper.

Create these when the third or fourth scenario starts copy-pasting the same setup prefix.

## Gotchas

- `Debugger.IsAttached ? Timeout.InfiniteTimeSpan : TimeSpan.FromSeconds(5)` — OK locally, leaks in CI if the agent attaches a debugger. Usually fine; note it.
- The Lambda test server starts and stops per invocation. Do not try to share it across tests — it's cheap.
- Don't forget to call `_testScope.Dispose()` — the scenario class should implement `IDisposable` and delegate.
- **Don't fake a typed HTTP/gRPC client interface.** If you see any `I*Client` whose implementation owns an `HttpClient` or a generated gRPC stub, intercept at the transport per the [outbound HTTP / gRPC variant](#variant-outbound-http--grpc). Faking the wrapper bypasses serialisation, auth, retry, and `BaseAddress` — which is exactly the path you want the test to exercise.
