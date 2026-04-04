# ValyKay's Claude Code Skills

Two Claude Code skills for automated code and plan review using specialized agents.

## Skills

### full-review

Run all five code review agents in parallel on your unstaged git diff. Agents cover style, bugs, error handling, type design, and comment quality. Supports iteration caps so repeated runs retire converged agents automatically.

Usage: `/full-review`

### plan-review

Run three specialized agents in parallel to audit an implementation plan before you start coding. Agents verify architecture, codebase assumptions, and error handling coverage.

Usage: `/plan-review`

## Dependencies

Both skills require these Claude Code plugins:

```
claude plugins install pr-review-toolkit
claude plugins install feature-dev
```

## Installation

Add skills from this repo to your Claude Code configuration:

```
claude skills add /path/to/full-review
claude skills add /path/to/plan-review
```
