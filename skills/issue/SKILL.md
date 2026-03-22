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

- `/issue <description>` — create a new issue, using the provided text as the initial description
- `/issue 123` — edit existing issue #123

Determine which flow to use by checking whether the argument is a number (edit) or text (create).

## Flow: Create New Issue (`/issue <description>`)

1. **Seed** `ISSUE.md` with the user-provided description as a starting point, using the target structure below as a skeleton.
2. **Interview** the user to flesh out all relevant details. Ask focused, one-at-a-time questions to clarify.
   After each decision or clarification, update `ISSUE.md` in the working directory with the current state of the issue description. Show the user what changed.
3. **When the user is satisfied**, create the issue:
   ```
   gh issue create --title "TITLE" --body-file "ISSUE.md"
   ```
   Keep `ISSUE.md` in the working directory — do NOT delete it. Downstream skills (e.g., `/go`) consume it.

## Flow: Edit Existing Issue (`/issue 123`)

1. **Fetch** the existing issue:
   ```
   gh issue view 123 --json title,body
   ```
2. **Write** the current body to `ISSUE.md` and present a summary to the user.
3. **Interview** the user about what needs to change, using the same iterative approach — ask questions, update `ISSUE.md` after each decision.
4. **When the user is satisfied**, update the issue:
   ```
   gh issue edit 123 --title "TITLE" --body-file "ISSUE.md"
   ```
   Keep `ISSUE.md` in the working directory.

## Target Structure for ISSUE.md

Guide the interview toward producing these sections. Not every section is required — skip what isn't relevant, but always produce **Problem**, **Desired behavior**, and **Acceptance criteria** at minimum.

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
```

### Section-by-section interview guidance

Work through these in roughly this order, but follow the natural conversation — don't force a rigid sequence.

1. **Problem & Desired behavior**: Start here. The user's initial description usually covers this partially. Ask clarifying questions until you could explain the task to a stranger.
2. **Acceptance criteria**: This is the most important section for downstream automation. Push for concrete, testable statements. Transform vague requirements ("it should be fast") into specific ones ("response time under 200ms for the p95 case"). Each criterion should be a clear pass/fail check.
3. **Scope**: Ask "Is there anything that might seem related but should NOT be part of this task?" This prevents scope creep during implementation.
4. **Constraints**: Ask about integration points, backwards compatibility, and anything the implementer must not break.
5. **Context**: Ask about relevant files, modules, or prior art in the codebase. Even rough pointers ("somewhere in the auth module") help the implementation agent orient.

## Interview Guidelines

- Use the `AskUserQuestion` tool to ask each interview question. Do not use plain text output for questions.
- Ask one question at a time. Do not overwhelm with multiple questions.
- Keep questions focused on understanding the problem and producing a clear description.
- Update `ISSUE.md` incrementally so the user can see progress and correct course.
- Use plain, direct language in the issue body — no filler.
- The issue title should be concise and specific.
- Do not ask about labels, assignees, priority, or project metadata — just the problem description.
- When writing acceptance criteria, prefer observable outcomes ("API returns 404 when resource not found") over implementation prescriptions ("add a null check in the handler"). Let the implementer decide *how*.
- If the user provides implementation hints or preferences, capture them in **Constraints** or **Context** — not as acceptance criteria.
