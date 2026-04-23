---
name: opus
description: Interview-driven planning and autonomous implementation, from spec to reviewed code
disable-model-invocation: true
user-invocable: true
argument-hint: "<issue-id-or-description>"
---

# /opus $ARGUMENTS

End-to-end flow: interview the user to produce a spec, review it, let the user resolve any open decisions, then hand off to autonomous implementation — plan, build, verify, and review. The skill stops at the review summary; **version control is entirely the user's responsibility.**

You are the **orchestrator** throughout. You dispatch subagents, read their output, and make decisions. You do not write implementation code yourself after the spec is finalized, and you do not perform any version-control operations (see Key Rules).

Use `.agent/work/<issue>/` as the folder for all working files. Do not remove it — it is used for traceability and inspection.

---

## Step 0: Resolve Input

`$ARGUMENTS` is either an issue reference (GitHub issue, or spec filename) or a prose description. Detect by format:

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

Create `.agent/work/<issue>/`. Do not write `ISSUE.md` yet — it is produced at the end of Step 3.

---

## Step 1: Interview

Seed the spec with the loaded content (or prose description) using the Target Structure below as a skeleton. Then interview the user to flesh out the details.

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
Known technical boundaries: must integrate with X, cannot change Y,
must remain backwards-compatible with Z, performance requirements, etc.

## Context
Relevant code paths, modules, prior discussion, related issues.
Anything that helps the implementer orient quickly.

## Implementation notes
(Populated in Step 3 from review findings — not during the interview.)

### Software considerations
<Idiomatic patterns, design decisions, simplification opportunities>

### Architecture considerations
<Structural impact, module boundaries, integration points>

### Agent workflow recommendation
<Suggested approach for implementation: parallelization strategy, specialist splits, high-risk areas>
```

### Section-by-section interview guidance

Work through these in roughly this order, but follow the natural conversation — don't force a rigid sequence. Always produce **Problem**, **Desired behavior**, and **Acceptance criteria** at minimum; skip other sections if they aren't relevant.

1. **Problem & Desired behavior.** The seed usually covers this partially. Ask clarifying questions until you could explain the task to a stranger.
2. **Acceptance criteria.** This is the most important section for downstream automation. Push for concrete, testable statements. Transform vague requirements ("it should be fast") into specific ones ("response time under 200ms for the p95 case"). Each criterion should be a clear pass/fail check.
3. **Scope.** Ask: "Is there anything that might seem related but should NOT be part of this task?" This prevents scope creep during implementation.
4. **Constraints.** Ask about integration points, backwards compatibility, and anything the implementer must not break.
5. **Context.** Ask about relevant files, modules, or prior art in the codebase. Even rough pointers ("somewhere in the auth module") help downstream agents orient.

### Interview guidelines

- Use the `AskUserQuestion` tool for every interview question. Do not use plain text for questions unless the user explicitly asks to chat about one.
- One question at a time. Do not overwhelm the user with multiple questions per turn.
- Keep questions focused on clarifying the problem and producing a clear spec.
- Update the spec incrementally so the user can see progress and correct course.
- Use plain, direct language in the spec — no filler.
- Prefer observable outcomes in acceptance criteria ("API returns 404 when resource not found") over implementation prescriptions ("add a null check in the handler"). Let the implementer decide *how*.
- If the user provides implementation hints or preferences, capture them in **Constraints** or **Context** — not as acceptance criteria.

**Never stop the interview without asking the user if we are finished, even if you have no more questions. Always confirm with the user before ending the interview.**

---

## Step 2: Review Phase

After the user confirms the interview is complete, launch THREE review agents in parallel. Each receives the current spec and has read access to the codebase.

### 1. Software Review (Agent subagent_type="general-purpose")

Provide: the spec contents + instruction to explore relevant parts of the codebase.

The agent should:
- Read the code areas referenced in the Context section (and discover adjacent code as needed)
- Identify idiomatic patterns already in use (language conventions, error handling style, naming, module structure)
- Flag design decisions the spec leaves open that should be resolved before implementation — e.g., "the spec says 'add validation' but doesn't specify where: handler level (consistent with existing pattern) or service level"
- Suggest simplifications: is there existing infrastructure (helpers, base classes, middleware) the implementer should reuse?
- Note any ambiguity in the acceptance criteria that could lead to divergent implementations

Output format:

```
## Software Review

### Existing patterns to follow
<What the codebase already does that the implementer should be consistent with>

### Open design decisions
<Choices the spec doesn't resolve that will affect implementation>

### Simplification opportunities
<Existing code to reuse, unnecessary complexity to avoid>

### Ambiguities
<Anything in the spec that could be read two ways>
```

### 2. Architecture Review (Agent subagent_type="general-purpose")

Provide: the spec contents + instruction to explore the broader application structure.

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

Provide: the spec contents + instruction to explore the codebase structure for parallelization assessment.

The agent should analyze the task's structure and recommend how implementation should be approached:
- Can the work be decomposed into independent parallel tasks, or is it inherently sequential?
- Are there natural specialist roles (backend, frontend, database, tests) that map to separate agents?
- Is this a good candidate for agent teams (coordination, shared discoveries) or subagents (independent work that reports back)?
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
<Rough sketch of how tasks could be batched, NOT a full plan>

### High-risk areas
<Tasks that warrant extra review attention or human checkpoints>
```

---

## Step 3: Triage & Resolution

After all three reviews complete, classify each finding into one of three categories.

### Category A: Open decisions and ambiguities requiring user input

Findings where only the user can answer — a design choice, an ambiguous requirement, a trade-off with no obvious default.

Examples:
- "The spec says 'add validation' but doesn't specify where — handler or service level?"
- "This could be a new endpoint or an extension of the existing one. Which?"
- "The acceptance criteria say 'returns an error' but don't specify the error format. The codebase uses both RFC 7807 and custom JSON — which should this follow?"

### Category B: Useful context for the implementer

Non-controversial findings that don't require a decision — just useful information. Existing patterns, reusable infrastructure, structural observations, the agent workflow recommendation.

### Category C: Not actionable or irrelevant

Noise. Drop it.

### Resolution flow

1. **Present all findings to the user**, organized by category. For Category A items, be explicit: "These need your input before the spec can be finalized."

2. **If there are Category A items → enter a focused clarification pass.** This is a narrower interview — you are NOT re-interviewing the whole problem, only resolving the specific open decisions and ambiguities the reviews surfaced.

   Follow the same interview guidelines as Step 1:
   - One question at a time via `AskUserQuestion`
   - Update the spec after each decision
   - Show the user what changed

   As decisions are made, update the appropriate section of the spec:
   - Resolved design decisions → **Constraints** or **Acceptance criteria** (depending on whether it's a constraint on implementation or a testable outcome)
   - Resolved ambiguities → clarify the **Acceptance criteria**
   - New scope boundaries discovered → **Scope**

3. **After all Category A items are resolved**, populate the **Implementation notes** section of the spec with Category B findings from all three reviews:
   - Software Review's findings → `### Software considerations`
   - Architecture Review's findings → `### Architecture considerations`
   - Agentic Workflow Review's findings → `### Agent workflow recommendation`

4. **If there are NO Category A items**, skip straight to populating Implementation notes with Category B findings.

### Persist the finalized spec

Write the finalized spec to `.agent/work/<issue>/ISSUE.md`. This is the source of truth for all downstream agents.

---

## Step 4: Checkpoint

Use `AskUserQuestion` to present the finalized spec and ask the user to proceed with implementation or stop here.

- **Proceed** → continue to Step 5.
- **Stop** → halt the skill. `ISSUE.md` remains in `.agent/work/<issue>/` for later use.

Do not proceed past this point without explicit confirmation.

---

## Step 5: Plan

### 5a. Create the implementation plan

Use the Agent tool with `subagent_type="general-purpose"` to create the plan.

Provide the subagent with:
- The contents of `.agent/work/<issue>/ISSUE.md`
- Instruction to explore the codebase and understand existing patterns

The plan agent must:
- Break work into small, independent tasks (one logical unit each, roughly 2–5 minutes of implementation work per task)
- For EACH task, specify:
  - **Goal**: what this task accomplishes (tied to specific acceptance criteria from `ISSUE.md`)
  - **Files to modify/create**: exact paths
  - **Test files**: exact paths
  - **Interface contracts**: any types, function signatures, or APIs this task produces that other tasks depend on
  - **Dependencies**: which other tasks must complete first (by task ID)
- Group tasks by file overlap — tasks touching the same files must be sequential
- Make TDD explicit in each task: write the failing test first, then the implementation
- Write the plan to `.agent/work/<issue>/plans/<feature>.md`

### 5b. Review the plan

Launch TWO review agents in parallel:

1. **Architecture review** (Agent subagent_type="general-purpose")

   Provide: `.agent/work/<issue>/ISSUE.md` + `.agent/work/<issue>/plans/<feature>.md`

   Check:
   - Tasks are atomic (2–5 min each)
   - File paths are exact and realistic
   - TDD steps are explicit
   - No unnecessary complexity (YAGNI)
   - Every acceptance criterion from `ISSUE.md` is covered by at least one task
   - Interface contracts between tasks are consistent

   Return `APPROVED` or `NEEDS_CHANGES` with a concrete list.

2. **Security review** (Agent subagent_type="general-purpose")

   Provide: `.agent/work/<issue>/ISSUE.md` + `.agent/work/<issue>/plans/<feature>.md`

   Check:
   - No hardcoded secrets planned
   - Input validation included where needed
   - Auth/authz considerations addressed
   - Logging won't expose PII
   - Error handling planned

   Return `APPROVED` or `NEEDS_CHANGES` with a concrete list.

### 5c. Fix loop (max 3 iterations)

If either reviewer returns `NEEDS_CHANGES`: dispatch a fixer agent with the specific feedback, then re-review.

Loop until BOTH return `APPROVED` — or abort after 3 fix iterations and report what's unresolved. Do not proceed to Step 6 without both approvals.

---

## Step 6: Implement (parallel batches)

Identify parallel batches from the plan. Tasks that touch different files can run in parallel.

### Dispatch rules

For each batch of independent tasks, dispatch ALL implementers in parallel using the Agent tool.

**Each implementer receives a scoped context — NOT the full plan.** Construct the prompt for each implementer with exactly:

1. **Overall goal** (one paragraph from `ISSUE.md` — the Problem and Desired behavior sections only)
2. **This task's specification** (copied from the plan: goal, files, test files)
3. **Interface contracts this task must respect** (types, signatures, APIs produced by already-completed tasks that this task depends on)
4. **Relevant code from already-completed tasks** (only if this task has dependencies — provide the specific files and the changes made, not the full history)

Do NOT include: the full plan, other tasks' specifications, review feedback from other tasks, or the full issue description.

### After each batch

Dispatch parallel code reviews for each completed task. Each reviewer receives the task specification + the files the implementer modified (paths and current contents) + the implementer's summary of what was changed. Not the full plan.

Each implementer is required to return, as part of its output, the list of files it created or modified and a short summary of the changes. The orchestrator aggregates these into a running manifest at `.agent/work/<issue>/changed_files.md` so downstream review agents can be given exact targets without running `git`.

### If NEEDS_CHANGES (max 3 iterations per task)

Dispatch a fix agent for the specific task with the review feedback, then re-review only the fixed task.

If a task fails 3 fix iterations, pause and report the situation. Do not continue to the next batch if a dependency has failed.

### Proceed to the next batch when the current batch is fully approved.

---

## Step 7: Verify

Dispatch a **verification agent** (Agent subagent_type="general-purpose") with a clean context.

Provide the agent with:
- The commands to run (build, test, lint — whatever the project uses)
- Instruction to run the commands and report raw results: exact pass/fail counts, exit codes, and any error output

The verification agent does NOT interpret or judge — it gathers evidence and reports.

**You (the orchestrator) then read the results and decide:**
- All tests pass and build succeeds → proceed to Step 8
- Any failures → dispatch a fixer agent with the failure output, then re-verify (max 3 attempts)
- After 3 failed verification cycles → stop and report

---

## Step 8: Review & Summarize

### 8a. Parallel review agents

Launch TWO review agents in parallel:

1. **Spec Compliance Agent** (Agent subagent_type="general-purpose")

   Provide: `.agent/work/<issue>/ISSUE.md` + `.agent/work/<issue>/changed_files.md` + the current contents of every file listed in the manifest

   Report:
   - **Delivered** — acceptance criteria fully met (reference specific criteria from `ISSUE.md`)
   - **Partially delivered** — criteria met with caveats
   - **Skipped** — criteria not addressed (with reason)
   - **Added beyond scope** — work done that wasn't in the original spec
   - **Decisions & trade-offs** — non-obvious choices made during implementation
   - **Deviations from plan** — where implementation diverged from `.agent/work/<issue>/plans/` and why

2. **Architecture Analysis Agent** (Agent subagent_type="general-purpose")

   Provide: the files listed in `.agent/work/<issue>/changed_files.md` + their surrounding module structure

   Report:
   - **Pattern consistency** — does the new code follow existing conventions?
   - **Duplication** — any new duplication introduced, or opportunities to consolidate?
   - **Coupling & cohesion** — are dependencies reasonable? Any tight coupling?
   - **Suggested follow-ups** — structural improvements worth considering (not blockers)

### 8b. Synthesize into SUMMARY.md

Dispatch a **Summary Writer Agent** (Agent subagent_type="general-purpose") with the output from both review agents. It must produce `.agent/work/<issue>/SUMMARY.md`:

```markdown
# Summary

## What was done
<Concise description of the delivered work>

## Acceptance criteria status
- [x] Criterion from spec — met
- [x] Criterion from spec — met
- [ ] Criterion from spec — not met (reason)

## Decisions & trade-offs
<Non-obvious choices and their rationale>

## Deviations from plan
<Where and why implementation diverged from the original plan>

## Architectural notes
<How the new code fits with the existing codebase, pattern consistency, and any concerns>

## Suggested follow-ups
<Refactoring, structural improvements, or future work worth considering>
```

`SUMMARY.md` is the final artifact of this skill. The user reads it and decides what to do next (commit, push, open a PR, or discard) — /opus does none of those things.

---

## Key Rules

1. **Orchestrator doesn't code** — after the interview, you dispatch, read results, and make decisions. You don't write implementation code yourself.
2. **Fresh subagent per logical unit** — clean context, no pollution.
3. **Scoped context per implementer** — each agent gets only what it needs: its task, the overall goal, and relevant interface contracts. Never the full plan.
4. **Maximize parallelization** — non-overlapping files = parallel dispatch.
5. **Review loops until approved** — never skip `NEEDS_CHANGES` feedback.
6. **Iteration limits everywhere** — max 3 fix iterations per task, per review, per verification. Fail loudly, don't grind.
7. **TDD is mandatory** — every implementer must demonstrate test-first.
8. **Verify in a clean context** — dispatch a fresh agent to gather evidence, then decide yourself.
9. **Summarize before stopping** — always produce `SUMMARY.md` as the final record of what was built.
10. **Interview questions go through `AskUserQuestion`** — not plain text.
11. **Always confirm before ending the interview.**
12. **Never bypass the Step 4 checkpoint** — the user must explicitly approve the spec before implementation begins.
13. **No version control — ever.** Neither the orchestrator nor any subagent runs `git` (any subcommand — no `diff`, `log`, `status`, `commit`, `push`, `branch`, `tag`, `stash`, nothing), invokes `gh` for anything that changes state (no `pr create`, no `issue edit`, no label or status changes), or otherwise touches the repository's version-control state. Read-only metadata fetches via `gh issue view` in Step 0 for loading seed content are the sole exception. Understanding what changed is done via the `.agent/work/<issue>/changed_files.md` manifest, not `git diff`. The user performs all version control themselves.

---

**Start now:** Run Step 0 with `$ARGUMENTS`.
