---
name: special-review
description: Drive a multi-pass code review loop alternating a Claude specialist cohort with Codex until findings converge. Use this ONLY when the user explicitly invokes the skill by name — "use special-review", "special-review this", "run the special-review workflow", or equivalent direct reference. This is a specialized tool, not a default. Do NOT trigger on general review requests, "review this code", "check my changes", or similar phrasing — those go to the default review workflow (/full-review). Only activate when the user names the skill.
---

# special-review

A multi-pass code review workflow. Invoke only on explicit user instruction when a diff exists and is ready for review. This skill does not write code — it orchestrates reviewers and presents findings. After each review pass you explicitly offer the user a choice of who applies fixes (you stepping outside this skill's reviewer role, Codex, another agent, or manual) before proceeding.

When active, you take two sequential roles: first **scope reader** (Phase 1), then **orchestrator** of a review loop (Phase 2). The user retains the right to stop, redirect, or override at any point.

## Dependencies

- `feature-dev` plugin (provides `code-reviewer` agent)
- `pr-review-toolkit` plugin (provides `code-reviewer`, `silent-failure-hunter`, `type-design-analyzer`, `comment-analyzer` agents)
- `codex` CLI on PATH (for Codex review passes)
- `jq` on PATH (for parsing Codex JSON output)

## Hard rules

1. **Do NOT write or modify code.** This skill is strictly a reviewer. Present findings and return control. If the user asks you to fix something during the loop, do so as normal Claude work outside this skill's scope, then resume the next review pass when the user is ready.
2. **Do NOT run Codex in parallel with Claude subagents in the same review pass.** The two lenses differ in shape. Parallel runs conflate findings and destroy per-vendor attribution. Each pass is either the Claude cohort OR Codex, never both.
3. **Do NOT invoke the same reviewer type more than two consecutive passes.** If passes N-1 and N both used the Claude cohort, pass N+1 must be Codex (or stop). Same in reverse. This forces cross-vendor coverage.
4. **User-supplied iteration counts are ceilings, not contracts.** If the user says "run 5 passes," that is an upper bound. Stop early when findings have converged — that is the whole point of having an AI orchestrator applying judgment.
5. **Do NOT auto-finalize.** When stopping, always return control to the user with a summary; do not declare the review "done" unilaterally.

## Phase 1: Kickoff

1. **Determine review scope.** Auto-detect or use user-specified scope:
   - If on a non-default branch with commits ahead of `main`/`master`: use `git diff main...HEAD` (all branch changes). If there are also unstaged changes, note them separately — the user may want to commit first.
   - If on the default branch with unstaged changes: use `git diff` (unstaged only).
   - If user specified a scope (commit range, PR number, specific files): use that.
   - If nothing to review (no diff in any scope): say so and stop.

2. **Run `git diff --stat`** (with the determined scope) to identify changed files and approximate scope size.

3. **Check the project root for orientation docs** — `CLAUDE.md` and `AGENTS.md`. Report what you found and identify which is authoritative:
   - **Both present:** read both. Look for explicit cross-references — whichever claims authority wins. If neither claims authority, ask the user.
   - **Only `CLAUDE.md`:** CLAUDE.md is the reference doc. **Codex reviews are blocked for this run** — Codex requires AGENTS.md in the project root to discover context on its cold-start subprocess invocations (see "Running Codex" for why). Warn the user upfront and offer two options: (a) stub an AGENTS.md that points at CLAUDE.md as authoritative, then continue with Codex enabled; (b) proceed cohort-only, accepting that the alternation rule degrades to cohort-only with convergence or cap as the only stops.
   - **Only `AGENTS.md`:** AGENTS.md is the reference doc. Codex available.
   - **Neither:** both reviewers will lack project context. Warn the user strongly.

4. **Check Codex availability.** Two preconditions must both be true:
   - AGENTS.md exists in the project root (determined above).
   - `codex` CLI is on PATH (`which codex` succeeds).
   Record a **Codex-available flag** (`true` only if both pass) and carry it into Phase 2.

5. **Summarize the review scope**: changed file count, approximate diff size (lines added/removed), notable areas of the codebase touched. Announce you are starting the review loop and move to Phase 2.

## Phase 2: Review loop

### Iteration unit

**One iteration = one review pass (cohort OR Codex) + your compilation and verification of findings + presentation to the user.**

Default iteration cap: **4**. The user can override with "run N passes", "run until stable", etc.

Between iterations, the user handles fixes. The skill resumes when the user triggers the next pass. In batch mode ("run N passes"), passes auto-continue without waiting for per-pass user confirmation — but fixes are *not* skipped by default. Instead, you ask the user for a **batch fix policy** at batch kickoff (see "Batch fix policy" below) and apply that policy between passes.

### Per-iteration algorithm

**Step 1. Decide which reviewer to run this iteration.**

Decision factors:

- **Opening pick:** default to the Claude cohort for pass 1. The five-agent cohort provides broad initial coverage across style, bugs, error handling, types, and comments.
- **Post-fix verification:** after the user has fixed findings from a prior pass, prefer the reviewer whose findings were fixed — it is best positioned to verify its own fixes and catch regressions introduced by the fixes.
- **Cross-lens coverage:** if only one reviewer type has run so far, switch to the other for independent verification.
- **Respond to the previous pass:** if the last pass surfaced findings the other reviewer is better positioned to verify or extend, switch. Example: cohort flagged "potential null dereference at file:line" → switch to Codex for mechanical verification.
- **Alternation constraint (hard rule 3):** never run the same reviewer three consecutive passes. If passes N-1 and N both used the cohort, this pass must be Codex (or stop). Same in reverse.

Briefly state your pick and reasoning before dispatching ("Pass 3: Codex. Prior two passes were cohort and surfaced error-handling gaps; Codex will mechanically verify the surrounding code paths.").

**Step 2. Re-read `git diff --stat`** with the review scope. The diff may have changed since the last pass due to fixes. If the diff has changed, note the delta (files added/removed from the changeset, lines changed). If the diff is now empty, stop — nothing left to review.

**Step 3. Run the chosen reviewer.** See "Running the Claude cohort" or "Running Codex" below.

**Step 4. Compile findings.**

- Deduplicate issues found by multiple agents (in cohort passes).
- Categorize by severity: **Critical** (security vulnerabilities, data loss, crashes) / **High** (logic errors, missing error handling, broken functionality) / **Medium** (style violations, suboptimal patterns, minor bugs) / **Low** (nits, formatting, documentation).
- For each finding: note which agent(s) or reviewer found it, the file and line, and a brief description.
- Keep per-reviewer attribution. If cohort found something Codex missed, or vice versa, note it — this feeds the cross-vendor lens signal the user cares about.

**Step 5. Verify each finding** before reporting. Read the actual code to confirm the issue is real. Agents (both Claude subagents and Codex) can hallucinate line numbers, misread context, or flag non-issues. Dismiss false positives with a one-line reason so they are visible in the transcript.

**Step 6. Present findings** to the user:
- Verified findings grouped by severity
- Dismissed findings (false positives) in a separate section with reasons
- Agent retirements that occurred this pass (if any)
- Finding count trend vs. prior passes (convergence signal)

**Step 6b. Resolve the fix handoff.** Immediately after the findings report (and before proposing the next pass), decide who applies fixes. Skip this step only when there are no actionable findings.

- **Default mode:** explicitly ask the user who should apply fixes. Present the options below.
- **Batch mode:** follow the batch fix policy set at kickoff (see "Batch fix policy" below). Do not re-prompt unless the policy says to pause. Always announce in the transcript what you are doing: "Policy: you-for-all. Applying fixes to all 4 verified findings, then triggering pass 3."

Options (for default mode, or when batch policy calls for a pause):

- **You (Claude)** — you step outside this skill's reviewer role and apply fixes as normal Claude work, then resume the next pass when done. Permitted by hard rule 1.
- **Codex** — the user hands the findings to Codex (e.g., via `codex exec` in their own shell, or a separate invocation they drive). You do not dispatch Codex for fixing from inside this skill.
- **Another agent** — any subagent the user prefers *that has write tools* (`Edit`/`Write`/`NotebookEdit`). Note: the `feature-dev` agents and most review-oriented agents are read-only and cannot apply fixes — check the agent's declared tools before suggesting one.
- **Manual** — the user fixes by hand.
- **No fixes this round** — skip fixing and run another review pass on the same diff (useful for cross-lens sweeps before touching code).

Wait for the user's call before proceeding. Their answer determines what happens next: if they pick you, do the fixes, then return to Step 7. If they pick any other option, return control and resume when they signal the next pass. This is the single moment where the skill's "user decides how and when to fix" promise becomes concrete — do not skip it silently.

**Step 7. Decide the next move.**

- **Convergence check:** apply the reviewer-specific convergence criteria (see "Convergence" below). If converged, stop and report.
- **Cap reached:** stop and report.
- **Default mode (no batch instruction):** Step 6b has already resolved how fixes get applied. Once fixes are in (or the user chose "no fixes this round"), propose the next pass — which reviewer, why — and return control to the user. Resume when they signal.
- **Batch mode (user said "run N passes"):** fixes between passes follow the batch fix policy set at kickoff (see "Batch fix policy" below). Auto-continue to the next pass within the cap, applying policy-driven fix actions between passes. If the policy requires a pause (e.g., "each reviewer fixes own team" pauses on Codex findings), surface the pause and wait for the user's call, then resume the batch. Still respect convergence — stop early if findings converge before N.
- **Single-lens batch:** if Codex is unavailable, batch mode runs consecutive cohort passes (alternation rule relaxed). But cohort-on-same-code converges quickly — expect at most 2 productive passes before findings plateau.
- **Ambiguous first pass:** if pass 1 surfaced no findings, report honestly: "Pass 1 returned no findings. That may mean the code is clean, or the reviewer missed something. Propose running a Codex pass for independent cross-check." Let the user decide.

### Batch fix policy

When the user triggers batch mode ("run N passes", "run until stable", etc.), ask up front how fixes should be applied between passes. Present the policy options explicitly — the user needs to pick one before the first pass starts. Their choice applies to the whole batch unless they interrupt.

Policy options:

- **you-for-all** — you (Claude) auto-apply fixes to every verified finding after each pass, stepping outside the skill's reviewer role for each fix interlude, then trigger the next pass. No per-pass pause. Fastest loop; best when the user trusts you to fix straightforwardly and wants hands-off iteration.
- **own-team** — after each pass, you auto-apply fixes to findings from the Claude cohort (your team) only. For Codex findings, you pause and offer the user the chance to trigger Codex (or another fixer) based on the presented findings list, before resuming the batch. Rationale: each reviewer is best positioned to act on findings it owns, and the user stays in the loop for cross-vendor handoffs. When the cohort didn't run that pass (it was a Codex pass), there are no own-team findings to fix — pause for Codex routing.
- **no-fixes** — cross-lens sweep mode: skip fixes between passes entirely. Useful for back-to-back cohort + Codex on the same diff before the user starts fixing at all.
- **custom** — any user-supplied rule, e.g., "pause after each findings report to decide on the spot" (degrades to default mode within the batch), "you fix Critical/High only, leave Medium/Low for me", "let Codex fix everything". Capture the rule verbatim in the transcript so it's auditable.

At batch kickoff, announce the policy choice and summarize what will happen: "Batch of 4, policy = own-team. Cohort findings → I fix; Codex findings → I pause and offer handoff. Alternation starts with cohort."

The user can override policy mid-batch ("switch to you-for-all", "stop batching, resume default mode"). Acknowledge and adjust on the next Step 6b.

### Running the Claude cohort

Five agents launched **in parallel** (single message, multiple `Agent` tool calls). Only agents that haven't been retired participate in a given pass.

| Agent | Focus | Session cap | Rationale |
|-------|-------|-------------|-----------|
| `pr-review-toolkit:code-reviewer` | Style, conventions, CLAUDE.md adherence | 3 | Style issues don't cascade deeply. |
| `pr-review-toolkit:silent-failure-hunter` | Error handling, silent failures, inappropriate fallbacks | No cap | Error handling has deep interaction surface — fixes can introduce new gaps. |
| `pr-review-toolkit:type-design-analyzer` | Type design: encapsulation, invariant expression, enforcement | 2 | Type shapes don't change between iterations. |
| `pr-review-toolkit:comment-analyzer` | Comment accuracy, completeness, long-term maintainability | 2 | Comments are small, finite scope. Converges fast. |
| `feature-dev:code-reviewer` | Bugs, logic errors, security vulnerabilities | No cap | Bug fixes can introduce new bugs. Worth running until clean. |

**Session cap**: hard ceiling on how many cohort passes an agent participates in across this skill invocation. When reached, the agent is retired regardless of findings.

Each agent's prompt should include:

- The review scope and changed files list (from Step 2)
- Instruction to review the changes in the current git diff (specify the exact diff command to use)
- Which iteration number this is for the agent and its cap (e.g., "This is iteration 2 of 3 for this agent")
- What was already found and fixed in prior iterations — so the agent focuses on **NEW** issues, not re-raising resolved findings
- Which orientation doc is authoritative (CLAUDE.md or AGENTS.md), with instruction to check conventions
- Instruction to cite file:line for every finding
- Instruction to be terse — don't propose full rewrites, only specific issues
- Instruction to report with severity labels (Critical / High / Medium / Low)

**Wait for ALL launched agents to complete before proceeding.** Do not start compiling or responding after individual agents finish — wait until every agent in the current pass has returned.

### Running Codex

**Precondition: Codex-available flag must be `true`.** If `false`:
- Do not invoke Codex for this pass.
- Fall back to the Claude cohort instead.
- Note in the transcript that Codex was skipped and why.
- Alternation rule relaxes — cohort may run consecutively. Still respect cap and convergence.

Codex must be invoked from the **project root** (absolute path in `cd`). Never assume the current shell CWD is already project root.

**Session resume across passes.** The first Codex pass uses a fresh session; subsequent Codex passes in the same loop resume via `thread_id`. This preserves conversation context and enables prompt caching.

**Step 1. Write the fresh-pass review prompt to a temp file:**

```bash
cat > /tmp/special_review_codex_prompt.txt <<'PROMPT'
You are reviewing code changes in the {project_name} project.

Review scope: {review_scope_description}
Changed files:
{changed_files_list}

Your job is to find problems in the code changes. Focus on your strengths: mechanical errors (wrong variable names, undefined references, broken imports, stale references), immediate-implication issues (null dereferences, off-by-one errors, race conditions, missing error paths, resource leaks), and anything that a careful reader grounded in the actual codebase would catch.

Read the changed files in full. Read surrounding context (callers, callees, related tests) to understand the changes in their environment. Verify every claim against the actual code before reporting.

Report findings with severity labels:
- **Critical** — security vulnerabilities, data loss, crashes
- **High** — logic errors, missing error handling, broken functionality
- **Medium** — style violations, suboptimal patterns, minor bugs
- **Low** — nits, formatting, naming

Shape per finding: `**[severity]** description (file:line)`.

Example: `**High** Missing null check on `user.profile` before accessing `profile.name` — will throw when profile hasn't loaded yet (src/components/UserCard.svelte:47)`.

Do NOT cap your findings. Surface ALL issues you encounter — every mechanical mismatch, every missing error path, every stale reference. Thoroughness is more valuable than brevity. Cite file:line for every claim. Do not propose full rewrites — only specific issues. Do not write any code.

If you find nothing at Critical or High severity, say so plainly — "no Critical/High findings" is a convergence signal the orchestrator reads, not a failure. Do not manufacture findings to fill space.
PROMPT
```

(Substitute `{project_name}`, `{review_scope_description}`, and `{changed_files_list}` before writing.)

**Reading Codex output — session-anchoring model:**

Codex's attention is session-scoped and file-anchored. A fresh session picks whichever changed files look most complex on first read and focuses there. A resumed session stays anchored to the files it already explored. This matters for convergence:

- **Normal convergence (most reviews):** finding count drops on resumed sessions, no new Critical/High. The files Codex anchored to are clean. For a typical 3-4 pass review, this is sufficient.
- **Late-stage convergence (10+ passes):** resumed-session convergence is no longer sufficient. Fresh sessions sample different files. A fresh session that produces only theoretical findings = global convergence. See special-plan's "Reading Codex output — session-anchoring model" for the full model; it applies identically here.

**Step 2. Invoke Codex.** First pass — fresh session:

```bash
cd /absolute/path/to/project/root
codex exec --json --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check "$(cat /tmp/special_review_codex_prompt.txt)" < /dev/null > /tmp/special_review_codex_out.jsonl 2> /tmp/special_review_codex_err.log
```

All flags required: `--json` for parseable output, `--dangerously-bypass-approvals-and-sandbox` for non-interactive invocation, `--skip-git-repo-check` for repos with unusual state, `< /dev/null` to suppress stdin warning.

**Step 2b. Extract and persist `thread_id`** immediately after the first pass:

```bash
jq -r 'select(.type == "thread.started") | .thread_id' /tmp/special_review_codex_out.jsonl | head -1 > /tmp/special_review_codex_thread_id
```

If the file is empty, extraction failed — fall back to a fresh session on the next Codex pass.

**Step 2c. Subsequent Codex passes — resume the thread:**

```bash
THREAD_ID=$(cat /tmp/special_review_codex_thread_id)
cd /absolute/path/to/project/root
codex exec resume --json --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check "$THREAD_ID" "$(cat /tmp/special_review_codex_prompt.txt)" < /dev/null > /tmp/special_review_codex_out.jsonl 2> /tmp/special_review_codex_err.log
```

Key differences from fresh: `exec resume` subcommand, `thread_id` as positional arg before prompt. Do NOT pass `--cd` or `-C`. All other flags stay the same.

**Follow-up prompt for resumed passes** — keep it tight:

```
The code has been modified since your last review. Re-review the changes and report any NEW issues:

1. Regressions introduced by fixes to prior findings (new bugs, broken imports, missing error paths).
2. Issues in newly changed code that wasn't in the prior diff.
3. Any Critical/High findings you missed last time.

Do NOT re-surface findings from prior passes — the orchestrator tracks those. "No new Critical/High findings" is a valid and useful response.

Severity conventions unchanged. Do NOT cap findings.
```

**Step 3. Extract response:**

```bash
jq -r 'select(.type == "item.completed" and .item.type == "agent_message") | .item.text' /tmp/special_review_codex_out.jsonl
```

**Step 4. Verify Codex's findings** against the actual code — same discipline as cohort findings.

**Failure handling.** If `codex exec` returns non-zero, read `/tmp/special_review_codex_err.log`. On first failure: report to user, suggest running cohort instead, continue. On repeated failures: escalate and pause. Do not silently skip.

**Resume failure modes** — same as special-plan:
- "Thread not found": fall back to fresh session, re-extract thread_id, note in transcript.
- Empty thread_id file: fall back to fresh session.
- Cross-session reuse: start fresh each skill invocation. `/tmp/special_review_codex_thread_id` is safely overwritten.

### Agent retirement

Two independent retirement mechanisms, both session-scoped (reset on each new skill invocation):

**1. Session cap.** When an agent hits its cap from the table above, it is retired. No exceptions. The cap is a hard ceiling on how many cohort passes an agent participates in.

**2. Strike rule.** If an agent returns zero substantive findings on two consecutive cohort passes where it was dispatched, retire it. A finding is substantive if it is **Critical or High** severity and survives verification. Medium/Low findings do not count — an agent that only surfaces nits is not producing load-bearing signal. Dismissed findings (false positives) do not count.

Two consecutive zeros — not one — to tolerate the normal case where an agent has a dry pass but catches something on the next.

**Report every retirement** in the transcript when it happens, with the specific reason and pass numbers: "Retiring `comment-analyzer` — reached session cap of 2 after passes 1 and 3" or "Retiring `silent-failure-hunter` — zero substantive findings in passes 3 and 5."

**Once retired, stay retired for this session.** Do not re-add an agent mid-loop even if the diff grows substantially due to fixes. If the user wants a retired agent back, they can explicitly ask.

**Degraded-cohort modes:**
- **Some agents retired:** cohort continues with remaining agents. The pass still counts as a cohort pass for alternation purposes.
- **All cohort agents retired, Codex available:** cohort passes become impossible. Alternation rule relaxes — Codex may run consecutively. Announce the mode change.
- **All cohort agents retired, Codex unavailable:** no reviewer available. Stop the loop and report explicitly that every reviewer has either retired or was blocked.

### Convergence

- **Cohort convergence**: two consecutive cohort passes where no non-retired agent surfaced a substantive finding (Critical or High that survived verification).
- **Codex convergence**: finding count dropped on a resumed session AND no new Critical/High findings. For late-stage reviews (10+ passes): a fresh session that produces only theoretical findings (unlikely preconditions, deployment-context-dependent) = global convergence.
- **Loop convergence**: both the cohort AND Codex have independently converged, each by their own criteria. If only one lens has converged, the other still has signal to offer.
- **Codex-unavailable convergence**: if Codex is blocked (no AGENTS.md or no CLI), the loop converges when the cohort has converged alone. Note the single-lens limitation in the stop report.

### Stopping and reporting

Stop the loop when any of these conditions hold:

- Findings have converged per the criteria above
- Iteration cap reached
- User interrupts or asks to stop
- No remaining reviewers (all retired + no Codex)
- Diff is empty (all changes reverted or committed to a different scope)

When stopping, report:

- **Actual iteration count** and reason for stopping (e.g., "stopped at 3/4 — last two passes surfaced only Low nits, findings converged")
- **Review scope** reminder (branch, commit range, or working tree)
- **Reviewer sequence** (e.g., "cohort → Codex → cohort")
- **Per-reviewer finding tally** — how many Critical/High/Medium/Low each source surfaced across all passes, verified vs. dismissed. This feeds the cross-vendor lens observation.
- **Agent retirement log** — which agents retired and when, with reasons
- **Open items** — unresolved findings the user has not yet addressed, scope-expansion suggestions flagged but not acted on
- **Recommendation** — is the code ready to merge, or does something need to happen first?

Do not declare the review "complete" or the code "ready" unilaterally. The user decides.

## Reviewer strengths

- **Claude cohort (5 agents):** broad coverage across style, bugs, error handling, type design, and comments. Strongest on architectural patterns, convention adherence, and semantic correctness. Five agents surface diverse findings on a single pass — the tradeoff is cost per pass.
- **Codex:** mechanical verification, variable references, import correctness, off-by-one errors, missing error branches, edge cases. Weaker at broad architectural judgment. On mature diffs (many passes), tends to chase theoretical edge cases — apply the judgment model from "Reading Codex output."

First pass → cohort (broad sweep). Second pass → Codex (mechanical verification). Then alternate based on findings. If observations drift from these heuristics (e.g., Codex catches architectural issues the cohort missed), update your judgment for that pass and note it in the transcript.

## Scope reminders

- When in doubt about whether to continue, stop early and defer to the user.
- The user may interrupt to fix things or change direction. Adapt and continue.
- Briefly state each decision (reviewer pick, finding verdict) as you go — the transcript is the audit record.
- This skill reviews. The user codes. Do not conflate the two roles.
