# Joao Spec Kit Preset

This repository is a single Spec Kit preset that overrides the stock commands:

- `/speckit.clarify`
- `/speckit.plan`

The preset replaces those commands in place. It does not add `speckit.joao.*` commands.

## Install

```bash
specify preset add --dev /Users/jantunes/dev/speckit-joao
```

Note: with the current `specify` CLI, installing a preset from the same project root that already contains `.specify/` may recurse during `--dev` copy. For local verification in this repository, use a filtered copy outside the project root.

## Remove

```bash
specify preset remove joao
```

Use this to uninstall the preset and restore the stock `speckit.clarify` and `speckit.plan` command resolution.

## Commands

### `/speckit.clarify`

Overrides the stock clarify workflow with a requirements-phase, provenance-aware variant:

- validates that an active spec exists
- keeps clarification focused on what must be true about the system
- defers implementation and architecture choices to planning unless they are true constraints
- preserves the stock hook checks, question loop, spec updates, and reporting structure

Use it when clarification should focus on what must be true, not how the implementation should work.

### `/speckit.plan`

Overrides the stock plan workflow with an implementation-focused clarification and planning variant:

- validates that an active spec exists
- blocks when unresolved clarification markers or equivalent readiness issues remain
- keeps the stock planning artifact generation structure
- uses an implementation clarification loop to resolve design-critical decisions
- uses recursive decomposition when implementation decisions are too broad or coupled
- requires visible checkpoints with 3 to 5 alternatives and explicit approval before continuing

Use it when planning should clarify implementation decisions directly, and break them down further only when needed, while still requiring explicit user approval at every decision checkpoint.

## Usage

```text
/speckit.clarify
/speckit.clarify Clarify only functional requirements and acceptance criteria.

/speckit.plan
/speckit.plan Plan this as an implementation clarification loop with recursive decomposition when needed.
```

## Install Verification

After installing, verify the overrides with:

```bash
specify preset list
specify preset info joao
cat .specify/presets/.registry
find .specify/presets/joao/commands -maxdepth 1 -type f | sort
```
