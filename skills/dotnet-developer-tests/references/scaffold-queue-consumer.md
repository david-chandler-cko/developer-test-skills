# Scaffold recipe: queue-consumer service (SQS / Kafka / Service Bus)

For a long-running service hosted in `Microsoft.Extensions.Hosting.IHost` whose entrypoint is **a background polling loop pulling messages off a queue**, not an HTTP request handler. Examples: an SQS consumer using `Checkout.Sqs.Consumers`, a Kafka consumer using `Confluent.Kafka` behind a hosted service, an Azure Service Bus processor.

This recipe layers on top of [`scaffold-api-xunit-di.md`](scaffold-api-xunit-di.md). Read that first — most of the infrastructure (TestContext/TestScope, AsyncLocal HTTP interceptor accessor, correlation-ID partitioning, fake patterns) is shared. This page only covers what's *different* about queue-consumer hosts.

## Key difference: bypass the polling pump

Production polls the real queue on a background `IHostedService`. Tests must **not** start that pump — it would either (a) hang the test host trying to connect to a real queue, or (b) race the test's enqueue with the pump's dequeue.

Two strategies; pick by what's least invasive in your codebase.

### Strategy A: invoke the handler directly via DI (recommended)

The cleanest pattern. Tests:

1. Strip every `IHostedService` from the production DI graph during test boot.
2. Resolve the message-handler service from DI (`IServiceProvider.GetRequiredService<IHandler<TMessage>>()`).
3. Synthesise the consumer library's `MessageContext` (or equivalent) wrapping the test message.
4. Await `handler.HandleAsync(context, ct)` and assert.

```csharp
public async Task<HandlerResult> SendMessageAsync<TMessage>(
    QueueId queue,
    Func<TMessage, TMessage> customize)
{
    var message = customize(new TMessage());
    var handler = _server.Host.Services.GetRequiredService<IHandler<TMessage>>();
    var context = new MessageContext(
        queueUrl: QueueUrlFor(queue),
        handledCount: 0,
        data: message,
        cancellationToken: _cts.Token);
    var result = await handler.HandleAsync(context);
    return result;
}
```

Tests assert on the handler's return value (`Processed` / `Retried` / `Deadlettered`) *first*, then snapshot fake state. See `maintain-add-scenario.md` "Queue-consumer entrypoint" for the scenario-side rule.

### Strategy B: drive the queue fake and let the production pump consume

Only when handler dispatch is genuinely opaque (production has a complex middleware/decorator chain you'd skip by going direct). Replace the real SQS client with `FakeSqsClient.PushMessage(...)`, then `await fake.WaitForProcessing(messageId)`. More fragile — the test depends on the pump's internal scheduling — but sometimes unavoidable.

If you reach for B, document why in the scaffold PR. Most queue-consumer codebases work fine with A.

## Strip `IHostedService` pollers during test boot

```csharp
// In Startup.ConfigureServices (Xunit.DI) or the equivalent test composition root:
services.RemoveAll<IHostedService>();
// or, more surgically, only the pollers you don't want:
var pollers = services
    .Where(d => typeof(IHostedService).IsAssignableFrom(d.ImplementationType ?? d.ServiceType))
    .Where(d => d.ServiceType.Name.Contains("Poller") || d.ServiceType.Name.Contains("Consumer"))
    .ToList();
foreach (var poller in pollers) services.Remove(poller);
```

`RemoveAll<IHostedService>()` is the right call if the service's hosted services exist *only* to poll queues. If production uses hosted services for legitimate test-relevant work (cache warmers, metrics flushers), keep those and remove only the queue pumps.

## TestScope shape

Same as scaffold-api-xunit-di, but the public surface is `SendMessageAsync<TMessage>(...)` instead of `CreateApiClient()`. Outbound fakes (a `FakeSnsPublisher` if the consumer republishes downstream, a `FakeReadModelRepository`, etc.) and the event-log fake (if the consumer emits domain events) live exactly as they do for the API recipe.

## Auth — usually unneeded

Queue-consumer services rarely validate JWTs (the queue ingestion already happened in another service). Skip the `FakeTokenIssuer` setup unless production explicitly reads claims off the message context.

## Gotchas specific to queue consumers

- **Cancellation semantics**: The handler's `CancellationToken` usually comes from the pump's lifetime. In tests, scope it to the `TestScope.Dispose()` so handlers don't outlive the test. Don't pass `CancellationToken.None`.
- **Per-message correlation ID propagation**: If production reads correlation IDs from message attributes (SQS `MessageAttributes`, Kafka headers), the test's synthesised `MessageContext` must include them — otherwise correlation-keyed fakes return empty.
- **Acceptance-result mapping**: The consumer library's "what to do next" enum (the result of `HandleAsync`) is the contract being tested. A green test that doesn't assert on it is testing nothing — the handler could be silently dead-lettering everything.
- **Dead-letter and retry pathways are first-class assertions**, not pitfalls to handle elsewhere. A scenario titled `WhenXMessageFails` should assert `result.Action.Should().Be(Action.Deadlettered)` — that's the contract.
