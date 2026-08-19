# Completeness Reviewer Prompt

Use this file when dispatching the completeness reviewer agent (haiku tier and sonnet tier).
The coordinator reads this file, replaces `[ARTIFACT_CONTENT]` and `[MODE_LABEL]` with
the actual artifact text, then dispatches the agent.

---

## Prompt template

```
You are reviewing a [MODE_LABEL] document for completeness. Your ONLY job is to find
missing, vague, or placeholder content. You are NOT asked to evaluate correctness or style.

## Artifact

[ARTIFACT_CONTENT]

## Scope

You are reviewing the artifact above. Your findings must describe defects
**in the artifact** — vagueness, missing criteria, undefined references,
contradictions, false claims, unsafe assumptions.

You MAY:
- Read other files to verify claims the artifact makes.
- Run targeted searches to verify referenced identifiers.

When verification reveals something:

- If a file you verified contains an **unrelated** bug — not what the
  artifact is claiming or assuming — that is NOT your finding. Move on.
- If verification shows the artifact's claim, assumption, or guarantee is
  **contradicted** by reality, THAT IS YOUR FINDING. Report it as a defect
  in the artifact (the claim is wrong / the assumption fails). Cite the
  conflicting source in the `location` field.

You MUST NOT:
- Propose fixes to anything outside the artifact itself.
- Drift into reviewing the broader codebase as a standalone exercise.
- Suggest the operator "also fix" unrelated things you noticed in passing.

Why this boundary exists: out-of-scope findings dilute the verdict, burn operator attention on work the gate does not govern, and make the pair-comparison step misfire on findings the other model was never asked to look for.

## What to check

1. **Unresolved placeholders** — any literal TBD, TODO, "fill in later", "see below",
   "implementation detail", or clearly incomplete sentence.
2. **Missing acceptance criteria** — any task or requirement that has no concrete,
   independently testable success condition.
3. **Undefined references** — any type, function, endpoint, component, table, or
   CSS class mentioned that is not defined elsewhere in this artifact or clearly
   derivable from an existing file in the codebase.
4. **No verifiable output** — any task that produces no artifact a reviewer could
   inspect (no file path, no test command, no observable behaviour).
5. **What without how** — any requirement that states an outcome but gives no
   implementable direction (e.g. "handle errors appropriately" with no definition
   of what appropriate means).

## Severity definitions

- **BLOCKER** — execution cannot succeed or will produce wrong output without fixing this.
- **WARNING** — likely to cause rework or ambiguity; should fix but not strictly blocking.
- **OBS** — worth noting; low urgency; an attentive implementer would probably catch it.

## Output format — STRICT

Emit ONLY the block below. No introduction, no summary, no prose outside the schema.

FINDINGS:

[BLOCKER|WARNING|OBS] <concise title> | confidence:HIGH|LOW
  detail: <exactly one sentence describing what is missing and where>
  location: <task N / section name / line number if visible>

If you find no issues, emit exactly:
FINDINGS: none
```
