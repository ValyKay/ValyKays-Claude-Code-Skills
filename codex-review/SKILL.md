---
name: codex-review
description: Run a single-turn OpenAI Codex review against an existing artifact — an implementation plan, a git diff, a pull request, specific files, or any other artifact the user names. Universal and stage-agnostic — works identically for plans and for already-written code. Use this ONLY when the user explicitly invokes the skill by name — "use codex-review", "run codex-review on this", "codex-review the plan", "codex-review the diff", or equivalent direct reference. This is a specialized tool, not a default. Do NOT trigger on general review or second-opinion requests, "what does Codex think", or similar phrasing unless the skill name is used. Only activate when the user names the skill. The skill preserves the Codex thread between invocations on the same artifact, so a follow-up invocation resumes rather than restarts.
---

# codex-review

A universal, single-turn Codex review dispatcher. One invocation = one Codex pass against an existing artifact. The skill does not draft, implement, or fix — it dispatches Codex, parses the output, verifies findings, and hands them back. When the user re-invokes the skill on the same artifact, the prior Codex thread is resumed so context and prompt caching are preserved.

The motivation: `special-plan` and `special-review` each wrap Codex in a stage-specific multi-pass loop (plan-drafting, diff-reviewing, alternating with a Claude cohort). You often want just the Codex lens on its own — one pass, one artifact, no cohort, no orchestration. This skill is that bare tool.

## Dependencies

- `codex` CLI on PATH (`command -v codex` must succeed)
- `jq` on PATH (for parsing Codex JSON output)
- `AGENTS.md` in the project root of the target repository (Codex cold-starts a fresh subprocess each invocation and discovers project context by reading `AGENTS.md` from its working directory — without it, the review is uninformed)

## Hard rules

1. **One Codex pass per invocation.** This skill does not loop. If the user wants another pass, they re-invoke the skill; the second invocation resumes the first's thread.
2. **Do NOT write or modify any artifact.** This skill is strictly a reviewer. It does not edit the plan, the diff, the code, or anything else. The user decides how and when to act on findings, with whichever tool they prefer (Claude, Codex, manual edit).
3. **Do NOT invoke Codex for coding inside this skill.** Codex is the reviewer here. If the user wants Codex to implement, that is a separate operation outside this skill's scope.
4. **Do NOT drift toward a cohort.** This skill is Codex-only by design — no Claude subagents, no parallel lenses. For cross-vendor coverage, the user should use `special-plan` or `special-review` instead.
5. **Do NOT declare the review "done" or "approved".** Hand findings back with a recommendation; the user decides what ready means.
6. **Report Codex output faithfully.** Do not re-rank, soften, or silently drop findings. A finding may be dismissed as a false positive, but only after verifying against the code and only with a one-line reason in the transcript.

## Phase 1: Capture the target and check for prior session

The user invokes the skill after the artifact already exists — a plan file on disk, a git branch with commits, a set of specific files. Do not draft anything. Confirm the target briefly and resolve gaps before dispatching.

### Target shapes this skill supports (adapt as needed)

- **Plan file** — a markdown implementation plan at a path in the repo.
- **Git diff** — unstaged changes, staged changes, a branch vs. `main`, a commit range, or a PR.
- **Specific files or directories** — one or more paths, reviewed as-is (not as a diff).
- **Free-form target** — anything else the user names: a design doc, a config, a migration script, a prose spec.

### Confirm in 2-4 bullets

1. **Target** — exact path, scope, or identifier (e.g., `plans/migrate-auth.md`, `git diff main...HEAD`, `src/auth/middleware.ts`).
2. **Focus** — what the user wants Codex to pay particular attention to (correctness, security, cross-section consistency, mechanical errors). Default: "whatever Codex surfaces, let the user filter." Do not over-constrain unless asked.
3. **Any hard exclusions** — files to skip, concerns to ignore.

A target like "review the auth stuff" is too vague — ask for a path, a commit, or a file list before dispatching. Codex burns real tokens; do not fire it on a fuzzy target.

### Compute a stable target key (for session resume)

Codex's session-resume mechanism depends on reusing the same `thread_id` across invocations. To find the prior thread (if one exists), derive a **target key** from the target in a way that is stable across invocations on the same artifact but unique per artifact. Use this approach:

- **Plan file / specific files** — absolute path, sha256 of the path string: `echo -n "/abs/path/to/plan.md" | sha256sum | cut -c1-16`.
- **Git diff** — the scope string after normalization: e.g., `diff:main...HEAD`, `diff:--cached`, `diff:unstaged`, `diff:abc123..def456`. Then sha256 of that string.
- **Free-form target** — the user's own name for the target plus the project root path, then sha256. Ask the user to name it consistently on re-invocation (e.g., "the auth migration") — record the name in the transcript so the user can see what to repeat.

The thread ID file lives at `/tmp/codex_review_thread_<target_key>`. This keeps multiple concurrent targets from stomping each other — reviewing a plan and a diff in alternating invocations keeps both threads alive in separate files.

Also store a small metadata file alongside — `/tmp/codex_review_meta_<target_key>` — containing one line per field: target description, project root, creation timestamp. This is for your own audit when re-invoked; read it on re-invoke to confirm the target you are about to resume is the one the user actually means.

### Detect prior session and decide fresh-vs-resume

- If `/tmp/codex_review_thread_<target_key>` exists and is non-empty, this is a **resume**. Read the metadata file, confirm with the user in one sentence: "Found a prior Codex thread for this target from {timestamp}. Resuming that thread." If the user objects (wants a fresh look), delete the thread file and metadata, then treat this as fresh.
- Otherwise, this is a **fresh** invocation.

Fresh and resume use different prompts and different Codex subcommands — see Phase 3.

### Preflight checks

Before dispatching Codex, verify all three:

- `command -v codex` succeeds.
- `command -v jq` succeeds.
- `AGENTS.md` exists in the **target repository root** (the directory Codex will `cd` into — not just anywhere).

If any check fails, stop and report. Do not try to "work around" a missing AGENTS.md — suggest the user either stub one (even a short pointer at `CLAUDE.md` works) or use a different reviewer. If `codex` is not installed, direct the user to install it rather than guessing a workaround.

## Phase 2: Build the prompt

Write the prompt to a temp file (`/tmp/codex_review_prompt_<target_key>.txt`) to avoid shell-escaping headaches. Fresh and resume use different prompt shapes; both have a universal spine with two slots varying by target shape.

### Fresh-pass prompt template

```
You are reviewing {target_description} for the {project_name} project.

Your job is to find problems. Focus on your strengths: mechanical errors (wrong file paths, stale line numbers, signature mismatches, undefined references, broken imports), immediate-implication issues (null dereferences, off-by-one errors, missing error paths, race conditions, resource leaks, edge cases not handled), and anything that a careful reader grounded in the actual codebase would catch.

{target_specific_instructions}

Read the target in full. Read the surrounding code (callers, callees, related tests, referenced files) to understand the target in its environment. Verify every specific claim against the actual code before reporting.

Report findings with severity labels:
- **Critical** — security vulnerabilities, data loss, crashes, broken invariants
- **High** — logic errors, missing error handling, broken functionality, contradictions
- **Medium** — suboptimal patterns, minor bugs, style violations that cost clarity
- **Low** — nits, formatting, naming, documentation

Additionally tag each finding with ONE category so findings can be grouped:
- [Error] — target is wrong (contradicts the code, references something that does not exist, breaks a flow)
- [Omission] — target is missing something load-bearing
- [Improvement] — target works but could be cleaner

Shape per finding: `**[severity] [category]** description (file:line)`.

Example: `**High [Error]** null dereference on user.profile when profile hasn't loaded yet (src/components/UserCard.svelte:47)`

Do NOT cap your findings. Surface ALL issues you encounter — every mechanical mismatch, every missing branch, every stale reference. Thoroughness is more valuable than brevity. Cite file:line for every claim. Do not propose full rewrites — only specific issues. Do not write any code.

If you find nothing at Critical or High severity, say so plainly — "no Critical/High findings" is a useful signal, not a failure. Do not manufacture findings to fill space.
```

### Filling `{target_description}` and `{target_specific_instructions}`

Keep the spine universal; vary only these two slots:

- **Plan file** → description: "the implementation plan at {path}". Instructions: "Read the plan in full. Check every file:line citation against actual code. Look for cross-section contradictions. If the plan has an invariant matrix near the top, verify each matrix entry against both the plan body and the actual code — matrix-body drift is a High finding."
- **Git diff** → description: "the code changes in {scope}" (e.g., "`git diff main...HEAD`"). Instructions: "Read the diff and the files it touches. Read surrounding context — callers, callees, related tests — to ground your review in the broader codebase, not just the diff."
- **Specific files** → description: "the code at {paths}". Instructions: "Read the files fully. Read callers and callees to understand how the code is used. You are reviewing the current state, not a diff — focus on defects in the code as it stands."
- **Free-form** → description: whatever the user named it ("the migration script at db/migrations/042.sql"). Instructions: matched to the artifact type. Err on the side of letting Codex use its judgment rather than over-constraining.

If the user named specific concerns in Phase 1, append one sentence: "Pay particular attention to {concerns}, but do not skip other issues you notice."

### Resume-pass prompt template

Much shorter. Codex already has the target, prior findings, and the full conversation in context:

```
The target at {target_description} may have changed since your last pass. Re-review and report any NEW issues:

1. Regressions introduced by edits to prior findings (new bugs, broken imports, stale references, drift from the prior design).
2. Issues in newly changed content that was not in the prior pass.
3. Any Critical/High findings you missed last time that become visible now.

Do NOT re-surface findings from prior passes — they are tracked. "No new Critical/High findings" is a valid and useful response.

Severity and category conventions unchanged. Do NOT cap findings.
```

Do not re-paste the full fresh template on resume — Codex already has it. Keep the resumed prompt scoped to "what changed, find new issues."

## Phase 3: Invoke Codex

Codex must run from the **target project root** (absolute path in `cd`). Do not assume the shell's current working directory.

### Fresh invocation

```bash
cd /absolute/path/to/project/root
codex exec --json --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check "$(cat /tmp/codex_review_prompt_<target_key>.txt)" < /dev/null > /tmp/codex_review_out_<target_key>.jsonl 2> /tmp/codex_review_err_<target_key>.log
```

All flags are required:
- `--json` — parseable event-stream output
- `--dangerously-bypass-approvals-and-sandbox` — non-interactive invocation
- `--skip-git-repo-check` — tolerates repos with unusual state
- `< /dev/null` — suppresses stdin warning

### Immediately extract and persist the `thread_id`

This is the mechanism that makes a later re-invocation a resume instead of a restart. Do it before any other work:

```bash
jq -r 'select(.type == "thread.started") | .thread_id' /tmp/codex_review_out_<target_key>.jsonl | head -1 > /tmp/codex_review_thread_<target_key>
```

If the file is empty after this step, extraction failed — the next invocation will have to start fresh. Note the extraction failure in the transcript so the user is not surprised when a follow-up does not resume. Do not attempt `codex exec resume ""` (empty arg misparses).

Also write the metadata file:

```bash
cat > /tmp/codex_review_meta_<target_key> <<EOF
target: {target_description}
project_root: /absolute/path/to/project/root
created: $(date -Iseconds)
EOF
```

### Resume invocation

```bash
THREAD_ID=$(cat /tmp/codex_review_thread_<target_key>)
cd /absolute/path/to/project/root
codex exec resume --json --dangerously-bypass-approvals-and-sandbox --skip-git-repo-check "$THREAD_ID" "$(cat /tmp/codex_review_prompt_<target_key>.txt)" < /dev/null > /tmp/codex_review_out_<target_key>.jsonl 2> /tmp/codex_review_err_<target_key>.log
```

Key differences from the fresh call:
- **`exec resume`** is the subcommand (not `exec` alone, not a top-level `codex resume`).
- **`thread_id` is a positional argument** between the flags and the prompt — order matters: `codex exec resume <flags> <thread_id> <prompt>`.
- **Do NOT pass `--cd` or `-C`** — `codex exec resume` does not accept it. Use a shell `cd` to the project root.
- **All other flags stay the same.**
- **The prompt is the resume template**, not the fresh template.

Do not re-extract the `thread_id` on resume — the same file keeps the original thread across subsequent resumes. Only update the metadata file's timestamp if you want to track the most recent resume.

### Extract Codex's response text

Same command for fresh and resume:

```bash
jq -r 'select(.type == "item.completed" and .item.type == "agent_message") | .item.text' /tmp/codex_review_out_<target_key>.jsonl
```

This prints Codex's review as plain text — the pass's findings.

## Phase 4: Verify findings and present

**Do not forward Codex's output raw.** Codex can hallucinate line numbers, misread context, or flag non-issues. Before presenting:

1. **Spot-check every file:line citation** against the actual code. Open the file, look at the cited region, confirm the claim matches reality.
2. **Dismiss false positives** with a one-line reason in the transcript. Keep them visible — the user should see what was filtered and why.
3. **Group surviving findings by severity** (Critical → High → Medium → Low) and, within severity, keep Codex's category tags ([Error] / [Omission] / [Improvement]) so the user can skim by shape.
4. **Note the finding count** and, on resume, the delta vs. the prior pass (if you can tell from the user's transcript or memory) — this is the convergence signal.

Present in this shape:

```
Codex review — {pass_type: fresh | resumed}

Target: {target_description}
Project root: {root}

Findings ({total} verified, {dismissed} dismissed):

Critical:
- [finding 1]
...

High:
- [finding 2]
...

Medium:
- ...

Low:
- ...

Dismissed as false positive:
- [finding X] — [one-line reason]

Session state: thread_id persisted at /tmp/codex_review_thread_<target_key>. Re-invoke codex-review on the same target to resume this thread.

Recommendation: {brief judgment — e.g., "several High-severity mechanical errors, worth fixing before implementation" or "no Critical/High findings, target looks clean to Codex on this pass"}
```

Do not mark the review "done" or "approved". The recommendation is a judgment, not a verdict. The user decides.

## Failure modes

**`codex exec` or `codex exec resume` returns non-zero.** Read `/tmp/codex_review_err_<target_key>.log`. Report the error to the user with a concrete next step (retry, check `codex --version`, check AGENTS.md). Do not silently skip.

**Resume-specific failures:**

- **"thread not found"** — the `thread_id` has been garbage-collected on Codex's side, or was never valid. Delete `/tmp/codex_review_thread_<target_key>` and `/tmp/codex_review_meta_<target_key>`, report the fallback to the user, and either stop or re-run as fresh per the user's preference. Do not attempt another resume with the same thread ID.
- **Empty `thread_id` file from a prior invocation** — extraction failed last time. Treat this invocation as fresh; re-extract a new `thread_id` after the pass.
- **Stale thread (e.g., artifact changed substantially since last pass)** — technically a resume will still work, but the prior context may mislead Codex. If the user says the artifact has been heavily rewritten, offer to start fresh instead of resuming. The user decides.

**Codex reviews the wrong thing** (cites files not in the target, discusses a different project). Usually means the working directory or AGENTS.md is misconfigured. Verify you `cd`'d to the project root and AGENTS.md is present. Retry; if it persists, report and stop.

**Codex's finding count explodes into noise.** Uncapped Codex can surface 15-25 findings on a large target. If the false-positive rate climbs above ~30%, the prompt's focus is off, the target is too broad, or Codex is chasing theoretical edge cases. Report this to the user and offer to narrow the target or tighten the prompt on the next invocation.

## Session lifecycle

- **One Codex pass per invocation, period.** The skill runs once and returns.
- **Thread persists across invocations via `/tmp/codex_review_thread_<target_key>`.** A follow-up invocation on the same target reads the file and resumes.
- **Thread reset** — if the user wants a fresh look at a target that already has a thread, delete `/tmp/codex_review_thread_<target_key>` (and the matching meta file) before dispatching. Confirm with the user first — resetting loses Codex's prior context and prompt cache for that target.
- **Cross-target isolation** — different targets have different keys, so concurrent threads for a plan and a diff coexist without stomping.
- **No cross-user, no cross-machine** — `/tmp` is local. Thread IDs issued by Codex on this machine are not portable.

## Scope reminders

- This skill reviews. The user acts on findings. Do not conflate the two roles.
- One invocation = one Codex pass. The user drives iteration by re-invoking.
- Briefly state each decision (fresh vs. resume, target key, dispatch command summary) as you go. The transcript is the audit record.
- When in doubt about whether a target is review-ready or a prior thread is still relevant, ask the user. Do not guess.
