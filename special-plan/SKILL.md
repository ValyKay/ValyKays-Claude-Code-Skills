---
name: special-plan
description: Draft an implementation plan and drive a multi-pass review loop (Claude specialist cohort alternating with Codex) until the plan is hardened enough to hand to a Codex or another AI implementer. Use this ONLY when the user explicitly invokes the skill by name — "use special-plan", "special-plan this", "run the special-plan workflow", or equivalent direct reference. This is a specialized tool, not a default. Do NOT trigger on general planning requests, "make a plan for X", "let's plan this out", or similar phrasing — those go to the default planning workflow. Only activate when the user names the skill.
---

# special-plan

A specialized plan-drafting and multi-pass review workflow. This skill does not replace any default behavior. Invoke only on explicit user instruction after the user has already discussed the task with you in normal conversation.

When active, you take two sequential roles: first **plan author** (Phase 2), then **orchestrator** of a review loop (Phase 3). The user retains the right to stop, redirect, or override at any point.

## Dependencies

- `feature-dev` plugin (provides `code-architect` and `code-explorer` agents)
- `pr-review-toolkit` plugin (provides `silent-failure-hunter` agent)
- `codex` CLI on PATH (for Codex review passes)
- `jq` on PATH (for parsing Codex JSON output)

## Hard rules

1. **Do NOT enter plan mode.** Operate entirely in normal mode. Write the plan file with `Write`, edit in place with `Edit`. Plan mode is a different instrument with its own on-disk home (`~/.claude/plans/`); this skill writes to the target repository, not there.
2. **Do NOT run Codex in parallel with Claude subagents in the same review pass.** The two lenses differ in shape (see "Empirical observation on reviewer lenses" below). Parallel runs would conflate findings and destroy per-vendor attribution. Each pass is either the Claude cohort OR Codex, never both.
3. **Do NOT invoke Codex for coding inside this skill.** Codex is strictly a reviewer here. Implementation dispatch is the user's decision and lives outside this skill's scope.
4. **Do NOT invoke the same reviewer more than two consecutive passes.** If passes N-1 and N both used the Claude cohort, pass N+1 must be Codex (or stop). Same in reverse. This forces cross-vendor coverage and prevents thrash on a single lens.
5. **Do NOT save per-iteration snapshots.** The plan file is edited in place across iterations. The session transcript is the audit.
6. **User-supplied iteration counts are ceilings, not contracts.** If the user says "run 5 iterations," that is an upper bound. Stop early when findings have converged — that is the whole point of having an AI orchestrator applying judgment.
7. **Do NOT auto-approve or auto-finalize.** When stopping, always return control to the user with a summary; do not mark the plan "done" unilaterally.

## Phase 1: Kickoff (interactive, lightweight)

By the time the user invokes the skill, the task has usually already been discussed in the conversation. Do not re-interrogate from scratch.

1. Summarize what you understood from the prior discussion in 3-5 bullets: objective, acceptance criteria, target repository, hard constraints, relevant existing files.
2. Ask the user to confirm or correct. Resolve any gaps with targeted questions.
3. **Check the project root for orientation docs** — `CLAUDE.md` and `AGENTS.md`. Report what you found and identify which (if any) is authoritative. Authority is a per-project choice, not a hard rule:
   - **Both present:** read both. Look for explicit cross-references — e.g., "AGENTS.md is the authoritative project doc; this CLAUDE.md only adds Claude-specific guidance" or the reverse. Whichever one claims authority wins. If neither claims authority, ask the user.
   - **Only `CLAUDE.md`:** CLAUDE.md is the reference doc. **Codex reviews are blocked for this run** — Codex requires AGENTS.md in project root to discover context on its cold-start subprocess invocations (see "Running Codex" for why). Warn the user upfront and offer two options: (a) stub an AGENTS.md that points at CLAUDE.md as authoritative, then continue with Codex enabled; (b) proceed cohort-only, accepting that the alternation rule degrades to "cohort-only, stop at cap or convergence."
   - **Only `AGENTS.md`:** AGENTS.md is the reference doc. Codex reviews available.
   - **Neither:** both reviewers will lack project context. Warn the user strongly; suggest creating at least one orientation doc before proceeding. If the user insists on proceeding, note the degraded context in the stop-report.
4. Record a **Codex-available flag** (`true` if AGENTS.md is in project root, `false` otherwise) and carry it into Phase 3 for the reviewer-selection logic.
5. If a plan file for this task already exists (user is re-invoking on an existing draft), read it first and tell the user you are picking up from the existing draft — do not overwrite silently.
6. When the understanding is complete, announce you are drafting the plan and move to Phase 2.

## Phase 2: Drafting

### Pick a descriptive filename

Not the silly-auto `<adjective>-<verbing>-<noun>.md` pattern that plan mode uses by default. Pick a name that tells a future reader what the plan is about. Use lowercase kebab-case, ending in `.md`.

Good: `add-power-function.md`, `extend-coordinator-to-execute-verify.md`, `migrate-auth-to-jwt.md`, `consolidate-retry-helpers.md`
Bad: `plan.md`, `draft.md`, `new-feature.md`, `serene-coalescing-liskov.md`

### Pick a location by detecting project conventions

Look for existing plan storage in this order:

1. Project-specific conventions documented in `CLAUDE.md` or `AGENTS.md` (read these first to check).
2. Directories already containing plan-shaped files: `docs/plans/`, `plans/`, `.plans/`, `handoff/`, or any directory already containing files named `PLAN*.md` or `*-plan.md`.
3. If none apparent, default to the **project root**.

Never write to `~/.claude/plans/` — that path belongs to plan mode. This skill's output lives in the target repository so it can be committed alongside the implementation if the user wants.

### Write at execution-spec altitude

Your implementer audience is **Codex** (unless the user has directed otherwise for this session). Write at **execution-spec altitude** — concrete enough that reviewers can verify every claim against the code and an implementer can execute without guessing:

- Specific file paths and line numbers where edits land
- Function signatures and data structures to use or extend
- Explicit step-by-step insertion / modification / deletion instructions
- Concrete test names with concrete assertions
- Distinctive error strings where those strings will be asserted in tests
- Named failure paths and exact text of termination reasons
- Cross-references between sections so the implementer can trace dependencies
- No "you know what I mean" or "the implementer can figure out the details"
- If a design decision is genuinely ambiguous, name it as `Decision: X (rationale: Y)` so the implementer knows it was a choice, not a detail to fill in

The plan sections to include (adapt as needed, but keep them named and explicit):

- **Goal** — one paragraph, what the change accomplishes
- **Preconditions** — what must be true about the repo / environment before starting
- **Changes** — the bulk of the plan, organized by file or subsystem
- **Tests** — concrete test names, distinctive assertions
- **Failure modes** — named error paths and exact reason strings
- **Verification** — what to run, in what order, to confirm success
- **Known limitations** — things deliberately out of scope, with rationale

### Build the invariant matrix

After writing the plan body but before finalizing, add an **invariant matrix** section near the top of the plan (between any preamble/context section and the first deliverable). The matrix is a set of tables that encode cross-section invariants — the structural claims that prose-level review repeatedly fails to hold consistent across sections. **The matrix is declared authoritative**: if an implementer finds a conflict between a matrix entry and a deliverable section, they must treat the matrix as correct and surface the discrepancy.

The matrix exists because review passes converge locally (within a section) faster than they converge globally (across sections). A reviewer anchored to the latest fix delta will catch per-section errors but miss cross-section drift: a snippet in section A that contradicts a validation rule in section B, a field described as "caller-supplied" in one place and "registry-constructed" in another, a commit summary that doesn't match its detailed deliverable. The matrix makes these invariants mechanically checkable in one place.

**Which tables to include depends on the plan.** Not every plan needs every table type. Include a table when the plan has the corresponding structural risk. The following are the known high-value table types — include all that apply:

1. **Shared-state ownership by phase/commit/step.** When a data structure, field, config key, or resource is created in one step and consumed in another, build a table with one row per field and one column per commit/phase showing: who supplies it (caller, framework, constructed internally, absent), and whether it is validated. This catches the class of bug where "field X is required in step 3 but validation in step 1 doesn't enforce it."

2. **Interface/payload shapes and validation.** When the plan defines callbacks, event handlers, API endpoints, or message contracts, build a table with one row per interface showing: exact payload shape, what is validated, what happens on invalid input, and what happens on handler error. This catches the class of bug where a validation rule in prose doesn't match the actual emission site, or where error handling is described differently in two sections.

3. **Code snippet fidelity labels.** For every code block in the plan, label it: **Literal** (copy exactly, character-for-character), **Sketch** (illustrative structure, adapt to context), or **Do not copy** (shows current state or anti-pattern for reference). This catches the class of bug where a snippet described as "exact" uses an undefined variable, references a wrong property name, or omits a required import — and an implementer copies it verbatim.

4. **Upstream/parent requirements status.** When the plan implements a subset of a larger spec (parent plan, RFC, design doc), build a table with one row per upstream requirement showing: **Implemented** (with phase reference), **Deferred** (with justification), or **Intentionally superseded** (with justification and divergence note in the plan body). This catches the class of bug where a parent-plan contract is partially reflected and the gap is invisible.

5. **Per-commit/step compilation scope.** When the plan has multiple ordered commits or steps, build a table with one row per file and one column per commit showing which files must compile (or pass lint/typecheck) at each commit boundary. This makes bisect-safety mechanically verifiable and catches "file X was edited in commit 2 but its new import from commit 3 doesn't exist yet."

6. **Cross-cutting guard sites.** When a function or lookup can receive special-case input (virtual IDs, null sentinels, test fixtures) that would break the normal path, build a table listing every call site where the special case can reach, what guard is needed, and the exact guard expression. This catches the class of bug where a new throwing lookup is added but one caller receives the special-case ID and needs a pre-check.

**Writing discipline for the matrix:**

- Each table entry must be verifiable against a specific section of the plan body. If you write a matrix entry that has no corresponding deliverable section, either the deliverable is missing or the matrix entry is wrong.
- When the plan body and matrix disagree during drafting, fix the plan body to match your intended design, then record the correct state in the matrix. Do not paper over the disagreement.
- Keep matrix entries terse — one row, one claim. The matrix is a checklist, not prose. Explanatory notes go in the plan body; the matrix points at them.
- The matrix is a living artifact: Phase 3 review edits must update both the plan body AND the corresponding matrix entries. A fix that changes a deliverable without updating the matrix is incomplete.

### Finalize drafting

Use `Write` to create the file at the chosen path. Do not use plan mode. Do not call `ExitPlanMode`.

Tell the user the plan path and proceed directly to Phase 3. The review loop is how the plan gets approved — you do not need user approval to start reviewing.

## Phase 3: Review loop

### Iteration unit

**One iteration = one review pass (cohort OR Codex) + your judgment on findings + your edits to the plan.**

Default iteration cap: **4**. The user can override with "run N iterations", "run until stable", etc.

The cap is a ceiling. Stop early when findings have converged. Convergence criteria differ by reviewer:

- **Cohort convergence**: two consecutive cohort passes surfaced no substantive findings (empty, or only nits that would not change implementation).
- **Codex convergence** has two tiers. **Local convergence**: finding count drops on a resumed session and no new Highs — that section of the plan is clean. This is the normal stopping point for a typical 3-4 pass review loop. **Global convergence** (late-stage, multi-session plans only): a fresh session's High findings are all theoretical (unlikely preconditions, deployment-context-dependent). This level takes 10-20+ passes across multiple sessions to reach and is not the default expectation. See "Reading Codex output — session-anchoring model" in the "Running Codex" section.
- **Cross-reviewer convergence**: the loop converges when both the cohort AND Codex have independently converged (each by their own criteria). If only one lens has converged, the other still has signal to offer.

### Per-iteration algorithm

**Step 1. Apply judgment on which reviewer to run this iteration.**

Decision factors:

- **Early rounds (architectural plan, broad design):** prefer the Claude cohort. Its strength is design fit, pattern consistency, abstraction choice, failure-mode enumeration.
- **Later rounds (mechanical execution spec):** prefer Codex. Its strength is file paths, line numbers, signature verification, immediate-implication bugs.
- **Respond to the previous pass:** if the last pass surfaced findings the *other* reviewer is better positioned to verify or extend, switch. Example: cohort flagged "you are referencing signatures that may not exist" → switch to Codex for signature verification on the next pass.
- **Alternation constraint (hard rule 4):** never run the same reviewer three consecutive passes. If passes N-1 and N both used the cohort, this pass must be Codex (or stop). Same in reverse.
- **Opening pick for a brand-new draft:** default to the Claude cohort for the first pass. Brand-new drafts tend to have architecture issues that cohort catches faster; Codex's mechanical checks are more valuable once the overall shape is settled.

Briefly state your pick and reasoning before dispatching ("Pass 3: Codex. The last two passes were cohort and surfaced architecture findings; the plan is now at execution altitude and needs mechanical verification on the signatures referenced in the new sections.").

**Step 2. Run the chosen reviewer pass.** See "Running the Claude cohort" or "Running Codex" below for the exact dispatch details.

**Step 3. Read and compile findings.**

- Categorize: **Errors** (plan is wrong), **Omissions** (plan is incomplete), **Improvements** (plan works but could be better).
- Before recording any finding as real, verify the claim against the actual code. Agents (both Claude subagents and Codex) can hallucinate line numbers, misread context, or flag non-issues. Dismiss false positives with a one-line reason so they are visible in the transcript.
- Keep per-reviewer attribution on findings that survive verification. If the cohort found something Codex missed, or vice versa, note it.

**Step 4. Apply judgment on which findings to act on.**

- Default: act on all verified Errors and Omissions.
- Act on Improvements only if they do not expand scope beyond the original objective. Scope-expanding improvements should be noted in the plan's "Known limitations" section, not implemented.
- If a finding would expand scope or change user-visible behavior, flag it to the user and wait for a decision rather than acting unilaterally.
- **Codex findings carry equal weight with cohort findings.** Do not unconsciously down-weight Codex's output because it came from an out-of-session subprocess rather than an in-session subagent. A Codex High finding is as actionable as a cohort Error; a Codex Low finding is no less credible than a cohort Improvement. The "who found it" attribution matters for the cross-vendor signal in the stop report, not for whether to act on the finding.

**Step 5. Edit the plan in place with `Edit`.** Make the fixes. Do not preserve earlier versions; the plan file *is* the working artifact. If a fix requires inserting a new section, insert it at the right position — do not append orphan sections at the bottom.

**Update the invariant matrix alongside every body edit.** When a fix changes a deliverable section, check whether any matrix table entry is affected. A field ownership change, a payload shape change, a snippet correction, a new guard site — all require the corresponding matrix row to be updated in the same edit pass. A fix that changes the plan body without updating the matrix is incomplete and will cause cross-section drift on the next review pass. This is the single most important discipline for preventing the "local convergence, global drift" pattern.

**Prefer deletion over reconciliation.** When a finding says "section A contradicts section B", change one side to match the other. Do not write a third section reconciling them. When a finding says "the rationale for Z is wrong", fix or delete the rationale. Do not layer a corrective paragraph on top.

After any fix, re-read and ask: "if I removed the prose I just added and kept only the mechanical change, would an implementer still get this right?" If yes, remove the prose.

**Step 6. Decide the next move.**

- **Convergence check:** apply the reviewer-specific convergence criteria (see "Iteration unit" above). For cohort: two consecutive passes with no substantive findings. For Codex: finding count dropped AND no High findings. For the loop as a whole: both lenses have independently converged. Stop early when the loop has converged. Report.
- **Ambiguous early-stop:** if this pass surfaced no findings but no prior pass did (it is the first pass), report honestly: "Pass 1 returned no findings. That may mean the plan is already solid, or the reviewer missed something subtle. Propose running a Codex pass next for an independent cross-check." Let the user decide.
- **Cap reached:** if iteration count has hit the cap, stop and report.
- **Default mode (no batch instruction):** propose the next pass (which reviewer, why) and wait for user confirmation before dispatching.
- **Batch mode (user said "run N"):** auto-continue to the next iteration within the cap. Still respect convergence — stop early if findings converge before N.

### Running the Claude cohort

**Default cohort is two agents, not three**: `feature-dev:code-architect` (with a dual-phase merged prompt that does BOTH mechanical verification and architectural review) + `pr-review-toolkit:silent-failure-hunter`. Launch in parallel (single message, multiple `Agent` tool calls — not sequential).

`feature-dev:code-explorer` is **not** part of the default cohort. It spawns conditionally in two cases (see "Conditional code-explorer dispatch" below): as a third parallel agent on **Pass 1 only** when the plan is a brand-new draft, or as a **targeted tiebreaker** on a subsequent pass when the merged architect reports unresolved mechanical uncertainty.

**Why two-agent by default**: `code-architect` and `code-explorer` overlap on mechanical plans — the merged dual-phase prompt captures both lenses in one agent. `silent-failure-hunter` is orthogonal (error handling gaps) and stays separate.

**`feature-dev:code-architect` — dual-phase merged prompt**. The prompt must force **Phase A (mechanical verification)** before **Phase B (architectural review)**, or the agent will skip straight to reasoning and the mechanical-ground-truth lens disappears. Use this structure:

```
You will do two phases. Phase A is a hard gate — do NOT begin Phase B until Phase A is complete and its output is present in your response.

Phase A — Mechanical verification
For every specific file:line citation in the plan (file paths, line numbers, function signatures, field lists, argument shapes), open the referenced file and confirm the claim. Maintain a verification log inline in your response — one row per claim, shape `- file:line — CONFIRMED` or `- file:line — MISMATCH (plan says X, actual is Y)`. No interpretation in Phase A. No architectural claims. Just ground truth. If a claim is ambiguous or a file cannot be read, mark it `UNCERTAIN` with a one-line reason — this feeds the tiebreaker-dispatch decision in the next pass.

Phase B — Architectural review
With Phase A's verified ground truth in hand, now assess:
- Does the plan follow existing patterns in this codebase? Does the abstraction choice fit?
- Are the right files targeted? Are there missing or unnecessary changes?
- Cross-section consistency: does section X contradict section Y? Does a commit summary match the corresponding detailed deliverable?
- **Invariant matrix cross-check**: if the plan contains an invariant matrix section, verify each matrix entry against the plan body AND against the actual code. A matrix entry that contradicts its deliverable section is a High finding. A deliverable section that was edited without updating the corresponding matrix entry is a High finding. A code snippet labeled "Literal" in the matrix that contains an undefined variable, wrong property name, or missing import is a High finding.
- Pattern match against CLAUDE.md / AGENTS.md / PluginSpec.md conventions.
- Failure-mode enumeration — does the plan cover the architecturally important failure paths?

Anchor every Phase B claim in Phase A's verification log where relevant. If Phase A marked a row MISMATCH, cite the row number when you report the architectural consequence.

Report format (both sections required):
[Mechanical findings] — MISMATCHes from Phase A, formatted as Errors with file:line
[Architectural findings] — Errors / Omissions / Improvements from Phase B, with file:line citations
```

**Orchestrator check**: when the merged agent returns, verify Phase A is present and non-empty. If the verification log is missing, empty, or obviously hand-wavy ("I skimmed the files and they look right"), **dismiss Phase B findings and re-dispatch with an explicit "Phase A is mandatory" instruction**. A skipped Phase A means the architectural findings have no mechanical anchor and should not be acted on.

**`pr-review-toolkit:silent-failure-hunter`** — error handling review. Does the plan account for failure modes? Are there missing error paths, silent failures, or inappropriate fallbacks in the proposed design? What can go wrong that the plan does not address? (Prompt unchanged from the original three-agent cohort.)

Each agent's prompt should also include:

- The full path to the plan file (not inline contents — the file is long; have the agent read it)
- Instruction to read the plan in full, then read the actual codebase files the plan references
- Instruction to read **both** `CLAUDE.md` and `AGENTS.md` if present — authority varies per-project (some treat AGENTS.md as authoritative and CLAUDE.md as a Claude-specific supplement, others do the opposite, and some only have one). Tell the agent which doc is authoritative for this project, based on what you determined in Phase 1 §3. The plan must respect the authoritative doc's conventions.
- Instruction to report Errors / Omissions / Improvements with file:line citations (for silent-failure-hunter; the merged architect uses the two-section format above)
- Instruction to be terse — under ~500 words for silent-failure-hunter, under ~700 words for the merged architect to accommodate the Phase A log — and cite exact code where relevant
- Instruction to not propose rewrites of the whole plan; only specific changes

**Wait for all dispatched agents to complete before compiling findings.** Do not start acting on partial results, and do not send more tool calls after spawning the cohort until every agent in the current pass has returned.

### Conditional code-explorer dispatch

`feature-dev:code-explorer` is dispatched in two specific situations, neither of which is the default cohort:

**1. Brand-new-draft Pass 1 triple dispatch.** On Pass 1 of a review loop against a brand-new plan draft (not a re-invocation on an existing plan), spawn `code-explorer` as a third parallel agent alongside the merged architect and silent-failure-hunter. From Pass 3 onward, default to the two-agent cohort. Pass 1 triple dispatch does NOT apply when the user re-invokes the skill on an existing draft (Phase 1 §5 case); in that case, treat Pass 1 as a mid-loop pass and use the two-agent default.

**2. Tiebreaker dispatch after mechanical uncertainty.** If the merged architect's Phase A log contains any `UNCERTAIN` rows — claims the agent could not verify — spawn `code-explorer` on the **next pass** as a targeted verification agent. The explorer prompt is narrow: "read the following claims and confirm each against the actual code. Do not assess architecture, do not surface new findings, do not propose changes. Only CONFIRMED or MISMATCH for each listed claim." Pass the `UNCERTAIN` rows as the explicit verification target. This is explorer operating as a strict mechanical-verification agent, its highest-value mode. The pass still counts as a cohort pass for alternation-rule purposes.

If both conditions fire in the same loop, they compose: Pass 1 gets the triple dispatch anyway; later passes spawn explorer only if the merged architect flagged `UNCERTAIN`. No double-spawning within a single pass.

### Retiring cohort agents mid-session

Within a single invocation of the skill (one review loop), individual cohort agents can be **retired** when they stop producing signal. The retirement rule:

- **Track per-agent substantive finding counts across cohort passes.** A finding counts as substantive if it is an Error or Omission that survives verification. Improvements do not count — an agent that only surfaces nits is not producing load-bearing signal. Findings that get dismissed as false positives also do not count.
- **Strike rule:** if an agent returns **zero substantive findings on two consecutive cohort passes where it was dispatched**, retire it from the rest of this session's cohort passes. Two consecutive zeros — not one — to tolerate the normal case where an agent happens to miss a pass but would catch something on the next.
- **Retirement is session-scoped, not persistent.** Every new skill invocation starts with all agents available. Retirements do not carry across sessions — a new artifact has a new findings surface.
- **Once retired, stay retired for this session.** Do not re-add an agent mid-loop even if the plan grows substantially. If the user wants a retired agent back, they can explicitly ask; otherwise the retirement stands.
- **Report the retirement in the session transcript** when it happens, with the specific pass numbers that triggered it. Example: "Retiring `silent-failure-hunter` — zero substantive findings in Passes 3 and 5; it will not run in subsequent cohort passes this session."

**Retirement semantics under the two-agent default cohort:**

- **Merged `code-architect` retirement**: if the merged architect returns zero substantive findings in BOTH its Mechanical and Architectural sections across two consecutive cohort passes, retire it. The cohort is then down to `silent-failure-hunter` alone. Do NOT split retirement by phase — the agent is retired as a unit because both phases share the same context read, and retiring only one phase's findings while keeping the other makes no cost sense.
- **`silent-failure-hunter` retirement**: same two-consecutive-zero rule. When retired, the cohort is down to the merged architect alone.
- **Conditional `code-explorer` is NOT subject to the standard retirement rule.** Explorer only runs in two specific modes (Pass 1 triple dispatch, or tiebreaker dispatch after UNCERTAIN rows). Both modes are by-dispatch, not by-default, so there is nothing to "retire" — explorer runs when the conditions fire and doesn't run when they don't. If explorer is dispatched as a tiebreaker and returns all CONFIRMED on the flagged claims, that's a successful verification, not a zero-finding strike.

**Degraded-cohort modes:**

- **One default agent retired** (either merged architect or silent-failure-hunter): cohort passes continue with the surviving agent. The pass still counts as a cohort pass for alternation-rule purposes. If the surviving agent ALSO hits two consecutive zeros, move to the next mode.
- **Both default cohort agents retired and Codex is available**: cohort passes become impossible. The alternation rule (hard rule 4) relaxes — Codex may run consecutively because there is no alternative. Announce the mode change to the user. Still respect the iteration cap and convergence stop. Note that conditional `code-explorer` tiebreaker dispatch is ALSO unavailable in this mode, because the merged architect (which produces the UNCERTAIN rows that trigger the tiebreaker) is retired.
- **Both default cohort agents retired and Codex is unavailable** (no AGENTS.md in project root): the skill has no remaining reviewer. Stop the loop and report the "converged in all available lenses" signal to the user. Do not silently end; say explicitly that every reviewer has either retired or was blocked.

### Running Codex

**Precondition: `AGENTS.md` must exist in the project root.** Codex is a cold-start subprocess — each invocation is a fresh OS process that discovers project orientation by reading `AGENTS.md` from its working directory on startup. Without AGENTS.md, Codex has no project context and its review will be uninformed and low-quality. This is **not** a soft recommendation: do not invoke Codex when AGENTS.md is absent.

**Session resume across passes.** The first Codex pass uses a fresh session; every subsequent Codex pass in the same loop resumes the first session's thread via `thread_id`. This preserves conversation context and enables prompt caching. See Step 2a / 2b / 2c below.

Before running Codex, confirm the Phase 1 Codex-available flag is `true`. If it is `false`:
- Do **not** invoke Codex for this pass.
- Fall back to the Claude cohort for this iteration instead.
- Note in the session transcript that Codex was skipped because AGENTS.md is missing.
- The alternation rule (hard rule 4) is relaxed in this degraded mode — cohort may run consecutively because there is no alternative. Still respect the iteration cap and convergence stop.

Codex must be invoked from **project root** (absolute path in `cd`). Never assume the current shell CWD is already project root.

**Step 1. Write the fresh-pass review prompt to a temp file** (avoids shell-escaping issues). Resumed passes (Step 2c) use a shorter follow-up template. Let Codex report in its native severity format (High / Medium / Low) and additionally tag each finding with a category:

```bash
cat > /tmp/special_plan_codex_prompt.txt <<'PROMPT'
You are reviewing an implementation plan at {plan_path} for the {project_name} project.

Your job is to find problems the plan does not account for. Focus on your strengths: mechanical errors (wrong file paths, stale line numbers, signature mismatches), immediate-implication issues (missing error branches, edge cases the plan does not handle, state invariants the plan breaks), and anything that a careful reader grounded in the actual codebase would catch.

Read the plan in full. Read the codebase files the plan references. Verify every specific claim before reporting it.

If the plan contains an invariant matrix section near the top, treat it as the structural cross-check. For each matrix entry, verify it against both the plan body AND the actual code. A matrix entry that contradicts its deliverable section is a High finding. A code snippet labeled "Literal" in the matrix that uses an undefined variable, wrong property, or missing import is a High finding. A deliverable section that was edited without a matching matrix update is a High finding.

Report findings in your default severity format (High / Medium / Low). For each finding, additionally tag it with ONE of these categories so the orchestrator can merge findings across reviewers:
- [Error] — plan is wrong (contradicts the code, breaks a flow, references a function or line that does not exist as described)
- [Omission] — plan is missing something load-bearing
- [Improvement] — plan works but could be cleaner

Shape per finding: `**[severity] [category]** description (file:line)`.

Example: `**High [Error]** gitignore "strip surrounding whitespace" rule is wrong because git does not strip leading whitespace from patterns (PLAN.md:37)`.

Do NOT cap your findings. Surface ALL issues you encounter — every mechanical mismatch, every cross-section inconsistency, every stale line number, every missing import. Thoroughness is more valuable than brevity. Cite file:line for every claim. Do not propose rewrites of the whole plan — only specific changes. Do not write any code.

If you find nothing at High severity, say so plainly — "no High findings" is a convergence signal the orchestrator reads, not a failure. Do not manufacture High findings to fill space.
PROMPT
```

(Substitute `{plan_path}` and `{project_name}` into the heredoc body before writing, or write the file and then use `sed` to replace placeholders.)

**Reading Codex output — session-anchoring model:**

Codex's attention is **session-scoped and section-anchored**. A fresh session picks whichever plan section looks most complex on first read and focuses there. A resumed session stays anchored to the region it already explored. This matters for interpreting convergence:

**Normal convergence (most reviews):** finding count drops on resumed sessions, no new Highs. This is *local* convergence — the section Codex anchored to is clean. For a typical 3-4 pass review within the default iteration cap, this is sufficient. Most plans are good enough to hand to an implementer at this point.

**Late-stage convergence (complex plans, 10-20+ passes across sessions):** when the user pushes past the default cap on a complex plan, resumed-session convergence is no longer sufficient because Codex has only ever examined the section it initially anchored to. At this stage:

1. **Fresh sessions find issues that resumed sessions miss** — a different section gets attention. If a fresh session surfaces mechanical Highs (undefined variables, missing imports, cross-section contradictions), those are real and were invisible to the resumed thread.

2. **Fresh sessions on mature plans chase theoretical concerns.** Once real issues are fixed, Codex on a fresh session will still find the most complex-looking section and try to produce findings. These tend to be technically correct but practically irrelevant — "what if an exception fires during the 50ms startup window", "what if a supervisor reads the exit code." The orchestrator MUST apply judgment: a finding is theoretical if it requires an unlikely precondition AND the plan's deployment context rules it out. **Theoretical-only High findings from a fresh session = global convergence** — the mechanical surface is clean.

3. **Resumed sessions remain efficient for verifying fixes** — Codex has the prior context, catches regressions introduced by edits. But once it reports only Medium/Low, a fresh session adds more value than another resume.

**Fresh-vs-resume decision rule (late-stage only — within the default cap, just resume):**
- After applying fixes from a Codex pass, **resume** to verify fixes and catch regressions.
- If the resumed pass surfaces only Medium/Low (no new Highs), **start fresh** — a new session samples a different plan section.
- If a fresh session surfaces only **theoretical** Highs, that is global convergence. Report them as "theoretical Highs, converged" and let the user decide.
- If a fresh session surfaces **mechanical** Highs, fix them and continue.

**Typical Codex output shape:** default Codex produces ~5-8 findings per pass. The explicit "do NOT cap, surface ALL issues" instruction in the prompt overrides this to ~8-20, depending on plan quality.

- **Severity is NOT symmetric with action priority.** A Medium finding may still be worth acting on; it means "Codex believes this is real but not critical." Apply the same verification discipline (confirm against code) before acting.
- **Category tags merge cleanly with cohort findings.** Claude cohort agents use Errors / Omissions / Improvements natively in their prompts. Codex tags findings with the same taxonomy, so per-category tallies across reviewers remain comparable even though the primary framing differs.

**Step 2. Invoke Codex** from the target project root. First pass uses a fresh session; subsequent passes resume via `codex exec resume <thread_id>`.

**2a. First Codex pass in the loop** — fresh session:

```bash
cd /absolute/path/to/project/root
codex exec --json --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check "$(cat /tmp/special_plan_codex_prompt.txt)" < /dev/null > /tmp/special_plan_codex_out.jsonl 2> /tmp/special_plan_codex_err.log
```

All flags are required: `--json` for parseable output, `--dangerously-bypass-approvals-and-sandbox` for non-interactive invocation, `--skip-git-repo-check` for repos with unusual state, `< /dev/null` to suppress stdin warning.

**2b. Extract and persist the `thread_id`** immediately after the first pass completes, before starting any other work:

```bash
jq -r 'select(.type == "thread.started") | .thread_id' /tmp/special_plan_codex_out.jsonl | head -1 > /tmp/special_plan_codex_thread_id
```

If the file is empty after this step, the extraction failed — fall back to a fresh session on the next Codex pass. Do not attempt `codex exec resume ""` (misparses).

**2c. Second and later Codex passes in the same loop** — resume the thread:

```bash
THREAD_ID=$(cat /tmp/special_plan_codex_thread_id)
cd /absolute/path/to/project/root
codex exec resume --json --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check "$THREAD_ID" "$(cat /tmp/special_plan_codex_prompt.txt)" < /dev/null > /tmp/special_plan_codex_out.jsonl 2> /tmp/special_plan_codex_err.log
```

Key differences from the fresh call:
- **`exec resume`** is the subcommand (not `exec` alone). `resume` is a subcommand of `exec`, not a top-level `codex resume`.
- **The `thread_id` is a positional argument**, immediately after the flags and **before** the prompt. Order matters: `codex exec resume <flags> <thread_id> <prompt>`.
- **Do NOT pass `--cd` or `-C`** — `codex exec resume` does not accept it. Use a shell `cd` to the project root instead, same as the fresh invocation.
- **All other flags stay the same** (`--json`, `--dangerously-bypass-approvals-and-sandbox`, `--skip-git-repo-check`, `< /dev/null`).
- **The follow-up prompt should be MUCH shorter** than the fresh-pass prompt — see "Follow-up prompt shape" below.

**Follow-up prompt shape — keep it tight on resumed passes.** On a resumed pass, Codex already has the plan, prior findings, and prior fixes in context. A resumed-pass prompt can be 5-10 sentences. Recommended template:

```
The plan at {plan_path} has been edited based on prior review findings (yours and the Claude cohort's). Please re-review the changes in the plan since your last pass and report any NEW issues, focusing on:

1. New mechanical regressions introduced by the edits (wrong line numbers, stale citations, broken cross-references between sections that were rewritten).
2. Contradictions between newly-added content and existing sections.
3. If the plan has an invariant matrix section: matrix entries that now contradict their deliverable sections (body was edited but matrix was not updated, or vice versa). Matrix-body drift is a High finding.
4. Any High-severity findings you missed last time that become visible now.

Do NOT re-surface findings you already reported in prior passes — the orchestrator is tracking those and will verify independently whether they're resolved. "No new High findings" is a valid and useful response.

Output format and severity conventions are unchanged from the first pass. Do NOT cap findings — surface everything new.
```

Do not re-paste the full prompt template on resumed passes — Codex already has it. Keep the resumed prompt scoped to "here's what changed, find new issues."

**Step 3. Extract the response text** from the JSON event stream (same for fresh and resumed passes):

```bash
jq -r 'select(.type == "item.completed" and .item.type == "agent_message") | .item.text' /tmp/special_plan_codex_out.jsonl
```

This returns Codex's review as plain text. Read it as the findings for this pass.

**Step 4. Verify Codex's findings against the code** before acting — same discipline as cohort findings. Codex can hallucinate line numbers; always spot-check.

**Codex failure handling.** If `codex exec` or `codex exec resume` returns a non-zero exit code, read `/tmp/special_plan_codex_err.log` for diagnostics. On the first failure: report to the user, suggest running the cohort for this pass instead, and continue. On repeated failures: escalate to the user and pause the loop. Do not silently skip Codex passes — if the alternation rule would require Codex and Codex is broken, the user needs to know.

**Resume-specific failure modes**:
- **"thread not found" or similar**: the `thread_id` has been garbage-collected or was never valid. Fall back to a fresh session on this pass (not a resume), re-extract a new `thread_id`, and continue the loop with the new thread from this point forward. Note the fallback in the session transcript — it means the next-pass continuation will reset Codex's conversation context.
- **Empty `thread_id` file**: Step 2b's extraction failed on the first pass (e.g., JSONL stream was malformed or truncated). Fall back to a fresh session on the next pass and re-extract; do not attempt `codex exec resume ""` because the empty argument will cause the command to misparse.
- **Cross-session thread reuse**: the `thread_id` is scoped to a single skill invocation. When the user re-invokes the skill on a new plan (or the same plan in a new conversation), start with a fresh session — do not attempt to resume a thread from a prior run. `/tmp/special_plan_codex_thread_id` can be safely overwritten each invocation.

### Stopping and reporting

Stop the loop when any of these conditions hold:

- Findings have converged per the reviewer-specific criteria (cohort: two consecutive passes with no substantive findings; Codex: resumed session finding count dropped + no new Highs — or, late-stage, fresh session produces only theoretical Highs; loop: both lenses independently converged)
- Iteration cap reached
- User interrupts or asks you to stop
- An unrecoverable error makes further progress impossible

When stopping, report:

- **Actual iteration count** and reason for stopping (e.g., "stopped at 3/4 — last two passes surfaced only nits, findings converged")
- **Final plan path**
- **Reviewer sequence** (e.g., "cohort → Codex → cohort")
- **Per-reviewer finding tally** — how many Errors / Omissions / Improvements did each source surface, verified vs. dismissed. This feeds the cross-vendor lens observation.
- **Any open items** the user should know about before handing the plan to an implementer (scope-expansion findings you flagged but did not act on; verification gaps; known limitations)
- **Recommendation** — is the plan ready for implementation, or does something else need to happen first?

Do not mark the plan "done" on your own. The user decides when it is ready to hand off.

## Reviewer strengths

- **Claude cohort:** architectural judgment, design fit, pattern consistency, failure-mode enumeration.
- **Codex:** mechanical errors, file paths, line numbers, signature verification, immediate-implication bugs, contradictions. Weaker at broad architectural judgment. On mature plans (10-20+ passes), tends to chase theoretical edge cases — apply the judgment model from "Reading Codex output."
- **Codex outputs ~8-20 findings per fresh pass** (with uncapped prompts). Finding count dropping on resumed sessions is a valid local convergence signal. A fresh pass on a late-stage plan that returns only theoretical Highs is the global convergence signal.

Early-draft architectural plan → cohort. Late-stage mechanical execution spec → Codex. Mixed signal → alternate per the alternation rule. If observations drift (e.g., Codex catches architecture issues the cohort missed), update your judgment for that pass and note it in the transcript.

## Scope reminders for the session

- When in doubt about whether the plan is "done", stop early and defer to the user.
- The user may interrupt mid-loop to change direction. Adapt and continue.
- Briefly state each decision (reviewer pick, finding verdict, edit summary) as you go — the transcript is the audit record.
