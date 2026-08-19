---
name: multi-agent-review
description: Use when a spec or plan needs review before execution begins - "review my spec", "is this plan ready to execute", "check this before I build it", "second opinion on this plan" - and specifically before writing-plans (spec mode) or before subagent-driven-development (plan mode). Reviews specs and plans, not code. Panels six reviewers across two model tiers (cross-vendor configurable) and three topics, invokes a reasoning-tier juror only when the tiers disagree, and gates the next workflow step with a fail-closed Blockers / Warnings / Observations verdict.
---

# Multi-Agent Review

Run a panel of independent reviewers on a spec or plan before execution starts.
Two model tiers (fast + standard) review each of three topics in parallel.
When the tiers disagree, a single reasoning-tier juror adjudicates.
The verdict gates whether the next workflow step proceeds.
Concrete model IDs come from the plugin manifest (Step 3); the tier names below are variables, not model names.

## When to invoke

| Command | Fire after | Proceeds to |
|---|---|---|
| `/multi-agent-review spec` | spec is written and committed | `writing-plans` |
| `/multi-agent-review plan` | plan is written and committed | `subagent-driven-development` |

## Invocation syntax

```
/multi-agent-review [mode] [path?] [--fast]

mode:    spec | plan          (required)
path:    explicit file path   (optional — omit to use most-recent artifact)
--fast:  fast tier only, skip standard tier and juror (cost-saving for fast iteration)
```

If the operator omits the mode, infer it from the artifact's path (`docs/superpowers/specs/` vs `docs/superpowers/plans/`) and confirm the inference before dispatching.

---

## Coordinator steps

### Step 1 — Locate the artifact

**spec mode:**
```bash
ls -t docs/superpowers/specs/*.md | head -1
```
Use the path returned. If a `path` arg was provided, use that instead.

Also check for a matching `docs/design/` subdirectory:
```bash
ls docs/design/ 2>/dev/null
```
If matching design mockups exist, read their filenames — pass them to the alignment reviewer as supplementary context.

**plan mode:**
```bash
ls -t docs/superpowers/plans/*.md | grep -v tasks.json | head -1
```
Also find the linked spec: read the plan file, look for a `Spec:` header line or `planPath`, and load that spec as supplementary context for the alignment reviewer.

### Step 2 - Validate, then read the artifact

**Pre-dispatch validation.** Before spawning anything: the artifact file exists and is non-empty, and every spec, mockup, or companion path it references resolves on disk. A missing or empty artifact burns a six-agent panel on false positives; report it to the operator instead of dispatching.

**Oversized artifacts.** Above roughly 2,000 lines, stop and ask the operator: proceed with the full artifact, or narrow to a named section. Six reviews of a document too large to hold degrade silently; the question costs one turn.

Read the **full** artifact content verbatim. **Never truncate or summarise any part of the artifact before passing it to agents** — an agent that receives an abbreviated spec will flag missing sections as BLOCKERs, producing false positives that poison the verdict. If the file is large, read it in chunks but assemble the full text before building prompts.

### Step 3 — Resolve model tiers, read companion files, read project rules

**Model tier resolution:**

Locate the plugin manifest (do NOT use a bare `.claude-plugin/plugin.json` relative path — that resolves against the user's project, not the plugin install). Try in order until one succeeds:

1. `${CLAUDE_PLUGIN_ROOT}/.claude-plugin/plugin.json` - Claude Code
2. `${CLAUDE_PLUGIN_ROOT}/.codex-plugin/plugin.json` - Codex if it sets this var
3. `${CLAUDE_PLUGIN_ROOT}/.cursor-plugin/plugin.json` - Cursor
4. `<dir of this SKILL.md>/../../.claude-plugin/plugin.json` - walk up two levels
5. `<dir of this SKILL.md>/../../.codex-plugin/plugin.json` - same fallback for Codex
6. `<dir of this SKILL.md>/../../.cursor-plugin/plugin.json` - same fallback for Cursor
7. Hardcoded fallback: `{ "fast": "haiku", "standard": "sonnet", "reasoning": "opus" }`

Read the `"models"` object from the first manifest that loads **and contains a `models` key**. A manifest that loads but has no `models` key does not stop the search - keep trying the next candidate, and use the hardcoded fallback only when the list is exhausted. (Without this rule, a manifest that merely exists resolves the tiers to nothing and Step 5 dispatches with undefined model IDs.)

Quick bash to try options 1-3:
```bash
for m in .claude-plugin .codex-plugin .cursor-plugin; do
  jq -e '.models' "$CLAUDE_PLUGIN_ROOT/$m/plugin.json" 2>/dev/null && break
done
```

Store `FAST_TIER`, `STANDARD_TIER`, `REASONING_TIER` from `models.fast`, `models.standard`, `models.reasoning`. Use these in Step 5 and Step 7 - never hardcode model names.

**Read companion prompt files:**

Read these four files from the `agents/` directory beside this SKILL.md:
- `agents/completeness-reviewer.md`
- `agents/alignment-reviewer.md`
- `agents/risk-reviewer.md`
- `agents/synthesis-agent.md`

Each contains a fenced prompt template. Extract the content inside the outermost ``` fence.

Also check for a `project-rules.md` file in the **project root** (same directory as CLAUDE.md):
```bash
cat project-rules.md 2>/dev/null || echo ""
```
If found, read it into `PROJECT_CONTEXT`. This file is where projects configure their standing rules, safety constraints, and codebase-specific conventions.

If absent, fall back to the project's `CLAUDE.md` (or `AGENTS.md`) rather than reviewing rules-blind, prefixed with this framing so reviewers do not treat working instructions as review criteria: "The following are the project's general working instructions, not purpose-built review rules. Apply only the ones that read as standing constraints on specs and plans; ignore instructions about tooling, workflow, or agent behavior." If neither file exists, `PROJECT_CONTEXT` is the empty string.

### Step 4 — Build the six agent prompts

For each of the three topic prompts (completeness, alignment, risk):
1. Replace `[ARTIFACT_CONTENT]` with the full artifact text.
2. Replace `[MODE_LABEL]` with `"spec"` or `"plan"`.
3. Replace `[PROJECT_CONTEXT]` with the content of `project-rules.md` (or empty string).
4. For the alignment reviewer only — replace `[SUPPLEMENTARY_CONTEXT]` with:
   - spec mode: list of mockup HTML filenames + their paths
   - plan mode: the linked spec content

### Step 5 — Dispatch six agents in parallel

Send all six in a **single message** with parallel Agent tool calls:

```
Agent(completeness-fast):     model=FAST_TIER,     prompt=completeness_prompt
Agent(completeness-standard): model=STANDARD_TIER, prompt=completeness_prompt
Agent(alignment-fast):        model=FAST_TIER,     prompt=alignment_prompt
Agent(alignment-standard):    model=STANDARD_TIER, prompt=alignment_prompt
Agent(risk-fast):             model=FAST_TIER,     prompt=risk_prompt
Agent(risk-standard):         model=STANDARD_TIER, prompt=risk_prompt
```

(Substitute `FAST_TIER` / `STANDARD_TIER` with the actual model IDs resolved in Step 3.)

**Cross-vendor dispatch.** A tier value prefixed `cli:` (for example `"standard": "cli:codex"`) means: dispatch that tier's three reviewers by running the named CLI with the built prompt, instead of the Agent tool. Same prompts, same output contract. This makes cross-model disagreement an independent second opinion rather than a same-vendor capability gap; a fast/standard pair from one vendor still catches real defects, but its disagreements partly measure model size, and the juror inherits that bias. Prefer a cross-vendor standard tier where a second vendor's CLI is available.

**--fast escalation guard.** Before honoring `--fast`, scan the artifact for high-risk markers: authentication, authorization, security, secrets, payment, billing, migration, data deletion, production infrastructure, or anything the project rules mark safety-critical. On a hit, refuse `--fast`, tell the operator which marker triggered the refusal, and run the full panel. The single-tier discount is never available for the work that needs the panel most.

If `--fast` was passed (and not refused): dispatch only the three `FAST_TIER` agents, then skip Steps 6 and 7 entirely and go to Step 8. With one model per topic there is no pair to compare and nothing to adjudicate, so `CONTESTED_LIST` does not exist in this mode.

⚠ **--fast safety note:** `--fast` is a single-tier review: no cross-model check and no juror. Do NOT use `--fast` on anything touching production-safety-critical or irreversible paths. Reserve it for lightweight non-safety artifacts (tooling, docs, UI copy).

**After dispatching, track which agents returned valid reports.** A valid report contains at least one line starting with `FINDINGS:`. Record which agents errored or timed out — this is used in Step 8's quorum check.

### Step 6 — Compare pairs, identify contested findings

**First — check for errored agents.** If a report is missing or malformed (no `FINDINGS:` line), treat that agent as having returned `FINDINGS: ERROR — agent did not respond`. Proceed to the quorum check in Step 8 before comparing.

For each topic pair (haiku report vs sonnet report for that topic):

A finding is **CONTESTED** if ANY of the following:
- Same finding appears in both reports but at different severity levels
- A finding appears in one report but NOT in the other (match by title keywords or detail content)
- One model emitted `FINDINGS: none` but the other emitted findings

Build a `CONTESTED_LIST` with this format for each contested item:
```
TOPIC: completeness | alignment | risk
HAIKU said: [severity] — <exact text> or "not raised"
SONNET said: [severity] — <exact text> or "not raised"
```

### Step 7 — Invoke juror (ONLY if CONTESTED_LIST is non-empty)

If `CONTESTED_LIST` is empty: skip directly to Step 8.

If `CONTESTED_LIST` has entries:
1. Build the juror prompt from `synthesis-agent.md`:
   - Replace `[ALL_SIX_REPORTS]` with the concatenated raw text of all six reports
   - Replace `[CONTESTED_LIST]` with the contested list built in Step 6
2. Dispatch a single juror agent: `model=REASONING_TIER` (resolved in Step 3)
3. Collect `JUROR RULINGS` response

**Juror failure fallback:** If the juror agent errors, times out, or returns a malformed response (no `JUROR RULINGS:` line), do NOT proceed to Step 8. Instead: promote every contested finding to BLOCKER severity (conservative fallback) and present them to the operator as "JUROR FAILED — treating all contested findings as BLOCKER". This is fail-closed: a dead juror cannot let a contested BLOCKER silently degrade.

### Step 8 — Quorum check + compile final verdict

**Quorum check (do this FIRST):**

Count valid reports (those that returned a `FINDINGS:` line, including `FINDINGS: none`).

`--fast` mode quorum: the panel is 3, not 6. Fewer than 2 valid reports = HALT (same message, with N/3); exactly 2 = add the panel-health WARNING; 3 = proceed normally.

Full-panel quorum:

```
if valid_reports < 3:
    HALT — present to operator:
    "⛔ REVIEW ABORTED — only N/6 agents returned valid reports.
     Cannot produce a reliable verdict with fewer than 3 reviewers.
     Options: (a) Re-run, (b) Check for API/rate-limit issues, (c) Override and proceed."
    Do NOT invoke the decision gate.

if valid_reports < 6:
    Add a panel-health WARNING to the verdict:
    [WARNING] Partial panel — N/6 agents responded
      detail: <N> of 6 reviewers failed to return a valid report; coverage gaps possible.
      location: Agent dispatch
```

**Then compile accepted findings:**

`--fast` mode compile rule: there are no pairs and no juror, so accept EVERY finding from every valid report, at the severity the emitting model assigned. This rule exists because the pair-agreement logic below would otherwise accept nothing in `--fast` mode, and an always-empty accepted_findings list silently produces a clean verdict on every run - the exact failure the panel-failure table forbids.

Full-panel compile:

```
accepted_findings = []

For each topic pair:
  For each finding where BOTH models agreed (same title + same severity):
    add to accepted_findings at agreed severity

If juror was invoked:
  For each RULING in JUROR RULINGS:
    add to accepted_findings
  For each SYNTHESISED finding (if any):
    add to accepted_findings

BLOCKERS  = findings with severity BLOCKER
WARNINGS  = findings with severity WARNING
OBS       = findings with severity OBS
```

**Cross-topic deduplication (before bucketing):** the same defect flagged by two topics (matching location and overlapping description) is one finding, kept at the highest severity assigned, annotated with both topics. Without this, one gap double-counts in the verdict and reads as two problems to fix.

### Step 9 — Decision gate

**Alongside every human verdict**, emit a fenced JSON block so downstream automation and the next skill can consume the gate result without parsing prose:

```json
{"verdict": "BLOCKED | WARNINGS | CLEAN | HALTED",
 "blockers": [], "warnings": [], "obs": [],
 "panel_health": [], "iteration": 1}
```

**Iteration cap.** Track the iteration count — first invocation = 1, each
operator-requested re-run increments it. The cap is **3 iterations**.

When `iteration_count >= 3`:
- Do NOT offer "fix + re-run" again.
- Present the current verdict with header:
  > "⛔ REVIEW EXHAUSTED — 3 iterations reached. Showing final findings."
- The operator's only choices are:
  - **(a) Override and proceed** — invoke next skill despite remaining
    blockers (log the override per the override-log rule below).
  - **(b) Abort** — do not invoke the next skill; the artifact is not ready.

Do not loop indefinitely. After 3 rounds we have enough signal — keep
chasing blockers only burns cycles and surfaces stylistic noise.

---

**If BLOCKERS exist:**

Present blockers clearly:
```
⛔ REVIEW BLOCKED — N blocker(s) must be addressed before proceeding.

BLOCKERS:
1. [title] (topic: X, confidence: Y)
   detail: ...
   location: ...
```

Ask the operator:
```
Two options:
  (a) Fix blockers in the artifact now, then I'll re-run the full review.
  (b) Override — acknowledge the blockers and proceed anyway (I'll note the override).
```

**Do NOT auto-proceed.** Wait for operator response.
- If (a): after operator confirms fixes, re-run from Step 1 (full loop).
- If (b): write the override record to stdout AND append a `git note` to the artifact's most-recent commit:
  ```bash
  git notes append -m "MULTI-AGENT-REVIEW OVERRIDE $(date -u +%Y-%m-%dT%H:%M:%SZ): operator acknowledged N blockers: [titles]" HEAD
  ```
  Then invoke next skill. The git note is durable and co-located with the artifact's commit history.

---

**If WARNINGS exist (no blockers):**

```
⚠ REVIEW PASSED WITH WARNINGS — N warning(s).

WARNINGS:
1. [title] (topic: X)
   detail: ...
   location: ...

Fix warnings before proceeding, or accept and continue?
```

Wait for operator response.
- Fix → re-run from Step 1 after operator confirms.
- Continue → invoke next skill.

---

**If clean (no blockers, no warnings):**

```
✅ Review passed — N findings (0 blockers, 0 warnings, M observations).
```

If OBS > 0, list them below the pass line.

Auto-invoke next skill:
- `spec` mode → invoke `writing-plans`
- `plan` mode → invoke `subagent-driven-development`

---

## Panel failure semantics

These rules govern what happens when the panel does not complete cleanly:

| Failure | Behaviour |
|---|---|
| < 3 valid reports | **HALT** — abort, present to operator, do not produce verdict |
| 3–5 valid reports | Add panel-health WARNING, continue with partial coverage |
| 6 valid reports | Proceed normally |
| Juror error/timeout | **Conservative fallback** — promote all contested findings to BLOCKER, present as "JUROR FAILED" |
| All agents error | **HALT** — empty accepted_findings must never silently trigger clean verdict |
| `--fast` mode | Single tier, nothing is contested: every finding from a valid report is accepted at its emitted severity; quorum is 2 of 3; the verdict carries a note that no cross-model adjudication ran |

**Never allow an incomplete panel to produce a clean verdict.** If any doubt, halt and present to operator.

---

## Model assignments

Model IDs are resolved at runtime from the active plugin manifest's `"models"` field.
Default Claude Code values shown; Codex and other platforms override via their own plugin.json.

| Tier variable | CC default | Codex default | Cursor default | Role |
|---|---|---|---|---|
| `FAST_TIER` | `haiku` | `gpt-5.6-luna` | `claude-haiku` | Fast reviewer (always used) |
| `STANDARD_TIER` | `sonnet` | `gpt-5.6-terra` | `claude-sonnet` | Standard reviewer (skipped with `--fast`) |
| `REASONING_TIER` | `opus` | `gpt-5.6-sol` | `claude-opus` | Juror (not overridable: adjudication on a weak model defeats its purpose) |

A tier value prefixed `cli:` (e.g. `"standard": "cli:codex"`) dispatches that tier through the named external CLI per Step 5's cross-vendor rule. Prefer it where a second vendor's CLI is available.

## What this skill does NOT do

- Does not repair artifacts — surfaces findings only; the operator makes changes
- Does not review code — that is `subagent-driven-development`'s job
- Defaults to the most recent spec or plan under `docs/superpowers/`; an explicit path argument overrides this

## Companion files

- `agents/completeness-reviewer.md` - placeholders, missing criteria, undefined refs
- `agents/alignment-reviewer.md` - mockup/codebase consistency, standing rules
- `agents/risk-reviewer.md` - production safety, fail-closed paths
- `agents/synthesis-agent.md` - juror prompt, only dispatched on contested findings
- `assets/project-rules.example.md` - template to copy into your project as `project-rules.md`
