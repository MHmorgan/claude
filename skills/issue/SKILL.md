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

1. **Seed** `ISSUE.md` with the user-provided description as a starting point.
2. **Interview** the user to flesh out all relevant details. Ask focused, one-at-a-time questions to clarify.
   After each decision or clarification, update `ISSUE.md` in the working directory with the current state of the issue description. Show the user what changed.
3. **When the user is satisfied**, create the issue:
   ```
   gh issue create --title "TITLE" --body-file "ISSUE.md"
   ```
   Clean up `ISSUE.md` after successful creation.

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
   Clean up `ISSUE.md` after successful update.

## Interview Guidelines

- Use the `AskUserQuestion` tool to ask each interview question. Do not use plain text output for questions.
- Ask one question at a time. Do not overwhelm with multiple questions.
- Keep questions focused on understanding the problem and producing a clear description.
- Update `ISSUE.md` incrementally so the user can see progress and correct course.
- Use plain, direct language in the issue body — no filler.
- The issue title should be concise and specific.
- Do not ask about labels, assignees, priority, or project metadata — just the problem description.
