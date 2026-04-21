---
name: debate-plan
description: End-to-end plan workflow optimized for automated/unattended runs (approvals off). Drafts an execution-spec plan, runs a two-team review (cohort + baseline-Codex independently), correlates findings, then debates substantive picks with Codex until convergence. The highest rung of the plan-review ladder (plan-review → special-plan → debate-plan); use when maximum rigor and full automation are wanted. Use ONLY when the user explicitly invokes by name ("use debate-plan", "debate-plan this task"). Do NOT trigger on general plan requests.
---

# debate-plan

Draft → two-team independent review → correlate → debate. Highest-rigor rung of the plan-review ladder. Use when approvals are off and maximum bug-catching plus design-judgment refinement is worth the extra Codex calls and orchestration time.

## Plan-review ladder

| Skill | Drafting | Review shape | Cost |
|---|---|---|---|
| `plan-review` | no | Single Claude pass | low |
| `special-plan` | yes | Alternating cohort ↔ Codex passes | mid |
| `debate-plan` | yes | Two-team independent → correlate → debate | high |

`debate-plan` is a true superset of `special-plan`: drafting protocol is identical, review phase is strictly more thorough. Choice between the two is preference-based.

## Dependencies

- `feature-dev` plugin (provides `code-architect` and `code-explorer` agents for the Phase 1 cohort)
- `pr-review-toolkit` plugin (provides `silent-failure-hunter` for the Phase 1 cohort)
- `synthesis-author` plugin (provides the `synthesis-author` subagent used in Phase 3)
- `codex` CLI on PATH (required — debate partner has no substitute)
- `jq` on PATH (for parsing Codex JSON output)
- `AGENTS.md` in project root (required for Codex — it reads this on cold-start subprocess invocations)

## Preconditions

- A task description exists (user request, prior conversation, or both). An existing plan path may be passed in, which skips Phase 0.
- Project `CLAUDE.md` already loaded by being in-project — do not re-read.
- Approvals off (or auto-accepted). The workflow is designed for unattended runs.

## Why two independent teams

Running cohort and baseline-Codex independently on the raw plan — then debating on the union of findings — produces strictly more coverage than giving Codex a synthesis to critique. Feeding Codex a synthesis narrows its attention onto the orchestrator's framing; a free mechanical scan does not. The teams hit complementary surfaces:

- **Cohort** — design fit, pattern conformance, failure-mode enumeration, architectural drift.
- **Codex baseline** — mechanical correctness, file/line citations, signature mismatches, declaration-vs-implementation drift, import survival.

Expected overlap: ~10–15%. Running both is strictly more coverage than either alone.

## Workflow

Ten steps across five phases. Each phase has a trivial-application checkpoint so the substantive set stays small. The final phase tightens the reviewed plan for prescription density.

### Phase 0 — Draft the plan

Skip if the user passed an existing plan path.

Otherwise, draft:

1. **Filename** — lowercase kebab-case, ends in `.md`, descriptive. Not adjective-verbing-noun auto-patterns.
2. **Location** — detect the project's plan directory in this order: plan-directory references in `CLAUDE.md`/`AGENTS.md`, then `docs/plans/`, `plans/`, `.plans/`, `handoff/`, then project root as fallback. Never use `~/.claude/plans/`.
3. **Altitude: execution-spec.** Concrete file paths, line numbers, function signatures, step-by-step edits, named failure modes, distinctive error strings. No "you-know-what-I-mean" handwaving.
4. **Invariant matrix** near the top — declared authoritative. Tables encode cross-section claims: shared-state ownership, interface payload shapes, code-snippet fidelity labels (Literal / Sketch / Do-not-copy), upstream requirement status, per-commit compilation scope, cross-cutting guard sites. Include only the row types that match the plan's structural risks. Rationale: review passes converge locally within a section faster than globally across sections; a matrix makes cross-section invariants mechanically checkable in one place.
5. Use `Write` to create the file. Do not use plan mode. Do not call `ExitPlanMode`.
6. Tell the user the plan path. Move directly to Phase 1 — drafting does not need approval; the review phases are how the plan gets validated.

If the task is ambiguous enough that drafting would require guessing on structural choices, ask one or two targeted clarifying questions before drafting. Otherwise draft on best understanding and let review catch misalignments.

### Phase 1 — Independent two-team review

**Strict serial ordering: Step 1 → Step 2 → Step 3 → Step 4.** Parallelism is allowed *within* Step 1 (three cohort agents in one message) but not *across* steps. Each trivial-application checkpoint exists to shrink the findings surface the next agent sees. Dispatching the baseline Codex in parallel with the cohort — or before applying cohort trivials — burns Codex's attention budget on issues already accepted, and adds redundant noise to the synthesis. This is a common temptation because the cohort wait feels like dead time; resist it. Stage prompts on disk if you want, but do not fire them.

#### Step 1. Cohort dispatch (three agents, parallel)

Dispatch three agents in a **single response** (multiple Agent tool-calls in one message — not serial):

- `feature-dev:code-architect` — design fit, pattern conformance, architectural drift, matrix-vs-body consistency.
- `pr-review-toolkit:silent-failure-hunter` — error paths, silent failures, inappropriate fallbacks in the proposed design.
- `feature-dev:code-explorer` — mechanical verification. Opens every file the plan cites; checks line numbers, function signatures, import survival, per-commit compilation claims, and matrix entries against real code.

All three always dispatched for execution-spec plans; none is optional.

Per-agent prompt template (customize the agent-specific directive):

```
Review the implementation plan at <plan-path>.

Authoritative project docs:
- <project-root>/AGENTS.md
- <project-root>/CLAUDE.md
- <other-relevant-docs>

The plan does <one-sentence summary>. It cites specific files in <list>.

<agent-specific angle — one paragraph naming this agent's lens>.

Verify every claim against actual code before reporting. Cite file:line for each finding.

Categorize:
- [Error] — plan is wrong
- [Omission] — plan is missing something load-bearing
- [Improvement] — plan works but could be cleaner

Severity: High / Medium / Low.

For every finding, propose a specific fix (which plan section to change, how, main tradeoff).

Do NOT edit the plan. Do not cap findings. Do not propose whole-plan rewrites — only specific changes.
```

Before moving to Step 2, confirm three dispatch records exist. Wait for all three to complete. **Do not dispatch Step 3's baseline Codex until Step 2's trivial-application is done** — see Phase 1 preamble.

#### Step 2. Apply cohort's trivials directly

Triage the combined cohort findings:

- **Trivial** — single target, one obviously correct fix, low-risk (wrong line, typo, matrix/body drift, missing import in a snippet). Apply with `Edit`.
- **Substantive** — multiple viable resolutions, design choice, framing question. Park as "cohort substantives."
- **Dismissed** — false positives. Drop with a one-line reason.

Announce: "Cohort: N findings — X trivial (applied), Y substantive (parked), Z dismissed."

#### Step 3. Baseline Codex review pass

**Precondition: Step 2 is complete.** Cohort trivials must be applied to the plan file before this step. Do NOT dispatch Codex until `Edit` calls from Step 2 have landed. If Step 2 produced zero trivials, announce "Cohort: 0 trivial" and proceed — the ordering rule still holds, but the application itself is a no-op.

Fresh Codex session, plain review prompt — NO synthesis context, NO cohort findings shared. Codex sees the post-cohort-trivial plan and finds what it finds. This is the anti-narrowing step: feeding Codex a synthesis to critique narrows its attention; a free mechanical scan does not.

**If Codex drifts (wrong plan, hallucinated structure, off-topic findings): diagnose, then run fresh.** Do NOT resume the polluted thread to "correct" it. Codex is a tool: garbage in, garbage out. Once the context contains detailed findings about the wrong artifact, those findings anchor attention — a correction message sits on top of that noise rather than purging it. The recovery is: (1) check your own prompt file to confirm the input wasn't garbage; (2) skim Codex's output to diagnose what it latched onto (often: an earlier plan in project history, a wrong path, or a sibling plan with similar structure); (3) draft a tighter prompt that names the plan explicitly and lists anti-confusion specifics ("this plan does NOT touch X"); (4) dispatch `codex exec` fresh with a new `thread_id`, not `codex exec resume`. Resume is reserved for legitimate continuation (Step 7 critique). Kill stale background runs before firing the fresh one.

**Codex proposes fixes but does not apply them.** Proposed fixes are load-bearing for the debate's bidirectional attack surface. Without them, only the orchestrator's picks are debatable — Codex can attack but can't be attacked.

Prompt template:

```
Review the implementation plan at <plan-path>.

Authoritative project docs (read for context):
- <project-root>/AGENTS.md
- <project-root>/CLAUDE.md
- <other-relevant-docs>

The plan introduces <one-sentence summary>. It cites specific files in <list>.

Find errors, bugs, wrong assumptions, missing failure modes, contradictions between sections, matrix-vs-body drift, and violations of project conventions. Verify every claim against actual code before reporting. Cite file:line for each finding.

Categorize:
- [Error] — plan is wrong
- [Omission] — plan is missing something load-bearing
- [Improvement] — plan works but could be cleaner

Severity: High / Medium / Low.

For every finding, propose a specific fix:
**Proposed fix:** <which plan section to change, what to add/remove/modify, main tradeoff>.

Do NOT edit the plan. Describe the change in enough detail that a synthesis phase can either adopt or counter it.

Do not cap findings. Do not propose whole-plan rewrites — only specific changes. Do not write any code.

If nothing lands at High severity, say so plainly.
```

Invocation (fresh, save thread_id for resume):

```bash
cd /absolute/path/to/project/root
codex exec --json --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check "$(cat /tmp/debate_plan_baseline.txt)" < /dev/null > /tmp/debate_plan_baseline_out.jsonl 2> /tmp/debate_plan_baseline_err.log
jq -r 'select(.type == "thread.started") | .thread_id' /tmp/debate_plan_baseline_out.jsonl | head -1 > /tmp/debate_plan_baseline_thread_id
jq -r 'select(.type == "item.completed" and .item.type == "agent_message") | .item.text' /tmp/debate_plan_baseline_out.jsonl
```

Optional resume pass to catch sections covered lightly — typically surfaces 30–50% of first-pass count in net-new findings:

```bash
THREAD_ID="$(cat /tmp/debate_plan_baseline_thread_id)"
codex exec resume "$THREAD_ID" --json --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check "$(cat /tmp/debate_plan_baseline_p2.txt)" < /dev/null > /tmp/debate_plan_baseline_out2.jsonl
```

Resume prompt: "Re-review. Find issues you missed on the first pass. Focus on sections covered lightly, cross-section drift, failure modes declared but not exercised by tests, and Literal snippets that would fail to compile. Do NOT re-surface first-pass findings."

#### Step 4. Apply baseline's trivials directly

Same triage rules as Step 2. Park substantives as "baseline substantives."

Announce: "Baseline Codex: N findings — X trivial (applied), Y substantive (parked), Z dismissed."

### Phase 2 — Correlate

#### Step 5. Dedupe and classify the union

Walk both substantive lists. For each finding:

- **Same issue, different framing** — merge. If both teams proposed fixes, keep both as options.
- **Different issues** — keep separate. Baseline-sourced findings carry Codex's Step-3 proposal verbatim — that proposal becomes input to the synthesis.
- **Same finding, different fix** — keep both options; decide in synthesis.

Re-classify the deduped union:

- **Trivial-after-correlation** — looked substantive in isolation but becomes trivial when both framings are visible (e.g., one team diagnoses, the other names the fix). Apply now.
- **Bug-shaped substantive** — real issue, one obviously correct answer. Debate converges fast. Synthesize tersely.
- **Design-judgment substantive** — multiple viable choices with real tradeoffs. The high-value debate fodder.
- **Dismissed** — false positives from either team.

Announce: "Union after correlation: N substantive — X bug-shaped, Y design-judgment, Z trivial-after-correlation (applied), W dismissed."

### Phase 3 — Debate

#### Step 6. Author the synthesis artifact via the `synthesis-author` subagent

The synthesis is authored by the **`synthesis-author` subagent** (Opus, dispatched via the `Agent` tool with `subagent_type: 'synthesis-author:synthesis-author'`). The orchestrator does NOT write the synthesis directly.

**Why a separate subagent.** The orchestrator knows the workflow shape (cohort → baseline → synthesis → Codex critique → implementation). That knowledge produces a measurable failure mode: when the orchestrator-as-author knows a critique pass follows, the author's effort drops — Picks ship as sketches, Uncertainty fields become parking lots for decisions the author could resolve but defers, file:line claims go unverified because "Codex will catch it." Splitting author from orchestrator removes the workflow context the failure mode feeds on. The subagent has no model of "later passes"; from its perspective the spec is the only deliverable.

**Location.** Same directory as the plan, suffix `-debate-synthesis.md`. Announce the path before dispatching.

**Inputs.** The subagent reads notes files from disk. Two notes files, one per source:

```bash
# Cohort notes — write the parked substantives from Step 2 here.
cat > /tmp/debate_plan_notes_cohort.md <<'NOTES'
# Cohort flags
... (parked cohort substantives)
NOTES

# Baseline notes — write the parked substantives from Step 4 here.
# Include Codex's Step-3 proposed fixes verbatim.
cat > /tmp/debate_plan_notes_baseline.md <<'NOTES'
# Baseline flags
... (parked baseline substantives, including Codex's "Proposed fix:" blocks)
NOTES
```

The orchestrator's job at this step is purely transcription — parked substantives go to disk in the form the subagent can consume. Trivials are NOT included (already applied). Dismissed flags are NOT included.

**Dispatch.** Single `Agent` tool call. The dispatch prompt provides the inputs and nothing else:

```
Repository: <absolute path>.
Artifact under review: the implementation plan at <plan-path>.
Notes files:
- /tmp/debate_plan_notes_cohort.md
- /tmp/debate_plan_notes_baseline.md
Output spec path: <synthesis-path>.

Produce the per-finding implementation spec. Verify every flag against the actual plan and codebase before writing.
```

**Hard rule on the dispatch prompt.** Do NOT mention: "review", "reviewer", "critique", "round", "debate", "another pass", "this will be implemented", "downstream verification", "Codex", or any other workflow-shape signal. The subagent must believe the spec is the final deliverable. The dispatch prompt is bounded to inputs + outputs + repo context.

**Read the produced synthesis.** The subagent writes the spec at the announced path. The orchestrator reads it before proceeding to the Codex critique. The structure inside the spec is the subagent's responsibility (Source / Evidence / Read / Options / Pick / Concrete change / Uncertainty per finding) — the orchestrator does not enforce shape, only reads what was produced.

If the spec is empty or markedly thinner than the input flags warranted, that's a signal — re-dispatch with sharper notes files OR fall back to orchestrator-authored synthesis with explicit acknowledgement of the regression.

#### Step 7. Dispatch Codex critique

Resume the baseline Codex thread (plan already in context). Critique prompt:

```
Read <synthesis-path>. It contains proposed positions on N substantive findings (A through Z), surfaced jointly by other reviewers and your own baseline review.

Critique the reasoning in the synthesis — do not re-review the plan. For each A–Z, find problems with the read, the options enumeration, the pick, the concrete change, the uncertainty.

**Baseline-sourced findings have two postures.** Findings where you proposed a fix in your Step-3 review show adoption-or-counter context in the synthesis Pick rationale:
- If the synthesis ADOPTED your proposed fix, verify the adoption didn't subtly change it (wrong section, weakened wording, altered scope). Plain-OK otherwise.
- If the synthesis COUNTERED your proposed fix, defend your original if you still stand by it. Concede cleanly if the counter lands.

Cohort-sourced findings get ordinary critique against the plan and code.

Verify specific claims against actual code before objecting. Cite file:line.

Plain-OK on any finding is the convergence signal. Do not manufacture objections. Do not soften critique to look generous.

End with a two-line summary: which findings you approve, which you push back on.
```

After each critique pass, the orchestrator adjudicates: per finding, integrate Codex's correction, counter when the verified code supports the spec's Pick (or a third option), or defer to the user. The orchestrator may also independently catch a spec Pick that Codex plain-OK'd but the verified code shows is wrong — append a revision in that case too. Append-only revisions to the synthesis. Iterate until convergence or a hard cap of four critique passes.

**Operate in pragmatic mode.** State positions, defend as long as the defense holds against verified code, fold cleanly when it doesn't. Zero counters across multiple passes is a flag — either Codex is right on everything (possible but suspicious), the spec was already strong enough that Codex finds nothing to correct (possible — the subagent's verification floor is high), or the orchestrator is conceding for tone reasons. Distinguish via per-finding evidence: mostly plain-OK without invasive changes is the spec-was-strong signal.

#### Step 8. Apply or hand off

Present:

- Findings landscape — cohort-only / baseline-only / both / debate-emergent.
- Convergence shape — pass count; which findings moved when.
- Verdict with per-finding tally (scoring below).
- Revisions ready to apply, with specific plan sections.

Wait for user instruction before editing the plan.

**Bidirectional verdict scoring.** Per substantive finding, score one of:

- **Orchestrator win** — (a) Codex plain-OK'd the Pick on a cohort-sourced finding; (b) the orchestrator countered Codex's baseline-proposed fix and the counter held; (c) Codex's objection was cosmetic and didn't change the Pick.
- **Codex win** — (a) Codex materially corrected the Pick (new option added, new constraint, changed diagnosis); (b) the orchestrator adopted Codex's baseline proposal and it held through debate; (c) the orchestrator's counter to a baseline-proposed fix failed and folded.
- **Option-pruning (no win)** — Codex eliminated an option the orchestrator had already listed, and the orchestrator fell back to another option from its own list.

Verdict: more wins = winner. Ties go to Codex (fresh-POV rationale). Announce with per-finding tally: "Codex 3–1 with 2 prunings (won D/G/K, lost H; pruned C/F)." The verdict is informational — the user decides what to apply.

### Phase 4 — Prescription pass

#### Step 9. Tighten the finalized plan

Runs once Step 8 revisions are applied and the plan is in its final reviewed form. This is the last step before exit — correctness is settled; this pass is purely about shape.

Goal: raise prescription density. The implementer needs concrete instructions, not a defense of decisions already made.

**Cut:**
- **Justification prose.** "We chose X because Y" — unless Y is a live constraint the implementer must not violate. Committed decisions don't need defense; they need instruction.
- **Historic reasoning.** "Previously this was Z, we're moving to W" — archaeology. `git log` owns the past. Describe the target state directly.
- **Framing prose.** Section preambles ("This section covers…"), hedge phrases ("Note that…", "It's worth mentioning…"), meta-commentary about the plan itself.
- **Alternatives-considered logs.** "We considered A and B but picked C" — unless A/B are likely re-litigation risks during implementation.

**Keep:**
- **Prescriptions.** Concrete edits: file paths, line numbers, function signatures, step-by-step instructions.
- **Constraints.** Invariants, guard sites, failure modes, distinctive error strings — the non-negotiable shape of the target state. The invariant matrix is load-bearing; do not touch.
- **Load-bearing rationale.** A one-line "why" that prevents a wrong implementation choice. Test: if cutting the line would let an implementer pick wrong, keep it.

**Style shift:**
- Indicative prose ("The handler validates X and returns Y") → imperative ("Validate X. Return Y.").
- Multi-sentence explanations of a single change → one-sentence prescription plus optional one-line constraint.
- Adjective-heavy framing ("This critical change ensures robust…") → stripped ("Change X to Y.").

Apply edits directly with `Edit`. Do not restructure sections, renumber steps, or alter the invariant matrix. One pass — no iteration, no review loop.

Test for any retained sentence: "Does the implementer need this to do the task correctly?" If no, cut.

Announce: "Prescription pass: N sections tightened." Then exit the flow.

## Calibration — when the skill is doing its job

Working as designed when:
- Cohort and baseline-Codex surface different things; overlap stays small (~10–15%).
- Baseline-Codex's Step-3 proposals give the orchestrator real attack surface — counters happen, and at least some hold.
- Codex's critique surfaces at least one material issue from the synthesis on most runs.
- Debate converges in 2–3 exchanges.
- Verdict distribution over multiple runs is genuinely bidirectional — not Codex-always-wins, not orchestrator-always-wins.

Malfunctioning when:
- Every Codex objection is rubber-stamped (orchestrator not participating). Fix: push back harder; verify every Codex claim before integrating.
- Every Codex objection is rejected (orchestrator defending reflexively). Fix: verify each objection against code with fresh eyes.
- Debate exceeds 4 exchanges. Fix: surface unresolved point to user; stop bouncing.
- Cohort and baseline-Codex overlap exceeds 50% across multiple runs. Fix: retune prompts to differentiate, OR drop one team if its unique-finding rate is consistently zero.
