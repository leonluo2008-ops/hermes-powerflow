# Pre-Commit Review Pipeline

> **Archived sibling**: `~/.hermes/skills/.archive/superpowers-requesting-code-review/` (full SKILL.md, 280 lines, recoverable).
>
> This reference summarizes the canonical 8-step pre-commit verification pipeline. Load the archived sibling directly if you need the full agent-prompt templates and Python pattern catalog.

## When to run

After implementing a feature or bug fix, before `git commit` or `git push`. Triggers:

- User says "commit", "push", "ship", "done", "verify", or "review before merge"
- After completing a task with 2+ file edits in a git repo
- After each task in `superpowers` Phase 3 (Subagent-Driven Development) — the two-stage review

**Skip for**: documentation-only changes, pure config tweaks, when user says "skip verification".

**Skill vs `github-code-review`**: this skill verifies YOUR changes before committing. `github-code-review` reviews OTHER people's PRs on GitHub with inline comments.

## The 8-step pipeline

### Step 1 — Get the diff

```bash
git diff --cached
# Empty? Try git diff, then git diff HEAD~1 HEAD.
# Still empty? git status — nothing to verify.
# > 15k chars? git diff --name-only, then per-file review.
```

### Step 2 — Static security scan (scan added lines only)

```bash
git diff --cached | grep "^+" | grep -iE "(api_key|secret|password|token|passwd)\s*=\s*['\"][^'\"]+['\"]"
git diff --cached | grep "^+" | grep -E "os\.system\(|subprocess.*shell=True"
git diff --cached | grep "^+" | grep -E "\beval\(|\bexec\("
git diff --cached | grep "^+" | grep -E "pickle\.loads?\("
git diff --cached | grep "^+" | grep -E "execute\(f\"|\.format\(.*SELECT|\.format\(.*INSERT"
```

### Step 3 — Baseline tests + linting

Detect project language, run the right tools, capture `baseline_failures` (stash → run → pop → restore). Only NEW failures from your diff block the commit.

| Language | Test | Lint |
|---|---|---|
| Python | `python -m pytest --tb=no -q 2>&1 \| tail -5` | `ruff check .` / `mypy . --ignore-missing-imports` |
| Node | `npm test -- --passWithNoTests` | `npx eslint .` / `npx tsc --noEmit` |
| Rust | `cargo test` | `cargo clippy -- -D warnings` |
| Go | `go test ./...` | `go vet ./...` |

### Step 4 — Self-review checklist

- No hardcoded secrets / API keys / credentials
- Input validation on user-provided data
- SQL queries parameterized
- File operations validate paths (no traversal)
- External calls have error handling (try/catch)
- No debug print/console.log left behind
- No commented-out code
- New code has tests

### Step 5 — Independent reviewer subagent (delegate_task)

The reviewer gets ONLY the diff + static-scan results. **No shared context** with the implementer. Fail-closed: unparseable response = fail.

Pass this JSON contract (full prompt in archived sibling):

```json
{
  "passed": true|false,
  "security_concerns": [],
  "logic_errors": [],
  "suggestions": [],
  "summary": "one sentence verdict"
}
```

**Auto-FAIL rules**: security_concerns non-empty → passed=false. logic_errors non-empty → passed=false. Cannot parse diff → passed=false.

### Step 6 — Evaluate results

Combine Steps 2 + 3 + 5. Any failure → Step 7.

### Step 7 — Auto-fix loop (max 2 cycles)

Spawn a THIRD agent (not implementer, not reviewer) that fixes ONLY the reported issues. Re-run Steps 1–6. If still failing after 2 cycles, escalate to user with `git stash` or `git reset` suggestions.

### Step 8 — Commit

```bash
git add -A && git commit -m "[verified] <description>"
```

The `[verified]` prefix tells reviewers an independent subagent approved this change.

## Pitfalls

- **Empty diff** — `git status`, nothing to verify
- **Not a git repo** — skip and tell user
- **Large diff (>15k chars)** — split per-file, review each
- **delegate_task returns non-JSON** — retry once with stricter prompt, else treat as FAIL
- **Auto-fix introduces new issues** — counts as new failure, cycle continues

## See also

- **Phase 4 (Systematic Debugging)** — methodology for fixing the issues this pipeline finds
- **Phase 5 (Finishing a Branch)** — what to do AFTER verification passes
- **Archived full skill**: `superpowers-requesting-code-review/` for the complete delegate_task prompt template and language-specific anti-patterns
