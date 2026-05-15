# NuGet versions

Recommended baseline as of 2026-04. Update as packages move.

## Core test framework

| Package | Version | Notes |
|---|---|---|
| `xunit.v3` | `3.2.0+` | **Preferred** for greenfield projects. |
| `xunit` | `2.9.3+` | Use only when blocked from v3. |
| `xunit.runner.visualstudio` | `3.1.5+` (for v3) / `2.8.2+` (for v2) | Required for test discovery. Mark `PrivateAssets=all`. |
| `Microsoft.NET.Test.Sdk` | `18.0.1+` (for v3) / `17.13.0+` (for v2) | Required. |
| `MartinCostello.Logging.XUnit.v3` | `0.7.0+` (for v3) | Routes logs to `ITestOutputHelper`. |
| `MartinCostello.Logging.XUnit` | `0.5.1+` (for v2) | v2 equivalent. |

## HTTP interception

| Package | Version | Notes |
|---|---|---|
| `JustEat.HttpClientInterception` | `4.3.0+` | Canonical HTTP interception library. |

## Lambda testing

| Package | Version | Notes |
|---|---|---|
| `MartinCostello.Testing.AwsLambdaTestServer` | `0.12.0+` | Lambda test server. |
| `Amazon.Lambda.Core` | `2.8.1+` | SDK baseline. |
| `Amazon.Lambda.Serialization.SystemTextJson` | `2.4.4+` | Required by the test scope to serialise events into the test server. |
| `Amazon.Lambda.TestUtilities` | `3.0.1+` | Event helpers. |
| `Amazon.Lambda.DynamoDBEvents` | `3.1.2+` | DynamoDB stream event model. |
| `Amazon.Lambda.KinesisEvents` | `3.0.2+` | Kinesis stream event model. |
| `Amazon.Lambda.SQSEvents` | `2.2.1+` | SQS event model. **Not on a 3.x track** as of 2026-04 — do not bump above 2.x without checking nuget.org. |

## API testing

| Package | Version | Notes |
|---|---|---|
| `Microsoft.AspNetCore.Mvc.Testing` | **MUST match** the test project's target framework: `net10.0` → `10.0.x`, `net9.0` → `9.0.x`, `net8.0` → `8.0.x`. Mismatch causes silent ref-pack failures. | `WebApplicationFactory<T>`. |
| `Microsoft.AspNetCore.TestHost` | Same matching rule as above. | `TestServer`. |
| `Alba` | `8.1.1+` | Only for the Alba variant. |
| `Refit` + `Refit.HttpClientFactory` | `8.0.0+` | Typed HTTP clients. |
| `Xunit.DependencyInjection` | `8.9.1+` (for v2) | Only for the Xunit.DI variant. **Not yet compatible with xUnit v3** — stay on xUnit v2 if using this. |
| `Testcontainers` + `Testcontainers.DynamoDb` / `Testcontainers.PostgreSql` / etc. | `4.0.0+` | Only when using a real dependency instead of a fake. |

## Assertions

| Package | Version | Notes |
|---|---|---|
| `AwesomeAssertions` | `9.3.0+` | A community fork of FluentAssertions that restored the pre-license-change behaviour. |
| `FluentAssertions` | `7.2.0+` | Acceptable if the project already uses it. Do not introduce alongside AwesomeAssertions. |
| `Shouldly` | latest | Acceptable alternative; pick one assertion library per project and stay consistent. |

## Test data

| Package | Version | Notes |
|---|---|---|
| `AutoFixture` | `4.18.1+` | Random data; seed as needed. |
| `AutoBogus` / `Bogus` | `2.13.1+` / `34.0.2+` | Fluent domain builders. |

## xUnit v2 vs v3 today

- **v3 if**: greenfield project, Lambda, no `Xunit.DependencyInjection` need.
- **v2 if**: you need `Xunit.DependencyInjection` (the Pattern D shape) — it has not shipped a v3-compatible release yet.

Check the current status of `Xunit.DependencyInjection` before committing to v2; v3 support is tracked upstream.

## Shared test-helpers libraries

Some organisations extract fakes, ID generators, and request builders into a shared NuGet package reused across services. Do not introduce one as part of scaffolding — consolidating test helpers across teams is a bigger decision than this skill should make. The duplication across teams is a deliberate trade-off in most setups.
