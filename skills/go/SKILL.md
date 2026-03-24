---
description: "Work on an issue: plan, implement with TDD, and create PR"
disable-model-invocation: true
argument-hint: "<issue-id>"
---

# /go $ARGUMENTS
 
You are the **orchestrator**. You dispatch subagents, review their output, and make decisions. You do NOT write implementation code yourself.
 
---
 
## Step 0: Load Issue
 
If `$ARGUMENTS` is provided it can be a **GitHub issue number/URL** or a **beads ID** (if beads is installed).
 
### Detect Issue Source by Format
 
| Input format | Source | Example |
|--------------|--------|---------|
| Pure integer | GitHub | `123`, `7316` |
| GitHub URL | GitHub | `https://github.com/owner/repo/issues/123` |
| GitHub shorthand | GitHub | `owner/repo#123` |
| Alphanumeric with letters | beads | `PROJ-123`, `abc-def` |
 
**For GitHub issues:**
```bash
gh issue view <issue-id> --json number,title,body,labels,state
```
 
**For beads issues:**
```bash
bd show <issue-id> --json
```
 
Write the issue body to `ISSUE.md` in the working directory if it doesn't already exist. This is the source of truth for the task — all downstream agents reference it.
 
### Mark as In Progress
 
**GitHub:** `gh issue edit <issue-id> --add-label "in-progress"`
**Beads:** `bd update <issue-id> --status=in_progress`


---

## Step 1: Plan

### 1a. Create Implementation Plan

Use the Agent tool with `subagent_type="general-purpose"` to create a plan.

Provide the subagent with:
- The contents of `ISSUE.md`
- Instruction to explore the codebase and understand existing patterns

The plan agent must:
- Break work into small, independent tasks (one logical unit each)
- For EACH task, specify:
  - **Goal**: what this task accomplishes (tied to specific acceptance criteria from `ISSUE.md`)
  - **Files to modify/create**: exact paths
  - **Test files**: exact paths
  - **Interface contracts**: any types, function signatures, or APIs this task produces that other tasks depend on
  - **Dependencies**: which other tasks must complete first (by task ID)
- Group tasks by file overlap — tasks touching the same files must be sequential
- Write plan to `.agent/plans/<feature>.md` (do not commit this)
- Commit the plan

### 1b. Review the Plan

Launch TWO review agents in parallel:

1. **Architecture review** (Agent subagent_type="general-purpose"):

   Provide: `ISSUE.md` + `.agent/plans/<feature>.md`

   Check:
   - Tasks are atomic (2-5 min each)
   - File paths are exact and realistic
   - TDD steps explicit
   - No unnecessary complexity (YAGNI)
   - Every acceptance criterion from `ISSUE.md` is covered by at least one task
   - Interface contracts between tasks are consistent

2. **Security review** (Agent subagent_type="general-purpose"):

   Provide: `ISSUE.md` + `.agent/plans/<feature>.md`

   Check:
   - No hardcoded secrets planned
   - Input validation included where needed
   - Auth/authz considerations addressed
   - Logging won't expose PII
   - Error handling planned

### 1c. Fix Loop (max 3 iterations)

If NEEDS_CHANGES from either reviewer: dispatch a fixer agent, then re-review.

Loop until BOTH return APPROVED — or abort after 3 fix iterations and report what's unresolved.

---

## Step 2: Implement (Parallel Batches)

Identify parallel batches from the plan. Tasks that touch different files can run in parallel.

### Dispatch Rules

For each batch of independent tasks, dispatch ALL implementers in parallel using the Agent tool.

**Each implementer receives a scoped context — NOT the full plan.** Construct the prompt for each implementer with exactly:

1. **Overall goal** (one paragraph summary from `ISSUE.md` — the Problem and Desired behavior sections only)
2. **This task's specification** (copied from the plan: goal, files, test files)
3. **Interface contracts this task must respect** (types, signatures, APIs produced by already-completed tasks that this task depends on)
4. **Relevant code from already-completed tasks** (only if this task has dependencies — provide the specific files/diffs, not the full history)

Do NOT include: the full plan, other tasks' specifications, review feedback from other tasks, or the full issue description.

### After Each Batch

Dispatch parallel code reviews for each completed task.

Each reviewer receives: the task specification + the diff produced by the implementer. Not the full plan.

### If NEEDS_CHANGES (max 3 iterations per task)

Dispatch fix agent for the specific task with the review feedback, then re-review only the fixed task.

If a task fails 3 fix iterations, pause and report the situation. Do not continue to the next batch if a dependency has failed.

### Proceed to next batch when current batch is fully approved

---

## Step 3: Verify

Dispatch a **verification agent** (Agent subagent_type="general-purpose") with a clean context.

Provide the agent with:
- The commands to run (build, test, lint — whatever the project uses)
- Instruction to run the commands and report raw results: exact pass/fail counts, exit codes, and any error output

The verification agent does NOT interpret or judge — it gathers evidence and reports.

**You (the orchestrator) then read the results and decide:**
- ALL tests pass and build succeeds → proceed to Step 4
- ANY failures → dispatch a fixer agent with the failure output, then re-verify (max 3 attempts)
- After 3 failed verification cycles → stop and report

---

## Step 4: Review & Summarize

### 4a. Parallel Review Agents

Launch TWO review agents in parallel:

1. **Issue Compliance Agent** (Agent subagent_type="general-purpose"):

   Provide: `ISSUE.md` + `git diff main...HEAD`

   Report:
   - **Delivered** — acceptance criteria fully met (reference specific criteria from `ISSUE.md`)
   - **Partially delivered** — criteria met with caveats
   - **Skipped** — criteria not addressed (with reason)
   - **Added beyond scope** — work done that wasn't in the original issue
   - **Decisions & trade-offs** — any non-obvious choices made during implementation
   - **Deviations from plan** — where implementation diverged from `.agent/plans/` and why

2. **Architecture Analysis Agent** (Agent subagent_type="general-purpose"):

   Provide: the new/changed files + their surrounding module structure

   Report:
   - **Pattern consistency** — does the new code follow existing conventions?
   - **Duplication** — any new duplication introduced, or opportunities to consolidate?
   - **Coupling & cohesion** — are dependencies reasonable? Any tight coupling?
   - **Suggested follow-ups** — structural improvements worth considering (not blockers)

### 4b. Synthesize into SUMMARY.md

Dispatch a **Summary Writer Agent** (Agent subagent_type="general-purpose") with the output from both review agents. It must produce `SUMMARY.md` in the repo root:

```markdown
# Summary

## What was done
<Concise description of the delivered work>

## Acceptance criteria status
- [x] Criterion from issue — met
- [x] Criterion from issue — met
- [ ] Criterion from issue — not met (reason)

## Decisions & trade-offs
<Non-obvious choices and their rationale>

## Deviations from plan
<Where and why implementation diverged from the original plan>

## Architectural notes
<How the new code fits with the existing codebase, pattern consistency, and any concerns>

## Suggested follow-ups
<Refactoring, structural improvements, or future work worth considering>
```

Do not commit `SUMMARY.md` to the branch.


---

## Step 5: Complete

**Only after Step 4 is done and Step 3 passed with evidence:**

```bash
git status
git push
```

### Create a Draft PR

```bash
gh pr create --draft --title "<type>(<scope>): <description>" --body "## Summary

<brief description of changes>

Closes #<issue-number>
"
```

The `Closes #<issue-number>` syntax automatically closes the issue when the PR is merged.


---

## Key Rules

1. **Orchestrator doesn't code** — you dispatch, read results, make decisions
2. **Fresh subagent per logical unit** — clean context, no pollution
3. **Scoped context per implementer** — each agent gets only what it needs: its task, the overall goal, and relevant interface contracts. Never the full plan.
4. **Maximize parallelization** — non-overlapping files = parallel dispatch
5. **Review loops until approved** — never skip NEEDS_CHANGES feedback
6. **Iteration limits everywhere** — max 3 fix iterations per task, per review, per verification. Fail loudly, don't grind.
7. **TDD is mandatory** — all implementers must demonstrate test-first
8. **Verify in a clean context** — dispatch a fresh agent to gather evidence, then decide yourself
9. **Summarize before completing** — always produce SUMMARY.md and use it for the PR description

---

**START NOW:** Run Step 0 with the provided issue ID.
