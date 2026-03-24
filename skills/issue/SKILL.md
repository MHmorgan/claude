---
name: issue
description: Create or edit GitHub issues through an interactive interview process
disable-model-invocation: true
user-invocable: true
arguments: "<issue_number_or_description>"
---

# Issue Skill

Help the user plan tasks by creating or editing GitHub issues through a structured interview process.

## Usage

- `/issue <description>` — create a new issue, using the provided description as the initial description
- `/issue 123` — edit existing issue #123

Determine which flow to use by checking whether the argument is a number (edit) or text (create).

## Flow: Create New Issue (`/issue <description>`)

1. **Seed** `ISSUE.md` with the user-provided description as a starting point, using the target structure below as a skeleton.
2. **Interview** the user to flesh out all relevant details (see Interview Phase below).
3. **Review** the completed issue with three parallel agents (see Review Phase below).
4. **Triage** review findings (see Triage & Resolution below). If there are open decisions or ambiguities, return to an interview focused on resolving them, then update `ISSUE.md`.
5. **When the user is satisfied**, create the issue:
   ```
   gh issue create --title "TITLE" --body-file "ISSUE.md"
   ```

## Flow: Edit Existing Issue (`/issue 123`)

1. **Fetch** the existing issue:
   ```
   gh issue view 123 --json title,body
   ```
2. **Write** the current body to `ISSUE.md` and present a summary to the user.
3. **Interview** the user about what needs to change (see Interview Phase below).
4. **Review** the updated issue with three parallel agents (see Review Phase below).
5. **Triage** review findings (see Triage & Resolution below). If there are open decisions or ambiguities, return to an interview focused on resolving them, then update `ISSUE.md`.
6. **When the user is satisfied**, update the issue:
   ```
   gh issue edit 123 --title "TITLE" --body-file "ISSUE.md"
   ```

---

## Target Structure for ISSUE.md

Guide the interview toward producing these sections. Not every section is required — skip what isn't relevant, but always produce **Problem**, **Desired behavior**, and **Acceptance criteria** at minimum. The **Implementation notes** section is populated during the Review Phase, not the interview.

```markdown
# <Title>

## Problem
What's wrong or missing today? What is the observable symptom or unmet need?

## Desired behavior
What should happen when this is done? Describe from the user's or caller's perspective.

## Acceptance criteria
- [ ] Concrete, testable statement
- [ ] Another concrete, testable statement
(Each must be independently verifiable. Prefer observable outcomes over implementation details.)

## Scope
What is explicitly NOT part of this task. Boundaries that the implementer should respect.

## Constraints
Known technical boundaries: must integrate with X, cannot change Y,
must remain backwards-compatible with Z, performance requirements, etc.

## Context
Relevant code paths, modules, prior discussion, related issues.
Anything that helps the implementer orient quickly.

## Implementation notes
(Populated by review agents — not part of the interview.)

### Software considerations
<Idiomatic patterns, design decisions, simplification opportunities>

### Architecture considerations
<Structural impact, module boundaries, integration points>

### Agent workflow recommendation
<Suggested agentic approach for /go: parallelization strategy, subagent vs agent team, etc.>
```

---

## Interview Phase

### Section-by-section interview guidance

Work through these in roughly this order, but follow the natural conversation — don't force a rigid sequence.

1. **Problem & Desired behavior**: Start here. The user's initial description usually covers this partially. Ask clarifying questions until you could explain the task to a stranger.
2. **Acceptance criteria**: This is the most important section for downstream automation. Push for concrete, testable statements. Transform vague requirements ("it should be fast") into specific ones ("response time under 200ms for the p95 case"). Each criterion should be a clear pass/fail check.
3. **Scope**: Ask "Is there anything that might seem related but should NOT be part of this task?" This prevents scope creep during implementation.
4. **Constraints**: Ask about integration points, backwards compatibility, and anything the implementer must not break.
5. **Context**: Ask about relevant files, modules, or prior art in the codebase. Even rough pointers ("somewhere in the auth module") help the implementation agent orient.

### Interview guidelines

- Use the `AskUserQuestion` tool to ask each interview question. Do not use plain text output for questions.
- Ask one question at a time. Do not overwhelm with multiple questions.
- Keep questions focused on understanding the problem and producing a clear description.
- Update `ISSUE.md` incrementally so the user can see progress and correct course.
- Use plain, direct language in the issue body — no filler.
- The issue title should be concise and specific.
- Do not ask about labels, assignees, priority, or project metadata — just the problem description.
- When writing acceptance criteria, prefer observable outcomes ("API returns 404 when resource not found") over implementation prescriptions ("add a null check in the handler"). Let the implementer decide *how*.
- If the user provides implementation hints or preferences, capture them in **Constraints** or **Context** — not as acceptance criteria.

NEVER stop the interview without asking the user if we are finished, even if you have more questions.
ALWAYS CONFIRM WITH THE USER BEFORE STOPPING THE INTERVIEW.

---

## Review Phase

After the user confirms the interview is complete, launch THREE review agents in parallel. Each receives the current contents of `ISSUE.md` and has read access to the codebase.

### 1. Software Review (Agent subagent_type="general-purpose")

Provide: contents of `ISSUE.md` + instruction to explore relevant parts of the codebase.

The agent should:
- Read the code areas referenced in the Context section (and discover adjacent code as needed)
- Identify the idiomatic patterns already in use (language conventions, error handling style, naming, module structure)
- Flag design decisions the issue leaves open that should be resolved before implementation — e.g., "the issue says 'add validation' but doesn't specify where: at the handler level (consistent with existing pattern) or at the service level"
- Suggest simplifications: is there existing infrastructure (helpers, base classes, middleware) the implementer should reuse rather than build from scratch?
- Note any ambiguity in the acceptance criteria that could lead to divergent implementations

Output format:
```
## Software Review

### Existing patterns to follow
<What the codebase already does that the implementer should be consistent with>

### Open design decisions
<Choices the issue doesn't resolve that will affect implementation>

### Simplification opportunities
<Existing code to reuse, unnecessary complexity to avoid>

### Ambiguities
<Anything in the spec that could be read two ways>
```

### 2. Architecture Review (Agent subagent_type="general-purpose")

Provide: contents of `ISSUE.md` + instruction to explore the broader application structure.

The agent should:
- Assess whether this change impacts module boundaries, dependencies, or the overall structure of the application
- Identify if new interfaces, contracts, or integration points are introduced
- Check whether the change requires updates to shared infrastructure (database schemas, API contracts, configuration, build pipeline)
- Flag any risk of unintended coupling or architectural drift from the existing design

Output format:
```
## Architecture Review

### Structural impact
<Which modules/layers are affected and how>

### New boundaries or contracts
<Any new interfaces, API changes, schema migrations, or config changes required>

### Risks
<Coupling concerns, architectural drift, or things that could go wrong at the structural level>

### Prerequisites
<Anything that must exist or be true before this work can proceed>
```

### 3. Agentic Workflow Review (Agent subagent_type="general-purpose")

Provide: contents of `ISSUE.md` + instruction to explore the codebase structure for parallelization assessment.

The agent should analyze the task's structure and recommend how `/go` should approach execution:
- Can the work be decomposed into independent parallel tasks, or is it inherently sequential?
- Are there natural specialist roles (backend, frontend, database, tests) that map to separate agents?
- Is this a good candidate for agent teams (agents need to coordinate, negotiate contracts, share discoveries) or subagents (independent work that reports back)?
- What's the likely batch structure? How many parallel batches, roughly how many tasks per batch?
- Are there any tasks that are high-risk and should get extra review attention?

Output format:
```
## Agent Workflow Recommendation

### Parallelization potential
<High/Medium/Low — with reasoning about task independence>

### Recommended approach
<Subagents or Agent Teams — and why>

### Suggested task decomposition
<Rough sketch of how tasks could be batched, NOT a full plan — that's /go's job>

### High-risk areas
<Tasks that warrant extra review attention or human checkpoints>
```

---

## Triage & Resolution

After all three reviews complete, classify each finding into one of three categories:

### Category A: Open decisions and ambiguities requiring user input

These are findings where the reviews surfaced a question that only the user can answer — a design choice, an ambiguous requirement, a trade-off with no obvious default.

**Examples:**
- "The issue says 'add validation' but doesn't specify where — handler or service level?"
- "This could be implemented as a new endpoint or an extension of the existing one. Which?"
- "The acceptance criteria say 'returns an error' but don't specify the error format. The codebase uses both RFC 7807 and custom JSON — which should this follow?"

### Category B: Useful context for the implementer

Non-controversial findings that don't require a decision — just useful information. Existing patterns, reusable infrastructure, structural observations, the agentic workflow recommendation.

### Category C: Not actionable or irrelevant

Noise. Drop it.

### Resolution flow

1. **Present all findings to the user**, organized by category. For Category A items, be explicit: "These need your input before we can finalize the issue."

2. **If there are Category A items → enter a focused interview.**
   This is a second interview pass, but narrower. You are NOT re-interviewing the whole problem — only resolving the specific open decisions and ambiguities the reviews surfaced.

   Follow the same interview guidelines as before:
   - One question at a time via `AskUserQuestion`
   - Update `ISSUE.md` after each decision
   - Show the user what changed

   As decisions are made, update the appropriate section:
   - Resolved design decisions → **Constraints** or **Acceptance criteria** (depending on whether it's a constraint on implementation or a testable outcome)
   - Resolved ambiguities → clarify the **Acceptance criteria**
   - New scope boundaries discovered → **Scope**

3. **After all Category A items are resolved**, populate the **Implementation notes** section with Category B findings from all three reviews.

4. **Show the user the final `ISSUE.md`** and confirm they're satisfied before creating/updating the GitHub issue.

5. **If there are NO Category A items**, skip straight to populating Implementation notes with Category B findings, show the final `ISSUE.md`, and confirm.
