# Pitfalls

Each entry: **symptom** → **cause** → **fix**.

---

### Fake singleton pollution (shared-singleton strategies only)
**Symptom.** Tests pass individually, fail in parallel. A fake returns data seeded by another test.
**Cause.** Fakes are registered as singletons but partition state by nothing (or by a value that multiple tests share, like a fixed merchant ID).
**Fix.** Partition by a per-test discriminator the service propagates (correlation ID, request ID). If no such value exists, move the fake to per-test-instance (strategy 1) or serialise tests with a `[Collection]`.

---

### HTTP interceptor subscriptions not disposed
**Symptom.** A test's assertions see HTTP requests from a different test, or `ThrowOnMissingRegistration` fires for a registration that "should" be there.
**Cause.** `HttpRequestObserver.Subscribe` returns an `IDisposable` that must be disposed when the TestScope ends. If not disposed, the subscription outlives the scope.
**Fix.** Store the subscription on the TestScope and dispose it in `Dispose()`. Also set `httpClientInterceptorOptionsAccessor.Current = null` so the AsyncLocal doesn't leak.

---

### Discriminator propagation gap
**Symptom.** Shared-singleton fake returns empty even though the test seeded it; or the service logs show one correlation ID and the fake's partitions use another.
**Cause.** The service reads the discriminator from one place (e.g., HTTP header `X-Correlation-Id`) but the test sets it somewhere else (or not at all).
**Fix.** Trace the full path: test → HTTP request header → middleware → Activity / context / DI → service → fake. The fake must read from the same place the middleware writes. Add a startup assertion or a test that explicitly checks propagation.

---

### Static state between tests
**Symptom.** Tests start failing after a recent change introduced a `static` cache, a `DateTime.UtcNow` capture, or a singleton reading from `Environment.GetEnvironmentVariable`.
**Cause.** Static state outlives the test scope.
**Fix.** Replace statics with injected abstractions (`IClock`, `IConfig`, `IMemoryCache`). In tests, inject a fake/disabled version. For caches you can't easily abstract, register a `DisabledCache` or similar no-op implementation.

---

### `TestServer.PreserveExecutionContext = false`
**Symptom.** `AsyncLocal<>`-based isolation (e.g., an `AsyncHttpClientInterceptorOptionsAccessor` that stores options in `AsyncLocal<HttpClientInterceptorOptions?>`) silently doesn't isolate. HTTP interception options bleed across tests.
**Cause.** ASP.NET's `TestServer` drops the execution context by default, so `AsyncLocal<>` values don't flow to the request handler.
**Fix.** `webHostBuilder.UseTestServer(options => options.PreserveExecutionContext = true);`

---

### `ThrowOnMissingRegistration` off
**Symptom.** Tests pass in CI despite the service now making a real outbound HTTP call to a URL no one mocked. Production breakage follows.
**Cause.** `JustEat.HttpClientInterception` defaults to passing unmocked requests through; if the target happens to be reachable (or times out silently), the test still passes.
**Fix.** Set `interceptorOptions.ThrowOnMissingRegistration = true` in the central interceptor setup. Override it deliberately in the (rare) cases where pass-through is wanted.

---

### Debugger-attached infinite timeouts leak into CI
**Symptom.** A test hangs in CI forever; logs show the Lambda test server never timed out.
**Cause.** Code like `Debugger.IsAttached ? Timeout.InfiniteTimeSpan : TimeSpan.FromSeconds(5)` is fine locally but a CI agent with a debugger attached (unlikely) or an explicit env override sets the infinite branch.
**Fix.** Gate on an env var, not on `Debugger.IsAttached`, if the risk is real. Otherwise, keep the pattern but make sure CI never sets the long branch.

---

### Health checks and warmup tasks during tests
**Symptom.** Fakes receive calls you didn't make; HTTP interceptor throws on a URL the test never invoked.
**Cause.** The application registers a health check that pings a dependency on startup, or a `WarmupTaskExecutor` runs before the first request.
**Fix.** In the test Startup / fixture, remove these registrations explicitly:
```csharp
services.PostConfigure<HealthCheckServiceOptions>(options => options.Registrations.Clear());
var warmup = services.Single(s => s.ImplementationType == typeof(WarmupTaskExecutor));
services.Remove(warmup);
```

---

### `appsettings.*.json` not in `bin/`
**Symptom.** Runtime says config key missing, but you see it in the JSON file.
**Cause.** `.csproj` doesn't copy the file to the output directory.
**Fix.**
```xml
<ItemGroup>
  <Content Include="appsettings.DeveloperTests.json" CopyToOutputDirectory="PreserveNewest" />
</ItemGroup>
```

---

### IAsyncLifetime ordering surprises
**Symptom.** Setup code references a resource that isn't ready yet; or cleanup runs before something has flushed.
**Cause.** The order is: fixture `InitializeAsync` → test class `InitializeAsync` → test method → test class `DisposeAsync` → fixture `DisposeAsync`. Test class construction runs BEFORE fixture `InitializeAsync` is awaited in some xUnit versions.
**Fix.** Don't do I/O in the test class constructor. Do it in `InitializeAsync`. Don't assume fixture state is ready in the constructor.

---

### Logging interleave under parallel tests
**Symptom.** Log output for test A contains lines from test B. Hard to debug.
**Cause.** Parallel tests, shared stdout.
**Fix.** Use `MartinCostello.Logging.XUnit` (or equivalent) to route logs through `ITestOutputHelper`. Each test has its own output helper; xUnit correlates lines to their test.

---

### Testcontainers not cleaned up
**Symptom.** CI disk fills up; containers accumulate across runs.
**Cause.** Fixture `DisposeAsync` doesn't stop the container.
**Fix.** Implement `IAsyncLifetime.DisposeAsync` on the fixture and call `container.StopAsync()` and `container.DisposeAsync()` explicitly. Ryuk (the Testcontainers reaper) is supposed to clean up on JVM/process exit, but CI agents often kill processes in ways that defeat it.

---

### Fake vs. mock confusion
**Symptom.** A "fake" that contains if/else business logic, or a test that only passes because its mock's `Verify` call was set up identically to the implementation.
**Cause.** Someone reached for Moq where a fake would do, or grew logic into a fake over time.
**Fix.** Fakes are dumb stores: seed, read, return. If you need "when X is called, assert it was called with Y", usually you can fake the dependency as a store, let the real code write to it, and query the store afterwards.

---

### `AsyncLocal` ≠ DI scope
**Symptom.** Using `AsyncLocal<T>` for per-test isolation works until you introduce a background worker, hosted service, or `Task.Run`. The AsyncLocal value is empty in the worker.
**Cause.** AsyncLocal flows down the async tree, but some execution contexts are intentionally cut (e.g., `Task.Run` without a captured execution context).
**Fix.** Prefer a proper DI scope over AsyncLocal when available. If AsyncLocal is the only option (HTTP interception options are accessed from `HttpMessageHandler` outside DI), make sure `TestServer.PreserveExecutionContext = true` and test explicitly that background-triggered paths see the right value.
