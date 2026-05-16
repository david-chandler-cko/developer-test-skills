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

### Variant — production exposes `IConfiguration`, not `IHostBuilder`

Some services skip `Host.CreateDefaultBuilder()` and build their `ServiceProvider` directly from an `IConfiguration` (e.g., AWS Lambda templates that pre-date the `Microsoft.Extensions.Hosting` story). The production signature is:

```csharp
public static async Task RunAsync(
    HttpClient? httpClient = null,
    IConfiguration? configuration = null,
    Action<IServiceCollection>? configureServices = null,
    CancellationToken cancellationToken = default)
{
    var services = ServiceRegistry.BuildServiceProvider(
        configuration ?? ConfigurationHelpers.BuildConfiguration(Directory.GetCurrentDirectory()),
        configureServices);
    /* ... LambdaBootstrapBuilder.Create(handler).UseHttpClient(httpClient).Build().RunAsync(...) ... */
}
```

This is functionally equivalent to the canonical shape — just keep the same seam contract:

- `httpClient` (nullable): test server's client, threaded to `UseHttpClient`.
- `configuration` (nullable): when tests want to override config keys without writing files, they pass a `new ConfigurationBuilder().AddJsonFile(prod-appsettings).AddInMemoryCollection(overrides).Build()`.
- `configureServices`: the `Action<IServiceCollection>` seam for fakes — same role as `modifyServices` above. Note that this delegate runs *after* the production registrations, so a fake registered as a singleton against the same interface overrides the real one (last registration wins for non-`IEnumerable<T>` resolution).
- `cancellationToken`: as before.

`TestScope.InvokeFunctionAsync` then looks like:

```csharp
await Function.RunAsync(
    httpClient: server.CreateClient(),
    configuration: _testConfiguration,
    configureServices: services =>
    {
        services.AddSingleton<IAmazonSimpleNotificationService>(FakeSns);
        services.AddSingleton<IDynamoDbClientBuilder>(FakeDynamoDbClientBuilder);
        // ...
    },
    cancellationToken: cts.Token);
```

If production currently has no such seam, *adding* the `IConfiguration` and `Action<IServiceCollection>` parameters is the scaffold's first PR — same forcing-function rationale as the canonical shape.

### Anything else — top-level `Program.cs`, plain `FunctionHandler` class, `Amazon.Lambda.Annotations`

Each of these starts with production owning a bootstrap path the test can't reach: top-level statements compile to an inaccessible `Program.<Main>$()`; an older `Function.FunctionHandler` is invoked by `Amazon.Lambda.RuntimeSupport` directly; annotations-generated code exposes a static method but no `modifyServices` seam.

**Refactor to the canonical shape.** Extract the DI wiring into a `ConfigureServices`, add a `RunAsync` overload with the canonical signature, and have the existing entry point (`Main`, the annotations adapter, or `FunctionHandler`) become a thin call into it. Tests are the right forcing function for this change.

Don't extract the handler class into the test directly — that loses the `LambdaBootstrapBuilder` and serializer registration production relies on. Don't `new Function()` either — its constructor/field initialisers usually do one-off DI setup that diverges from the runtime path.

### Variant — production already owns a class-based `LambdaEntryPoint` / `Startup`

Common when the service inherits an internal base class (e.g., `APIGatewayHttpApiV2ProxyFunction`-style adapters that pair an entry-point class with a `Startup`-style configurator). The existing shape is:

```csharp
internal static class Program
{
    private static async Task Main() { /* bootstrap LambdaEntryPoint here */ }
}

public class LambdaEntryPoint : APIGatewayHttpApiV2ProxyFunction { /* eager field init */ }

public class Startup { public void ConfigureServices(IServiceCollection s) { ... } }
```

`LambdaEntryPoint`'s constructor often eagerly resolves an `IServiceProvider`, instantiates loggers, and registers handlers — you cannot replace its DI without re-running its constructor, but you also don't want two entry-point classes drifting. The fix:

1. Keep `LambdaEntryPoint` and `Startup` exactly as production has them.
2. Add a `Startup.ConfigureHosting(Action<IHostBuilder>)` seam, or use whatever method the `Startup` base already exposes to influence the host builder. If no such seam exists yet, adding one is part of the scaffold — it should be a tiny method that captures the delegate and invokes it inside the base class's existing host-building flow.
3. Add the canonical `Program.RunAsync` overload that wires `modifyHostBuilder` / `modifyServices` *through* `Startup`, then bootstraps `LambdaEntryPoint` exactly as `Main` did:

   ```csharp
   public static async Task RunAsync(
       HttpClient? httpClient = null,
       Action<IHostBuilder>? modifyHostBuilder = null,
       Action<IServiceCollection>? modifyServices = null,
       CancellationToken cancellationToken = default)
   {
       var startup = new Startup();
       if (modifyServices is not null)
       {
           startup.ConfigureHosting(builder =>
           {
               modifyHostBuilder?.Invoke(builder);
               builder.ConfigureServices((_, services) => modifyServices(services));
           });
       }
       else if (modifyHostBuilder is not null)
       {
           startup.ConfigureHosting(modifyHostBuilder);
       }

       using var entryPoint = new LambdaEntryPoint(startup);
       using var bootstrap = LambdaBootstrapBuilder
           .Create(entryPoint.FunctionHandlerAsync, new DefaultLambdaJsonSerializer())
           .UseHttpClient(httpClient)
           .Build();
       await bootstrap.RunAsync(cancellationToken);
   }

   private static async Task Main() => await RunAsync();
   ```

Production behaviour is unchanged (`Main` calls `RunAsync()` with no arguments). Tests pass the `LambdaTestServer.CreateClient()` and a `modifyServices` delegate that registers fakes.

**Check sibling services first.** If another Lambda in the same solution already exposes a `Program.RunAsync(...)` of this shape, copy it verbatim — neighbouring services have usually already solved this, and matching their convention keeps the scaffold one consistent pattern rather than several near-duplicates.

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

## Variant: multi-trigger Lambda (e.g., ALB + SQS in one binary)

A single Lambda binary that handles two trigger types (sync HTTP via ALB or API Gateway, async events via SQS / DynamoDB Streams / Kinesis) sharing one DI graph is common. The base `TestScope.InvokeFunction(TEvent)` shape (single trigger type, single response type) collides with that. The fix is two thin invoke methods on `TestScope`, sharing one private worker:

```csharp
public Task<ApplicationLoadBalancerResponse?> InvokeAlbAsync(ApplicationLoadBalancerRequest request)
    => InvokeAsync<ApplicationLoadBalancerResponse>(request);

public Task<SQSBatchResponse?> InvokeSqsAsync(SQSEvent sqsEvent)
    => InvokeAsync<SQSBatchResponse>(sqsEvent);

private async Task<T?> InvokeAsync<T>(object lambdaEvent) where T : class
{
    // Same body: serialise → enqueue → Program.RunAsync(testServer.CreateClient(), ...) → read response
}
```

Notes:

- One `LambdaTestServer` per invocation is fine — they're cheap and isolate cleanly. Don't try to share one across both trigger types.
- The interceptor options accessor (`HttpClientInterceptorOptionsAccessor`) and fakes are shared across both invoke methods on the same `TestScope` — a scenario can seed historical records via an SQS invocation, then validate a new one via an ALB invocation, exercising both trigger paths against the same in-memory state.
- Use the appropriate Lambda serializer (e.g., `DefaultLambdaJsonSerializer`) to serialise the input — `JsonSerializer.Serialize(lambdaEvent)` will miss source-generator-aware shaping on some event types.
- Don't compose the two events into one — drive the two triggers independently. If a scenario *needs* to seed via the production SQS path rather than directly faking the repository, see the "Direct seed vs. round-trip" rule in `references/maintain-add-scenario.md`.

## Variant: outbound HTTP / gRPC

Apply this on top of the base scaffold when Step 1 turned up a typed HTTP client or a gRPC client. The principle (`outside-in-testing/references/decision-rules.md`): exercise the real outbound transport and let the test control the response — don't replace the typed client with a fake. Faking an `I*Client` that wraps `HttpClient` skips the auth headers, retry policy, serialiser, and `BaseAddress` resolution that the production code actually relies on.

Because each Lambda test rebuilds its own DI graph (isolation strategy 1), the interception setup here is simpler than the API variants — you don't need an `AsyncLocal<>`-based accessor that partitions options by call context. But the accessor *pattern* (a singleton holder bound into DI, whose `Current` is set per invocation) is still required: the `HttpClientInterceptionFilter` and the `HttpMessageHandler` chain run on threads the test method's `AsyncLocal` doesn't reach (the Lambda test server's worker context). A plain `services.AddSingleton(myInterceptorOptions)` works only if you rebuild the host every invocation *and* set `Current` immediately before each `Program.RunAsync` call.

The simplest shape: `TestScope` owns an `HttpClientInterceptorOptionsAccessor` (a small class with a single mutable `Current` property — no `AsyncLocal<>` involved), registers it as the singleton, and writes `_optionsAccessor.Current = BuildOptions()` right before each `Program.RunAsync`. A plain shared-property accessor is correct here precisely because per-test DI rebuild gives you the isolation that `AsyncLocal<>` would normally provide — and `AsyncLocal<>` wouldn't propagate anyway, because the Lambda test server's worker context doesn't inherit values from the test thread.

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

## Variant: AWS SDK clients (`IAmazon*`) — direct or via a factory

The same outside-in rule applies — intercept at the boundary — but AWS SDK interfaces are usually too large to hand-fake comfortably (`IAmazonDynamoDB` and `IAmazonSimpleNotificationService` are >50 methods each). Two patterns work; pick deliberately.

**Option A: hand-fake only the methods production actually calls.** A `FakeDynamoDb : IAmazonDynamoDB` that implements `GetItemAsync`, `PutItemAsync`, `DescribeTableAsync` and lets every other method default to throwing (the C# compiler will refuse to compile until you stub them, so write a one-line `=> throw new NotImplementedException();` for each). The fake's *state* (a `Dictionary<string, Dictionary<string, AttributeValue>>` keyed by table + partition key) is the part you actually use; everything else is wallpaper. Most Lambda services touch fewer than five SDK methods per client.

**Option B: introduce a project-specific port interface in `src/`** (e.g., `IPaymentIdStore` instead of `IAmazonDynamoDB`). The production code wraps the SDK behind a narrow interface that exposes only the operations the domain needs; tests fake the narrow interface, not the SDK. Right answer when (a) the SDK is touched in many places, (b) the operations have a clear domain meaning, or (c) the SDK surface is genuinely large. The downside: it's a small `src/` refactor before the test work begins.

Option A is the right default for first-test-suite scaffolds where production code is stable; option B is the right answer when you're already touching the SDK call sites.

### AWS SDK clients hidden behind a factory or builder

A common production pattern is to wrap the SDK client construction (cross-account role assumption, custom endpoints, retry config) in a factory interface:

```csharp
public interface IDynamoDbClientBuilder { IAmazonDynamoDB Build(); }
public class PaymentIdStore(IDynamoDbClientBuilder builder) { ... builder.Build().GetItemAsync(...) ... }
```

Fake the factory, not the SDK client directly. The factory hands back the SDK fake on every call (or the same instance — the SDK clients in production are usually thread-safe and reused, so handing back the same fake matches behaviour):

```csharp
public class FakeDynamoDbClientBuilder(FakeDynamoDb client) : IDynamoDbClientBuilder
{
    public IAmazonDynamoDB Build() => client;
}

// in ConfigureServices:
services.AddSingleton<IDynamoDbClientBuilder>(new FakeDynamoDbClientBuilder(FakeDynamoDb));
```

The TestScope exposes `FakeDynamoDb` (the inner client) as the seam tests seed against; the builder is invisible plumbing.

## Configuration: how the TestScope wires `IConfiguration`

If production reads from `appsettings.json` and the entry point exposes an `IConfiguration` parameter ([Variant — production exposes `IConfiguration`, not `IHostBuilder`](#variant--production-exposes-iconfiguration-not-ihostbuilder)), the TestScope owns a config built from the same file plus in-memory overrides:

```csharp
private static IConfiguration BuildTestConfiguration()
{
    // The production csproj must already copy appsettings.json to its bin/ folder.
    // The test reads from there so config keys are real, and overrides only what it
    // needs to know deterministically (topic ARNs, table names, role ARNs).
    var prodAssemblyDir = Path.GetDirectoryName(typeof(MyFunction).Assembly.Location)
        ?? throw new InvalidOperationException("Production assembly location unknown.");

    return new ConfigurationBuilder()
        .SetBasePath(prodAssemblyDir)
        .AddJsonFile("appsettings.json", optional: false)
        .AddInMemoryCollection(new Dictionary<string, string?>
        {
            ["AWSSNS:TopicArn"] = "arn:aws:sns:eu-west-1:123456789012:test-topic",
            ["DynamoDb:TableName"] = "TestTable",
            // override anything the scenario asserts on
        })
        .Build();
}
```

Two reasons to override in-memory rather than ship a `appsettings.Tests.json`:
1. Overrides live next to the test that depends on them — easy to find the source-of-truth for a magic value.
2. No second config file to drift from production keys.

Use a separate `appsettings.Tests.json` only when there's a large block of config the tests deliberately diverge on (e.g., enabling/disabling feature flags wholesale).

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
