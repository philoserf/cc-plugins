# use-trueup Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a Claude Code skill that detects drift between implementation and spec, reviews decisions with the user, and reconciles the spec before commit.

**Architecture:** Single SKILL.md file containing all behavioral instructions. No code, no CLI, no external dependencies. The skill guides Claude Code's native capabilities (read files, edit files, run commands, ask questions) to perform drift detection, decision review, and spec reconciliation. A sidecar JSONL file holds pending/rejected decisions between invocations.

**Tech Stack:** Markdown (SKILL.md), JSONL (sidecar), git (diff commands)

---

## File Structure

- Create: `.claude/skills/use-trueup/SKILL.md` — the skill file
- Create: `.claude/skills/use-trueup/coverage.md` — coverage check instructions (loaded on demand)

---

### Task 1: Skill Frontmatter and Overview

**Files:**

- Create: `.claude/skills/use-trueup/SKILL.md`

- [ ] **Step 1: Create the skill file with frontmatter**

````markdown
---
name: use-trueup
description: Detects drift between implementation and spec before committing. Use when about to commit, after implementation work, or when asking "is the spec still accurate?" Reads staged diffs and compares against superpowers specs to surface decisions that need review.
---

# true-up

Detect decisions made during implementation that aren't in the spec. Review them. Reconcile. Then commit.

## When to Use

- Before committing, after implementation work
- When asked "is the spec still accurate?"
- When superpowers' 1% rule triggers on commit-related activity

## Overview

true-up sits between implementation and commit:

```text
brainstorming → writing-plans → [implementation] → true-up → commit
```
````

It catches choices made in code that the spec doesn't reflect — architecture decisions, behavior changes, data model choices — and ensures the spec stays true before the commit lands.

````

- [ ] **Step 2: Verify frontmatter parses correctly**

Run: `head -4 .claude/skills/use-trueup/SKILL.md`
Expected: valid YAML frontmatter with `name` and `description` fields

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/use-trueup/SKILL.md
git commit -m "feat: create use-trueup skill with frontmatter and overview"
````

---

### Task 2: Spec Discovery Logic

**Files:**

- Modify: `.claude/skills/use-trueup/SKILL.md`

- [ ] **Step 1: Add spec discovery section**

Append to SKILL.md:

````markdown
## Spec Discovery

Find the spec files to reconcile against. Check in this order:

1. If a `.true-up` file exists in the project root, read it as JSON and use `spec_paths`:
   ```json
   {
     "spec_paths": ["docs/spec.md", "docs/design/"]
   }
   ```
````

2. If `docs/superpowers/specs/` exists, use all `*-design.md` files in it.
3. If neither exists, ask the user: "Where are your spec files? I need a path to check implementation decisions against."

Store the resolved spec paths for the rest of this invocation. Do not cache across sessions.

````

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/use-trueup/SKILL.md
git commit -m "feat: add spec discovery logic to use-trueup"
````

---

### Task 3: Drift Detection Flow

**Files:**

- Modify: `.claude/skills/use-trueup/SKILL.md`

- [ ] **Step 1: Add the core drift detection section**

Append to SKILL.md:

```markdown
## Drift Detection

### Step 1: Check for staged changes

Run `git diff --cached --stat`. If empty, tell the user "No staged changes to review" and stop.

### Step 2: Read the full staged diff

Run `git diff --cached` to get the complete diff.

### Step 3: Read all spec files

Read each spec file found during discovery. Hold the full content in context.

### Step 4: Check for pending decisions

If `docs/superpowers/specs/.true-up/decisions.jsonl` exists, read it. Any entries with `"status": "pending"` are carried forward into this review session.

### Step 5: Identify new decisions

Compare the staged diff against the spec content. Look for prescriptive choices in the diff that are NOT already captured in the spec. A "decision" is a choice that affects:

- **System behavior** — what the software does
- **Architecture** — how components relate
- **Data models** — what's stored, how it's structured
- **External interfaces** — APIs, protocols, formats

**Ignore** changes that are only:

- Formatting or style
- Variable or function naming
- Import ordering
- Tooling or editor configuration
- Diagnostic or debug output

For each decision found, determine:

- A plain-English **question** framing the choice (e.g., "How should API responses be cached?")
- The **decision made** in the code (e.g., "In-memory LRU cache with 5-minute TTL")
- Which **spec file** it relates to
- Which **spec section** it belongs in (the nearest relevant header)
- The **diff context** (file and line range)

### Step 6: Deduplicate

If there are pending decisions from the sidecar (Step 4), compare each new decision against them. If a new decision is substantively the same as a pending one (same choice, possibly different wording), drop the new one. Use your judgment — different words for the same choice is a duplicate; a related but distinct choice is not.
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/use-trueup/SKILL.md
git commit -m "feat: add drift detection flow to use-trueup"
```

---

### Task 4: Decision Review Loop

**Files:**

- Modify: `.claude/skills/use-trueup/SKILL.md`

- [ ] **Step 1: Add the review loop section**

Append to SKILL.md:

````markdown
## Decision Review

If no decisions are found (no new decisions and no pending decisions from sidecar), tell the user "No spec drift detected. Ready to commit." and proceed to the Commit section.

Otherwise, present each decision one at a time using this format:

```text
true-up found [N] decision(s) to review.

Decision [X of N]
Question: [question]
Decision made: [decision]
Spec: [spec_file] § [spec_section]
Diff: [file:lines]

How would you like to handle this?
  - Approve — update the spec to reflect this decision
  - Reject — modify the staged code to undo this decision
  - Edit — rewrite the decision before approving
  - Skip — leave pending for next invocation
```
````

Wait for the user's response before proceeding to the next decision.

### Approve

1. Edit the spec file using the Edit tool
2. Insert the decision as a natural requirement in the identified section
3. The spec should read as if it was always written this way — no "Decision:" markers, no metadata, no timestamps
4. If the right section is ambiguous (decision could fit in multiple places), ask the user which section to update
5. Tell the user what you changed: "Updated [spec_file] § [section] to reflect: [decision]"
6. Remove this decision from the sidecar if it was loaded from there

### Reject

1. Ask: "What's the reason for rejecting this?"
2. After getting the reason, modify the staged code to undo the decision
3. Run the project's test command to verify the change doesn't break anything
4. If tests pass: stage the modified files with `git add`, tell the user what changed
5. If tests fail: show the test output, tell the user "Automatic reversal broke tests. Please resolve manually." Write the decision to the sidecar with `"status": "rejected"` and the rejection reason
6. Remove from sidecar if it was a pending decision that's now resolved

### Edit

1. Ask: "How would you rewrite this decision?"
2. Take the user's rewritten text
3. Follow the same flow as Approve, but use the edited text instead of the original

### Skip

1. Write the decision to the sidecar file with `"status": "pending"`
2. Tell the user: "Skipped. Will resurface next time you run true-up."
3. Move to the next decision

**All decisions must be explicitly reviewed.** Never auto-approve. Never skip without the user saying to skip.

````

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/use-trueup/SKILL.md
git commit -m "feat: add decision review loop to use-trueup"
````

---

### Task 5: Sidecar File Management

**Files:**

- Modify: `.claude/skills/use-trueup/SKILL.md`

- [ ] **Step 1: Add sidecar management section**

Append to SKILL.md:

````markdown
## Sidecar File

The sidecar stores pending and rejected decisions between invocations.

**Location:** `docs/superpowers/specs/.true-up/decisions.jsonl`

**Format:** One JSON object per line:

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
  "created_at": "2026-04-06T14:32:00Z"
}
```
````

### Writing to the sidecar

- Create `docs/superpowers/specs/.true-up/` if it doesn't exist
- Generate `id` as `dec-` followed by 8 random hex characters
- Set `created_at` to current ISO 8601 timestamp
- Append one JSON line per decision — never rewrite the whole file

### Reading from the sidecar

- If the file doesn't exist, there are no pending decisions
- Read all lines, parse as JSON
- Only consider entries with `"status": "pending"` as active
- Entries with `"status": "rejected"` are kept for reference but not re-presented

### Cleaning the sidecar

- After a decision is approved or edited (spec updated), remove its line from the sidecar
- After all decisions are resolved and the commit lands, delete the sidecar file if it's empty

### Gitignore

On first invocation, check that `docs/superpowers/specs/.true-up/` is gitignored. If not, append this line to the project's `.gitignore`:

```text
# true-up working state
docs/superpowers/specs/.true-up/
```

````

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/use-trueup/SKILL.md
git commit -m "feat: add sidecar file management to use-trueup"
````

---

### Task 6: Commit Completion

**Files:**

- Modify: `.claude/skills/use-trueup/SKILL.md`

- [ ] **Step 1: Add commit section**

Append to SKILL.md:

```markdown
## Commit

After all decisions have been reviewed (none remaining):

1. If any spec files were edited during this session, stage them: `git add [spec files]`
2. Tell the user: "All decisions reviewed. Spec is up to date. Ready to commit."
3. Proceed with the commit the user originally intended

If any decisions were skipped, they remain in the sidecar. Warn the user:
"[N] decision(s) were skipped and will resurface next time. Proceeding with commit."

The user decides whether to commit with skipped decisions or go back and resolve them.
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/use-trueup/SKILL.md
git commit -m "feat: add commit completion flow to use-trueup"
```

---

### Task 7: Coverage Check

**Files:**

- Create: `.claude/skills/use-trueup/coverage.md`
- Modify: `.claude/skills/use-trueup/SKILL.md`

- [ ] **Step 1: Create the coverage check document**

````markdown
# true-up: Coverage Check

Spec-to-test coverage analysis. Invoked via `/use-trueup coverage` or when the user asks "what's missing?"

## Process

### Step 1: Read all spec files

Use the same discovery logic as the main skill (see Spec Discovery in SKILL.md).

### Step 2: Extract requirements

For each spec file, identify individual testable requirements. A requirement is an atomic statement of behavior — something that could have exactly one test. Skip headings, context paragraphs, and non-behavioral content.

### Step 3: Find test files

Locate test files in the project. Check common locations:

- `tests/`
- `test/`
- `__tests__/`
- Files matching `*_test.*`, `*.test.*`, `*.spec.*`

### Step 4: Match requirements to tests

For each requirement, check if a test covers it:

1. **Marker match:** Look for requirement markers in test files (e.g., `# req: <text>` or test names that reference the requirement)
2. **Content match:** If no markers, reason about whether any existing test exercises the requirement based on test names and assertions

### Step 5: Report

Present a summary:

```text
Spec-to-Test Coverage: [covered]/[total] requirements ([percentage]%)

Covered:
  ✓ [requirement text] — [test file:test name]
  ✓ [requirement text] — [test file:test name]

Not covered:
  ✗ [requirement text] (from [spec file] § [section])
  ✗ [requirement text] (from [spec file] § [section])
```
````

Do not generate tests. Do not offer to generate tests. Report the gaps and let the user decide what to do. TDD owns test creation.

````

- [ ] **Step 2: Add coverage reference to SKILL.md**

Append to SKILL.md:

```markdown
## Coverage Check

When invoked as `/use-trueup coverage` or when the user asks about spec coverage, read and follow the instructions in `coverage.md` in this skill directory.
````

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/use-trueup/SKILL.md .claude/skills/use-trueup/coverage.md
git commit -m "feat: add spec-to-test coverage check to use-trueup"
```

---

### Task 8: Error Handling and Edge Cases

**Files:**

- Modify: `.claude/skills/use-trueup/SKILL.md`

- [ ] **Step 1: Add error handling section**

Append to SKILL.md:

```markdown
## Error Handling

- **No spec files found:** Tell the user. Ask where their specs live. Do not proceed without specs.
- **No staged changes:** Tell the user "No staged changes to review." Stop.
- **No drift detected:** Tell the user "No spec drift detected. Ready to commit." Proceed.
- **Sidecar directory missing:** Create `docs/superpowers/specs/.true-up/` silently.
- **Ambiguous section placement:** Ask the user which spec section the decision belongs in. Never guess.
- **Spec file has been deleted:** If a pending decision references a spec file that no longer exists, tell the user and ask whether to drop the decision or point it at a different spec.

## Rules

- Never auto-approve decisions. Every decision needs explicit user action.
- Never edit `.claude/memory/` — design decisions belong in the spec, not in memory.
- Never modify superpowers skill files or workflow.
- Never write to `~/.claude/` global directories.
- Never generate tests — flag coverage gaps only.
- The spec should read naturally after edits. No "Decision:" prefixes, no metadata markers, no timestamps in the spec.
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/use-trueup/SKILL.md
git commit -m "feat: add error handling and rules to use-trueup"
```

---

### Task 9: Final Review and Verification

**Files:**

- Read: `.claude/skills/use-trueup/SKILL.md`
- Read: `.claude/skills/use-trueup/coverage.md`

- [ ] **Step 1: Read the complete SKILL.md and verify against spec**

Read the full SKILL.md. Check each requirement from `docs/superpowers/specs/2026-04-06-true-up-design.md`:

| Spec requirement                                           | Covered in SKILL.md?                |
| ---------------------------------------------------------- | ----------------------------------- |
| Detect drift (diff → decisions)                            | Drift Detection section             |
| Decision review (approve, reject, edit, skip)              | Decision Review section             |
| Spec reconciliation (edit spec on approval)                | Approve subsection                  |
| Spec-to-test coverage reporting                            | Coverage Check + coverage.md        |
| Spec discovery (superpowers default + config)              | Spec Discovery section              |
| Sidecar file (JSONL, pending/rejected only)                | Sidecar File section                |
| Decision filtering (behavior/architecture/data/interfaces) | Drift Detection Step 5              |
| Deduplication against pending decisions                    | Drift Detection Step 6              |
| Gitignore the sidecar                                      | Sidecar File / Gitignore subsection |
| No auto-approve                                            | Rules section                       |
| No test generation                                         | Rules section                       |
| No memory overlap                                          | Rules section                       |
| Error handling (no specs, no changes, ambiguous section)   | Error Handling section              |

- [ ] **Step 2: Read coverage.md and verify completeness**

Read coverage.md. Confirm it covers: spec reading, requirement extraction, test file discovery, matching, and reporting format.

- [ ] **Step 3: Verify frontmatter is valid**

Run: `head -4 .claude/skills/use-trueup/SKILL.md`
Expected: valid YAML with `name: use-trueup` and `description` field

- [ ] **Step 4: Final commit if any fixes were needed**

```bash
git add .claude/skills/use-trueup/SKILL.md .claude/skills/use-trueup/coverage.md
git commit -m "chore: final review polish for use-trueup skill"
```
