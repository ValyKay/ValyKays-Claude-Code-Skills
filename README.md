# ValyKay's Claude Code Skills

Ever catch yourself wondering if the grass is greener on the other side of the AI fence? Lying awake at night thinking *"what if 'the other guy' is actually better?"* Convinced you're missing out on the best agents while everyone else is shipping flawless code?

Worry no more.

These skills teach Claude to stop being a lonely coder and start acting like a seasoned department head — delegating to a hand-picked roster of specialists, weighing their reports, and then (when the job calls for a second opinion only an outsider can give) shelling work out to its trusted subcontractor across the street: none other than **OpenAI's Codex**.

Five skills, one philosophy: more eyes, better verdicts, less bikeshedding.

## Skills

### full-review

Run all five code-review specialists in parallel against your unstaged git diff. The crew:

- **the stylist** (`pr-review-toolkit:code-reviewer`) — keeps your conventions in line
- **the bug hunter** (`feature-dev:code-reviewer`) — sniffs out logic errors and security holes
- **the paranoid one** (`pr-review-toolkit:silent-failure-hunter`) — refuses to trust any error path
- **the type theorist** (`pr-review-toolkit:type-design-analyzer`) — judges your invariants from on high
- **the comment auditor** (`pr-review-toolkit:comment-analyzer`) — calls out lies in your docstrings

Iteration caps mean repeated runs retire converged agents automatically — nobody wants the comment auditor showing up to the fifth meeting with nothing new to say.

Usage: `/full-review`

### special-review

The review counterpart to `special-plan`. A multi-pass code-review loop that alternates the in-house cohort with **Codex** pass after pass, applies judgment between rounds, and stops the moment both lenses have run out of new things to say. No auto-fixes — the skill reviews and hands findings back; between passes you pick who applies the edits (Claude, Codex, or you). User-supplied iteration counts are ceilings, not contracts.

Requires the `codex` CLI on PATH and an `AGENTS.md` in the project root for Codex passes. Without them, the skill degrades gracefully to a cohort-only loop.

Invoke explicitly: `use special-review` (it deliberately won't trigger on generic "review this" requests — `/full-review` is the default for that).

### plan-review

Three specialists audit your implementation plan **before** you write a line of code. The architect checks design fit, the explorer verifies that the file paths and signatures you cited actually exist, and the silent-failure-hunter asks the uncomfortable "what could possibly go wrong" questions.

Usage: `/plan-review`

### special-plan

The full department-head experience. Claude drafts an execution-grade implementation plan, then automatically drives a multi-pass review loop alternating its in-house cohort with **Codex** as the cross-vendor second opinion. Each pass picks the right reviewer for the moment, applies judgment to the findings, edits the plan in place, and stops when both lenses have independently converged. Built for plans you intend to hand to Codex (or another AI implementer) with zero hand-waving.

Requires the `codex` CLI on PATH and an `AGENTS.md` in the project root for Codex passes. Without them, the skill degrades gracefully to a cohort-only loop.

Invoke explicitly: `use special-plan` (it deliberately won't trigger on generic "make me a plan" requests — this is a specialized tool, not a default).

#### Dangerous

For complete autonomy, run Claude Code with the `--dangerously-skip-permissions` argument. Otherwise, Codex invocations must be manually approved.

### codex-review

The bare Codex lens. One invocation, one Codex pass, against any artifact you name — a plan file, a diff, a PR, a specific set of files. No cohort, no orchestration, no loop. On re-invocation against the same artifact, the Codex thread is resumed rather than restarted, so context and prompt caching survive. For when you want the cross-vendor second opinion without booking the whole department meeting.

Requires the `codex` CLI on PATH, `jq`, and an `AGENTS.md` in the project root.

Invoke explicitly: `use codex-review` (intentionally inert to generic "what does Codex think" phrasing — this is a named tool, not a default).

## Dependencies

All skills require these Claude Code plugins:

```
claude plugins install pr-review-toolkit
claude plugins install feature-dev
```

`special-plan`, `special-review`, and `codex-review` additionally need:

- `codex` CLI on PATH (for Codex passes)
- `jq` on PATH (for parsing Codex JSON output)
- `AGENTS.md` in the project root of any repo you target (Codex cold-starts a fresh subprocess each invocation and orients by reading `AGENTS.md` from its working directory — without it, the review is uninformed)

## Installation

Add skills from this repo to your Claude Code configuration:

```
claude skills add /path/to/full-review
claude skills add /path/to/plan-review
claude skills add /path/to/special-plan
claude skills add /path/to/special-review
claude skills add /path/to/codex-review
```
