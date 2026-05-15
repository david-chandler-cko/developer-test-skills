# developer-test-skills

A Claude Code plugin bundling two skills for writing **outside-in black-box developer tests**:

| Skill | Purpose |
|---|---|
| [`outside-in-testing`](skills/outside-in-testing/SKILL.md) | Language-agnostic methodology: principles, decision rules, pitfalls. |
| [`dotnet-developer-tests`](skills/dotnet-developer-tests/SKILL.md) | .NET scaffolder: creates `*.Developer.Tests` projects, scenarios, fakes, and `httpclient-interception` setup. |

The methodology in short: drive one service end-to-end through its real entrypoint (HTTP, Lambda, message consumer, port). In-process dependencies run for real, but anything that crosses an expensive boundary (a network call, database query, etc.) is replaced with a test double.

The approach is based on https://checkout.atlassian.net/wiki/spaces/MSE/pages/5611225699/Developer+Testing+approach

## Install

This repo is packaged as a Claude Code plugin. Install it from any Claude Code session:

```
/plugin marketplace add david-chandler-cko/developer-test-skills
/plugin install developer-test-skills@developer-test-skills
```

The first command registers the marketplace defined in `.claude-plugin/marketplace.json`; the second installs the plugin (`<plugin>@<marketplace>`). Both skills are auto-discovered from `skills/` on activation.

### Verify

In the same session, look for the skills in the available-skills list. You should see:

- `developer-test-skills:outside-in-testing`
- `developer-test-skills:dotnet-developer-tests`

(Skills installed via a plugin are namespaced as `<plugin>:<skill>`.)

## Use

Skills auto-trigger on matching phrases in your prompt — you don't usually need to invoke them by name.

**Methodology / review questions** — triggers `outside-in-testing`:

- "Review this developer test"
- "How should I isolate fakes between parallel tests?"
- "Should I mock this dependency?"

**.NET code generation** — triggers `dotnet-developer-tests`:

- "Scaffold a Developer.Tests project for this Lambda"
- "Add a `When<Situation>.cs` scenario for the refund endpoint"
- "Add a fake for `IProcessorRepository`"
- "Set up httpclient-interception"

For .NET work, the scaffolder reads the methodology skill first, then either runs the **Scaffold flow** (greenfield project — asks about Lambda/API/ports-and-adapters, host framework, isolation strategy) or the **Maintain flow** (existing project — adds scenarios/fakes/interceptions, or audits against the conformance checklist).
