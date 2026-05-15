# Maintain: add an HTTP interception

Use when the service code now makes an outbound HTTP call that tests must control. Do this via `JustEat.HttpClientInterception`, not by faking the HTTP client.

## Checklist

1. **Find the central interceptor setup.** Usually `Infrastructure/Extensions/` or `TestContext.cs`. Look for `RegisterSuccessfulXxxResponse(this HttpClientInterceptorOptions ...)` extension methods grouped by target service.

2. **Add a new extension method** for the new dependency. Group by target service file — one file per external service being called.

   ```csharp
   // Infrastructure/Extensions/ExternalPricingApiInterceptorExtensions.cs
   public static class ExternalPricingApiInterceptorExtensions
   {
       public static HttpClientInterceptorOptions RegisterSuccessfulPriceLookup(
           this HttpClientInterceptorOptions options,
           string correlationId,
           decimal price = 100m)
       {
           new HttpRequestInterceptionBuilder()
               .Requests()
               .ForHttps()
               .ForHost("pricing.internal.example.com")
               .ForPath("/prices")
               .ForQuery($"correlation_id={correlationId}")
               .Responds()
               .WithJsonContent(new { price })
               .RegisterWith(options);
           return options;
       }

       public static HttpClientInterceptorOptions RegisterPriceLookupReturningNotFound(
           this HttpClientInterceptorOptions options,
           string correlationId)
       {
           new HttpRequestInterceptionBuilder()
               .Requests()
               .ForHttps()
               .ForHost("pricing.internal.example.com")
               .ForPath("/prices")
               .ForQuery($"correlation_id={correlationId}")
               .Responds()
               .WithStatus(HttpStatusCode.NotFound)
               .RegisterWith(options);
           return options;
       }
   }
   ```

   Note: `RegisterWith` returns the `HttpRequestInterceptionBuilder`, not the options. The two-statement form above is the working pattern.

3. **Register the happy-path variant as a default** in the central setup (e.g., `TestContext.CreateScope` composes it into every test):

   ```csharp
   ConfigureInterceptor = interceptorOptions =>
   {
       // ... existing defaults
       interceptorOptions.RegisterSuccessfulPriceLookup(correlationId);

       foreach (var action in testScopeOptions.InterceptorConfigurations)
       {
           action(correlationId, interceptorOptions);
       }

       interceptorOptions.ThrowOnMissingRegistration = true;
   };
   ```

4. **Override per-scenario when needed**:

   ```csharp
   using var testScope = _testContext.CreateScope(options =>
   {
       options.ConfigureInterceptor((correlationId, interceptorOptions) =>
       {
           // Replace the default with a NotFound for this test
           interceptorOptions.Deregister(/* matching criteria */);
           interceptorOptions.RegisterPriceLookupReturningNotFound(correlationId);
       });
   });
   ```

5. **Sanity-check.**
   - `ThrowOnMissingRegistration = true` is still set at the end of the configuration.
   - The happy-path default is registered for every test, not only the scenarios that happen to care.
   - The correlation ID is on the match predicate when state must partition (strategy 2).

## Gotchas

- **URL differs by environment**: interceptor matches the full URL. If the service uses `httpClientFactory` with a configured `BaseAddress`, the interceptor must match that exact host. Don't intercept `http://localhost:9999` if production is `https://x.y.z.com` — the test should hit the production URL, and `appsettings.DeveloperTests.json` should NOT redirect it to localhost (that's an anti-pattern; the interceptor makes the real URL safe).
- **HTTPS vs HTTP**: `.ForHttps()` or `.ForHttp()` must match how the service dials.
- **Query-string matching**: some services add correlation IDs as query params, some as headers. The interceptor registration must match the real request shape or it will fail.
- **`Deregister` is rare**: usually you add specialised overrides (more-specific path or headers) rather than removing the default.
