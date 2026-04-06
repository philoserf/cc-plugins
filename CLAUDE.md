# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A self-hosted Claude Code plugin marketplace (`philoserf/cc-plugins`). Plugins are installed via `claude plugin marketplace add philoserf/cc-plugins`, then individual plugins via `claude plugin install <name>@cc-plugins`.

Currently ships one plugin: **use-trueup** — detects drift between implementation and spec before committing.

## Repository Layout

- `marketplace.json` — plugin registry (name, version, description for each plugin)
- `use-trueup/` — plugin directory
  - `.claude-plugin/plugin.json` — plugin metadata
  - `design.md` — design spec (kept in sync using the plugin itself)
  - `skills/use-trueup/SKILL.md` — the skill definition (core logic)
  - `skills/use-trueup/references/coverage.md` — sub-command for spec-to-test coverage

## Architecture

Each plugin follows this structure: `.claude-plugin/plugin.json` for metadata, `skills/<skill-name>/SKILL.md` for the skill definition, and optional `references/` for sub-command docs. The skill file IS the implementation — no separate code, no CLI, no external dependencies. Claude reasons about diffs and specs inline.

`marketplace.json` at the repo root indexes all plugins with source paths, versions, and descriptions.

## Adding a New Plugin

1. Create `<plugin-name>/` with `.claude-plugin/plugin.json`, `skills/<skill-name>/SKILL.md`
2. Add an entry to `marketplace.json` with name, source path, description, version, license
3. Add a row to the root `README.md` plugins table

## Version Tracking

Plugin versions live in two places that must stay in sync:

- `<plugin>/`.claude-plugin/plugin.json`→`version` field
- `marketplace.json` → plugin entry `version` field

## Commands

```bash
task              # list available tasks
task validate     # check marketplace.json validity and version sync
task fmt          # format all markdown with prettier
```
