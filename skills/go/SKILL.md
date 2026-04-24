---
name: go
description: Execute a planned epic, branch, or leaf by dispatching leaf orchestrators against the `ae` plan
disable-model-invocation: true
user-invocable: true
arguments: "<id>"
---

# Epic Work Skill

You are the **epic orchestrator**. You dispatch leaf orchestrators, monitor
their results, and update epic state in `ae`. You do NOT write code
yourself. You do NOT perform version control actions (commit, push, branch,
PR) unless the user explicitly asks.

The plan already lives in `ae` — every leaf has a finalized body, composed
context, and Implementation notes from `/epic-plan`. Your job is to walk
the plan: pick ready leaves, dispatch work, apply status transitions, and
summarize on completion.

---

## Flow: `/go <id>`

1. **Resolve scope** — determine whether `<id>` is the epic root, a
   branch, or a leaf; establish which leaves are in scope.
2. **Execute leaves** — loop through in-scope leaves in dependency order,
   dispatching a leaf orchestrator for each.
3. **Summarize** — scope-dependent finalization.
4. **Report** — tell the user what happened.

---

## Step 0: Resolve scope

Verify the task exists and find its shape:

```
ae task <id>                 # does it exist?
ae task:list parent=<id>     # does it have children?
```

Classify `<id>`:

- **Epic root** — no colons in the id (e.g., `my-epic`). All descendant
  leaves are in scope.
- **Branch** — has children. All leaves under this subtree are in scope.
- **Leaf** — no children. Exactly one leaf is in scope.

Scope determines what "Step 2: Summarize" does later.

### Single-leaf scope — inline, no dispatch

If exactly one leaf is in scope (either the id itself is a leaf, or the
whole epic was never split), do not spawn a separate orchestrator. Read
`leaf-orchestrator.md` adjacent to this SKILL.md and follow it yourself
against the target leaf. You still own the status transitions described
in Step 1c.

---

## Step 1: Execute leaves (multi-leaf scope)

### 1a. Collect the next batch of ready leaves

```
ae task:next <epic>
```

This returns the first `pending` leaf whose sibling dependencies are
satisfied. Filter to your scope:
- Epic scope: any leaf in the epic qualifies.
- Branch scope: only leaves whose id starts with `<branch>:`.

**Assemble a parallel batch.** Start the returned leaf with
`ae task:start <leaf>` so it becomes `active`, then call `ae task:next`
again. If it returns another in-scope leaf, start that too, and repeat.
Stop when `task:next` returns empty, a leaf outside your scope, or you
hit a reasonable batch cap (e.g. 4 parallel leaves).

**File-conflict assumption.** `task:next` uses sibling `:after` edges,
not file analysis. Parallel dispatch is safe only if the planner encoded
file conflicts as `:after` deps during `/epic`. If a batch member's
body (`ae show <leaf>`) reveals it touches files another batch member is
also modifying, don't dispatch them together — stop the batch at the
last non-conflicting member, leave the rest pending for the next round.

If the initial `task:next` returns empty, or every ready leaf is out of
scope — all in-scope work is terminal. Go to Step 2.

### 1b. Dispatch leaf orchestrators in parallel

Read `leaf-orchestrator.md` (adjacent to this SKILL.md) once per session
— you'll reuse its content across dispatches.

In a **single message**, dispatch one fresh `general-purpose` subagent
per leaf in the batch. Each prompt contains:
- The full content of `leaf-orchestrator.md` (operating instructions).
- The leaf id to work on.
- The required return format:
  ```
  {
    "status": "done" | "blocked" | "abandoned",
    "reason": "<only if blocked or abandoned>",
    "summary": "<one or two sentences>"
  }
  ```

Wait for all subagents in the batch to return. Each handles its own
planning, implementation, verification, reviews, records, and handoff
context.

### 1c. Apply results

For each returned result, apply status:

- **done** → `ae task:done <leaf>`.
- **blocked** → `ae task:block <leaf> <reason>`.
- **abandoned** → `ae task:abandon <leaf> <reason>`.

Then decide how to proceed:

- **No blocks in the batch** → back to 1a for the next batch.
- **Any block in the batch** → **STOP.** Wait for the remaining
  siblings in this batch to finish (they're already running), apply
  their results, then go to Step 3. Do not assemble another batch. The
  user decides whether to resume.

Blocks halt the whole run. Abandons do not.

### Single-leaf batches

When only one ready leaf exists, 1a returns a batch of one and 1b
dispatches a single orchestrator. No special-case logic needed.

---

## Step 2: Summarize

Reached only when Step 1 completes normally (all in-scope leaves are
terminal — done or abandoned — with no blocks).

### Epic scope → epic summary attribute

Gather material:
- Records: `ae task:records <epic>`
- Each leaf's handoff context: `ae context <leaf>`

Compose the summary (markdown, moderate length) covering:
- What was accomplished
- Anything abandoned, with reason
- Non-obvious decisions made during execution
- Suggested follow-ups

Write it:

```
echo "$summary" | ae attr:set <epic> summary
```

### Branch scope → branch context rollup

Update the branch's context with a delta-focused rollup:

```
echo "$markdown" | ae task:set-context <branch>
```

Short. The branch context still has to compose with downstream planning,
so include only:
- What this subtree delivered (one or two lines)
- Unresolved or abandoned items that affect neighboring work
- Decisions made mid-execution that other subtrees need to respect

Don't duplicate the body or what's in children's handoff contexts.

### Leaf scope → nothing

The leaf orchestrator already wrote its handoff context. You're done.

---

## Step 3: Report

Tell the user:
- The scope you worked on and the outcome (all completed / stopped at a
  blocked leaf).
- Per leaf: status + one-line summary (pull from the structured result
  returned in 1b, or from handoff contexts).
- Where any written summary lives (epic attribute, branch context).
- Anything that needs human attention: the blocking leaf and reason,
  abandoned items, suggested follow-ups.

**No git actions.** If the user wants to commit, push, or open a PR,
they'll ask separately.

---

## Key rules

1. **Orchestrator doesn't code** — dispatch, read results, apply status.
2. **No version control actions** — no commits, pushes, PRs, or branch
   work unless the user explicitly asks.
3. **Status transitions pass through you** — `task:start` before dispatch;
   `task:done` / `task:block` / `task:abandon` after the result. The leaf
   orchestrator reports; you apply.
4. **Records and handoff context are written by the leaf orchestrator** —
   you don't touch them.
5. **Blocks stop everything.** Report to the user and wait.
6. **Parallel leaves when deps allow.** Assemble a batch via repeated
   `ae task:next` + `ae task:start`, then dispatch all leaf orchestrators
   in a single message. The planner is responsible for encoding
   file-level conflicts as sibling `:after` deps during `/epic`; if the
   plan didn't capture them, parallel runs may collide — stop the batch
   before conflicting members.
7. **Single-leaf scope is inline** — skip the dispatch layer.

---

## `ae` command reference (subset used by this skill)

```
ae task <id>                            # read a task (JSON)
ae show <id>                            # read task body (plain text)
ae context <id>                         # read composed context (plain text)
ae task:list parent=<id>                # list children (shape check)
ae task:next <epic>                     # next ready pending leaf
ae task:start <leaf>                    # pending/blocked → active
ae task:done <leaf>                     # active → done
ae task:block <leaf> <reason>           # active → blocked
ae task:abandon <leaf> <reason>         # → abandoned
ae task:records <id>                    # read subtree records
printf ... | ae task:set-context <id>   # write context (any task, stdin)
printf ... | ae attr:set <epic> summary # write epic summary (stdin)
```

Task commands return `{"ok": bool, "data": ..., "error": ...}`. Check
`ok` before proceeding; surface errors to the user.
