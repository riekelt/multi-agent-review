# Project rules example

Copy the rules below to your project root as `project-rules.md`, then replace
them with your own. Delete everything above the divider. The
`multi-agent-review` skill reads that file and injects it into the alignment
and risk reviewers as `[PROJECT_CONTEXT]`.

---

## Standing rules (alignment reviewer uses these)

- **Source-language rule:** task titles, step descriptions, and commit message
  suggestions must NOT contain phase IDs (Phase N), task IDs (T17), milestone
  labels, or spec section references. These belong in planning docs, not code.
- **No inline hex in component files:** component code in `.tsx`/`.ts` files
  takes colors from CSS classes or design tokens, never from `#rrggbb` literals.
- **No Co-Authored-By trailer:** generated commit messages must omit
  `Co-Authored-By:` lines.
- **Record constructor convention:** new Java records require an explicit
  canonical constructor that calls `Validators.requireValid(this)`.

## Risk rules (risk reviewer uses these)

- **Trading-path fallbacks are BLOCKER-level:** any handler or service that
  catches an exception on a trading-critical path and returns a stale/default
  value instead of propagating the error is a BLOCKER, not a WARNING.
- **DB schema changes require a migration task:** any plan task that adds or
  alters a table or column must include a corresponding Liquibase changeset task.
  Missing it is BLOCKER-level.
- **No hardcoded secrets:** source files and plan steps contain no API keys,
  credentials, or tokens. They read secrets from environment variables or a
  secrets manager.
