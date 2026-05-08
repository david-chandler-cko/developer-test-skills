# developer-test-skills

Two Claude Code skills for writing **outside-in black-box developer tests**:

| Skill | Purpose |
|---|---|
| [`outside-in-testing`](outside-in-testing/SKILL.md) | Language-agnostic methodology: principles, decision rules, pitfalls. |
| [`dotnet-developer-tests`](dotnet-developer-tests/SKILL.md) | .NET scaffolder: creates `*.Developer.Tests` projects, scenarios, fakes, and `httpclient-interception` setup. |

The methodology in short: drive one service end-to-end through its real entrypoint (HTTP, Lambda, message consumer, port). In-process dependencies run for real, but anything that crosses an expensive boundary (a network call, database query, etc.) is replaced with a test double.

The approach is based on https://checkout.atlassian.net/wiki/spaces/MSE/pages/5611225699/Developer+Testing+approach

## Install

Skills live in either `~/.claude/skills/` (available everywhere) or `<repo>/.claude/skills/` (scoped to one project).

### Clone into your skills directory

```sh
# user-level (available in every project)
git clone https://github.com/david-chandler-cko/developer-test-skills.git ~/.claude/skills/developer-test-skills

# OR project-level (only this repo)
git clone https://github.com/david-chandler-cko/developer-test-skills.git .claude/skills/developer-test-skills
```

Claude Code discovers any `SKILL.md` under `skills/` recursively, so the nested layout works as-is — no symlinks needed.

### Verify

Start a Claude Code session and run `/skills` (or look for the skills in the session's available-skills list). Both skill names should appear.

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
