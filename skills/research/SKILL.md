---
description: "Deep research on a topic using Agent Teams"
disable-model-invocation: true
argument-hint: "<research question>"
---

# /research $ARGUMENTS

You are the **orchestrator**. You create a research plan, coordinate an Agent Team to execute it, and deliver a synthesized result. You do NOT perform research yourself — you plan, dispatch, and review.

---

## Step 0: Setup

### Research Directory

Create a directory for this research session:

```bash
mkdir -p .agent/research/<slugified-topic>
```

Use a short, descriptive slug derived from the research question (e.g., `.agent/research/event-sourcing-kotlin`, `.agent/research/postgres-vs-cockroachdb`).

All research artifacts live here. This directory serves two purposes:
1. **Working memory for the team** — teammates write findings here to manage context across long sessions
2. **Audit trail for the user** — the raw evidence behind the final synthesis, reviewable after delivery

### Output Format

The default output format is **Markdown** (`.md` file).

Ask the user to confirm this default or specify an alternative before proceeding. Use the `AskUserQuestion` tool. Do not continue until confirmed.

### Codebase Relevance

Assess whether the research question relates to the local codebase. Include codebase exploration in the plan if:
- The question references "our code," "this project," "our service," or similar
- The question involves a technology or pattern the codebase uses
- The question is about implementation approaches where existing code patterns would inform the answer

If including codebase exploration, note this in the plan so the user can see it.

---

## Step 1: Reconnaissance

Before planning, do a quick survey of the research space. This is lightweight — enough to understand the shape of the topic, not to answer the question.

Dispatch a single subagent (NOT an Agent Team member — this is pre-planning work):

Provide: the research question + instruction to do a broad, shallow survey.

The subagent should:
- Search the web for key concepts, major perspectives, and the structure of the topic
- If codebase-relevant: do a quick scan of relevant code areas to understand current state
- Identify the main sub-topics, open debates, and key dimensions of the question
- Write a brief (1-2 page) overview to `.agent/research/<topic>/00-reconnaissance.md`
- Return a summary to the orchestrator

This gives you the map you need to create a good plan. Without it, you're planning blind.

---

## Step 2: Plan

### Create a Research Plan

Based on the reconnaissance, create a research plan and write it to `.agent/research/<topic>/PLAN.md`.

The plan must include:

```markdown
# Research Plan: <topic>

## Research question
<The user's original question, restated clearly>

## Scope
<What this research covers and — critically — what it does NOT cover.
This is the natural governor for research depth.>

## Codebase integration
<Yes/No. If yes, what aspects of the codebase are relevant and why.>

## Research areas
For each area:
### <Area name>
- **Question**: What specific sub-question does this area answer?
- **Approach**: Web research, codebase exploration, or both?
- **Key angles**: What perspectives or sources should be sought?
- **Depends on**: Any other areas that should complete first? (most should be independent)

## Team structure
- Number of teammates needed
- Which teammate owns which research areas
- Any areas where teammates should actively share findings with each other

## Expected depth
<A plain-language statement of how deep this goes.
e.g., "3 teammates, 5 research areas, expect ~10-20 minutes.
This is a survey-depth exploration, not an exhaustive literature review.">

## Refinement strategy
<After initial research, what gaps are likely?
What would a second pass focus on?>
```

### Present Plan for Approval

Show the user the plan. They must explicitly approve before you proceed.

If the user wants changes:
- Adjust the plan
- Show the updated plan
- Ask for approval again

Loop until the user approves. Do not start the Agent Team until you have approval.

---

## Step 3: Execute with Agent Teams

### Launch the Team

Create an Agent Team based on the approved plan. The team lead coordinates research teammates.

**Team lead instructions:**

You are the lead of a research team. Your job is to coordinate teammates, track progress, and drive toward a complete answer to the research question.

Your mandate:
1. Assign research areas to teammates per the plan
2. Monitor progress — read teammate findings as they're written
3. Encourage cross-pollination: if one teammate discovers something relevant to another's area, tell them
4. After initial research is complete, identify gaps and direct targeted follow-up
5. When you judge the research is comprehensive enough to answer the original question, move to synthesis

Your constraints:
- Follow the approved plan's scope. Do not expand beyond it without reason.
- All findings go to `.agent/research/<topic>/` — this is the shared knowledge base
- You drive refinement iterations. If a research area is thin, direct the teammate to go deeper. If a new angle emerges, assign follow-up.

**Teammate instructions (per teammate):**

You are a research teammate. You investigate your assigned research areas and write detailed findings.

Working practices:
- Write findings to `.agent/research/<topic>/<area-slug>.md` as you go — do NOT wait until you're "done"
- Structure findings with clear sections: key facts, sources, analysis, open questions
- If you discover something relevant to another teammate's area, message them directly
- If you hit a dead end or discover the question needs reframing, message the team lead
- When doing web research: search, read sources, extract key information, note source URLs
- When exploring the codebase: read code, note patterns, reference specific files and line ranges
- Prefer primary sources (official docs, papers, original announcements) over secondary summaries
- When sources disagree, note the disagreement and the evidence for each position

Writing to `.agent/research/<topic>/`:
- Use descriptive filenames: `event-sourcing-patterns.md`, `kafka-exactly-once.md`, `our-current-architecture.md`
- Write for a knowledgeable reader — your audience is a senior engineer, not a beginner
- Include source URLs inline so the synthesis can cite them
- It's fine to write rough notes first and refine later — working memory, not polished prose

### Iterative Refinement

The team lead drives refinement. After the initial research pass:

1. **Gap analysis**: The team lead reads all findings in `.agent/research/<topic>/` and identifies:
   - Sub-questions that weren't adequately answered
   - Areas where sources conflict and more evidence is needed
   - Angles that emerged during research that the original plan didn't cover (within scope)
   - Connections between research areas that need to be explored

2. **Follow-up assignments**: The team lead directs teammates to address specific gaps. This is targeted — not "research more," but "find evidence for X" or "check whether Y applies to our codebase."

3. **Convergence check**: The team lead assesses whether the findings are sufficient to answer the original research question within the defined scope. If yes, move to synthesis. If not, another refinement pass.

There is no fixed iteration limit. The plan's scope is the natural governor. When the research areas defined in the plan have been adequately covered, the research is complete.

### Synthesis

When the team lead judges the research is complete:

1. **The team lead produces a synthesis** by reading all findings in `.agent/research/<topic>/` and writing a structured draft to `.agent/research/<topic>/DRAFT.md`.

   The draft should:
   - Answer the original research question directly
   - Organize findings into a coherent narrative, not just concatenate teammate outputs
   - Resolve or clearly present conflicting information
   - Reference source URLs from the underlying findings
   - If codebase-relevant: connect general findings to the specific codebase context
   - Note remaining uncertainties or areas where the evidence is thin

2. **Peer review**: The team lead asks teammates to review the draft against their own findings. Each teammate checks:
   - Are their findings accurately represented?
   - Is anything important missing or mischaracterized?
   - Are there nuances that got lost in synthesis?

   Teammates write review comments to `.agent/research/<topic>/REVIEW.md`.

3. **Final draft**: The team lead incorporates review feedback and writes the final version to `.agent/research/<topic>/FINAL.md`.

4. **The team lead signals completion** and the orchestrator (you) regains control.

---

## Step 4: Deliver

### Review the Output

Read `.agent/research/<topic>/FINAL.md`. This is the team's synthesized answer.

You (the orchestrator) are the last quality gate. Check:
- Does it actually answer the original research question?
- Is the scope consistent with what was approved in the plan?
- Is the structure clear and the reasoning sound?

If the output has significant issues, you can send feedback to the team lead and request revisions. But for minor issues, fix them yourself in the final output — don't restart the team for polish.

### Write the Final Output

Copy `.agent/research/<topic>/FINAL.md` to the user-facing output location. The default is a Markdown file in the working directory, named descriptively (e.g., `event-sourcing-research.md`).

Present the output to the user with a brief summary of:
- What was researched (scope)
- How deep the team went (number of sources, research areas covered)
- Where the raw findings live (`.agent/research/<topic>/`) for deeper review
- Any caveats or areas of uncertainty

### Preserve the Research Directory

Do NOT clean up `.agent/research/<topic>/`. It contains:
- `00-reconnaissance.md` — initial survey
- `PLAN.md` — the approved research plan
- Individual finding files from each teammate
- `DRAFT.md` — the synthesis draft
- `REVIEW.md` — teammate review comments
- `FINAL.md` — the final synthesis

This is the audit trail. The user can review any layer of depth — from the polished output down to the raw findings that support each claim.

---

## Key Rules

1. **Orchestrator doesn't research** — you plan, dispatch the team, and review the output
2. **Plan approval is mandatory** — never start the team without user approval of the plan
3. **Agent Teams, not subagents** — research benefits from cross-pollination between teammates
4. **Everything goes to `.agent/research/`** — findings, drafts, reviews. This is both working memory and audit trail
5. **Scope governs depth** — no arbitrary limits. The approved plan defines what's in scope; the team researches until that scope is covered
6. **The team lead drives refinement** — gap analysis and follow-up happen within the team, not by the orchestrator micro-managing
7. **Synthesis is a team effort** — the lead drafts, teammates review against their own findings, lead incorporates feedback
8. **Raw findings are preserved** — the user can always trace the final output back to the evidence

---

**START NOW:** Read the research question from `$ARGUMENTS`. Confirm output format with the user, then run Step 1 (Reconnaissance).
