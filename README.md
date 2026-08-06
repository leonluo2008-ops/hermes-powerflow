# Hermes Powerflow

> Spec-first development pipeline for [Hermes Agent](https://hermes-agent.nousresearch.com). Evolved from [obra/superpowers](https://github.com/obra/superpowers), heavily customized for real-world Hermes workflows.

## What it is

A mandatory multi-phase pipeline for complex tasks — not suggestions, but a structured workflow that prevents guesswork and enforces evidence-based execution.

```
Idea → Brainstorm → Plan → Subagent-Driven Build (TDD) → Code Review → Finish Branch
```

## Key features (beyond upstream superpowers)

- **Fact Detection (Step 0)** — Fetch authoritative docs + probe local environment (ports, deps, OS) before writing any code
- **CBM-First** — Code exploration defaults to [Codebase Memory](https://github.com/nousresearch/cbm) (`search_graph` / `get_architecture`) instead of raw grep
- **CBM Index Refresh** — Automatically refresh code graph after merge/push when code files changed
- **MoA Advisory** — Recommends Mixture-of-Agents mode for complex tasks (multi-model brainstorm + adversarial review)
- **Adversarial Review Gate** — Multi-round plan review until "pass" — not just one round
- **User-Capability-Aware Execution** — Progress reports every step, no silent skips, no process-level interruptions

## Pipeline phases

| Phase | What happens |
|-------|-------------|
| **1. Brainstorm** | Explore context (CBM + skills_list) → ask clarifying questions → propose approaches → design doc |
| **2. Plan** | Task-by-task implementation plan, each 2-5 min, TDD mandatory |
| **3. Build** | Subagent-driven: implementer → spec-reviewer → quality-reviewer per task |
| **4. Debug** | Root cause investigation → pattern analysis → hypothesis testing → fix at root |
| **5. Finish** | Verify tests → merge/push → CBM index refresh → cleanup |

## Usage

This skill is loaded automatically by Hermes Agent when trigger words are detected:

- `powerflow` / `powerflow 流程`
- `superpowers 流程` (backward compat)

## Structure

```
SKILL.md                          # Main pipeline definition
references/
├── brainstorming.md              # Phase 1 deep dive (Step 0 fact detection + Step 1 CBM-first)
├── writing-plans.md              # Phase 2 deep dive
├── subagent-development.md       # Phase 3 deep dive
├── tdd.md                        # TDD methodology
├── systematic-debugging.md       # Phase 4 deep dive
├── debug-toolkit.md              # Real debuggers (pydbg, node inspect)
├── finishing-branch.md           # Phase 5 deep dive (CBM index refresh)
├── pre-commit-review.md          # Code review before merge
├── user-capability-aware-execution.md  # Progress reporting + acceptance checklist
├── user-context.md               # User-specific context
├── repository-cohesion.md        # Single-repo multi-entry architecture
└── simplify.md                   # Post-merge cleanup
```

## Installation

This skill lives at `~/.hermes/skills/superpowers/` (directory name kept for compatibility).

```bash
git clone https://gitee.com/luo-xiansen2023/hermes-powerflow.git ~/.hermes/skills/superpowers
```

## Version

**v1.0.0** — Initial release as independent project.

## License

MIT
