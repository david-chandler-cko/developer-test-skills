# Ports-and-adapters entrypoint

If the service is structured so that HTTP/Lambda adapters are thin and the real behaviour lives in an application-level handler (`ICommandHandler<T>`, `IRequestHandler<TReq, TRes>`, MediatR `IRequest<T>`, etc.), you may drive the test at the handler directly and skip the transport.

## When this is appropriate

- **Adapter is trivial** (deserialise → call handler → serialise).
- **No meaningful middleware** (auth, validation, error-mapping) in the adapter layer.
- **Handler is directly constructible** in the test DI.

## When to keep using the transport entrypoint instead

- The adapter does auth, custom model binding, or error transformation.
- You want to test the HTTP contract shape (routes, headers, status codes).
- Integration / acceptance tests already drive via the transport and you want consistency.

## How to do it

1. **Expose the DI composition root** in a way the test can call. Usually the service already has this (e.g., `ServiceCollectionExtensions.AddDomainServices(IServiceCollection)`).

2. **Build a minimal test DI container** with the real composition + fakes:

   ```csharp
   public sealed class TestScope : IAsyncDisposable
   {
       private readonly ServiceProvider _provider;

       public TestScope()
       {
           var services = new ServiceCollection();
           services.AddLogging();
           services.AddDomainServices();       // the real DI extension from src/

           // Fakes
           services.AddSingleton<FakeThingRepository>();
           services.AddSingleton<IThingRepository>(sp => sp.GetRequiredService<FakeThingRepository>());

           _provider = services.BuildServiceProvider();
           FakeThingRepository = _provider.GetRequiredService<FakeThingRepository>();
       }

       public FakeThingRepository FakeThingRepository { get; }

       public Task<TResponse> Handle<TRequest, TResponse>(TRequest request, CancellationToken ct = default)
           where TRequest : IRequest<TResponse>
       {
           var handler = _provider.GetRequiredService<IRequestHandler<TRequest, TResponse>>();
           return handler.Handle(request, ct);
       }

       public ValueTask DisposeAsync() => _provider.DisposeAsync();
   }
   ```

3. **In scenarios**:

   ```csharp
   public class WhenSomethingHappens : IAsyncLifetime
   {
       private readonly TestScope _scope = new();
       private SomeResponse _response = null!;

       public async Task InitializeAsync()
       {
           _scope.FakeThingRepository.AddThing(new Thing("id-1", ...));
           _response = await _scope.Handle<CreateThingCommand, SomeResponse>(new CreateThingCommand(...));
       }

       [Fact]
       public void ThenThingIsPersisted() { ... }

       public async Task DisposeAsync() => await _scope.DisposeAsync();
   }
   ```

## Isolation

Same three strategies as transport-driven tests. Per-test DI container (strategy 1) is the natural fit because building the DI container is cheap.

## Outbound deps

Fakes and interceptors still apply. HTTP-calling code inside the handler still needs `httpclient-interception` (register the `IHttpMessageHandlerBuilderFilter` in the test DI and use `AsyncLocal<>` exactly as in the API variants).
