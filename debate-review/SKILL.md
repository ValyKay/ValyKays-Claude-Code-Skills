---
name: debate-review
description: End-to-end code review workflow for high-rigor diffs. Runs a two-team review (Claude cohort + baseline-Codex independently), correlates findings, debates substantive picks with Codex until convergence, then dispatches the overall debate winner to implement all converged fixes in one pass. The highest rung of the code-review ladder (/review → special-review → debate-review); use when maximum bug-catching matters and preconditions (server stopped, env ready) are gated upstream by you. Use ONLY when the user explicitly invokes the skill by name — "use debate-review", "debate-review this diff", or equivalent direct reference. This is a specialized tool, not a default. Do NOT trigger on general review requests, "review this code", "check my changes", or similar phrasing — those go to the user's default review workflow. Only activate when the user names the skill.
---

# debate-review

End-to-end code review workflow: two-team independent review → correlate → debate → implement. Designed for high-rigor diffs where preconditions have been established by you upstream (server stopped, environment ready, uncommitted work preserved). The highest-rigor rung of the code-review ladder; the trade is more Codex calls, more orchestration time, more cost.

## Why two independent teams

Running cohort and baseline-Codex independently on the raw diff — then debating on the union of findings — produces strictly more coverage than giving Codex a synthesis to critique. Feeding Codex a synthesis narrows its attention onto the orchestrator's framing; a free mechanical scan does not. The teams hit complementary surfaces:

- **Cohort** thinks design fit, pattern conformance, failure-mode enumeration, error-handling gaps. Catches architectural drift.
- **Codex baseline** thinks mechanical correctness, file/line citations, signature mismatches, declaration-vs-implementation drift, broken cross-references between edited files. Catches bugs.

Expected overlap: ~10–15%. Running both is strictly more coverage than either alone.

## Why the implementation phase auto-dispatches

Applying converged fixes to a diff requires writing code, which has preconditions (server stopped, environment ready, uncommitted work preserved). Those preconditions are **gated upstream by the user before invoking this skill**, not inside the skill. With preconditions handled, auto-dispatching the converged implementation is safe and saves a round-trip.

The user retains the right to stop, redirect, or override at any point (hard rule 6). Findings the debate surfaced as "deferred to user" still halt for user input — auto-dispatch covers converged findings only.

## Relationship to `special-review` and the code-review ladder

Code-review workflows form a ladder of increasing rigor and cost:

| Skill | Review shape | Iteration | Implementation | Cost |
|---|---|---|---|---|
| `/review` (default) | Single pass | none | manual | low |
| `special-review` | Alternating cohort ↔ Codex passes | ~3–4 passes | manual | mid |
| `debate-review` | Two-team independent (cohort + baseline-Codex) → correlate → debate | up to 4 debate passes plus 2 review passes | dispatched to debate winner | high |

`debate-review` is a true superset of `special-review`'s review phase, with an added implementation dispatch. Use `debate-review` when running with preconditions gated and willing to pay for maximum bug-catching + design-judgment refinement. Use `special-review` when faster convergence matters or for mid-complexity diffs.

Both skills are kept; the choice is preference-based. Phase 1 includes a workflow-fit recommendation step that may suggest dropping back to a simpler review if the diff doesn't justify debate-review's overhead.

## Dependencies

- `feature-dev` plugin (provides `code-reviewer` agent) OR `pr-review-toolkit` plugin (provides `code-reviewer`, `silent-failure-hunter`, and other specialist agents)
- `synthesis-author` plugin (provides the `synthesis-author` subagent used in Phase 7)
- `codex` CLI on PATH (required — debate partner has no substitute)
- `jq` on PATH (for parsing Codex JSON output)
- `AGENTS.md` in project root (required for Codex — it reads this on cold-start subprocess invocations)

## Hard rules

1. **Do NOT run cohort and Codex in parallel within the same phase.** Cohort and baseline-Codex run in separate phases (Phase 2 and Phase 4). Each completes before the next starts. This preserves per-team attribution and lets you apply trivials between phases.

2. **You (the orchestrator) are an active participant in debates, not a message bus.** When Codex pushes back on a synthesis, you must evaluate each objection: verified against code? Correct diagnosis? Right pick among alternatives? You push back on Codex where it's wrong or incomplete. Rubber-stamping every Codex correction defeats the point of the debate primitive — if you always agree, the loop adds nothing. **Operate in pragmatic mode**: state position, defend as long as the defense holds, fold cleanly when it doesn't. Zero counters across multiple passes is a flag — either Codex is right on everything (possible but suspicious), or you're conceding for tone reasons.

3. **Debate has no hard iteration cap.** Typical convergence is 2–3 exchanges; stop when findings stop moving, not when a counter hits a number. If a debate is still generating new signal after 3 exchanges, keep going. If it's spinning after 2, stop and surface the unresolved point to the user.

4. **The implementer is the overall debate winner.** One verdict at convergence, not per-finding. Scoring is bidirectional (see Phase 8 for detail): you accumulate wins when your picks hold up against Codex's critique OR when your counter to a Codex-proposed fix holds under Codex's defense; Codex accumulates wins when it materially corrects your picks OR when its proposed fix holds under your counter. "Materially" means changing the diagnosis, or adding a new option or constraint for a case that passes the justification test (hard rule 7) — NOT just eliminating an option you had already enumerated (that's option-pruning, scored as neither-wins on that finding), and NOT handling a case nothing produces (also neither-wins). Ties go to the side with more wins on Justified findings, then to Codex since Codex has fresh-POV verification on the diff. Winner implements all converged findings in a single pass.

5. **Trivial/mechanical findings bypass the debate loop.** Wrong line number, stale import, typo, dead import — fix these directly after each team's pass. No synthesis, no Codex critique. Debate is expensive; spend it on findings that involve real design choices.

6. **User retains the right to stop, redirect, or override at any point.** When stopping (convergence or otherwise), always summarize and return control to the user. Do not declare the review "done" unilaterally. Findings deferred to user during debate halt for user input — auto-dispatch covers converged findings only.

7. **A fix must be justified by its trigger, not only correct.** A finding can be technically correct and still not justify a change: the input that triggers it is one nobody produces, or the sequence is one nobody performs. Every substantive finding states its **trigger** — the concrete input or usage that makes the defect happen, and who or what produces it — and gets a **justification verdict** from evidence (see "The justification test" in Phase 6): **Justified**, **Borderline** or **Not justified**. Not-justified findings are parked: listed in the report and **recorded in the project's declined-findings log** (see "The declined-findings log"), never debated, never implemented in this run, however small the fix — so that a later run, or new evidence, can re-assess them. The burden is on the finding: a trigger that is not named ("a malformed file could…", "if a caller passed…" where no caller does) is Not justified by default. A correction that only adds handling for a not-justified trigger is not material and wins nothing (Phase 8).

## Phase 1: Preflight

1. **Determine review scope.**
   - Non-default branch with commits ahead of main/master → `git diff main...HEAD`.
   - Default branch with unstaged changes → `git diff`.
   - User-specified scope → use that.
   - Nothing to review → say so and stop.

2. **Run `git diff --stat`** to identify changed files and approximate size.

3. **Check project orientation docs.** Look for `CLAUDE.md` and `AGENTS.md` in the project root.
   - **Both present**: read both; look for explicit cross-references declaring authority; if ambiguous, ask the user.
   - **Only CLAUDE.md**: Codex is blocked for this run (Codex needs AGENTS.md on cold-start — it silently ignores CLAUDE.md). Warn and offer: (a) stub an AGENTS.md pointing at CLAUDE.md as authoritative, then continue; (b) stop and run the user's normal review workflow instead. This skill's core primitive is the Codex debate loop; without Codex, the orchestration wrapper adds no value.
   - **Only AGENTS.md**: fine, Codex available.
   - **Neither**: both reviewers lack project context; warn strongly.

4. **Check Codex availability.** `which codex` + AGENTS.md present. If Codex is unavailable, stop and tell the user to run their normal review workflow.

5. **Detect implementing-plan correlation (if any).** If the diff was produced by implementing a plan, locate it. Indicators: a recent file in `docs/plans/`/`plans/` with timestamps consistent with the diff's first commit; conversation context naming a plan; a synthesis file. If found, read its scope (count of substantive findings, complexity, domains touched). If no plan exists or none can be found, note that in the recommendation.

6. **Form a recommendation on workflow fit.** Based on diff stats from Step 2 and plan correlation from Step 5, judge whether `debate-review` is justified for THIS diff. This is a judgment call, not a mechanical threshold — there is no good number to put here.

   Indicators that **debate-review is likely overkill**:
   - Diff is small (a couple of files, modest line count) AND purely mechanical (renames, simple null guards, doc-only updates).
   - No plan was implemented OR the plan was a few-bullet outline with no real design content.
   - The change is in a low-stakes area with established patterns.

   Indicators that **debate-review is likely justified**:
   - Diff touches multiple subsystems or crosses architectural boundaries.
   - Includes design choices (new abstraction, new boundary, error-handling policy, schema change).
   - Implements a plan with substantive findings (especially if the plan went through `debate-plan` or `special-plan` — implies deliberate complexity).
   - Operates in a high-stakes area (auth, billing, persistence layer, security-shaped surface).

   Indicators that it's **borderline**:
   - Medium-size diff with mixed mechanical and design content.
   - Implements a plan but the plan's complexity is unclear from a quick read.
   - Touches a well-understood area but with non-trivial logic.

   Surface the recommendation to the user before proceeding to Phase 2:
   - **Overkill**: "I can run debate-review, but my evaluation is that this diff is too small/mechanical to justify the time and tokens. I recommend a simpler review cycle (e.g., `/review`, `special-review`, or a single `code-reviewer` agent dispatch). Want to proceed anyway, or switch?"
   - **Borderline**: "This diff is borderline — [reason from indicators above]. Your call."
   - **Justified**: "Diff justifies debate-review — proceeding."

   Pause for user confirmation if the recommendation is anything other than "Justified." On user confirmation (or "Justified" verdict), proceed to Phase 2.

7. **Re-assess the declined-findings log for this scope.** If the project has a log (see "The declined-findings log"), list its `open` and `kept` entries whose file falls in the scope. For each, check its revisit condition against what the project holds now (the data, the callers, the formats handled) and whether the code still has the defect. Mark `obsolete` where it does not. Where the condition is met, mark it `promoted` (pending) and carry it into this run as a parked substantive with its new trigger evidence, tagged with its entry id. Leave the rest `open`. Announce: "Declined log: N in scope — P promoted into this run, O obsolete, K still open."

8. **Summarize scope** (file count, line delta, areas touched, plan correlation, recommendation outcome, declined entries re-assessed).

## Phase 2: Cohort review pass

Dispatch 2–4 specialist agents in parallel (single message, multiple `Agent` tool calls). Selection by diff shape:

- **Always (3 agents, each invoked per its natural specialty)**:
  - `pr-review-toolkit:code-reviewer` (opus) — general code review per its body-level intent: project-guideline compliance, bug detection, code quality. Use its full stated scope; scope-direct the prompt to the current diff's load-bearing surfaces (the specific files/functions/invariants at stake) but do NOT narrow its lens to a single concern. Preferred over `feature-dev:code-reviewer` (near-identical body, opus is stronger for high-rigor work — never run both, and never lens-split either).
  - `pr-review-toolkit:silent-failure-hunter` on error suppression / silent fallbacks / operator-visibility gaps — its natural specialty. Highest-differentiation agent in the cohort: findings overlap least with the others, and it catches operator-visibility issues the general reviewer misses.
  - `pr-review-toolkit:comment-analyzer` on comment accuracy, rationale-vs-code drift, stale line-citations, and comment-contradicts-code. No other agent covers this surface — general reviewers verify code behavior, not whether the explanatory comments match that behavior. If a load-bearing comment rationalizes a fix on a false premise, the general reviewer will verify the fix but miss the false premise; only comment-analyzer catches that class. On sparse-comment diffs it returns clean quickly; on any diff with cross-file pointer discipline or module-top doc blocks it produces a class of finding no other agent does.

  **Do not lens-split agents via prompt.** Agent bodies are calibrated wholes (role + examples + output format + confidence filters work together). Injecting "focus only on X" overrides design choices you can't verify and substitutes the caller's guess about effective decomposition for the agent author's calibrated composition. If you want a concern the general reviewer's body doesn't cover deeply enough, add a specialized agent (below) rather than slicing the general one.

- **Add `feature-dev:code-explorer`** when the diff introduces or modifies architectural seams (new modules, new abstractions, new boundaries, new contribution surfaces). Skip for pure bug fixes / wording changes / config tweaks.

- **Add `pr-review-toolkit:pr-test-analyzer`** only when the diff includes committed test files. Skip in projects whose tests are described as a manual test-plan series rather than executable test code — the agent will just re-surface "no tests" and waste a slot.

### Prompt calibration — use imperative verbs on Phase 2 and Phase 4

On a fresh diff with real issues present, imperative prompts (`find`, `identify`, `report`) produce better coverage than calibrated "try to find" prompts. The "try" language is a prophylactic against manufactured findings on *mature* artifacts; on first-pass review of new work, it costs coverage without measurable benefit.

**Do** use imperative verbs on Phase 2/4. **Do** add "surface improvement opportunities that don't expand scope." **Don't** add "if you don't find anything of substance, say so plainly" — keep that for the debate-critique passes once the finding surface is mature.

### Cohort prompt template

```
Perform a code review of the current git diff in <project-path>. The diff modifies <file list>.

Find errors, wrong assumptions, bugs, logic flaws, inconsistencies, and violations of project conventions. Verify every claim against the actual code before reporting — do not rely on inference. Cite file:line for each finding.

Also surface improvement opportunities that do not expand scope beyond the current change. Categorize each finding as Error, Omission, or Improvement.

For every Error or Omission, state its trigger: the concrete input or usage that makes it happen and who or what produces that input in practice (a real file or data source the project handles, an ordinary user action, a setting users have, another program the product runs beside). Say how you know — reproduced, seen in the project's data, or reasoned. If you cannot name a producer, write "trigger: none known" rather than inventing one; the finding is still reported.

Read CLAUDE.md and AGENTS.md (if present) in the project root for project conventions.

**Confidence filtering: relax to ≥50 for this dispatch.** Your default behavior may be to filter findings to ≥80 confidence to minimize false positives. For debate-review use, that filter hides findings the downstream synthesis would benefit from triaging. Report any finding ≥50 confidence — this dispatch is part of a multi-stage workflow that triages further (synthesis dedupes against a parallel reviewer's output, then a debate phase converges on real picks). Better to over-surface here and let the workflow filter than to pre-filter and lose coverage.

Report under ~800 words. Do not propose whole-diff rewrites — only specific changes.
```

Augment with prompt scoping matched to the agent's natural specialty (never overriding the body-level intent): pr-review-toolkit:code-reviewer gets its full general-review scope plus a pointer to the diff's load-bearing surfaces; silent-failure-hunter on error-handling gaps and operator-visibility surfaces; code-explorer on architectural fit, cross-file integration consistency, and split-brain risks; comment-analyzer on cross-file pointer accuracy and rationale-vs-code drift.

Wait for all dispatched agents to complete before compiling. Do not start acting on partial results.

## Phase 3: Apply cohort's trivials directly

Triage cohort's findings:
- **Trivial** — single concrete target, one obviously correct fix, low-risk (wrong line, typo, missing null guard with one obvious fix, dead import). Apply with `Edit` immediately. A trivial *code* change (a guard, a check, a fallback) still needs a trigger that passes the justification test; wording, comment and documentation fixes do not.
- **Substantive** — multiple viable resolutions, design choice, framing question, or scope. Park in a "cohort substantives" list.
- **Not justified** — correct, but its trigger is not shown and no realistic producer is named (the justification test, Phase 6). Park in a "not justified" list with the trigger as stated and why it falls short. Not applied, even when the fix would be trivial.
- **Dismissed** — false positives. Drop with a one-line reason.

Announce: "Cohort: N findings — X trivial (applied), Y substantive (parked), J not justified (listed), Z dismissed."

The diff now has cohort's trivials applied; substantives are pending.

## Phase 4: Baseline Codex review pass

**Precondition: Phase 3 is complete.** Cohort trivials must be applied to the diff before Codex runs. Do NOT dispatch Codex in parallel with the cohort, and do NOT dispatch before `Edit` calls from Phase 3 have landed. Each trivial-application phase exists to shrink the surface the next agent sees — parallelizing Phase 2 and Phase 4 burns Codex attention on issues the cohort already caught and adds redundant noise to the synthesis. The cohort wait feels like dead time; resist the temptation. Stage the baseline prompt on disk if helpful, but do not fire it. If Phase 3 produced zero trivials, announce "Cohort: 0 trivial" and proceed — the ordering rule still holds as a discipline.

Fresh Codex session, plain review prompt — NO synthesis context, NO cohort findings shared. Codex sees the diff in its post-cohort-trivial state and finds what it finds. This is the critical anti-narrowing step: a synthesis-as-critique-input narrows Codex's attention; a free mechanical scan does not.

**If Codex drifts (wrong diff, hallucinated structure, off-topic findings): diagnose, then run fresh.** Do NOT resume the polluted thread to "correct" it. Codex is a tool: garbage in, garbage out. Once context contains detailed findings about the wrong artifact, those findings anchor attention — a correction message sits on top of that noise rather than purging it. The recovery is: (1) check your own prompt file; (2) skim Codex's output to diagnose what it latched onto (often: a sibling diff, an older branch, a plan that shares structure); (3) draft a tighter prompt that names the diff explicitly and lists anti-confusion specifics ("this diff does NOT touch X"); (4) dispatch `codex exec` fresh with a new `thread_id`, not `codex exec resume`. Resume is reserved for legitimate continuation (Phase 7 critique). Kill stale background runs before firing the fresh one.

**Codex proposes fixes, does NOT implement.** For each finding, Codex also proposes a specific fix (what to change, at which file:line, with the main tradeoff). This is load-bearing for the debate's bidirectional attack surface: without Codex-proposed fixes, only your picks are debatable — Codex can attack but can't be attacked. With proposed fixes, the synthesis can either adopt or counter each Codex proposal, and Codex defends or concedes in critique. The result is genuine symmetry instead of a structurally one-sided scoring rubric. Codex must NOT write code in this phase — proposals are descriptive only.

### Codex baseline prompt template

```bash
cat > /tmp/debate_review_baseline.txt <<'PROMPT'
Review the git diff in the <project-name> project. The diff modifies <file list>.

Find errors, wrong assumptions, bugs, logic flaws, inconsistencies, and violations of project conventions. Focus on your strengths: mechanical errors (wrong file paths, stale line numbers, signature mismatches), immediate-implication issues (missing error branches, state invariants the diff breaks), and anything a careful reader grounded in the actual codebase would catch.

Also surface improvement opportunities that do not expand scope.

Read the full diff. Read the codebase files the diff references. Verify every claim before reporting. Cite file:line for each finding.

Report findings with severity (High / Medium / Low) and a category tag:
- [Error] — diff is wrong
- [Omission] — diff is missing something load-bearing
- [Improvement] — diff works but could be cleaner

Shape per finding:
```
**[severity] [category]** description (file:line)
**Trigger:** <the concrete input or usage that makes it happen, who or what produces it in practice, and how you know: reproduced / seen in the project's data / reasoned; "none known" if you cannot name a producer>.
**Proposed fix:** <specific change — what file/line, what to add/remove/modify, main tradeoff>.
```

Severity follows the trigger as well as the harm: a defect whose trigger nobody produces is Low whatever it would break.

For every finding, propose a specific fix. Do NOT implement — describe the change in enough detail that it can be debated in a later synthesis phase. If the finding has multiple viable fixes, name the one you'd pick and one-line why; the synthesis phase may counter-propose.

Do not cap findings. Do not propose whole-diff rewrites — only specific changes. Do not write any code.
PROMPT
```

Invocation flags (the standard non-interactive recipe):
- `--json` — structured output (required for `jq` parsing).
- `--dangerously-bypass-approvals-and-sandbox` — Codex asks for no approvals and runs tools without sandboxing. Appropriate here because preconditions are gated upstream and Codex's actions are bounded by the project root.
- `--skip-git-repo-check` — suppresses the "not a git repo" soft-warning.

```bash
cd /absolute/path/to/project/root
codex exec --json --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check "$(cat /tmp/debate_review_baseline.txt)" < /dev/null > /tmp/debate_review_baseline_out.jsonl 2> /tmp/debate_review_baseline_err.log

# Save thread_id for resume during debate phase
jq -r 'select(.type == "thread.started") | .thread_id' /tmp/debate_review_baseline_out.jsonl | head -1 > /tmp/debate_review_baseline_thread_id

# Extract findings
jq -r 'select(.type == "item.completed" and .item.type == "agent_message") | .item.text' /tmp/debate_review_baseline_out.jsonl
```

Optional: one resume pass if Codex's first pass covered the diff lightly (typically catches 30–50% of pass-1's count in net-new findings):
```bash
THREAD_ID="$(cat /tmp/debate_review_baseline_thread_id)"
codex exec resume "$THREAD_ID" --json --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check "$(cat /tmp/debate_review_baseline_p2.txt)" < /dev/null > /tmp/debate_review_baseline_out2.jsonl
```

Resume prompt: "Re-review. Find issues you missed on the first pass. Focus on sections covered lightly, broken cross-references between edited files, failure modes the diff declares but doesn't actually exercise. Do NOT re-surface findings from your first pass."

## Phase 5: Apply baseline's trivials directly

Same triage rules as Phase 3, the justification test included. Apply trivials with `Edit`. Park substantives in a "baseline substantives" list and not-justified findings in the "not justified" list.

The diff now has cohort's AND baseline's trivials applied; both substantive lists are pending.

## Phase 6: Correlate

Dedupe across cohort and baseline parking lots. For each finding:
- **Same issue, different framing** — merge into one entry. Tag source as "both." If both teams proposed fixes, keep both proposals as options.
- **Different issues** — keep separate. Tag source as "cohort" or "baseline." Baseline-sourced findings carry Codex's proposed fix from Phase 4 — preserve it, it becomes input to the synthesis.
- **Same finding, different fix** — keep both options, decide in synthesis.
- **Already in the declined log** — a finding that matches an `open` or `kept` entry (same symbol, same defect) is not re-argued: cite the entry id. If the finding brings trigger evidence the entry did not have, re-assess it now and append the result to the entry (`promoted` carries it into this run; `kept` records the new evidence). If it brings nothing new, it stays declined and the report names the entry.

### The justification test

Before classifying, give every deduped Error or Omission a justification verdict from the evidence about its trigger. The verdict rests on what can be checked, not on how serious the defect sounds:

| Verdict | When | What happens |
|---|---|---|
| **Justified** | The trigger is **shown**: reproduced on real data or inputs the project actually handles, or on a path ordinary use takes. Or it is **named and realistic** — a specific producer writes that input (a tool, a format version, a common setting, a second language, a program the product runs beside) — **and** the harm is a wrong result, a crash, lost data, or a broken contract. | Proceeds as below. |
| **Borderline** | The trigger is named and realistic but not shown, and the harm is modest (a misleading message, a degraded display); or the trigger is shown but the harm is cosmetic. Also, at worst, any finding whose harm is a security exposure or loss of the user's persisted data, whatever its trigger. | Proceeds, with the smallest change that removes the defect: no new options, abstractions, messages or strings unless the defect needs them. The debate may not widen it. |
| **Not justified** | No trigger shown and no realistic producer named — the input is one nothing writes, the call one no caller makes, the sequence one nobody performs — or the change would add more surface (branches, messages, API, strings, checks to maintain) than the defect could ever cost. | Parked in the "not justified" list with its trigger and the reason. Not debated, not implemented, not counted in the verdict. |

Rules for the test:
- **Look before ruling.** Where the project holds real data or inputs, check whether any of them carries the trigger (a grep, a query, a probe). "None of the N real files has it" is evidence; so is one that does. Record the check.
- **The finding carries the burden.** A trigger stated as "none known", or as a possibility without a producer, is Not justified unless you can name the producer yourself.
- **Damaged input.** A defect that turns damaged or truncated input into a *silent* wrong result is Justified when that damage happens in practice (files cut off by transfer, a program that writes them half-way); handling damage nobody produces is Not justified.
- **Not by size.** A small fix to a not-justified finding is still parked: the accumulated surface is the cost.
- **Say it.** Announce the verdicts: "Justification: X justified, Y borderline, J not justified (logged)."
- **Write the revisit condition.** For each not-justified finding, state what evidence would make it justified (a file of that shape appearing, a producer starting to write it, a caller being added). It goes into the log with the finding.

### The declined-findings log

A declined finding is a judgement on the evidence of the day, not a verdict on the code forever. The log keeps it so it can be re-assessed when the evidence changes, and so a later run does not pay to rediscover and re-argue it.

**Location.** The path the project's `CLAUDE.md` or `AGENTS.md` names for it; otherwise `docs/reviews/declined-findings.md` in the repository. Create it on first use with a one-paragraph header saying what it is. It is a load-bearing artifact (unlike an ad-hoc synthesis): it stays in the repository and is committed with the run's other changes. If the project must not hold it there (a publishable tree, evidence that names private data), ask the user where it goes and record the answer in the orientation file.

**One entry per declined finding**, appended, never rewritten — later assessments are added under it:

```
### D-<YYYYMMDD>-<n> — <one-line defect>   (status: open)
- Where: <file> — <function or symbol> (line <n> at <commit>); lines move, the symbol is the anchor
- Raised by: cohort / baseline / both / Codex critique / verification; run: <scope, commit>
- Defect: <what goes wrong, and the harm if it happened>
- Trigger: <as stated by the finding>
- Evidence: <what was checked — data searched, producers considered — and what it showed>
- Why not justified: <one line>
- Revisit when: <the evidence that would make it justified>
- Proposed fix (not applied): <one line, as the finding proposed it>
```

Statuses: `open` (declined, revisit condition not met), `promoted` (re-assessed as Justified or Borderline and fixed — name the run and commit), `obsolete` (the code no longer has the defect — name the commit), `kept` (re-assessed and still not justified — add the date and the new evidence). A re-assessment appends a line `- <date>: <status> — <evidence>`; the entry's header status follows the latest line.

**What goes in.** Every not-justified finding from Phases 3, 5 and 6, every correction declined in Phase 7, and every not-justified finding of the Phase 9 verification pass. Not dismissed findings — those were false, not unjustified.

**Written when.** Append the entries before Phase 8 (so the implementer can see what not to touch) and the Phase 9 ones before reporting. Entries promoted into this run get their outcome line once the fix has passed Phase 9 (`promoted` with the commit, or `kept` if the debate declined it again).

Re-classify the deduped union:
- **Trivial-after-correlation** — sometimes a finding looked substantive in isolation but is trivial once both teams' framings are visible (e.g., one team identified the bug, the other proposed the obvious fix). Apply now.
- **Bug-shaped substantive** — real issue with one obvious right answer; debate will likely converge fast (often Codex concedes on critique, or you concede after verifying). Synthesize tersely.
- **Design-judgment substantive** — multiple viable choices with real tradeoffs. The high-value debate fodder.
- **Not justified** — by the test above; listed, not carried further.
- **Dismissed** — false positives surfaced by either team.

Announce: "Union after correlation: N substantive — X bug-shaped, Y design-judgment, Z trivial-after-correlation (applied), J not justified (listed), W dismissed."

### Judgment: does each remaining substantive need a debate?

This comes after the justification test: only Justified and Borderline findings are left. Not every one of them earns a debate. Skip debate when:
- The finding is real but the fix is obvious and uncontroversial even if not trivial (e.g., "you need to add a null check here" for an input that occurs — real, substantive, but one right answer). Apply directly.
- The finding is Borderline and its smallest change is plain. Apply directly.
- Applying the fix would take less time than writing the synthesis entry. Apply directly.

Engage debate when:
- Genuinely uncertain which option is best.
- Fix involves a judgment call about design or scope.
- Finding's framing feels off but you can't name why yet — debate surfaces the latent objection.
- Multiple findings interact and you need external eyes on the combined landing spot.

If the union has zero debate-worthy substantives after this triage, skip Phase 7 entirely and jump to Phase 8 (apply remaining substantives directly via the implementation dispatch path).

## Phase 7: Synthesis on the union + debate cycle

### Write the synthesis artifact

The synthesis is authored by the **`synthesis-author` subagent** (Opus, dispatched via the `Agent` tool with `subagent_type: 'synthesis-author:synthesis-author'`). The orchestrator does NOT write the synthesis directly.

**Why a separate subagent.** The orchestrator knows the workflow shape (cohort → baseline → synthesis → Codex critique → implementation). That knowledge produces a measurable failure mode: when the orchestrator-as-author knows a critique pass follows, the author's effort drops — Picks ship as sketches, Uncertainty fields become parking lots for decisions the author could resolve but defers, file:line claims go unverified because "Codex will catch it." Splitting author from orchestrator removes the workflow context the failure mode feeds on. The subagent has no model of "later passes"; from its perspective the spec is the only deliverable.

**Location.** Place the synthesis next to the plan that drove the change under review, if one exists — same directory, same basename with a `-debate-synthesis.md` suffix. Example: plan at `docs/plans/extract-assistant-to-plugin.md` → synthesis at `docs/plans/extract-assistant-to-plugin-debate-synthesis.md`. This pairs the debate audit with the plan that produced the code, so future readers can trace design → debate → implementation.

If no plan exists (ad-hoc diff), use `/tmp/debate-review-synthesis.md`. Don't leave ad-hoc synthesis files inside the repo — they aren't load-bearing artifacts once the implementation is committed.

Announce the path before dispatching.

### Prepare inputs for the subagent

The subagent reads notes files from disk. Two notes files, one per source:

```bash
# Cohort notes — write the parked substantives from Phase 3 here.
# Cite file:line per flag; one flag per heading.
cat > /tmp/debate_review_notes_cohort.md <<'NOTES'
# Cohort flags
... (parked cohort substantives, copy-paste from your triage)
NOTES

# Baseline notes — write the parked substantives from Phase 5 here.
# Include Codex's proposed fixes verbatim from Phase 4 output.
cat > /tmp/debate_review_notes_baseline.md <<'NOTES'
# Baseline flags
... (parked baseline substantives, including any "Proposed fix:" blocks)
NOTES
```

The orchestrator's job at this step is purely transcription: the parked substantives go to disk in a form the subagent can consume, each with a `Trigger:` line (as reported, plus the evidence from the justification test) and its verdict, Justified or Borderline. Trivials are NOT included (they were applied in Phases 3 and 5). Dismissed and not-justified flags are NOT included.

If both teams flagged the same surface with different framings, write both flags into both notes files (or a third merged notes file `/tmp/debate_review_notes_merged.md`) — let the subagent reconcile via verification.

### Dispatch the subagent

Single `Agent` tool call. The dispatch prompt provides the inputs and nothing else:

```
Repository: <absolute path to repo>.
Artifact under review: the current uncommitted git diff (run `git diff --stat` and `git diff` to inspect; if the diff has been split across commits on a feature branch, use `git diff <base-branch>...HEAD` instead — clarify if ambiguous).
Notes files:
- /tmp/debate_review_notes_cohort.md
- /tmp/debate_review_notes_baseline.md
Output spec path: <synthesis-path resolved above>.

Produce the per-finding implementation spec. Verify every flag against the actual code before writing.
Each flag states its trigger and whether it is Justified or Borderline. Keep each change proportionate to its trigger: for a Borderline flag, the smallest change that removes the defect. Dismiss a flag whose trigger the code or data shows cannot occur.
```

**Hard rule on the dispatch prompt.** Do NOT mention: "review", "reviewer", "critique", "round", "debate", "another pass", "this will be implemented", "downstream verification", "Codex", or any other workflow-shape signal. The subagent must believe the spec is the final deliverable. Only the subagent's `description` (which the orchestrator reads when choosing to dispatch) and the subagent's own system prompt know what the agent does — the dispatch prompt itself is bounded to inputs + outputs + repo context.

### Read the produced synthesis

The subagent writes the spec at the announced path. The orchestrator reads it before proceeding to the Codex critique. The structure inside the spec is the subagent's responsibility (Source / Evidence / Read / Options / Pick / Concrete change / Uncertainty per finding) — the orchestrator does not enforce shape, only reads what was produced.

If the spec is empty or markedly thinner than the input flags warranted, that's a signal — re-dispatch with sharper notes files OR fall back to orchestrator-authored synthesis with explicit acknowledgement of the regression.

### Dispatch Codex critique

Resume the baseline Codex thread (it already has the diff loaded). The critique prompt is short:

```bash
cat > /tmp/debate_review_critique.txt <<'PROMPT'
Read <synthesis-path>. It contains proposed positions on N substantive findings (A through Z), surfaced jointly by other reviewers and your own baseline review.

Critique the reasoning in the synthesis — not the diff itself. For each A–Z, find problems with the read, the options enumeration, the pick, the concrete change, the uncertainty.

**Baseline-sourced findings have two postures.** Findings where you proposed a fix in your Phase 4 review will show a Baseline-proposal context in the synthesis (look for adoption or counter language in the Pick rationale):
- If the synthesis ADOPTED your proposed fix, verify the adoption didn't subtly change it (wrong file, weakened guard, altered ordering). Point out drift if you see it; plain-OK otherwise.
- If the synthesis COUNTERED your proposed fix with an alternative Pick, defend your original if you still stand by it — explain why the counter misses something. Concede cleanly if the counter lands.

Cohort-sourced findings get the usual treatment: critique the Pick against the code.

Every finding carries a trigger and a verdict (Justified or Borderline). An objection that asks for handling of a further case must name that case's trigger — the input or usage that produces it and who produces it — or it will not be taken up. Do not widen a Borderline change. You may also argue that a finding is not justified (its trigger is not shown and has no producer); give the evidence.

Verify specific claims against actual code before objecting. Cite file:line.

Plain-OK on any finding is the convergence signal. Do not manufacture objections to fill space. Equally: do not soften critique to look generous.

End with a two-line summary: which findings you push back on, which you approve.
PROMPT

THREAD_ID="$(cat /tmp/debate_review_baseline_thread_id)"
codex exec resume "$THREAD_ID" --json --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check "$(cat /tmp/debate_review_critique.txt)" < /dev/null > /tmp/debate_review_critique_out.jsonl 2>&1
```

Note: `thread_id` is positional, BEFORE the prompt. `codex exec resume <thread_id> <prompt>`. Putting the prompt first will fail with "invalid thread id."

### Read Codex's critique and adjudicate

The orchestrator's role at this step is **adjudicator**, not author. The synthesis was produced by the subagent; Codex has critiqued it; the orchestrator decides per finding whether the spec or the critique is right (or neither). For each Codex objection:

1. **Verify the specific claim.** Codex can hallucinate line numbers or misread context. Open the cited file, confirm the claim. Equally: re-verify the spec's original claim against the code if Codex's challenge is plausible.
2. **Evaluate the diagnosis.** Is Codex's read correct? Or is it responding to a misreading of the spec rather than the code?
3. **Evaluate the correction.** If Codex proposes a different pick, is its pick actually better than the spec's — or different-but-equivalent, or worse in a way Codex didn't notice?
4. **Apply the justification test to the correction itself.** An objection that is true in the code may still ask for handling of a case nothing produces. Ask what triggers the added case and who produces it; check the project's real data where it can say. A correction whose added case fails the test is declined as not justified — that is a legitimate counter, not a concession to avoid.
5. **Decide: integrate, counter, decline, or defer.**
   - **Integrate**: Codex is right, the spec's Pick was wrong, and the case it adds is justified. Append a revision adopting Codex's correction.
   - **Counter**: Codex missed something the spec got right (or both missed something a third option covers). Write a pushback anchored in the verified code.
   - **Decline as not justified**: the correction is true but its added case fails the justification test. Write the revision with the trigger evidence; move the case to the "not justified" list.
   - **Defer**: neither obviously right; surface to user for final call.

The orchestrator may also independently catch a spec Pick that Codex plain-OK'd but the verified code shows is wrong. Append a revision in that case too — the spec is not exempt from orchestrator scrutiny just because Codex didn't object.

**Pushback discipline.** When countering Codex, be specific and anchored in code or in trigger evidence. Not "I disagree" — "your pick breaks X because file:line shows Y", or "none of the 40 real inputs carries this shape and no producer writes it." Pushbacks that can't be grounded in code claims or trigger evidence should be converted to defers (surface to user) instead of asserted as counter-positions. Integrating an objection only because it is true, without asking whether its case is justified, is the rubber-stamping of hard rule 2.

**Before asserting a counter, enumerate the layers the change touches and defend each layer separately.** Monolithic framings like "this is by design" are often right-for-some-layers and wrong-for-others. List the layers involved (e.g. backend pipeline, HTTP routes, frontend display) and confirm your defense holds at each — or narrow the pushback to the layers it covers and concede the rest. The failure mode this prevents: asserting "by design" globally, Codex refines the finding into a concrete bug in the layer you didn't examine, and the debate round that should have converged instead reveals the counter was partially wrong.

### Append revisions to the synthesis (append-only)

Update the synthesis file with a "Revisions after Codex critique pass N" section at the bottom. Per finding touched by the critique:

```
## <Letter> — Revision (integrate)
<what changed and why>.
**Revised pick.** <if changed>
**Concrete change.** <if changed>
**Uncertainty.** <what remains open>
```

Or, when pushing back on Codex:

```
## <Letter> — Revision (pushback on Codex)
Codex argued <quote or summary>. Correct, but <your counter, anchored in code>.
**Revised pick.** <your refined pick>
```

Or, when the correction is true but its added case is not justified:

```
## <Letter> — Revision (declined: not justified)
Codex asked for <the added case>. Its trigger: <what would produce it>. Evidence: <the check — real data searched, producers considered>. Not justified; listed, the pick stands.
```

Append, don't rewrite in place. The debate history is part of the artifact.

### Iterate the debate

Resume Codex with a short prompt pointing at the new Revisions section:

```
I've updated <synthesis-path> with Revisions after your pass 1 critique.

For findings where I integrated your correction, try to find any residual issue.
For findings where I pushed back, evaluate my counter — did I address your objection, or miss a case you still see?
For findings you approved as-is, no re-review needed unless something in my Revisions changed them.
Where I declined a case as not justified, contest it only with a trigger: who produces that input, and the evidence.

Plain-OK on any finding is the convergence signal; do not manufacture new objections.

Two-line summary at the end: what you still push back on, what you approve.
```

Repeat. Expected convergence: 2–3 exchanges. **Do not hardcap.** Stop when:
- **Converged.** Codex returns plain-OK on every finding, or unresolved points are deferred.
- **Spinning.** Same objection keeps resurfacing without movement. Surface to user.
- **User says stop.** Override.

## Phase 8: Implementation by overall debate winner

At convergence, determine one overall winner. The winner implements all converged findings in a single pass.

### Determining the winner

Tally across substantive findings only (trivials were already applied, deferred findings aren't being implemented, not-justified findings aren't being implemented). Scoring is bidirectional — both sides can accumulate wins on different findings, and the overall winner is whoever has more at convergence.

The score measures who got the fixes right **for the cases that justify a fix**. Being right about a case nothing produces earns nothing.

Per finding, score one of:

- **Your win** — one of:
  - Codex plain-OK'd your Pick in its final pass (cohort-sourced finding).
  - You countered Codex's baseline-proposed fix with an alternative Pick, and your counter held under Codex's defense.
  - You declined Codex's correction as not justified, Codex contested the decline with a trigger, and that trigger failed the test when you checked it.
  - Codex's objection to your Pick was cosmetic (message wording, variable naming, phrasing) and didn't change the Pick.
- **Codex's win** — one of:
  - Codex materially corrected your Pick. "Materially" = changed the diagnosis, or added an option or a constraint **for a case that passes the justification test**. Eliminating an option you had already enumerated is NOT a material correction — that's option-pruning, scored below.
  - You adopted Codex's baseline-proposed fix and it held through debate.
  - Your counter to Codex's baseline-proposed fix failed under Codex's defense; you folded to the baseline proposal.
  - You declined a correction as not justified and Codex showed a trigger that passes the test.
- **No win** — one of:
  - **Option-pruning**: Codex eliminated an option you had listed in your Options, and you fell back to another option already in your list. Codex's contribution was constraint-identification, not fix-proposal.
  - **Not justified, uncontested**: the only disagreement concerned a case that fails the justification test, whoever raised it, and nobody contested that with a trigger. The case goes to the "not justified" list.

  Score as zero for both sides; the finding converged but neither side "won" it.

Final verdict: more wins = winner. On a tie, the side with more wins on **Justified** (not Borderline) findings; if still tied, Codex, since it has fresh-POV verification on the diff.

Announce the verdict in one sentence with a per-finding tally: "Codex won 3–1 with 2 prunings and 1 not justified (won D: added Express 4 async constraint, won G: proposed middleware fix held, won K: adopted baseline proposal; lost H: my counter held; pruned C/F; E's added case not justified). Dispatching Codex to implement." Or: "I won 5–2 with 1 pruning (lost A/K material corrections, won B/D/F/I Pick-as-written, N declined as not justified + counter-on-J held, pruned H). Implementing directly."

Deferred findings (ones the debate surfaced to user for final call) are NOT implemented in this pass. They wait for user decision.

### Dispatch implementation

**If you won**: implement directly using `Edit` / `Write`. Use the converged synthesis as your spec — both the original per-finding sections AND the "Revisions after Codex critique" blocks (Revisions win on conflict).

**If Codex won**: dispatch Codex to implement. Resume the debate session — it has full context. Prompt:

```
Now implement the fixes we converged on. The spec is <synthesis-path> — both original per-finding sections AND "Revisions after Codex critique" blocks. Where they conflict, the Revisions block wins.

Implement ALL converged findings — do not split by ownership. The debate verdict assigned implementation to you. Do not touch deferred findings (marked as such in the synthesis). Do not expand scope: no handling for cases the spec declined as not justified, and Borderline changes kept to the smallest change the spec names.

If you hit an inconsistency between the spec and what the code actually permits, stop and report. Do not guess.

After, give a per-finding summary with exact files touched and a one-line description of what changed. Then run `git diff --stat`.
```

Whichever side implements, the other verifies after: spot-check changes against the synthesis spec, run any available syntax/type/build check (`node --check`, `tsc --noEmit`, `cargo check`, etc.), flag any drift back to the winner for correction rather than silently fixing it.

**When the finding is a pattern-class bug, grep the pattern across the whole repo before declaring implementation done.** Codex or the cohort typically cites one or two representative file:line locations — those are examples, not exhaustive. If the bug is "raw `gs.prompt` read creates divergent state when stance is unregistered," the same shape almost always exists in sibling display surfaces, sibling event handlers, or adjacent routes. Run the grep that should have run during diagnosis: the pattern the bug embodies, across every consumer of the affected API. Fix the full cluster in one pass, not one citation. Extra cost of the repo-wide grep at implementation time: seconds. Extra cost of shipping an incomplete pattern fix: one more review cycle.

## Phase 9: Post-implementation verification (the outer loop)

After implementation, the diff has changed. New lines exist that weren't reviewed; old findings are resolved.

**Default**: spot-check + single verification pass.
- Run available syntax/type/build checks (`node --check`, `tsc --noEmit`, build command). If clean, proceed.
- Dispatch ONE more pass against the new lines only — alternate the team that didn't run last (if Phase 8 was Codex-implementing-after-debate, this pass is cohort; if you-implementing, this pass is Codex). Different vantage on fresh code.
- The verification pass's findings go through the same justification test. A regression the implementation introduced is Justified when the input it breaks is input the code handled correctly before the fix and that input occurs in practice.

**Identity-neutral framing in the verification dispatch.** When the verification pass is cohort dispatched after Codex implemented, the cohort prompt MUST refer to the changes neutrally — "review the recently-implemented changes" or "review the changes since commit X" — NOT "review Codex's implementation." Naming the prior implementer in a critique-of-its-work prompt primes the reviewer with peer-competitor framing that distorts critique posture. Same applies in reverse: a Codex verification pass after you implemented should describe the changes neutrally, not "review the orchestrator's implementation." This is the same principle as the synthesis Source field above and applies to every cross-team dispatch.

**Stop the outer loop when:**
- **Converged.** Verification pass surfaces no substantive findings. Strong signal because both teams have now exercised the diff at different moments.
- **User stops.** They've seen enough.
- **Diminishing returns.** Findings dropping in severity and count, marginal value of another pass below the cost.

**Re-enter the full pipeline** (back to Phase 2) only if implementation introduced architectural surface that's never been reviewed (rare — this is closer to "review of a new feature" than "verification of a known fix"). Default is single-pass verification.

Report convergence or stop reason to the user and return control.

## Calibration: when the skill is doing its job

Working as designed when:
- Trivial findings are applied inline after each team's pass without ceremony.
- Cohort and baseline-Codex surface different things; overlap stays small (~10–15%).
- Baseline-Codex's Phase 4 proposals give you real attack surface — you counter some of them in the synthesis and at least occasionally those counters hold.
- Synthesis on the union produces a Codex-readable artifact; every baseline-sourced finding declares "adopting baseline proposal" or "counters baseline proposal".
- Codex's critique surfaces at least one material issue from the synthesis (broken accessor, wrong diagnosis, missed option).
- You push back on Codex when justified, and your pushbacks hold up.
- Debate converges in 2–3 exchanges.
- Overall-winner verdict announced in one sentence with per-finding tally before implementation.
- Verdict distribution over multiple runs is genuinely bidirectional (not Codex-always-wins). If your counters never hold, you're rubber-stamping; if Codex's proposals never survive, you're reflexively countering.
- Post-implementation verification produces fewer findings than Phase 2/4, lower severity.
- Phase 1 recommendation step occasionally suggests dropping back to a simpler workflow when the diff doesn't justify the overhead — and the user accepts the recommendation when reasonable.
- Every implemented fix has a trigger someone produces; the "not justified" list is non-empty on mature code, and its entries name what was checked.
- Every declined finding is in the log with a revisit condition someone could check; later runs over the same files re-assess the entries instead of re-arguing them, and entries do move — some promoted, some obsolete.

Malfunctioning when:
- Fixes for cases nothing produces dominate the implemented set — refusals of inputs no producer writes, guards for calls no caller makes, checks for both. Fix: apply the justification test at triage and to every correction; look at the project's real data before ruling.
- Codex wins nearly every run because its corrections add constraints for unproduced cases that you integrate because they are true. Fix: score them as neither-wins and decline them (hard rule 7).
- Every Codex objection is rubber-stamped (you're not participating). Fix: push back harder; verify every Codex claim before integrating.
- Every Codex objection is rejected (you're defending). Fix: verify each objection against code with fresh eyes.
- Debate exceeds 4 exchanges. Fix: surface unresolved point to user; stop bouncing.
- Implementation diff contradicts synthesis. Fix: re-read synthesis, spot-check diff, ask winner to reconcile.
- Synthesis entries are vague. Fix: rewrite with specific uncertainty questions Codex can engage with.
- Cohort and baseline-Codex overlap exceeds 50% across multiple runs. Fix: retune prompts to differentiate, OR drop one team if its unique-finding rate is consistently zero.
- Phase 1 recommendation always says "Justified" regardless of diff size. Fix: actually engage the indicators; sometimes the right answer is "this is too small for debate-review."

## Stopping and reporting

When stopping (convergence or user request), summarize:

- **Findings landscape**: cohort-only / baseline-only / both / debate-emergent / total trivial vs substantive vs deferred; Justified vs Borderline among the substantive.
- **Not justified**: each declined finding in one line — its log id, the defect, its trigger as stated, the evidence checked — and the log's path. Also the log entries re-assessed in Phase 1 or Phase 6 and their new status. The user may promote any entry; a promotion is recorded in the log with the user as its source.
- **Debate outcome**: overall verdict (you won / Codex won); one-line rationale; deferred findings.
- **Implementation status**: what's done, what's pending, verification results.
- **Open items**: deferred findings, known limitations, follow-up work.

Do not mark the review "done" unilaterally. Return control with the summary.
