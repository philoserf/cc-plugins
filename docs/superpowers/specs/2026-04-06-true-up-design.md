# true-up: Design Spec

**Version:** 0.1.0

## Problem

When building software with an AI coding agent, decisions get made during implementation that never flow back to the spec. "Use a dict instead of Redis." "Skip pagination for now." "Add retry with exponential backoff." These choices happen in conversation or in code, and the spec silently becomes a lie.

Superpowers solves the bookends — brainstorming captures intent before implementation, code review catches issues after. But the messy middle, where implementation decisions accumulate unnoticed, has no coverage.

## Purpose

true-up detects decisions made during implementation that aren't reflected in the spec, and reconciles them before commit.

## Where It Sits

```text
brainstorming → writing-plans → [implementation] → true-up → commit
                                       ↑                |
                                       TDD              |
                                       code-review      |
                                                        ↓
                                              spec is current
```

true-up fills the gap between "start building" and "commit." It is not part of superpowers — it is a standalone local skill that complements the superpowers workflow.

## What It Is

A single Claude Code skill file, invoked as `/use-trueup`. No separate package, no CLI, no git hook, no external LLM calls. The skill _is_ the LLM — it reasons about diffs and specs inline.

Superpowers' existing "1% rule" in `using-superpowers` triggers it naturally when the user is about to commit, based on the skill description.

## What It Does

### 1. Detect Drift

When invoked, the skill:

1. Reads the staged diff (`git diff --cached`)
2. Locates the relevant spec(s) in `docs/superpowers/specs/`
3. Identifies decisions in the diff that aren't captured in the spec

A "decision" is a prescriptive choice that affects system behavior, architecture, or data models. Not variable names, not import ordering, not formatting. Examples:

- Chose SQLite over Postgres
- Added a 30-minute token expiry
- Skipped input validation on internal endpoints
- Used an in-memory cache instead of Redis

### 2. Review Decisions

Presents each detected decision to the user, one at a time:

```text
true-up found 3 decision(s) in staged changes.

Decision 1 of 3
Question: Should API responses be cached?
Decision made: Added in-memory LRU cache with 5-minute TTL.
Spec section: docs/superpowers/specs/2026-04-01-api-design.md § Caching

How would you like to handle this?
  - Approve — update the spec to reflect this decision
  - Reject — modify the staged code to undo this decision
  - Edit — rewrite the decision before approving
  - Skip — leave for later
```

All decisions require explicit user action. Nothing is auto-approved.

### 3. Reconcile

**Approve:** The skill edits the spec file directly using the Edit tool. The decision becomes a natural requirement in the appropriate section. No "Decision: ..." markers, no metadata in the spec. The spec reads like it was always written that way.

**Reject:** The skill asks for a reason, modifies the staged code to undo the decision, runs tests, and stages the result if tests pass. If tests fail, the user resolves manually.

**Edit:** The user rewrites the decision text. The skill then treats it as an approval with the edited text.

**Skip:** The decision is recorded in the sidecar file and remains pending for the next invocation.

### 4. Commit

Once all decisions are reviewed (none pending), the skill proceeds with the commit.

## Coverage Check

On demand — user asks "what's missing?" or invokes `/use-trueup coverage`.

- Reads all specs in `docs/superpowers/specs/`
- Reads test files
- Reports which spec requirements have no corresponding test
- Uses requirement markers in tests when present, falls back to LLM matching when not

This is spec-to-test coverage only. Code coverage is pytest's job. Test creation is TDD's job. true-up only reports gaps.

## Spec File Discovery

Default: `docs/superpowers/specs/`. The skill looks here first.

If a project uses a different location, the skill checks for a `.true-up` config in the project root:

```json
{
  "spec_paths": ["docs/spec.md", "docs/design/"]
}
```

If neither exists, the skill asks the user where specs live on first invocation.

## Sidecar File

`docs/superpowers/specs/.true-up/decisions.jsonl` — holds pending and rejected decisions only. Approved decisions go directly into the spec and are not retained here.

```json
{
  "id": "dec-<short-uuid>",
  "status": "pending",
  "spec_file": "docs/superpowers/specs/2026-04-01-auth-design.md",
  "spec_section": "## Token Management",
  "question": "Should tokens expire on inactivity or only on logout?",
  "decision": "Tokens expire after 30 minutes of inactivity.",
  "diff_context": "src/auth.py:42-58",
  "rejection_reason": null,
  "created_at": "<ISO timestamp>"
}
```

The sidecar is gitignored. It is working state, not project history. The spec is the record.

## Boundaries

### What true-up owns

- Drift detection (diff → decisions)
- Decision review (present, approve, reject, edit, skip)
- Spec reconciliation (edit spec on approval)
- Spec-to-test coverage reporting

### What true-up does NOT own

- **Test creation** — TDD's job. true-up only flags gaps.
- **Code review** — requesting-code-review / receiving-code-review handle this.
- **Spec authoring** — brainstorming creates specs. true-up only amends them.
- **Implementation planning** — writing-plans handles this.
- **Memory** — design decisions belong in the spec, not in `.claude/memory/`. true-up enforces this by putting approved decisions where they belong.
- **Git hooks** — no hook. The skill is the trigger.
- **External LLM calls** — no SDK, no DSPy, no API keys. The skill reasons inline.

### What true-up does NOT do

- Auto-approve decisions
- Generate tests
- Modify superpowers' workflow or skill files
- Write to global `~/.claude/` directories
- Create separate decision logs as project history

## Decision Filtering

Not every code change is a decision. true-up filters for choices that affect:

- System behavior (what the software does)
- Architecture (how components relate)
- Data models (what's stored, how it's structured)
- External interfaces (APIs, protocols, formats)

It ignores:

- Formatting and style choices
- Variable and function naming
- Import ordering
- Tooling and editor configuration
- Diagnostic findings and debug observations

## Deduplication

When the sidecar contains existing pending decisions, the skill checks new decisions against them before presenting duplicates. Same decision expressed differently should not create noise. The skill handles this through inline reasoning — no separate deduplication pipeline.

## Skill File

The skill is a single file: `.claude/skills/use-trueup/SKILL.md` (project-local) or `~/.claude/skills/use-trueup/SKILL.md` (user-global).

### Skill Description (for superpowers 1% rule triggering)

```text
Detects drift between implementation and spec before committing. Use when
about to commit, after implementation work, or when asking "is the spec
still accurate?" Reads staged diffs and compares against superpowers specs
to surface decisions that need review.
```

## Error Handling

- If no spec files are found, the skill tells the user and asks where they are.
- If no staged changes exist, the skill says so and exits.
- If the sidecar directory doesn't exist, the skill creates it.
- If a spec edit would be ambiguous (decision maps to multiple sections), the skill asks the user which section to update.

## Out of Scope for v0.1.0

- Git hook integration
- Multi-branch decision tracking
- Decision history / audit trail beyond the sidecar
- Spec-to-code coverage
- Code-to-test coverage
- Team / multi-user workflows
- Non-markdown spec formats
- Automatic triggering (requires explicit invocation or superpowers 1% rule)
