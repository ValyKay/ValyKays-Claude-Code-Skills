# ValyKay's Claude Code Skills

A suite of skills that elevates Claude Code from solo coder to department head: dispatching specialist agents for focused passes, weighing their findings, and — when a cross-vendor perspective is worth the cost — delegating to OpenAI's Codex as an outside reviewer.

Seven skills organized across two ladders of increasing rigor: plan authoring (`plan-review` → `special-plan` → `debate-plan`) and code review (`full-review` → `special-review` → `debate-review`). One supporting plugin (`synthesis-author`) backs the top rung of each ladder. One additional skill (`codex-review`) provides the bare Codex lens for ad-hoc use.

## Skills

### full-review

Runs five code-review specialists in parallel against the current unstaged git diff:

- `pr-review-toolkit:code-reviewer` — project conventions and style compliance
- `feature-dev:code-reviewer` — logic errors and security issues
- `pr-review-toolkit:silent-failure-hunter` — error handling, silent fallbacks, operator-visibility gaps
- `pr-review-toolkit:type-design-analyzer` — type invariants and API design
- `pr-review-toolkit:comment-analyzer` — comment accuracy and rationale-vs-code drift

Iteration caps retire converged agents automatically, so repeated runs stop re-surfacing resolved findings.

Usage: `/full-review`

### plan-review

Audits an implementation plan before implementation begins. Three specialists run in parallel: `feature-dev:code-architect` checks design fit, `feature-dev:code-explorer` verifies cited file paths and signatures against the actual codebase, and `pr-review-toolkit:silent-failure-hunter` surfaces unaccounted-for failure modes.

Usage: `/plan-review`

### special-plan

Claude drafts an execution-grade implementation plan, then drives a multi-pass review loop alternating the in-house cohort with Codex as a cross-vendor second opinion. Each pass selects the reviewer better suited to the current state of the plan, applies judgment to the findings, edits the plan in place, and stops when both lenses have independently converged.

Intended for plans that will be handed to Codex or another AI implementer, where hand-waving is not tolerable.

Requires the `codex` CLI on PATH and `AGENTS.md` in the project root for Codex passes; degrades to a cohort-only loop without them.

Invoke explicitly: `use special-plan` (does not trigger on generic "make a plan" requests).

### special-review

The review counterpart to `special-plan`. A multi-pass code-review loop that alternates the cohort with Codex, applies judgment between rounds, and stops when both lenses have run out of new findings. No auto-fixes — findings are returned; between passes the implementer is chosen explicitly (Claude, Codex, or the user). User-supplied iteration counts are treated as ceilings, not contracts.

Requires the `codex` CLI on PATH and `AGENTS.md` in the project root for Codex passes; degrades to a cohort-only loop without them.

Invoke explicitly: `use special-review` (generic "review this" requests fall through to `/full-review` by default).

### debate-plan

One rung above `special-plan`. Claude drafts the execution-spec plan, then runs two independent review teams on the raw plan — the in-house cohort and a baseline Codex pass — in strict sequence. Findings are correlated into a union; the substantive picks are resolved by the `synthesis-author` subagent (authored blind to the rest of the workflow, by design) and then debated with Codex until convergence. A final prescription pass strips justification prose from the settled plan so the implementer receives instructions rather than archaeology.

Higher cost than `special-plan`, strictly more review coverage. Optimized for unattended runs with approvals off.

Invoke explicitly: `use debate-plan`.

### debate-review

The review counterpart to `debate-plan`, and the top rung of the code-review ladder. Two independent review teams run on the raw diff (cohort and baseline Codex), findings are correlated, the substantive picks are resolved by the `synthesis-author` subagent, and a debate between orchestrator and Codex converges the picks. At convergence an overall debate winner is declared — and whoever won implements all converged fixes in a single pass. One post-implementation verification pass by the team that did not implement runs before exit.

Designed for diffs where environmental preconditions (server stopped, environment ready, uncommitted work preserved) are already handled upstream, so the implementation dispatch can auto-fire. Higher cost than `special-review`; dispatched implementation instead of returned findings.

Invoke explicitly: `use debate-review`.

### codex-review

The bare Codex lens. One invocation, one Codex pass, against any artifact named — a plan file, a diff, a pull request, a specified set of files. No cohort, no orchestration, no loop. On re-invocation against the same artifact, the Codex thread is resumed rather than restarted, preserving context and prompt caching.

For cases where a cross-vendor second opinion is wanted without the overhead of a full review cycle.

Requires the `codex` CLI on PATH, `jq`, and `AGENTS.md` in the project root.

Invoke explicitly: `use codex-review` (does not trigger on generic "what does Codex think" phrasing).

## Supporting plugins

### synthesis-author

A supporting plugin, not a standalone skill. Provides the `synthesis-author` subagent used by `debate-plan` and `debate-review` to convert reviewer-notes files into a per-finding implementation spec. The subagent verifies every cited file:line against the actual code before adopting, countering, or dismissing each flag. Its system prompt is deliberately blind to the rest of the workflow, which prevents the "I'll let the next reviewer catch it" failure mode where an orchestrator-author sheds effort in anticipation of a downstream critic.

Not invoked directly by users; installed alongside the debate skills.

## Dependencies

All skills require these Claude Code plugins:

```
claude plugins install pr-review-toolkit
claude plugins install feature-dev
```

`special-plan`, `special-review`, `debate-plan`, `debate-review`, and `codex-review` additionally require:

- `codex` CLI on PATH
- `jq` on PATH (for parsing Codex JSON output)
- `AGENTS.md` in the project root of any repo under review (Codex cold-starts a fresh subprocess on each invocation and orients by reading `AGENTS.md` from its working directory; without it, the review is uninformed)

`debate-plan` and `debate-review` additionally require the `synthesis-author` plugin in this repository.

### Unattended operation

For fully autonomous runs — required for `debate-plan`, recommended for `debate-review` — run Claude Code with `--dangerously-skip-permissions`. Otherwise Codex invocations and orchestrated tool calls will pause for manual approval.

## Installation

### Plugin

This repository is itself a Claude Code plugin marketplace — **ValyKays Claude Code Plugins** (`valykays-claude-code-plugins`) — that ships the `synthesis-author` plugin. Install it through the `/plugin` manager, following the flow described in the [Claude Code plugin docs](https://code.claude.com/docs/en/discover-plugins):

1. Run `/plugin` to open the plugin manager.
2. Switch to the **Marketplaces** tab and choose **Add Marketplace**. When prompted for a source, enter one of:
   - `ValyKay/ValyKays-Claude-Code-Skills` — GitHub short form
   - `git@github.com:ValyKay/ValyKays-Claude-Code-Skills.git` — SSH
   - `./path/to/ValyKays-Claude-Code-Skills` — local checkout of this repo
3. Switch to the **Discover** tab, find `synthesis-author` under the `valykays-claude-code-plugins` marketplace, and press **Enter** to install. Pick a scope (user, project, or local).
4. Run `/reload-plugins` to activate the subagent without restarting Claude Code.

Once installed, `debate-plan` and `debate-review` dispatch the `synthesis-author:synthesis-author` subagent automatically — no further wiring required.

### Skills

Skills in this repository are added individually:

```
claude skills add /path/to/full-review
claude skills add /path/to/plan-review
claude skills add /path/to/special-plan
claude skills add /path/to/special-review
claude skills add /path/to/debate-plan
claude skills add /path/to/debate-review
claude skills add /path/to/codex-review
```
