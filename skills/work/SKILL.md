---
description: "Work on an issue: plan, implement with TDD, and create PR"
disable-model-invocation: true
argument-hint: "<issue-id>"
---

# /work $ARGUMENTS

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

### Mark as In Progress

**GitHub:** `gh issue edit <issue-id> --add-label "in-progress"`
**Beads:** `bd update <issue-id> --status=in_progress`

---

## Step 1: Trivial Check

Is this trivial? (ALL must be true):
- Single file change OR config-only
- No new interfaces, classes, or public APIs
- Under 20 lines of code
- No security implications

**If trivial:** Skip to Step 3 (implement directly).
**If NOT trivial:** Continue to Step 2.

---

## Step 2: Plan

### 2a. Create Implementation Plan

Use the Agent tool with `subagent_type="general-purpose"` to create a plan:

- Explore the codebase to understand existing patterns
- Break work into small, independent tasks (one logical unit each)
- For EACH task, specify exact file paths, language (C#/TypeScript), and test file paths
- Group tasks by file overlap — tasks touching the same files must be sequential
- Write plan to `.agent-plans/<feature>.md`
- Commit the plan

### 2b. Review the Plan

Launch TWO review agents in parallel:

1. **Architecture review** (Agent subagent_type="general-purpose"):
   - Tasks are atomic (2-5 min each)
   - File paths are exact and realistic
   - TDD steps explicit
   - No unnecessary complexity (YAGNI)

2. **Security review** (Agent subagent_type="general-purpose"):
   - No hardcoded secrets planned
   - Input validation included where needed
   - Auth/authz considerations addressed
   - Logging won't expose PII
   - Error handling planned

### 2c. Fix Loop

If NEEDS_CHANGES from either reviewer: dispatch a fixer agent, then re-review. Loop until BOTH return APPROVED.

---

## Step 3: Implement (Parallel Batches)

Identify parallel batches from the plan. Tasks that touch different files can run in parallel.

For each batch of independent tasks, dispatch ALL implementers in parallel using the Agent tool.

After each batch completes, dispatch parallel code reviews.

### If NEEDS_CHANGES

Dispatch fix agent for specific task, then re-review only the fixed task.

### Proceed to next batch when current batch is fully approved

---

## Step 4: Verify

Run full verification yourself (orchestrator).

**Read the actual output.** Evidence before claims:
- Count test results (X passed, Y failed)
- Check exit codes
- If ANY failures, dispatch fixer agent — do NOT claim success

---

## Step 5: Review & Summarize

Produce a `SUMMARY.md` that captures decisions, deviations, and architectural implications.

### 5a. Parallel Review Agents

Launch TWO review agents in parallel:

1. **Issue Compliance Agent** (Agent subagent_type="general-purpose"):
   - Re-read the original issue description (from Step 0) and the implementation plan (from Step 2)
   - Compare against the actual changes (`git diff main...HEAD`)
   - Report:
     - **Delivered** — requirements fully met
     - **Partially delivered** — requirements met with caveats
     - **Skipped** — requirements not addressed (with reason)
     - **Added beyond scope** — work done that wasn't in the original issue
     - **Decisions & trade-offs** — any non-obvious choices made during implementation
     - **Deviations from plan** — where implementation diverged from `.agent-plans/` and why

2. **Architecture Analysis Agent** (Agent subagent_type="general-purpose"):
   - Read the new/changed files and their surrounding modules
   - Analyze how the new code relates to existing patterns, conventions, and structure
   - Report:
     - **Pattern consistency** — does the new code follow existing conventions?
     - **Duplication** — any new duplication introduced, or opportunities to consolidate with existing code?
     - **Coupling & cohesion** — are dependencies reasonable? Any tight coupling introduced?
     - **Suggested follow-ups** — broader refactoring or structural changes the team should consider as a result of this work (not blockers, but worth tracking as future issues)

### 5b. Synthesize into SUMMARY.md

Dispatch a **Summary Writer Agent** (Agent subagent_type="general-purpose") with the output from both review agents. It must produce `SUMMARY.md` in the repo root with the following structure:

```markdown
# Summary

## What was done
<Concise description of the delivered work>

## Decisions & trade-offs
<Non-obvious choices and their rationale>

## Deviations from plan
<Where and why implementation diverged from the original plan>

## Architectural notes
<How the new code fits with the existing codebase, pattern consistency, and any concerns>

## Suggested follow-ups
<Refactoring, structural improvements, or future work worth considering>

Closes #<issue-number>
```

Commit `SUMMARY.md` to the branch.

---

## Step 6: Complete

**Only after Step 5 is done and Step 4 passed with evidence:**

```bash
git status
git push
```

### Create a Draft PR (Default)

Use the content of `SUMMARY.md` as the PR body:

```bash
gh pr create --draft --title "<type>(<scope>): <description>" --body "$(cat SUMMARY.md)"
```

The `Closes #<issue-number>` line in `SUMMARY.md` automatically closes the issue when the PR is merged.

### Alternative: Close Issue Directly

Only for trivial issues or when no PR is needed:

**GitHub:** `gh issue close <issue-id> --comment "<summary>"`
**Beads:** `bd sync && bd close <issue-id> --reason="<summary>"`

---

## Key Rules

1. **Orchestrator doesn't code** — you dispatch, read results, make decisions
2. **Fresh subagent per logical unit** — clean context, no pollution
3. **Maximize parallelization** — non-overlapping files = parallel dispatch
4. **Review loops until approved** — never skip NEEDS_CHANGES feedback
5. **TDD is mandatory** — all implementers must demonstrate test-first
6. **Verify before done** — run and read build/test output yourself
7. **Summarize before completing** — always produce SUMMARY.md and use it for the PR description

---

**START NOW:** Run Step 0 with the provided issue ID.
