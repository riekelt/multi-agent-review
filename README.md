# multi-agent-review

[![skills.sh](https://skills.sh/b/riekelt/multi-agent-review)](https://skills.sh/riekelt/multi-agent-review)

A Claude Code, Codex, and Cursor skill that reviews **specs and plans, not code**. Six review agents cover three topics before you execute, and a reasoning-tier juror is invoked only when the model tiers disagree. The gate is **fail-closed**: a dead juror promotes contested findings to BLOCKER, a broken panel halts, and an empty panel never yields a clean verdict. Use it when you want a plan reviewed before execution, a spec checked before planning, or a second opinion before you build.

## Install

### Claude Code, plugin manager

```
/plugin marketplace add riekelt/multi-agent-review
/plugin install multi-agent-review@multi-agent-review
```

### Claude Code, manual install

The plugin lives in the `plugins/multi-agent-review/` subdirectory, not at the repo root:

```bash
git clone https://github.com/riekelt/multi-agent-review /tmp/multi-agent-review
cp -r /tmp/multi-agent-review/plugins/multi-agent-review ~/.claude/plugins/multi-agent-review
```

### Codex

Clone the repo and copy the plugin directory into your Codex plugins directory:

```bash
git clone https://github.com/riekelt/multi-agent-review /tmp/multi-agent-review
cp -r /tmp/multi-agent-review/plugins/multi-agent-review <your-codex-plugins-dir>/multi-agent-review
```

Codex reads the plugin metadata and the model tiers from `.codex-plugin/plugin.json` inside the copied directory.

### Cursor

Clone the repo and copy the plugin directory into your Cursor plugins directory:

```bash
git clone https://github.com/riekelt/multi-agent-review /tmp/multi-agent-review
cp -r /tmp/multi-agent-review/plugins/multi-agent-review <your-cursor-plugins-dir>/multi-agent-review
```

Cursor reads the plugin metadata and the model tiers from `.cursor-plugin/plugin.json` inside the copied directory.

## Usage

```
/multi-agent-review spec    # after brainstorming, before writing-plans
/multi-agent-review plan    # after writing-plans, before subagent-driven-development
```

Six agents run in parallel: a fast and a standard tier each review **completeness**, **alignment**, and **risk**. If the two tiers disagree on a finding, a single reasoning-tier juror adjudicates. If they agree, no juror is invoked.

The verdict sorts findings into **Blockers**, **Warnings**, and **Observations**, then gates the next step:
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

Without Superpowers, invoke it directly before any plan execution: pass an explicit path argument, since the default artifact discovery assumes the Superpowers layout (`docs/superpowers/specs/`, `docs/superpowers/plans/`). That coupling is deliberate: this skill is a slot in that workflow, and the auto-invoked next skills (`writing-plans`, `subagent-driven-development`) are Superpowers skills by name. If Superpowers renames its paths or skills, Step 1 and Step 9 of `plugins/multi-agent-review/skills/multi-agent-review/SKILL.md` are the two places to update.

## Project-specific rules

Copy `plugins/multi-agent-review/skills/multi-agent-review/assets/project-rules.example.md` to your project root as `project-rules.md`. The skill injects it into the alignment and risk agents. Without it, the skill falls back to the project's `CLAUDE.md` or `AGENTS.md` and applies only the parts that read as standing constraints on specs and plans.

## Flags

| Flag | Effect |
|---|---|
| `--fast` | Fast-tier only: three agents, no cross-model check, no juror; every finding is accepted at its emitted severity. Safety limit: the skill refuses `--fast` when the artifact mentions authentication, authorization, security, secrets, payment, billing, migration, data deletion, or production infrastructure. It runs the full panel instead. |

## Panel failure semantics

| Failure | Behaviour |
|---|---|
| < 3 valid agent reports | Halt: no verdict produced |
| 3 to 5 valid reports | Continue with panel-health WARNING |
| Juror error/timeout | Conservative fallback: all contested findings become BLOCKER |
| All agents fail | Halt: an empty panel never triggers a clean verdict |

Both fail-closed paths have run in operation on Claude Code and Codex: the juror fallback promoting contested findings to BLOCKER, and the quorum halt on too few valid agent reports (operator-verified, 2026-08-19).

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
  evals/evals.json                     Persisted review test prompts
  evals/trigger-evals.json             Skill-routing trigger measurements
  evals/fixtures/                      Planted-defect artifacts the evals review
```

## Model tiers

Configured per platform in that platform's `plugin.json` under `models`. Claude Code: `haiku` / `sonnet` / `opus`. Codex: `gpt-5.6-luna` / `gpt-5.6-terra` / `gpt-5.6-sol`. Cursor: `claude-haiku` / `claude-sonnet` / `claude-opus`.

## License

MIT
