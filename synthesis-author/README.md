# synthesis-author

Single-agent plugin. Consumes a code diff or implementation plan plus N notes files of flagged issues, verifies each claim against the actual code, and produces a per-finding implementation spec at a caller-specified path.

## What it is

One subagent: `synthesis-author` (Opus). Tools: `Read`, `Grep`, `Glob`, `Bash`, `Write`.

The agent's system prompt is **operational**, not aspirational. It does not contain identity flattery ("you are a world-class expert with decades of experience"). It does contain hard requirements: verify every cited file:line before asserting; reserve the Uncertainty field only for what tools cannot resolve; pick must be implementable; dismiss explicitly when a flag does not survive verification.

The system prompt is also **blind to the workflow shape** that dispatches it. The agent does not know whether its output goes straight to implementation, gets critiqued by another reviewer, or sits on disk for human review. From the agent's perspective the spec is the final deliverable. This is a deliberate countermeasure against the "I'll let the next reviewer catch it" failure mode where an orchestrator that knows about a downstream critic produces weaker first-pass synthesis work.

## Intended invokers

- `debate-review` skill (synthesis phase on a code diff).
- `debate-plan` skill (synthesis phase on an implementation plan).
- Any other skill or workflow that has collected reviewer notes and needs them resolved into a per-finding spec.

Invokers are responsible for assembling the inputs (notes files on disk, artifact reference, output path) and for whatever they do with the resulting spec. The agent owns only the verification + resolution + write step.

## Not for

- Primary code review (use a `code-reviewer` agent or equivalent — this agent consumes flags, it does not generate them).
- Open-ended exploration (use the `Explore` agent).
- Implementing the spec (the spec is the agent's output; the implementer is a separate dispatch).

## Usage

The agent is dispatched via Claude Code's `Agent` tool, with `subagent_type: synthesis-author`. The dispatch prompt should provide:

1. The path to the artifact under review (diff reference or plan file).
2. Paths to one or more notes files of flagged issues.
3. The output path where the spec should be written.
4. The repository working directory if not the current one.

The agent expects to find the inputs on disk and writes the spec on disk. Nothing structured needs to be passed inline.

## Installation

Symlinked from this repo into `~/.claude/plugins/synthesis-author`. The repo is the canonical source; the symlink makes it available at user level so any session can dispatch the agent without further setup.
