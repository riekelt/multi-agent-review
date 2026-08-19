# multi-agent-review

[![skills.sh](https://skills.sh/b/riekelt/multi-agent-review)](https://skills.sh/riekelt/multi-agent-review)

A Claude Code + Codex skill that panels six independent review agents across three topics before you execute a plan or finish a spec. When model tiers disagree, a reasoning-tier juror adjudicates. The result is a gated **Blockers / Warnings / Observations** verdict.

Designed to work with the [Superpowers](https://github.com/obra/superpowers) workflow:
**brainstorming → spec → `/multi-agent-review spec` → writing-plans → `/multi-agent-review plan` → subagent-driven-development**

## Install

### Claude Code — via plugin manager

```
/plugin marketplace add riekelt/multi-agent-review
/plugin install multi-agent-review@multi-agent-review
```

### Claude Code — manual

The plugin lives in the `plugins/multi-agent-review/` subdirectory, not at the repo root:

```bash
git clone https://github.com/riekelt/multi-agent-review /tmp/multi-agent-review
cp -r /tmp/multi-agent-review/plugins/multi-agent-review ~/.claude/plugins/multi-agent-review
```

### Codex

Add to your Codex plugins via `plugins/multi-agent-review/.codex-plugin/plugin.json`.

## Usage

```
/multi-agent-review spec    # after brainstorming, before writing-plans
/multi-agent-review plan    # after writing-plans, before subagent-driven-development
```

Six agents run in parallel — haiku and sonnet tiers each review **completeness**, **alignment**, and **risk**. If the two tiers disagree on a finding, a single reasoning-model juror adjudicates. If they agree, no juror is invoked.

The verdict gates the next step:
- **Blockers** → stops execution, presents to operator (fix + re-run, or override with logged note)
- **Warnings** → presents to operator (fix or accept and continue)
- **Clean** → auto-proceeds to next skill

## Superpowers integration

This skill is designed to slot into the [Superpowers](https://github.com/obra/superpowers) / [Superpowers Extended CC](https://github.com/pcvelz/superpowers) workflow:

```
brainstorming skill
  → spec written
  → /multi-agent-review spec          ← this skill
  → writing-plans skill
  → /multi-agent-review plan          ← this skill
  → subagent-driven-development skill
```

Without Superpowers, invoke it directly before any plan execution.

## Project-specific rules

Copy `plugins/multi-agent-review/skills/multi-agent-review/assets/project-rules.example.md` to your project root as `project-rules.md`. The skill injects it into alignment and risk reviewers. Without it, only generic checks run.

## Flags

| Flag | Effect |
|---|---|
| `--fast` | Fast-tier only: three reviewers, no cross-model check, no juror; every finding is accepted at its emitted severity. ⚠ Do not use on plans that touch production-safety paths. |

## Panel failure semantics

| Failure | Behaviour |
|---|---|
| < 3 valid agent reports | Halt — no verdict produced |
| 3–5 valid reports | Continue with panel-health WARNING |
| Juror error/timeout | Conservative fallback — all contested findings → BLOCKER |
| All agents fail | Halt — empty panel never triggers clean verdict |

## Repository layout

```
.claude-plugin/
  marketplace.json     Claude Code marketplace entry

plugins/multi-agent-review/
  .claude-plugin/plugin.json   Claude Code plugin metadata (model tier config)
  .codex-plugin/plugin.json    Codex plugin metadata (model tier config)
  .cursor-plugin/plugin.json   Cursor plugin metadata (model tier config)
  skills/multi-agent-review/
    SKILL.md                           Coordinator
    agents/completeness-reviewer.md    Completeness agent prompt
    agents/alignment-reviewer.md       Alignment agent prompt
    agents/risk-reviewer.md            Risk agent prompt
    agents/synthesis-agent.md          Juror prompt
    assets/project-rules.example.md    Project-specific rules template
  evals/evals.json                     Persisted test prompts and fixtures
```

## Model tiers

Configured per platform in that platform's `plugin.json` under `models`. Claude Code: `haiku` / `sonnet` / `opus`. Codex: `gpt-5.6-luna` / `gpt-5.6-terra` / `gpt-5.6-sol`. Cursor: `claude-haiku` / `claude-sonnet` / `claude-opus`.

## License

MIT
