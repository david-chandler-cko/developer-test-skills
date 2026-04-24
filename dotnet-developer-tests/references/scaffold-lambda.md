# Scaffold recipe: .NET Lambda test project

Matches Pattern A in the methodology skill's `examples/example-patterns.md` (xUnit v3, an assertion library, per-test fake instances, no HTTP interception).

## Assumptions

- The Lambda entrypoint is a class (usually `EntryPoint`) with a static method like `RunAsync(HttpClient handler, Action<IServiceCollection>? modifyServices, CancellationToken ct)` that `MartinCostello.Testing.AwsLambdaTestServer` can drive. If the service was built with `Amazon.Lambda.Annotations` or a plain `FunctionHandler` class, adapt the invocation call site — the scope of change is small.
- Fakes are per-test-instance (strategy 1). Each test gets its own `TestScope`, which owns its own `new Fake*()` fields. No discriminator, no shared state.
- `AwesomeAssertions` for assertions.

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

2. **For each one, create a fake** in `Infrastructure/`:
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
