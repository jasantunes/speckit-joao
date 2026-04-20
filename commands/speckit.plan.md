---
description: "Execute the implementation planning workflow using an implementation clarification loop and recursive decomposition when needed to generate design artifacts."
compatibility: "Requires spec-kit project structure with .specify/ directory"
metadata:
  author: "Joao Antunes"
  source: "joao preset override for speckit.plan"
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before planning)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_plan` key.
- If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally.
- Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
- For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
  - If the hook has no `condition` field, or it is null/empty, treat the hook as executable.
  - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation.
- For each executable hook, output the following based on its `optional` flag:
  - **Optional hook** (`optional: true`):
    ```
    ## Extension Hooks

    **Optional Pre-Hook**: {extension}
    Command: `/{command}`
    Description: {description}

    Prompt: {prompt}
    To execute: `/{command}`
    ```
  - **Mandatory hook** (`optional: false`):
    ```
    ## Extension Hooks

    **Automatic Pre-Hook**: {extension}
    Executing: `/{command}`
    EXECUTE_COMMAND: {command}

    Wait for the result of the hook command before proceeding to the Outline.
    ```
- If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently.

## Outline

1. **Setup**: Run `.specify/scripts/bash/setup-plan.sh --json` from repo root and parse JSON for `FEATURE_SPEC`, `IMPL_PLAN`, `SPECS_DIR`, and `BRANCH`. For single quotes in args like "I'm Groot", use escape syntax such as `'I'\''m Groot'`.

2. **Load context**: Read `FEATURE_SPEC` and `.specify/memory/constitution.md`. Load the `IMPL_PLAN` template that was copied by setup.

3. **Plan-readiness gate**: Inspect the active spec before exploring options. If the spec contains any of the following, stop and direct the user to `/speckit.clarify` first:
   - `[NEEDS CLARIFICATION]` markers
   - unresolved `TODO` or `TBD` placeholders
   - contradictory decisions that materially affect architecture, interfaces, data flow, testing, acceptance criteria, or implementation decomposition
   - open questions that would force later behavior choices

4. **Execute plan workflow** as an implementation clarification loop and recursive decomposition workflow:
   - use implementation clarification to surface the next implementation-critical unknown, decision, or tradeoff that blocks safe planning progress
   - use recursive decomposition when the current implementation decision is still too broad, too coupled, or too risky to answer directly
   - keep the current focus, selected decisions, and completion status visible in the conversation
   - define the current implementation question or decomposed subproblem in terms of inputs, outputs, constraints, and success criteria before making decisions
   - treat each implementation clarification, decomposition choice, strategy choice, restructuring choice, and execution-method choice as a separate controlled checkpoint
   - decompose only when needed and prefer independent subproblems when feasible
   - present exactly 3 to 5 numbered alternatives at every checkpoint
   - include for each alternative: description, pros, and cons
   - recommend exactly one alternative with explicit justification
   - pause after every checkpoint and wait for explicit user approval before continuing
   - if the user's choice is ambiguous, request clarification rather than proceeding
   - do not silently batch multiple decision nodes into one checkpoint
   - do not silently skip checkpoints
   - do not silently introduce new classes, services, abstractions, ownership boundaries, or comparable internal design commitments unless they were covered by an approved checkpoint
   - validate each completed step against success criteria and return to an earlier checkpoint with a new set of alternatives if validation fails or new information invalidates the current implementation framing or decomposition

5. **Follow the structure in the `IMPL_PLAN` template** to:
   - fill Technical Context and mark unknowns as `NEEDS CLARIFICATION`
   - fill Constitution Check from the constitution
   - evaluate gates and error if violations are unjustified
   - Phase 0: generate `research.md` and resolve all `NEEDS CLARIFICATION`
   - Phase 1: generate `data-model.md`, `contracts/`, and `quickstart.md`
   - Phase 1: update agent context by running the agent script
   - re-evaluate Constitution Check post-design
   - continue the standard planning workflow only after all required approvals

6. **Stop and report**: Command ends after Phase 2 planning. Report branch, `IMPL_PLAN` path, and generated artifacts.

7. **Check for extension hooks** after reporting:
   - Check if `.specify/extensions.yml` exists in the project root.
   - If it exists, read it and look for entries under the `hooks.after_plan` key.
   - If the YAML cannot be parsed or is invalid, skip hook checking silently and continue normally.
   - Filter out hooks where `enabled` is explicitly `false`. Treat hooks without an `enabled` field as enabled by default.
   - For each remaining hook, do **not** attempt to interpret or evaluate hook `condition` expressions:
     - If the hook has no `condition` field, or it is null/empty, treat the hook as executable.
     - If the hook defines a non-empty `condition`, skip the hook and leave condition evaluation to the HookExecutor implementation.
   - For each executable hook, output the following based on its `optional` flag:
     - **Optional hook** (`optional: true`):
       ```
       ## Extension Hooks

       **Optional Hook**: {extension}
       Command: `/{command}`
       Description: {description}

       Prompt: {prompt}
       To execute: `/{command}`
       ```
     - **Mandatory hook** (`optional: false`):
       ```
       ## Extension Hooks

       **Automatic Hook**: {extension}
       Executing: `/{command}`
       EXECUTE_COMMAND: {command}
       ```
   - If no hooks are registered or `.specify/extensions.yml` does not exist, skip silently.

## Phases

### Phase 0: Outline and Research

1. Extract unknowns from Technical Context:
   - for each `NEEDS CLARIFICATION`, create a research task
   - for each dependency, create a best-practices task
   - for each integration, create a patterns task

2. Generate and dispatch research agents:

   ```text
   For each unknown in Technical Context:
     Task: "Research {unknown} for {feature context}"
   For each technology choice:
     Task: "Find best practices for {tech} in {domain}"
   ```

3. Consolidate findings in `research.md` using:
   - Decision
   - Rationale
   - Alternatives considered

**Output**: `research.md` with all `NEEDS CLARIFICATION` resolved.

### Phase 1: Design and Contracts

**Prerequisites:** `research.md` complete.

1. Extract entities from the feature spec into `data-model.md`:
   - entity name, fields, relationships
   - validation rules from requirements
   - state transitions if applicable

2. Define interface contracts in `/contracts/` when the project has external interfaces:
   - identify what interfaces the project exposes to users or other systems
   - document the contract format appropriate for the project type
   - skip if the project is purely internal

3. Update agent context:
   - update the plan reference between `<!-- SPECKIT START -->` and `<!-- SPECKIT END -->` markers in `AGENTS.md` to point to the generated plan file

**Output**: `data-model.md`, `/contracts/*`, `quickstart.md`, and updated agent context.

## Key Rules

- Use absolute filesystem paths for operations and project-relative paths for references in documentation and agent context.
- Error on gate failures or unresolved clarifications.
- Do not continue past any decision node without explicit user approval.
