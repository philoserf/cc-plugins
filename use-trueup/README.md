# use-trueup

A Claude Code plugin that detects drift between implementation and spec before committing.

When building software with an AI coding agent, decisions get made during implementation that never flow back to the spec. "Use a dict instead of Redis." "Skip pagination for now." "Add retry with exponential backoff." These choices happen in conversation or in code, and the spec silently becomes a lie.

use-trueup fills the gap between "start building" and "commit" by surfacing those decisions and reconciling them with the spec.

## What It Does

1. **Detect drift** — reads staged diffs and compares against spec files to identify implementation decisions not captured in the spec
2. **Review decisions** — presents each decision for explicit user action: approve, reject, edit, or skip
3. **Reconcile specs** — approved decisions are edited directly into the spec as natural requirements (no metadata markers)
4. **Coverage check** — reports which spec requirements have no corresponding test

## Usage

```text
/use-trueup:use-trueup
/use-trueup:use-trueup coverage
```

The skill triggers naturally via superpowers' 1% rule when you're about to commit, or invoke it directly to ask "is the spec still accurate?"

## Origin

use-trueup is a lightweight reimagining of [plumb-dev](https://pypi.org/project/plumb-dev/), a Python library and CLI tool that keeps specs, tests, and code in sync using DSPy programs and the Anthropic Claude SDK. Where plumb-dev uses external LLM calls, git hooks, and a full decision pipeline, use-trueup does the same core job — drift detection and spec reconciliation — as a single Claude Code skill file with no external dependencies. The skill _is_ the LLM.

## License

MIT
