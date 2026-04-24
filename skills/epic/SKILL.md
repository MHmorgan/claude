---
name: epic
description: Plan an epic through structured interviews, recursively splitting into tasks using the `ae` CLI
disable-model-invocation: true
user-invocable: true
arguments: "<description>"
---

# Epic Plan Skill

Plan a full epic: create it, interview the user to flesh out the work, split
it into sub-tasks where needed, and finalize each leaf with enough detail
that a separate implementation session (`/go`) can execute it without
further questions.

All planning state lives in `ae`. This skill uses `ae` commands for every
write so the user can exit and resume mid-plan.

Do not implement anything. Planning ends with every leaf having a finalized
body and a handoff context.

---

## Flow: `/epic <description>`

1. **Resolve the epic** — `ae epics` first. If the description matches an
   existing in-progress epic, offer to resume it. Otherwise create a slug
   (lowercase, `[a-z][a-z0-9-]*`), then `ae task:new-epic <slug>`.
2. **Explore the codebase** once for the whole epic — see *Codebase
   exploration* below. Seeds every downstream interview with concrete file
   references instead of abstractions. Skip only when resuming an
   in-progress epic whose root context already contains exploration notes.
3. **Plan the root task** via the per-task loop below.
4. **Recurse depth-first** into any children created by splits — finish
   child `:1` and its entire subtree before starting `:2`.
5. **Review each branch** once all its descendants are planned.
6. **Confirm completion** when the whole tree is planned.

---

## Codebase exploration

Run this once per epic, before planning the root task. The distilled
output propagates down through ancestor context composition, so every
later per-task loop starts with concrete file references.

1. Launch **2–3 `code-explorer` agents in parallel**, each targeting a
   different angle. Pick angles that best fit the epic's seed — for
   example:
   - "Find features similar to <epic> and trace through their
     implementation"
   - "Map the architecture and abstractions for the area this epic will
     touch"
   - "Analyze the current implementation of the feature/module being
     modified"
   - "Identify UI patterns, testing approaches, or extension points
     relevant to <epic>"

   Each agent returns a prose summary plus **5–10 key files** to read.

2. **Read every flagged file** (deduplicated) with the `Read` tool. This
   builds real context — don't reason from second-hand summaries.

3. **Distil into the epic context** (≤ 15 lines): key patterns, relevant
   files/modules, architectural landmarks, extension points. Write with
   `echo "$notes" | ae task:set-context <epic>`. Longer raw
   findings can be appended as records via `ae task:record <epic>` for
   traceability.

---

## Per-task planning loop

Run this on every task, starting with the root.

1. **Draft a short seed.** Based on available context — the user's
   description for the root, or the parent body plus ancestor contexts
   (`ae context <parent>`) for children — write a minimal
   seed: a title, one paragraph capturing the task's intent, and if the
   shape is obvious, a provisional outline of the work. Keep it brief;
   this is scaffolding, not the plan. `echo "$seed" | ae task:set-body <id>`.
2. **Scope: branch or leaf?** Ask the user explicitly, surfacing your
   read as the recommended option:
   - **Leaf** — one chunk of work, a single implementation session could
     finish it, acceptance criteria all address one concern.
   - **Branch** — multiple chunks with natural seams (distinct
     subsystems, clear ordering, parallel streams), or a single session
     couldn't reasonably finish it.

   The user's judgment wins. The scoping decision commits the interview
   to a template. It can be revisited mid-interview if the other shape
   starts looking obviously right — re-scope, restructure the body,
   continue.
3. **Restructure to the chosen template** and write it:
   `echo "$body" | ae task:set-body <id>`. Slot the seed content into the
   appropriate sections. See *Target body — leaf task* or *Target body —
   branch task (pre-split)* below.
4. **Interview into the template.** Refine the body through questions
   (see *Interview guidelines*). Rewrite the full body after each
   meaningful answer via `ae task:set-body` so the user sees progress.
5. **Finalize:**

   **Branch path:**
   - Confirm the `---`-separated section structure with the user.
     Remind them: the body is frozen after split.
   - Avoid accidental `---` elsewhere (code blocks, frontmatter
     examples). The tool splits on literal `---` lines with no
     markdown awareness.
   - `ae task:split <id>`.
   - **Dependencies (one-shot prompt):** ask once whether any children
     are ordered. Record with `ae task:after <child> <pred>`. Deps
     stay editable anytime with `task:after` / `task:unafter` —
     re-prompt only if structure materially changes.
   - **Branch context:** `echo "$context" | ae task:set-context <id>` with shared info
     children will need (cross-cutting decisions, terminology,
     references). Keep it ≤ 15 lines.
   - Recurse into child `:1`, finish its entire subtree, then `:2`, etc.

   **Leaf path:**
   - Run the **Leaf review** below.
   - **Triage** findings; resolve Category A via a focused follow-up
     interview.
   - Populate **Implementation notes** with Category B findings.
   - Write the finalized body: `echo "$body" | ae task:set-body <id>`.
   - Write the handoff context: `echo "$context" | ae task:set-context <id>`. Keep it ≤
     15 lines — what the implementer needs that isn't obvious from
     body or ancestor contexts.

---

## Interview guidelines

- Use `AskUserQuestion` for every interview question. No plain-text
  questions unless the user asks to chat.
- One question at a time. Do not batch.
- Rewrite the body after each meaningful answer via `ae task:set-body` so the
  user sees progress and can correct course.
- Plain, direct language. No filler.
- Acceptance criteria: concrete, testable, observable outcomes. Transform
  "it should be fast" into "p95 response time under 200ms".
- Implementation hints the user volunteers go in **Constraints** or
  **Context**, never in **Acceptance criteria**.
- NEVER end an interview phase without confirming with the user. Always
  ask "anything else before we move on?" before proceeding to finalize
  (split for branches, review for leaves).

---

## Target body — leaf task

```markdown
# <Title>

## Problem
What's wrong or missing today? Observable symptom or unmet need.

## Desired behavior
What should happen when this is done? From the user's or caller's
perspective.

## Acceptance criteria
- [ ] Concrete, testable statement
- [ ] Another concrete, testable statement

## Scope
What is explicitly NOT part of this task.

## Constraints
Known technical boundaries: must integrate with X, cannot change Y,
performance requirements, etc.

## Context
Relevant code paths, modules, prior discussion, related issues.

## Implementation notes
(Populated after leaf review — not part of the interview.)

### Software considerations
<Idiomatic patterns, design decisions, simplification opportunities.>

### Architecture considerations
<Structural impact, module boundaries, integration points.>

### Agent workflow recommendation
<Parallelization strategy, subagents vs agent teams, risk areas.
Advice for the orchestrator that runs this leaf.>
```

**Problem**, **Desired behavior**, and **Acceptance criteria** are
required. Skip any other section that isn't relevant.

---

## Target body — branch task (pre-split)

```markdown
# <Title>

<Short intro: what this branch covers, why it exists as a unit.>

---

# <Child 1 title>

<Enough content to seed a productive interview when child 1 is planned.
Full detail is not required — the child's body will be refined during its
own planning loop.>

---

# <Child 2 title>

<Child 2 seed content.>
```

The first section (before the first `---`) is the **branch description**
— it is NOT split into a child task. Children are created from section 2
onward. Populated directly during the branch interview — each
`---`-separated section after the first names one sub-task. The branch
body is frozen after split, so each section should name its seam clearly
and give the child's interview a meaningful starting draft.

---

## Context authoring

`ae context <leaf>` returns the composition:
epic context → parent context → terminal sibling contexts → self context.
They stack, so keep each context ≤ 15 lines.

- **Branch context** — shared info for all children in this subtree:
  cross-cutting constraints, terminology, references. Curate and update
  as child planning reveals new shared information.
- **Leaf context (planning)** — handoff seed for the implementing agent:
  what they need to know that isn't in the body or ancestor contexts.
  The implementer will overwrite this with results and unresolved issues
  after execution.

---

## Leaf review

After the user confirms the leaf interview is complete, launch THREE review
agents in parallel. Each receives:
- The body (`ae show <id>`) as artifact
- The composed context (`ae context <id>`) as supporting context
- Read access to the codebase

### 1. Spec clarity review (`plan-review` subagent)

- **Artifact:** the leaf body
- **Lens:** `spec-clarity`
- **Mode:** `findings`

Surfaces ambiguity in acceptance criteria, underspecified edge cases,
existing patterns the implementer should follow, and reusable
infrastructure (helpers, middleware, base classes) the spec doesn't
mention. Findings return pre-categorized as A / B / C.

### 2. Architecture review (`plan-review` subagent)

- **Artifact:** the leaf body
- **Lens:** `architecture`
- **Mode:** `findings`

Surfaces structural impact: which modules/layers are affected, new
interfaces or contracts, coupling risks, architectural drift,
prerequisites that must exist before this work can proceed. Findings
return pre-categorized as A / B / C.

### 3. Agent Workflow Review (`general-purpose` subagent)

No `plan-review` lens fits this — it produces orchestrator advice, not
spec feedback. Advice for the leaf orchestrator `/go` will spawn when it
picks this task:
- Parallelization potential (high/medium/low with reasoning).
- Subagents vs agent teams, and why.
- Rough decomposition sketch — not a full plan, the orchestrator decides.
- High-risk areas warranting extra review or human checkpoints.

---

## Branch review (lightweight)

Once every descendant leaf of a branch has been planned, run a single
`plan-review` agent:

- **Artifact:** the branch body (frozen, pre-split snapshot) + the body
  of each immediate child (`ae show <child>`) concatenated
- **Lens:** `architecture`
- **Mode:** `gate`
- **Supporting context:** the branch context

The agent returns `APPROVED` or `NEEDS_CHANGES` with a concrete action
list. Typical findings:
- Does the split still make sense given what the children turned into?
- Are any obvious pieces missing? → `ae task:add-child <branch>` and plan
  the new child.
- Are sibling dependencies correct, complete, and cycle-free?
- Does the branch context adequately support the children, or should it
  be extended? → `echo "$context" | ae task:set-context`.

---

## Triage & Resolution (leaf reviews)

Both `plan-review` agents (spec-clarity and architecture) return findings
already grouped as Category A / B / C. The Agent Workflow agent returns
free-form advice — fold it into Category B as **Implementation notes →
Agent workflow recommendation**.

- **A. Open decisions & ambiguities** — user input required. Design
  choices, ambiguous requirements, trade-offs with no obvious default.
- **B. Useful context for the implementer** — non-controversial
  information. Goes into **Implementation notes** in the body.
- **C. Noise** — drop.

Briefly validate the categorization (move findings between categories if
the agent miscategorized), then:

1. Present findings to the user, organized by category. Be explicit that
   Category A items need their input.
2. If Category A items exist → focused follow-up interview (one question
   at a time via `AskUserQuestion`, rewrite body after each answer).
   Resolutions update **Constraints**, **Acceptance criteria**, or
   **Scope** as appropriate.
3. **Architectural Category A escape hatch.** If a Category A item is a
   genuine architectural choice with no obvious default — two plausible
   approaches with different trade-offs — optionally dispatch **2–3
   `code-architect` agents in parallel** (framings: minimal changes /
   clean architecture / pragmatic balance), present the options via
   `AskUserQuestion`, and encode the chosen direction into
   **Implementation notes → Architecture considerations**. Skip when the
   choice is small-scale or a clear default exists.
4. After Category A is resolved (or if none existed), populate
   **Implementation notes** with Category B findings from all three
   reviews.
5. Write the finalized body: `echo "$body" | ae task:set-body <id>`.
6. Show the final body and confirm before writing the handoff context.

---

## `ae` command reference (subset used by this skill)

```
ae epics                                 # list epics (resume check)
ae task:new-epic <slug>                  # create a new epic
ae show <id>                             # read body (plain text)
echo "$body" | ae task:set-body <id>  # write body (stdin, leaf only)
ae context <id>                          # read composed context (plain text)
echo "$ctx" | ae task:set-context <id>  # write context (stdin, any task)
ae task:list parent=<id>                 # list immediate children
ae task:split <id>                       # split leaf on `---`
ae task:add-child <parent>               # add a child to a branch
ae task:after <id> <pred>                # sibling dependency edge
ae task:unafter <id> <pred>              # remove dependency edge
echo "$text" | ae task:record <id>    # append an agent note (stdin)
ae help                                  # tool help
```

Write commands (`task:*`) return a JSON envelope `{"ok": bool, "data": ..., "error": ...}`.
Check `ok` before continuing; surface errors to the user.
`ae show` and `ae context` return plain text directly.

