# Claude Code Agent Patterns

Production-grade agent definitions for [Claude Code](https://claude.com/claude-code), demonstrating patterns I've found useful when building Opus-class agents for real work.

This repo is a showcase of **agent authoring patterns**, not a framework. Each agent here is a complete, self-contained prompt you can drop into `.claude/agents/` and use immediately — but the real value is in the patterns they demonstrate.

## Patterns on display

Every agent in this repo uses some combination of:

- **Mode-based task routing** — the agent declares which mode it's operating in (`QUICK` / `DEEP` / `SECURITY` / `ARCHITECTURE`) before executing, so the user knows what they're getting and the agent stays disciplined.
- **Context discovery before action** — agents read project conventions (CLAUDE.md, surrounding code) before producing output. No blind generation.
- **Structured output with severity levels** — 🔴 blocking / 🟠 high / 🟡 medium / 🟢 low — so findings are triageable, not a wall of text.
- **Spec-first execution** — for creative work, the agent returns a mini-spec for confirmation before implementing.
- **Explicit rules of engagement** — what the agent *won't* do (no style nits unless asked, no flattery, no hedging) is as important as what it will do.
- **Sign-off checklists** — each agent has a concrete checklist it runs through before declaring work done.

## Agents

| Agent | Purpose | Modes |
|-------|---------|-------|
| [`code-review-architect`](agents/code-review-architect.md) | Senior code review with rigor | QUICK · DEEP · SECURITY · ARCHITECTURE |

More agents coming — the roadmap lives in [FUTURE.md](#roadmap) at the bottom.

## Usage

Drop any agent file into your project's `.claude/agents/` directory:

```bash
mkdir -p .claude/agents
cp agents/code-review-architect.md .claude/agents/
```

Then invoke it from Claude Code:

```
> Review the staged changes using code-review-architect in SECURITY mode
```

Claude Code will route the task to the agent automatically based on the `description` field in each agent's frontmatter.

## Why these patterns?

I've been building agents for a codebase where the cost of a wrong answer is high — incorrect analysis, missed issues in critical code paths, architectural drift that compounds. The patterns here came out of iteration: things that worked across many different tasks, with many different diff shapes, in many different modes.

The recurring theme: **constrain the agent's shape, not its reasoning**. A mode declaration, a severity scale, a sign-off checklist — these are scaffolding that keeps output consistent without limiting what the agent can discover inside that scaffolding.

## Design principles

Each agent in this repo follows these principles:

1. **One agent, one job.** If an agent has four modes, they're four facets of the same job, not four jobs crammed together.
2. **Discoverable behavior.** The user should be able to read the agent file and predict what the agent will do.
3. **No flattery, no hedging.** Agents that soften bad news waste the user's time. Agents that hedge everything waste it twice.
4. **Fail loud.** If the agent can't do the job (missing context, diff too large, unclear requirements), it says so rather than producing low-confidence output.
5. **Results over process.** The output section is concrete (file:line, severity, suggested fix). Not "I analyzed this and found some concerns."

## Roadmap

Likely next additions:

- `spec-driven-project-architect` — DISCOVER / SPEC / IMPLEMENT / AUDIT modes for taking a requirement from sketch to shipped.
- `api-design-agent` — RESEARCH / DESIGN / STRESS-TEST / DOCUMENT modes for designing and hardening backend APIs.
- `migration-planner` — for rolling schema, framework, or language-version migrations with rollback plans.

Open an issue if there's a pattern you'd like to see demonstrated.

## License

MIT. Use these as starting points, fork them, rewrite them for your codebase. Attribution appreciated but not required.
