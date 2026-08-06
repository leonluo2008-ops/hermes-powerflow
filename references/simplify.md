# Simplify Code — Parallel 3-Agent Review

> **Archived sibling**: `~/.hermes/skills/.archive/superpowers-simplify-code/` (full SKILL.md, 175 lines, recoverable).
>
> This reference summarizes the parallel 3-agent review pattern. Load the archived sibling for the full delegate_task prompts.

## When to use

Trigger this when the user says any of:

- "simplify" / "simplify my changes" / "simplify these changes"
- "review my code" / "review my recent changes" / "clean up my changes"
- "/simplify" (carrying the Claude Code habit)

Optional modifiers:
- **Focus**: `reuse` / `quality` / `efficiency` — runs only that reviewer
- **Dry run**: "just report" / "don't change anything" — run reviewers, present findings, apply NOTHING
- **Scope**: "the last commit" / "staged changes" / "src/foo.py" — narrow diff source

**Don't auto-run after every edit** — costs three subagents' worth of tokens.

## Core principle

Three narrow reviewers beat one broad reviewer. Each deeply searches the codebase for a single class of problem (reuse, quality, efficiency) without diluting attention. They run **concurrently** in batch mode — you pay the latency of one review, not three.

## Process

### Phase 1 — Identify changes

Default order:
1. `git diff` (working tree, tracked)
2. `git diff HEAD` (includes staged)
3. `git diff --staged` / `git diff HEAD~1` / `git diff main...HEAD` / `git diff -- src/foo.py` (scoped)

Capture the full diff text. If very large (>2000 changed lines), warn the user that three subagents each carrying the full diff is token-heavy; offer to scope down per-directory/per-commit.

### Phase 2 — Three reviewers in parallel

Use `delegate_task` **batch mode** — pass all three tasks in one `tasks` array so they run concurrently. Each gets the **complete diff** + absolute repo path + `terminal`/`file`/`search` toolsets.

**Reviewer 1 — Code Reuse**:
> Review the diff for code that duplicates functionality already in the codebase. Search utility modules, shared helpers, adjacent files. Flag new functions that duplicate existing ones; hand-rolled logic that an existing utility already does (string/path manipulation, custom env checks, ad-hoc type guards, re-implemented parsing). For each, name the existing thing and where it lives.

**Reviewer 2 — Code Quality**:
> Review the diff for quality problems: redundant state, parameter sprawl, copy-paste-with-variation, leaky abstractions, stringly-typed code where a constant/enum/registry exists. For each, give the concrete refactor.

**Reviewer 3 — Efficiency**:
> Review the diff for efficiency problems: unnecessary work (redundant computation, repeated file reads, N+1 access patterns), missed concurrency, hot-path bloat, TOCTOU anti-patterns, memory issues, overly broad reads. For each, give the concrete fix and why it's faster or lighter.

### Phase 3 — Aggregate and apply

1. **Merge** findings into one list, dedupe overlaps.
2. **Discard false positives** — you have the most context, drop weak/wrong findings silently.
3. **Resolve conflicts** (Reviewer 1 "use existing util X" vs Reviewer 3 "X is slow, inline it"). Default order: **correctness > user's stated focus > readability/reuse > micro-perf**.
4. **Apply** the surviving fixes with `patch`/`write_file` (unless dry-run).
5. **Verify** by running the project's targeted tests for touched files + linter/type check.
6. **Summarize** applied fixes grouped by reviewer category + any skipped findings with reasons.

## Pitfalls

- **Don't fan out wider than ~3.** More reviewers = more cost + more conflicts to reconcile.
- **Give the WHOLE diff to each reviewer.** Splitting defeats the design — cross-file duplication and N+1s only show up with the full picture.
- **Reviewers search, they don't guess.** Require `file:line` evidence; drop findings that lack it.
- **Apply ≠ rewrite.** Cleanup of recent changes, not a license to refactor the whole module.
- **Respect project conventions.** Fold `AGENTS.md` / `CLAUDE.md` / `HERMES.md` rules into reviewer prompts.
- **Large diffs blow context.** Scope down before delegating — three subagents each carrying a 5000-line diff is expensive and may truncate.

## See also

- **Pre-Commit Review pipeline** (`references/pre-commit-review.md`) — for *before-commit* verification (yours, single reviewer)
- **Phase 4 (Systematic Debugging)** — for *fixing* issues found during cleanup
- **Archived full skill**: `superpowers-simplify-code/` for complete delegate_task prompts
