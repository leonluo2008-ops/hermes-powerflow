---
name: hermes-powerflow
description: "Hermes powerflow pipeline — spec-first dev: brainstorm, plan, build, review, finish. 触发词: powerflow, powerflow 流程, 开启 powerflow, superpowers 流程, superpowers."
---

# Hermes Powerflow

> Evolved from [obra/superpowers](https://github.com/obra/superpowers). Heavily customized for Hermes Agent. Mandatory workflow — not suggestions.

## The Pipeline

```
Idea → Brainstorm → Plan → Subagent-Driven Build (TDD) → Code Review → Finish Branch
```

Every coding task follows this pipeline. "Too simple to need a design" is always wrong.

### ⭐ MoA 建议（2026-08-06 新增）

powerflow 处理的是复杂任务，单模型在 brainstorm / 对抗审查 / code review 阶段容易出盲区。**强烈建议用户在启动 powerflow 流程前用 MoA 模式**：

- 一次性：发 `/moa` 跑单个 prompt
- 持久：发 `/model my-council --session` 切到 MoA preset

如果用户已经在 MoA 模式，各阶段天然受益于多模型聚合——brainstorm 多视角、对抗审查跨模型。

> **注意**：Agent 运行时无法程序化切换 MoA——这是用户侧决策（gateway 层 slash command），不是 agent 能自动触发的。

---

## Phase 1: Brainstorming

**Trigger:** User wants to build something. Activate before touching any code.

**See:** [references/brainstorming.md](references/brainstorming.md)

**Summary:**
0. **事实探测** — 权威文档（0a）+ 本地环境探测（0b），避免凭印象猜测
1. Explore project context — **CBM 优先**（search_graph/get_architecture），fallback 到 grep
2. Ask clarifying questions — **one at a time**, prefer multiple choice
3. Propose 2–3 approaches with trade-offs + recommendation
4. Present design in sections, get approval after each
5. Write design doc → `docs/plans/YYYY-MM-DD-<topic>-design.md` → commit
6. Hand off to **Phase 2: Writing Plans**

**HARD GATE:** Do NOT write any code until user approves design.

### ⭐ 对抗审查必须多轮直到"通过"（2026-07-24 用户强调）

**用户原话**："再次对抗审查。记住，你修复后要提交对抗审查，不是你说修复就修复了"

**反模式**：第一轮审查发现 3 个问题，自己 patch 修复后直接说"修完了，可以开始执行"——**没有提交第二轮审查验证修复是否真正落地**。

**铁律**：
1. **Plan 修正后必须重新提交对抗审查**——不能口头说"已修复"就进入执行阶段
2. **审查轮数不设上限**——直到审查结果为"通过，可以开始执行"才结束
3. **每轮审查的重点不同**：
   - 第 1 轮：验证方案逻辑是否成立（致命断点、严重遗漏）
   - 第 2 轮：验证修正是否真正落地 + 是否引入新问题
   - 第 3 轮（终审）：用真实数据 dry-run 代码 + 验证注入点行号匹配 + 整体一致性
4. **审查子 agent 必须用真实数据验证**——不能只读 plan 文本就下结论，必须读项目实际 JSON / 代码文件 / 行号做交叉确认
5. **代码实现后的验证也不能信子 agent 自报**——主 agent 必须独立回读关键改动点（grep 函数定义 + 行号）+ 独立跑端到端验证

**实战数据**（2026-07-24 P5+P6）：3 轮审查分别发现 3+3+0 个问题（第 3 轮通过），核心映射链路用真实数据 dry-run 验证才确认跑通。

### ⚠ 用户 mid-session 更新硬约束时立即调整方案（2026-07-30 实战）

**场景**：方案刚定完（单文件离线 HTML），对抗审查也过了，实现做到一半时用户说"单文件离线约束已经过期了，可以打包发送，可以联网"。

**反模式**：继续按旧约束实现，或者假装没听到把新约束塞到"未来 v5.1"。

**铁律**：
1. **`[OUT-OF-BAND USER MESSAGE]` 是高优先级指令**——和原始请求同权威，不是建议
2. **立即评估影响面**——哪些已完成的代码需要改？哪些设计决策需要推翻？
3. **先完成当前操作的验证**（确保不 break），再按新约束调整
4. **不要重头再来**——已完成的工作（如模板抽取）通常在新约束下仍有价值，只改架构方向
5. **方案变更要跟用户确认一句**——"按新约束调整：不再 inline JS/CSS，改用外部引用"，然后直接干

**实战数据**（2026-07-30 course-builder v5.0）：用户取消单文件约束后，delivery 从 inline 模式改为 `--multi` 多文件模式（index.html + assets/），HTML 从 26MB 缩到 76KB。之前完成的 JS/CSS 模板抽取工作直接复用（只是从 `read_text()` 内联改为 `<script src>` 外部引用）。

**反例**（seedance 本地 cache 清零 · 实战）：写完 335 行 plan（12 Task、独立 SOP、拍板项）才事后发现 `agent-debugging-methodology` v1.2.0 已有：
- 铁律 7（先查官方文档再决定是否造本地机制）
- 完整 `references/seedance-cache-removal-2026-06-13.md` case study（5 步 SOP + 自检问题）
- 5 步流程：查官方 → 列影响面 → 拍板项 → 执行 → grep 验证

**铁律**：
- **Phase 1 Brainstorm 末尾** 必先 `skills_list`（按 category 扫） → `skill_view` 每个 class-level umbrella
- 查到的相关 skill → plan 段首引用 "本任务在 `X skill` 已有铁律 `Y` 基础上补充 `Z`"，**不**重复造轮子
- 查不到相关 → plan 段首加 "本任务在 `X` 领域未发现现有方法论，本 plan 创新方法论如下"
- **删除/清退类任务**额外必查 `agent-debugging-methodology` —— 几乎肯定有相关铁律（铁律 6/6b/7 涵盖历史报告修订、官方文档优先、本地机制清零）

**Plan 漏项自检**（写 plan 后、commit 拍板前问自己）：
1. plan 涉及的领域有没有现成 class-level skill？`skills_list` 列了吗？
2. plan 是不是在重新发明已有方法论？（如 "先查官方" → 铁律 7 已写）
3. plan 涉及**删除 / 清零 / 修订**时，是不是在重写一个已有 case study？
4. **plan 超过 100 行还没引用过任何 skill → 停下来去查 skills** —— 大概率有遗漏

详见 `references/writing-plans.md` §"Plan 阶段必先 `skills_list` 查已有方法论"扩展节 + `agent-debugging-methodology` 铁律 8。

---

## Phase 2: Writing Plans

**Trigger:** Design approved. Activated by brainstorming phase.

**See:** [references/writing-plans.md](references/writing-plans.md)

**Summary:**
- Write a detailed task-by-task implementation plan
- Each task = 2–5 minutes: write test → watch fail → implement → watch pass → commit
- Save to `docs/plans/YYYY-MM-DD-<feature>.md`
- Announce: `"I'm using the writing-plans skill to create the implementation plan."`
- After saving, offer two execution modes:
  - **Subagent-driven (current session):** `sessions_spawn` per task + two-stage review
  - **Manual execution:** User runs tasks themselves

---

## Phase 3: Subagent-Driven Development

**Trigger:** Plan exists, user chooses subagent-driven execution.

**See:** [references/subagent-development.md](references/subagent-development.md)

**Per-task loop (OpenClaw):**
1. `sessions_spawn` an implementer subagent with task + full plan context
2. Wait for completion announcement
3. `sessions_spawn` a spec-reviewer subagent → must confirm code matches spec
4. `sessions_spawn` a code-quality reviewer subagent → must approve quality
5. Fix any issues, re-review if needed
6. Mark task done, move to next
7. Final: dispatch overall code reviewer → hand off to Phase 5

**TDD is mandatory in every task.** See [references/tdd.md](references/tdd.md).

### ⭐ Subagent 接到 task 的第一条指令（2026-06-13 补充）

主 agent 派 task 给 subagent 时，**第一条指令**应该含：
> "先 `skills_list` + `skill_view` 查 `agent-debugging-methodology` + 任务相关 skill，避免重蹈主 agent 漏查覆辙"

避免 subagent 在隔离 context 里**重蹈主 agent 已经踩过的坑**（plan 漏查、铁律 6 违反等）。

---

## Phase 4: Systematic Debugging

**Trigger:** Bug, test failure, unexpected behaviour — any technical issue.

**See:** [references/systematic-debugging.md](references/systematic-debugging.md)

**HARD GATE:** No fixes without root cause investigation first.

**Four phases:**
1. Root Cause Investigation (read errors, reproduce, check recent changes, trace data flow)
2. Pattern Analysis (find working examples, compare, identify differences)
3. Hypothesis + Testing (one hypothesis at a time, test to prove/disprove)
4. Fix + Verification (fix at root, not symptom; verify fix doesn't break anything)

**⚠️ 涉及"黑盒 API / 跨进程 / 跨服务"行为的判断时**，**额外必查** `agent-debugging-methodology` 铁律 1（先实测再下结论）+ 铁律 2（根因可能在中间层）+ 铁律 6（动手前查既有实战沉淀）。

---

## Phase 5: Finishing a Branch

**Trigger:** All tasks complete, all tests pass.

**See:** [references/finishing-branch.md](references/finishing-branch.md)

**Summary:**
1. Verify all tests pass
2. Determine base branch
3. Present 4 options: merge locally / push + PR / keep / discard
4. Execute choice
5. **CBM 索引刷新**（仅 merge/push 时，改了代码仓才触发）
6. Clean up

---

## OpenClaw Subagent Dispatch Pattern

When dispatching implementer or reviewer subagents, use `sessions_spawn`:

```
Goal: [one sentence]
Context: [why it matters, which plan file]
Files: [exact paths]
Constraints: [what NOT to do — no scope creep, TDD only]
Verify: [how to confirm success — tests pass, specific command]
Task text: [paste full task from plan]
```

Run `sessions_spawn` with the task as a detailed prompt. The sub-agent announces results automatically.

---

## Key Principles

- **One question at a time** during brainstorm
- **TDD always** — write failing test first, delete code written before tests
- **YAGNI** — remove unnecessary features from all designs
- **DRY** — no duplication
- **Systematic over ad-hoc** — follow the process especially under time pressure
- **Evidence over claims** — verify before declaring success
- **Frequent commits** — after each green test

---

## Reference Map

Phase-specific deep dives live in `references/`. The seven sibling skills that used to be standalone are archived under `~/.hermes/skills/.archive/superpowers-*/` and merged into the matching reference below.

| Phase | Reference | Archived sibling (recoverable) |
|---|---|---|
| 1 — Brainstorming | `references/brainstorming.md` | — |
| 1 — Spike / idea exploration | `references/writing-plans.md` §Spike (pre-plan validation) | `superpowers-spike/` |
| 2 — Writing plans | `references/writing-plans.md` | `superpowers-writing-plans/` |
| 3 — TDD (mandatory in every task) | `references/tdd.md` | `superpowers-test-driven-development/` |
| 3 — Subagent-driven implementation | `references/subagent-development.md` | `superpowers-subagent-driven-development/` |
| 4 — Systematic debugging | `references/systematic-debugging.md` | `superpowers-systematic-debugging/` |
| 4 — Debug toolkits (real debuggers) | `references/debug-toolkit.md` | `superpowers-debug-toolkit-{python-debugpy,node-inspect-debugger,debugging-hermes-tui-commands}/` |
| 5 — Finishing a branch | `references/finishing-branch.md` | — |
| (separate) — Pre-commit review | `references/pre-commit-review.md` | `superpowers-requesting-code-review/` |
| (separate) — After-the-fact cleanup | `references/simplify.md` | `superpowers-simplify-code/` |
| (UX gating) — Plan mode | `references/writing-plans.md` §Discussion gate | `software-development/plan/` *(protected, kept standalone)* |

**Why this map exists**: before this consolidation (2026-06-23), the seven archived siblings were each their own SKILL.md with overlapping content. They're now labeled subsections of the same pipeline — single source of truth, with the originals recoverable from `.archive/` if any of them need to be revived as standalone again.

## `plan` skill is intentionally NOT archived

The protected `plan` skill (`software-development/plan/`) covers Plan-Mode-UX — the "user said `/plan`, write markdown plan to `.hermes/plans/`, no execution" workflow. It is *adjacent* to but distinct from this umbrella's Phase 2 "writing-plans". Keep them separate: `plan` is a slash-command entry point, this umbrella's Phase 2 is a brainstorm-to-execution handoff.

## User-Capability-Aware Execution（2026-06-01 用户信号）

> **用户原话**："我其实没法 review，我也看不太懂，我也只能看实际效果"

### ⛔ 进度汇报铁律（2026-08-03 用户强烈反馈 · 零容忍）

> **用户原话**："你又不向我确认，又不向我汇报，又不显示进度，你以前都不会像这样工作的" + "我都会把你卸载掉了"

**"不打断执行流" ≠ "不汇报进度"**。这两件事是独立的：

| | 问用户 | 汇报给用户 |
|---|---|---|
| 方向选择（选 A/B） | ✅ 问（拍板项模式） | — |
| 过程细节（commit 粒度） | ❌ 自动决定 | — |
| **每步完成** | — | ✅ **必须报**（≤3 行） |
| **长耗时操作（>2 min）** | — | ✅ **必须定期报**（每 3-5 min） |
| **阻塞/超时/方案偏离** | ✅ **必须打断**告知 | — |

**长耗时操作汇报规则**（镜像拉取、构建、大文件下载等）：
1. 启动时说"正在做 X，预计 Y 分钟"
2. 每 3-5 分钟主动查进度并报："已下载 XX MB / 共 YY MB"
3. **不等用户问**——用户问了 = 你已经汇报晚了
4. 阻塞/EOF/超时立刻报，不自己换方案跳过

**汇报模板**（≤3 行）：
```
做了什么：<1 句>
当前状态：<数字/百分比/具体结果>
下一步：<1 句>
```

详见 `references/user-capability-aware-execution.md` §进度汇报铁律。

**翻译成 superpowers 流程的调整**：

| 阶段 | 调整 |
|------|------|
| **Phase 1 Brainstorm** | 仍按"一次问一个问题"——用户**愿意**参与方向选择 |
| **Phase 2 Plan 写完** | **不**问 "Subagent 还是 Manual"——**直接 Subagent**（或直接干，看任务性质） |
| **Phase 3 Task 执行中** | **不**问"这一步要 X 还是 Y"——**按 superpowers 原则自行决定** |
| **Phase 3 每个 task 完成** | **不**问用户"这一步 OK 吗"——**自我验证**后直接报结果（≤3 行汇报：做了什么+状态+下一步，详见 `references/user-capability-aware-execution.md` §进度汇报铁律） |
| **Phase 5 Finish** | **不**问 4 options——**默认 Subagent-Driven + push 后用户自己验** |
| **最后** | 给一份**用户验收清单**（具体命令 + 预期输出）——用户自己跑 |

**判断标准**：用户"会回答方向选择"（Q1/Q2 这种），但**不**会回答"这一步要不要 commit"——后者属于过程性细节，**自动决定**。

**不适用**：纯对话问答 / 闲聊 / 一次性查询（这种**不**进 superpowers 流程）。

### 拍板项模式（2026-06-05 验证）

**Plan 写完，列出 N 个 yes/no 拍板项让用户一次性确认**——比"我开始执行了哈" + 边做边问更高效。

**模板**:
```markdown
## 待拍板（N 项 yes/no）
1. 整体方案 OK 吗？
2. <路径 1> OK 吗？
3. <路径 2> OK 吗？
```

**用户回复模式**:
- 答 `1` = 全部 yes → 直接开干
- 答 `2 1 1` = 第 1 个 no，其他 yes → 微调后开干
- 长答 = 提新约束 → 重新写 plan 拍板项

**为什么有效**:
- 用户**不**愿意在执行中被打断（"我其实没法 review"）——但**愿意**在写完后一次性 ack 方向
- 拍板项是**方向决策**（不是过程性细节）——正好命中用户的"会答 Q1/Q2，不会答 X/Y"判断标准
- 0 中断执行 = 用户体验显著提升（实测一个 PR 全程 0 打断）

**错误做法**:
- ❌ "Plan 完成，你要继续吗？"（用户被迫 yes/no 一次就打断）
- ❌ 写完 plan 不列拍板项直接执行（用户失去方向掌控感）
- ❌ 拍板项 > 5 个（用户会烦躁，只关心 top 3）

### ⭐ 拍板项 = "方向决策 · 含多仓/多仓影响面"（2026-06-13 补充）

**反例**（seedance cache 清零 · 实战）：plan 拍板项只列了"种子仓 5 个改动点"——用户额外补拍板"绘本仓一起改"+"用户纠错链路段删除"，迫使 plan 二次扩展。

**铁律**：
- 拍板项**必须包含"多仓影响面"段**——任何引用 X 机制的仓 / 项目 / 文档都要列出来单独问
- 拍板项**必须包含"修订深度"段**——"删错误事实" vs "改写整段" vs "保留原文 + 加 PATCH 注释"是 3 个不同拍板方向
- 拍板项**写到 ≥ 5 个**时**主动收敛**——把"修订深度" 3 个选项缩成 2 个（"删 vs 改写"二选一）让用户容易答

**自检问题**（列拍板项时问自己）：
1. 涉及的仓 / 项目 / 文档**全部列出来**了吗？有没有跨仓引用？
2. 每个拍板项的"删 / 改 / 留"方向**有明确二元选择**吗？还是含糊的"调整一下"？
3. 拍板项里有没有**"修订深度"**这个维度？

---

## Applicable Beyond Coding

This workflow applies to **any complex multi-step task**, not just software development:

| Domain | Brainstorm | Plan | Execution | Review |
|--------|-----------|------|-----------|--------|
| **Content creation** | 目标观众是谁？核心冲突？ | 剧本→分镜→生成→发布 | 子代理各环节执行 | 成品检视 |
| **Video production** | 故事线/风格/参考 | 分镜任务分解 | 分图生成→视频生成 | 整体质量 |
| **Business planning** | 真正要解决什么？限制条件？ | 里程碑→任务 | 执行并review | 成果验收 |
| **Workflow design** | 当前流程的问题？目标状态？ | 步骤拆解 | 实现各环节 | 验证有效性 |

**Trigger phrases for non-coding use:** "我想做一个XXX"、"我们要做XXX内容"、"帮我规划一下这个项目"、"这个工作流有问题"

**Key difference from coding:** TDD and git commits are coding-specific. For content/production, replace with:
- Write/produce a draft → review against spec → revise → final approval
- No test commands needed, but the *spirit* of TDD applies: define what "done" looks like before producing
## See also:

- [references/user-context.md](references/user-context.md) — user-specific mapping for Douyin content creation and user preferences
- [references/user-capability-aware-execution.md](references/user-capability-aware-execution.md) — 详细模式：当用户「没法 review / 只能看效果」时，每个 task 完成后的自验清单 + 用户验收清单模板 + 错误处理
- [references/repository-cohesion.md](references/repository-cohesion.md) — Repository 架构口径一致性（**单仓库多入口共享单真源**）—— 设计仓库 / 写 README / 决策 spike 转正时必读
- **`agent-debugging-methodology` v1.2.0** —— 8 条排错铁律（含铁律 8 "plan 阶段必先 skills_list"）。任何"X 行为异常 / X 没渲染 / X 自造机制是否必要 / X 怎么清零"类问题，**先读这个 skill**（含 4 个 case study）

## Repository 架构口径（2026-06-11 拍板）

> 仓库是**一个工具集**，不是"多个并列入口"。

**判断标准**：MCP / shared lib / spike 是 **CLI 的工具子集**（不是"3 个并列产品"），全部在**一个仓库**管理。

**写 README / 决策 spike 转正时**必读：[references/repository-cohesion.md](references/repository-cohesion.md)

## Pitfall: 新 Web 服务不复用已有端口（2026-07-28 实战）

设计新 Web 服务时"复用已有 skill 的默认端口"会导致同时运行时冲突。

**实战**：course-builder Web Console 设计端口 8765（web-reader-local-panel 默认端口）。决策专员指出两个服务会同时运行，第二个启动时 `Address already in use`。改为 8766。

**铁律**：新 Web 服务不复用已有服务的默认端口，推荐连号（8765→8766）。决策专员审查时必查端口冲突。
