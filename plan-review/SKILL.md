---
name: plan-review
description: Run review agents in parallel on the current implementation plan
---

# Plan Review

Run three specialized agents in parallel to audit the current implementation plan, then compile their findings.

## Dependencies

This skill requires the following Claude Code plugins:

- **feature-dev** — `claude plugins install feature-dev`
- **pr-review-toolkit** — `claude plugins install pr-review-toolkit`

## Steps

1. Read the current plan file (from the active plan path, or ask the user which plan to review)
2. Read CLAUDE.md for project conventions and architecture context
3. Launch all three agents **in parallel** (single message, multiple Agent tool calls):
   - `feature-dev:code-architect` — architecture review: does the plan follow existing patterns? Are the right files targeted? Are there missing or unnecessary changes? Does the data flow make sense?
   - `feature-dev:code-explorer` — codebase verification: read the actual files referenced in the plan and verify line numbers, function signatures, data structures, and insertion points are correct. Flag any assumptions that don't match the code.
   - `pr-review-toolkit:silent-failure-hunter` — error handling review: does the plan account for failure modes? Are there missing error paths, silent failures, or inappropriate fallbacks in the proposed design?
4. Each agent should be told:
   - The full plan contents
   - To read the actual codebase files referenced in the plan to verify assumptions
   - To check CLAUDE.md for conventions the plan must follow
   - To report errors, omissions, wrong assumptions, and areas of improvement
5. **Wait for ALL three agents to complete before proceeding.** Do not start compiling or responding after individual agents finish — wait until every agent has returned.
6. **Compile findings** into a single report:
   - Deduplicate issues found by multiple agents
   - Categorize: Errors (plan is wrong), Omissions (plan is incomplete), Improvements (plan works but could be better)
   - For each finding: note which agent(s) found it, the relevant plan section, and a brief description
   - Separate "real issues" from "false positives" with reasoning
7. **Verify each finding** before reporting it — read the actual code to confirm the issue is real. Agents can hallucinate line numbers, misunderstand context, or flag non-issues. Dismiss false positives with a brief reason.
8. Present the compiled report to the user. Do NOT auto-apply fixes to the plan — wait for instructions.
