---
name: opus
description: Interview-driven planning and autonomous implementation, from spec to reviewed code
disable-model-invocation: true
user-invocable: true
argument-hint: "<issue-id-or-description>"
---

# /opus $ARGUMENTS

End-to-end flow: explore the codebase, interview the user to produce a spec, review it, let the user pick an architecture, then hand off to autonomous implementation — plan, build, verify, and review. The skill stops at the review summary; **version control is entirely the user's responsibility.**

You are the **orchestrator** throughout. You dispatch subagents, read their output, and make decisions. You do not write implementation code yourself after the spec is finalized, and you do not perform any version-control operations (see Key Rules).

Use `.agent/work/<issue>/` as the folder for all working files. Do not remove it — it is used for traceability and inspection.

---

## Step 0: Resolve Input

`$ARGUMENTS` is either an issue reference (GitHub issue, beads ID, or spec filename) or a prose description. Detect by format:

| Input format | Source | Example |
|--------------|--------|---------|
| Pure integer | GitHub | `123`, `7316` |
| GitHub URL | GitHub | `https://github.com/owner/repo/issues/123` |
| GitHub shorthand | GitHub | `owner/repo#123` |
| Matches file in `~/.claude/plans/` | spec file | `api-rate-limiting`, `api-rate-limiting.md` |
| Alphanumeric with letters, no spec match | beads | `PROJ-123`, `abc-def` |
| Anything else | prose description | `add dark mode toggle to settings page` |

### Determine `<issue>` slug

- GitHub: issue number (e.g. `123`)
- Beads: the ID as given (e.g. `PROJ-123`)
- Spec file: filename without `.md`
- Prose: auto-generate a short kebab-case slug from the description (3–5 words, lowercase, hyphenated). Do not prompt the user.

### Load seed content

- GitHub: `gh issue view <issue-id> --json number,title,body,labels,state`
- Spec file: read `~/.claude/plans/<filename>` (append `.md` if needed)
- Prose: use the description itself as seed

### Initialize the working directory

Create `.agent/work/<issue>/`. Do not write `ISSUE.md` yet — it is produced at the end of Step 4.

---

## Step 1: Codebase Exploration

**Goal:** build real understanding of the relevant code *before* asking clarifying questions, so the interview can be grounded in concrete files instead of abstractions.

Launch **2–3 code-explorer agents in parallel**, each targeting a different angle. Pick the angles that best fit the seed content — for example:

- "Find features similar to the described task and trace through their implementation"
- "Map the architecture and abstractions for the area this task will touch"
- "Analyze the current implementation of the feature/module being modified"
- "Identify UI patterns, testing approaches, or extension points relevant to this task"

Each agent must return a prose summary of what it found **plus a list of 5–10 key files the orchestrator should read**.

After the agents return, **the orchestrator reads every flagged file** (deduplicated) with the `Read` tool. This builds concrete context so later steps aren't reasoning over second-hand summaries.

Write a short notes file at `.agent/work/<issue>/exploration.md` capturing the key patterns, relevant files, and architectural landmarks for later agents to reference.

---

## Step 2: Interview

With the codebase context fresh in mind, interview the user to flesh out the spec. Use the Target Structure below as a skeleton.

### Target Structure for the spec

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
What is explicitly NOT part of this task. Boundaries the implementer should respect.

## Constraints
Known technical boundaries: must integrate with X, cannot change Y, backwards-compatibility,
performance requirements, etc.

## Context
Relevant code paths, modules, prior discussion, related issues.
Include the key findings from Step 1.

## Implementation notes
(Populated in Step 4 from review findings — not during the interview.)
```

### Section-by-section interview guidance

Work through these in roughly this order, but follow the natural conversation — don't force a rigid sequence. Always produce **Problem**, **Desired behavior**, and **Acceptance criteria** at minimum; skip other sections if they aren't relevant.

1. **Problem & Desired behavior.** The seed usually covers this partially. Ask clarifying questions until you could explain the task to a stranger.
2. **Acceptance criteria.** The most important section for downstream automation. Push for concrete, testable statements. Transform vague requirements ("it should be fast") into specific ones ("p95 response time under 200ms").
3. **Scope.** Ask: "Is there anything that might seem related but should NOT be part of this task?"
4. **Constraints.** Ask about integration points, backwards compatibility, and anything the implementer must not break.
5. **Context.** Reference Step 1's findings and ask about additional files, modules, or prior art worth noting.

### Interview guidelines

- Use `AskUserQuestion` for every interview question. Do not use plain text for questions unless the user explicitly asks to chat about one.
- One question at a time. Do not overwhelm the user.
- Update the spec incrementally so the user can see progress.
- Use plain, direct language — no filler.
- Prefer observable outcomes in acceptance criteria over implementation prescriptions. Let the implementer decide *how*.
- If the user provides implementation hints, capture them in **Constraints** or **Context** — not as acceptance criteria.
- Use the codebase context from Step 1 to ask sharper, concrete questions ("I saw auth middleware at `src/auth/middleware.ts` — should this reuse that, or is it separate?") rather than generic ones.

**Never stop the interview without asking the user if we are finished. Always confirm with the user before ending the interview.**

---

## Step 3: Spec Clarity Review

After the interview is confirmed complete, dispatch a single **`plan-review`** agent:

- **Artifact**: the finalized spec
- **Lens**: `spec-clarity`
- **Mode**: `findings`
- **Supporting context**: `.agent/work/<issue>/exploration.md`

The agent returns findings pre-categorized into Category A (decisions requiring human input), Category B (useful context), and Category C (drop).

> Architecture-level concerns and parallelization strategy are deferred to Step 6 (architecture proposals) and Step 7 (plan), where they're handled concretely rather than as abstract review findings.

---

## Step 4: Triage & Resolution

The `plan-review` agent returns findings already grouped:

- **Category A** — Open decisions and ambiguities only the user can resolve
- **Category B** — Useful context for the implementer (reusable infrastructure, patterns to follow)
- **Category C** — Not actionable; drop it

Briefly validate the categorization (move findings between categories if the agent miscategorized), then:

1. Present findings to the user, organized by category. For Category A items, be explicit: "These need your input before the spec can be finalized."

2. **If there are Category A items**, enter a focused clarification pass — a narrower interview resolving only the surfaced decisions. Same guidelines as Step 2: one `AskUserQuestion` at a time, update the spec after each decision, show the user what changed.

3. After Category A items are resolved (or immediately, if none), fold Category B findings into the spec's **Context** and **Implementation notes** sections.

4. Write the finalized spec to `.agent/work/<issue>/ISSUE.md`. This is the source of truth for all downstream agents.

---

## Step 5: Checkpoint

Use `AskUserQuestion` to present the finalized spec and ask the user to proceed with architecture proposals or stop here.

- **Proceed** → continue to Step 6.
- **Stop** → halt the skill. `ISSUE.md` remains in `.agent/work/<issue>/` for later use.

Do not proceed past this point without explicit confirmation.

---

## Step 6: Architecture Proposals

**Goal:** produce multiple implementation approaches with different trade-offs so the user can pick the one they want.

Launch **three code-architect agents in parallel**, each with a different framing:

1. **Minimal changes** — smallest diff, maximum reuse of existing code, lowest risk. Deprioritizes elegance when reuse means awkward fits.
2. **Clean architecture** — maintainability, clear abstractions, idiomatic patterns. Accepts more new code when it improves the design.
3. **Pragmatic balance** — speed + quality. Reuses where sensible, introduces new abstractions where they clearly pay off.

Each agent receives: `ISSUE.md` + `exploration.md`. Each must return:
- A one-paragraph summary of the approach
- Key design decisions (what abstractions are introduced, what's reused, what's changed)
- Concrete files affected (created / modified / deleted)
- Trade-offs: what this approach gives up relative to the others

### Orchestrator synthesis

Read all three proposals. Form an opinion on which fits best given the task's size, risk profile, and urgency. Present to the user via `AskUserQuestion`:

- A one-line summary of each approach
- Your recommendation, with a brief reason
- The three proposals as options

**Wait for the user's choice.** If the user picks "Other" with "whatever you think is best", confirm your recommendation explicitly before proceeding. Save the chosen proposal to `.agent/work/<issue>/approach.md`.

---

## Step 7: Plan

### 7a. Decompose the chosen approach

Dispatch a **code-architect** agent with `ISSUE.md` + `approach.md` + `exploration.md`. The agent produces the task breakdown.

Each task must specify:
- **Goal** — what it accomplishes (tied to specific acceptance criteria from `ISSUE.md`)
- **Files to modify/create** — exact paths
- **Test files** — exact paths
- **Interface contracts** — any types, function signatures, or APIs this task produces that others depend on
- **Dependencies** — which tasks must complete first (by task ID)

Requirements: tasks atomic (roughly 2–5 minutes of implementation work each); tasks touching the same files grouped as sequential; TDD explicit (failing test first, then implementation). Write the plan to `.agent/work/<issue>/plans/<feature>.md`.

### 7b. Self-review

Launch two **`plan-review`** agents in parallel. Both receive the plan as artifact and `ISSUE.md` + `exploration.md` as supporting context. Both use `mode: gate`.

- Lens `architecture` — tasks atomic, file paths realistic, TDD explicit, YAGNI, every acceptance criterion covered, interface contracts consistent.
- Lens `security` — no hardcoded secrets, input validation where needed, auth/authz addressed, logging won't expose PII, error handling planned.

Each returns `APPROVED` or `NEEDS_CHANGES` with a concrete action list.

### 7c. Fix loop (max 3 iterations)

If either returns `NEEDS_CHANGES`: dispatch a fixer agent with the specific feedback, then re-review. Loop until both approve — or abort after 3 iterations and report what's unresolved. Do not proceed to Step 8 without both approvals.

---

## Step 8: Implement

Identify parallel batches from the plan. Tasks that touch different files run in parallel.

### Dispatch rules

For each batch, dispatch all implementers in parallel. **Each implementer receives a scoped context — NOT the full plan.** Construct each prompt with exactly:

1. **Overall goal** — one paragraph from `ISSUE.md` (Problem + Desired behavior only)
2. **This task's specification** — goal, files, test files from the plan
3. **Interface contracts this task must respect** — from already-completed tasks it depends on
4. **Relevant code from already-completed tasks** — only if this task depends on them; specific files and changes, not full history

Do NOT include: the full plan, other tasks' specifications, review feedback from other tasks, or the full issue description.

Each implementer returns the list of files it created or modified and a short summary of changes. The orchestrator aggregates these into a running manifest at `.agent/work/<issue>/changed_files.md` so downstream agents can be given exact targets without running `git`.

### Per-task review (after each batch)

For each completed task, dispatch **three code-reviewer agents in parallel**, each with a different focus:

- **Simplicity / DRY / elegance** — is this the simplest implementation that works? Any duplication worth consolidating? Could it be more readable?
- **Bugs / correctness** — logical errors, missed edge cases, off-by-ones, incorrect assumptions, test adequacy.
- **Conventions** — consistency with patterns identified in Step 1, naming, error handling style, module structure.

Each reviewer receives: the task specification + the files the implementer modified (paths and current contents) + the implementer's change summary. Not the full plan.

Each returns `APPROVED` or `NEEDS_CHANGES` with specifics. A task is approved only when all three approve.

### Fix loop (max 3 iterations per task)

If any reviewer returns `NEEDS_CHANGES`, dispatch a fix agent with the consolidated feedback, then re-review only that task. After 3 failed fix iterations, pause and report. Do not continue to the next batch if a dependency has failed.

---

## Step 9: Verify

Dispatch a **verification agent** (`general-purpose`) with a clean context.

Provide:
- The commands to run (build, test, lint — whatever the project uses)
- Instruction to run them and report raw results: exact pass/fail counts, exit codes, and error output

The agent does NOT interpret or judge — it gathers evidence and reports.

**You decide:**
- All tests pass and build succeeds → proceed to Step 10
- Any failures → dispatch a fixer agent with the failure output, then re-verify (max 3 attempts)
- After 3 failed verification cycles → stop and report

---

## Step 10: Review & Summarize

### 10a. Parallel review

Launch two **`plan-review`** agents in parallel, both with `mode: gate`. Each receives the files listed in `.agent/work/<issue>/changed_files.md` (paths and current contents) as artifact, plus `ISSUE.md` + `.agent/work/<issue>/plans/<feature>.md` as supporting context.

- Lens `acceptance-criteria` — which criteria were delivered, partially delivered, skipped, or added beyond scope; decisions and trade-offs made during implementation; deviations from the plan.
- Lens `architecture` — pattern consistency with the existing codebase, new duplication or consolidation opportunities, coupling and cohesion, suggested follow-ups.

### 10b. Fix loop (max 3 iterations)

If either reviewer returns `NEEDS_CHANGES`, dispatch a fixer agent with the consolidated feedback, re-run Step 9 (Verify), then re-run Step 10a on the updated files. Loop until both approve — or after 3 iterations, proceed to the summary with the outstanding issues documented under "Known issues" in `SUMMARY.md`.

### 10c. SUMMARY.md

Dispatch a **Summary Writer Agent** (`general-purpose`) with both review outputs (including any unresolved `NEEDS_CHANGES` items from Step 10b). It produces `.agent/work/<issue>/SUMMARY.md`:

```markdown
# Summary

## What was done
<Concise description>

## Acceptance criteria status
- [x] Criterion from spec — met
- [ ] Criterion — not met (reason)

## Decisions & trade-offs
## Deviations from plan
## Architectural notes
## Known issues
<Unresolved NEEDS_CHANGES items from Step 10b, if any>
## Suggested follow-ups
```

`SUMMARY.md` is the final artifact of this skill. The user reads it and decides what to do next (commit, push, open a PR, or discard) — /opus does none of those things.

---

## Key Rules

1. **Orchestrator doesn't code** — after the interview, you dispatch, read results, and make decisions. You don't write implementation code yourself.
2. **Fresh subagent per logical unit** — clean context, no pollution.
3. **Scoped context per implementer** — each agent gets only what it needs: its task, the overall goal, relevant interface contracts. Never the full plan.
4. **Read what agents flag** — when exploration or review agents return a list of files, read them with the `Read` tool to build real context, not just scan their summaries.
5. **Maximize parallelization** — non-overlapping files = parallel dispatch.
6. **Review loops until approved** — never skip `NEEDS_CHANGES` feedback.
7. **Iteration limits everywhere** — max 3 fix iterations per task, per review, per verification. Fail loudly, don't grind.
8. **TDD is mandatory** — every implementer must demonstrate test-first.
9. **Verify in a clean context** — dispatch a fresh agent to gather evidence, then decide yourself.
10. **Summarize before stopping** — always produce `SUMMARY.md` as the final record.
11. **Interview questions go through `AskUserQuestion`** — not plain text.
12. **Always confirm before ending the interview.**
13. **Never bypass user checkpoints** — Step 5 (spec approval) and Step 6 (architecture choice) both require explicit user input.
14. **Use specialized agents where they fit:**
    - `code-explorer` — codebase surveys in Step 1.
    - `code-architect` — architecture proposals (Step 6) and plan decomposition (Step 7a).
    - `plan-review` — artifact-level reviews (Steps 3, 7b, 10a), invoked with the appropriate `lens` and `mode` (`findings` for triage, `gate` for approval).
    - `code-reviewer` — per-task post-implementation review (Step 8), three parallel dispatches with different focuses.
    - `general-purpose` — implementers, fixers, the verification agent, and the summary writer.
15. **No version control — ever.** Neither the orchestrator nor any subagent runs `git` (any subcommand — no `diff`, `log`, `status`, `commit`, `push`, `branch`, `tag`, `stash`), invokes `gh` for anything that changes state (no `pr create`, no `issue edit`, no label or status changes), or otherwise touches the repository's version-control state. Read-only `gh issue view` in Step 0 for loading seed content is the sole exception. Understanding what changed is done via the `.agent/work/<issue>/changed_files.md` manifest, not `git diff`. The user performs all version control themselves.

---

**Start now:** Run Step 0 with `$ARGUMENTS`.
