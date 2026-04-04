---
name: full-review
description: Run all review agents in parallel on the current diff
---

# Full Review

Run all five code review agents in parallel on the unstaged git diff, then compile their findings.

## Dependencies

This skill requires the following Claude Code plugins:

- **pr-review-toolkit** — `claude plugins install pr-review-toolkit`
- **feature-dev** — `claude plugins install feature-dev`

## Iteration Caps

When `full-review` is run multiple times in a session, track how many iterations each agent has completed. Retire agents that have reached their cap — do not launch them in subsequent iterations.

| Agent | Cap | Rationale |
|-------|-----|-----------|
| `pr-review-toolkit:comment-analyzer` | 2 | Comments are small, finite scope. Converges fast. |
| `pr-review-toolkit:type-design-analyzer` | 2 | Type shapes don't change between iterations. |
| `pr-review-toolkit:code-reviewer` (style) | 3 | Style issues don't cascade deeply. |
| `pr-review-toolkit:silent-failure-hunter` | No cap | Error handling has deep interaction surface — fixes can introduce new gaps. |
| `feature-dev:code-reviewer` (bugs) | No cap | Bug fixes can introduce new bugs. Worth running until clean. |

When all capped agents are retired and the uncapped agents return clean, the review is complete.

## Steps

1. Run `git diff --stat` to identify changed files
2. Launch agents **in parallel** (single message, multiple Agent tool calls) — only agents that haven't reached their iteration cap:
   - `pr-review-toolkit:code-reviewer` — style, conventions, CLAUDE.md adherence (cap: 3)
   - `pr-review-toolkit:silent-failure-hunter` — error handling, silent failures, inappropriate fallbacks (no cap)
   - `pr-review-toolkit:type-design-analyzer` — type design quality: encapsulation, invariant expression, enforcement (cap: 2)
   - `pr-review-toolkit:comment-analyzer` — comment accuracy, completeness, long-term maintainability (cap: 2)
   - `feature-dev:code-reviewer` — bugs, logic errors, security vulnerabilities (no cap)
3. Each agent should be told:
   - To review the unstaged changes in the current git diff
   - Which files changed (from step 1)
   - To ignore files that are clearly unrelated documentation-only changes
   - Which iteration number this is for the agent (e.g., "This is iteration 2 of 3 for this agent")
   - What was already found and fixed in prior iterations (so the agent focuses on NEW issues only)
4. **Wait for ALL launched agents to complete before proceeding.** Do not start compiling or responding after individual agents finish — wait until every agent has returned.
5. **Compile findings** into a single report:
   - Deduplicate issues found by multiple agents
   - Categorize by severity (Critical / High / Medium / Low)
   - For each issue: note which agent(s) found it, the file and line, and a brief description
   - Separate "real issues to fix" from "non-issues / false positives" with reasoning
   - Note which agents were retired this iteration due to reaching their cap
6. **Verify each finding** before reporting it — read the actual code to confirm the issue is real. Agents can hallucinate line numbers, misunderstand context, or flag non-issues. Dismiss false positives with a brief reason.
7. Present the compiled report to the user. Do NOT auto-fix anything — wait for instructions.
