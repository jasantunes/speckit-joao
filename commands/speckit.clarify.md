---
description: "Identify underspecified areas in the current feature spec by asking up to 5 highly targeted, requirements-only clarification questions and encoding answers back into the spec."
compatibility: "Requires spec-kit project structure with .specify/ directory"
metadata:
  author: "Joao Antunes"
  source: "joao preset override for speckit.clarify"
---

## User Input

```text
$ARGUMENTS
```

You **MUST** consider the user input before proceeding (if not empty).

## Pre-Execution Checks

**Check for extension hooks (before clarification)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.before_clarify` key.
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

## Clarification Boundary

Clarify only what must be true about the system:

- the problem being solved and desired outcomes
- required system behavior and user-visible results
- scope boundaries and explicit out-of-scope decisions
- user, dependent-system, and operational expectations
- external contracts and compatibility commitments
- measurable non-functional requirements
- security, privacy, and compliance requirements
- preserve-existing-behavior expectations
- acceptance criteria and other testable completion signals

Defer to planning:

- component, class, service, module, file, or ownership decisions
- where logic lives or when it runs internally
- sequencing, phase placement, and execution mechanics
- retries, fallback, reselection, and recovery mechanics
- abstraction choice, module boundaries, and decomposition
- persistence design, schema shape, or storage approach unless externally mandated
- rollout mechanics and operational implementation tactics
- detailed test implementation strategy or fixture structure

Implementation detail is allowed only when it is already established and must be preserved as a requirement rather than chosen as a design preference. Valid sources are:

- the original user input
- repo facts that establish current behavior or compatibility obligations
- external contracts, regulatory requirements, or compliance mandates

## Outline

Goal: Detect and reduce requirements-phase ambiguity or missing decision points in the active feature specification and record the clarifications directly in the spec file without drifting into implementation or architecture design.

Note: This clarification workflow is expected to run (and be completed) before invoking `/speckit.plan`. If the user explicitly states they are skipping clarification (for example, an exploratory spike), you may proceed, but must warn that downstream rework risk increases.

Execution steps:

1. Run `.specify/scripts/bash/check-prerequisites.sh --json --paths-only` from repo root **once**. Parse minimal JSON payload fields:
   - `FEATURE_DIR`
   - `FEATURE_SPEC`
   - optionally capture `IMPL_PLAN` and `TASKS` for future chained flows
   - if JSON parsing fails, abort and instruct the user to re-run `/speckit.specify` or verify the feature branch environment
   - for single quotes in args like "I'm Groot", use escape syntax: `'\''` or double quotes

2. Load the current spec file. Perform a structured ambiguity and coverage scan using this taxonomy. For each category, mark status: Clear / Partial / Missing. Produce an internal coverage map used for prioritization.

   Functional Scope and Behavior:
   - core user goals and success criteria
   - explicit out-of-scope declarations
   - user roles and persona differentiation

   Domain and Data Model:
   - entities, attributes, relationships
   - identity and uniqueness rules
   - lifecycle and state transitions
   - data volume or scale assumptions

   Interaction and UX Flow:
   - critical user journeys and sequences
   - error, empty, and loading states
   - accessibility or localization notes

   Non-Functional Quality Attributes:
   - performance targets
   - scalability limits
   - reliability and availability expectations
   - observability requirements
   - security and privacy requirements
   - compliance or regulatory constraints

   Integration and External Dependencies:
   - external services or APIs and their failure modes
   - data import or export formats
   - protocol or versioning assumptions

   Edge Cases and Failure Handling:
   - negative scenarios
   - rate limiting and throttling
   - conflict resolution

   Constraints and Tradeoffs:
   - technical constraints that are externally imposed or already committed
   - explicit tradeoffs or rejected alternatives that affect required behavior

   Terminology and Consistency:
   - canonical glossary terms
   - avoided synonyms or deprecated terms

   Completion Signals:
   - acceptance criteria testability
   - measurable definition-of-done indicators

   Miscellaneous and Placeholders:
   - TODO markers or unresolved decisions
   - ambiguous adjectives lacking quantification

   For each category with Partial or Missing status, add a candidate question opportunity only if:
   - the clarification materially changes behavior, validation, compatibility, or acceptance criteria
   - the answer is a requirements-phase decision rather than a planning or design choice
   - repo facts are used only for current-state summaries, compatibility constraints, or preserve-existing-behavior framing, never to invent new desired behavior

3. Generate an internal prioritized queue of candidate clarification questions with a maximum of 5 total questions across the whole session. Apply these constraints:
   - each question must be answerable with either:
     - a short multiple-choice selection with 2 to 5 distinct, mutually exclusive options, or
     - a one-word or short-phrase answer with an explicit `<=5 words` constraint
   - only include questions whose answers materially impact required behavior, outcomes, constraints, compatibility, operational expectations, or acceptance criteria
   - exclude plan-level execution details unless they are true constraints already established by the user, repo facts, or external obligations
   - avoid unsupported inferred behavior; if the spec lacks evidence for a desired behavior, ask instead of assuming
   - if more than 5 categories remain unresolved, select the top 5 by impact times uncertainty

4. Sequential questioning loop:
   - present exactly one question at a time
   - for multiple-choice questions:
     - analyze all options and determine the most suitable option based on best practices, common patterns, risk reduction, and alignment with explicit spec goals
     - present your recommendation prominently at the top with 1 to 2 sentences of reasoning
     - format as `**Recommended:** Option [X] - <reasoning>`
     - render options as a Markdown table
     - after the table, add `You can reply with the option letter (for example, "A"), accept the recommendation by saying "yes" or "recommended", or provide your own short answer.`
   - for short-answer questions:
     - provide your suggested answer and brief reasoning
     - format as `**Suggested:** <answer> - <reasoning>`
     - then output `Format: Short answer (<=5 words). You can accept the suggestion by saying "yes" or "suggested", or provide your own answer.`
   - after the user answers:
     - if the user replies with `yes`, `recommended`, or `suggested`, use the previously stated recommendation or suggestion
     - otherwise, validate the answer maps to one option or fits the `<=5 words` constraint
     - if ambiguous, ask for quick disambiguation and do not advance
     - once satisfactory, record it in working memory and move to the next queued question
   - stop asking further questions when:
     - all critical ambiguities are resolved early
     - the user signals completion
     - you reach 5 asked questions
   - never reveal future queued questions in advance
   - if no valid questions exist at start, immediately report no critical ambiguities

5. Integration after each accepted answer:
   - maintain an in-memory representation of the spec plus the raw file contents
   - for the first integrated answer in this session:
     - ensure a `## Clarifications` section exists
     - under it, create a `### Session YYYY-MM-DD` subheading for today if absent
   - append a bullet line immediately after acceptance: `- Q: <question> → A: <final answer>`
   - then immediately apply the clarification to the most appropriate section:
     - functional ambiguity: update Functional Requirements
     - user interaction or actor distinction: update User Stories or Actors section
     - data shape or entities: update Data Model
     - non-functional constraint: add or modify measurable criteria in Success Criteria
     - edge case or negative flow: update Edge Cases or Error Handling
     - terminology conflict: normalize the canonical term across the spec
   - if the clarification invalidates an earlier ambiguous statement, replace that statement instead of duplicating it
   - preserve formatting and keep each clarification minimal and testable
   - save the spec file after each integration

6. Validation after each write and at the end:
   - clarifications session contains exactly one bullet per accepted answer
   - total accepted questions is at most 5
   - updated sections contain no lingering vague placeholders the new answer was meant to resolve
   - no contradictory earlier statement remains
   - markdown structure remains valid; only new headings allowed are `## Clarifications` and `### Session YYYY-MM-DD`
   - terminology stays consistent across updated sections
   - no inserted text introduces new plan-level ownership, placement, sequencing, retries, fallback mechanics, abstraction choice, storage design, rollout mechanics, or detailed test-structure decisions unless they are proven constraints

7. Write the updated spec back to `FEATURE_SPEC`.

8. Report completion:
   - number of questions asked and answered
   - path to updated spec
   - sections touched
   - a coverage summary table listing each taxonomy category with status: Resolved, Deferred, Clear, or Outstanding
   - if any Outstanding or Deferred remain, recommend whether to proceed to `/speckit.plan` or run `/speckit.clarify` again later
   - suggested next command

Behavior rules:

- If no meaningful ambiguities are found, respond: `No critical ambiguities detected worth formal clarification.` and suggest proceeding.
- If the spec file is missing, instruct the user to run `/speckit.specify` first.
- Never exceed 5 total asked questions.
- Avoid speculative tech stack questions unless the absence blocks functional clarity and is already a real requirement constraint.
- Respect user early termination signals.
- If no questions are asked due to full coverage, output a compact coverage summary and suggest advancing.
- If quota is reached with unresolved high-impact categories remaining, explicitly flag them under Deferred with rationale.

Context for prioritization: $ARGUMENTS

## Post-Execution Checks

**Check for extension hooks (after clarification)**:
- Check if `.specify/extensions.yml` exists in the project root.
- If it exists, read it and look for entries under the `hooks.after_clarify` key.
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
