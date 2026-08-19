# multi-agent-review

[![skills.sh](https://skills.sh/b/riekelt/multi-agent-review)](https://skills.sh/riekelt/multi-agent-review)

A Claude Code + Codex skill that reviews **specs and plans, not code**: six review agents across three topics before you execute, a reasoning-tier juror invoked only when the model tiers disagree, and a **fail-closed** gate: a dead juror promotes contested findings to BLOCKER, a broken panel halts, and an empty panel never yields a clean verdict. The result is a gated **Blockers / Warnings / Observations** verdict. Use it when you want a plan reviewed before execution, a spec checked before planning, or a second opinion before you build.

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

Six agents run in parallel: a fast and a standard tier each review **completeness**, **alignment**, and **risk**. If the two tiers disagree on a finding, a single reasoning-model juror adjudicates. If they agree, no juror is invoked. A tier can be pointed at another vendor's CLI (see Model tiers), which turns cross-tier disagreement into a genuinely independent second opinion instead of a same-vendor capability gap.

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

Without Superpowers, invoke it directly before any plan execution: pass an explicit path argument, since the default artifact discovery assumes the Superpowers layout (`docs/superpowers/specs/`, `docs/superpowers/plans/`). That coupling is deliberate: this skill is a slot in that workflow, and the auto-invoked next skills (`writing-plans`, `subagent-driven-development`) are Superpowers skills by name. If Superpowers renames its paths or skills, Step 1 and the decision gate are the two places to update.

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

The fail-closed paths are not theoretical: they have been exercised in operation on Claude Code and Codex (operator-verified, 2026-08-19). Contrast with the common pattern elsewhere of degrading gracefully to fewer reviewers, which fails open.

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

A tier value prefixed `cli:` dispatches that tier through an external CLI instead of the platform's own models, e.g. `"standard": "cli:codex"` on Claude Code makes the standard tier a second vendor. Recommended where a second vendor's CLI is installed: cross-vendor disagreement is what the juror is for.

## License

MIT
