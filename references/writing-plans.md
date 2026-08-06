# Writing Plans Reference

Source: obra/superpowers writing-plans skill

## Overview

Write a detailed implementation plan before touching code. Each task must be granular enough
for an agent with no project context to execute correctly.

Announce at start: "I'm using the writing-plans skill to create the implementation plan."

## Plan File Location

`docs/plans/YYYY-MM-DD-<feature-name>.md`

## Plan Document Header (required)

```markdown
# [Feature Name] Implementation Plan

> **For implementer:** Use TDD throughout. Write failing test first. Watch it fail. Then implement.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2–3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## Task Granularity

Each step = one action, 2–5 minutes:

```markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py`
- Test: `tests/exact/path/to/test_file.py`

**Step 1: Write the failing test**
[paste exact test code]

**Step 2: Run test — confirm it fails**
Command: `pytest tests/path/test.py::test_name -v`
Expected: FAIL — "function not defined" or similar

**Step 3: Write minimal implementation**
[paste exact implementation code]

**Step 4: Run test — confirm it passes**
Command: `pytest tests/path/test.py::test_name -v`
Expected: PASS

**Step 5: Commit**
`git add <files> && git commit -m "feat: <description>"`
```

## Rules

- Exact file paths always — no vague references
- Complete code in plan — not "add validation here"
- Exact commands with expected output
- DRY, YAGNI, TDD, frequent commits after each green test

## Execution Handoff

After saving plan, offer:

> "Plan saved to `docs/plans/<filename>.md`. Two execution options:
>
> 1. **Subagent-Driven** — I dispatch a fresh sub-agent per task, review between tasks
> 2. **Manual** — You run the tasks yourself
>
> Which approach?"

If Subagent-Driven: proceed to subagent-development phase.

---

## ⭐ Plan 修正后必须重新提交对抗审查（2026-07-24 用户教训）

**用户原话**："再次对抗审查。记住，你修复后要提交对抗审查，不是你说修复就修复了"

**反模式**：第一轮审查发现 3 个问题，自己 patch 修复后直接说"修完了，可以开始执行"——**没有提交第二轮审查验证修复是否真正落地**。

**铁律**：
1. **Plan 修正后必须重新提交对抗审查**——不能口头说"已修复"就进入执行阶段
2. **审查轮数不设上限**——直到审查结果为"通过，可以开始执行"才结束
3. **每轮审查的重点不同**：
   - 第 1 轮：验证方案逻辑是否成立（致命断点、严重遗漏）
   - 第 2 轮：验证修正是否真正落地 + 是否引入新问题（如死引用、签名遗漏）
   - 第 3 轮（终审）：用真实数据 dry-run 代码 + 验证注入点行号匹配 + 整体一致性
4. **审查子 agent 必须用真实数据验证**——不能只读 plan 文本就下结论，必须读项目实际 JSON / 代码文件 / 行号做交叉确认
5. **代码实现后的验证也不能信子 agent 自报**——主 agent 必须独立回读关键改动点（grep 函数定义 + 行号）+ 独立跑端到端验证

**实战数据**（2026-07-24 P5+P6）：3 轮审查分别发现 3+2+0 个问题（第 3 轮通过），核心映射链路用真实数据 dry-run 验证才确认跑通。

---

## Plan 阶段用户回复的两种模式（**实战 2026-06-05**）

### 模式 A: 多选项单选（澄清工具默认行为）

典型：`clarify` 给 4 个独立选项，用户选 1 个。代表：**「用 claude_chat 跑 dogfood / 暂停 / 国际版 / 暂不跑」**。

**正确处理**：照选中的干，不要追问"那要不要先……"。

### 模式 B: 序贯决策（**用户给的是 A→B→C 流水线**）

典型回复："A. 先 dry run 一下; b. 检查。c. 用现有对话挖 40 个真场景"。

**关键判断**：用户**已经把整个 plan 拍板**，3 个步骤是**流水线节点**——不是"选 A 还是 B"。

**正确处理**：
1. 按 A → B → C 顺序串行干
2. 不要再开 `clarify` 问"接下来 X 还是 Y"
3. 任何一步**发现 fail-fast 条件**（如 dry run 出意外 bug）→ **立即升级报告**，不要硬上

**反模式**（踩过的坑）：把序贯决策误读成单选，问"接下来跑 C 吗？"——浪费 1 轮对话。

### 模式 C: 引用 / 提示

典型："昨晚我不是叫你写了一份专门配置 M3 模型的 skill 吗？里面有配置方式"。

**关键信号**：用户**提示**有现成资源（skill / 文件 / 命令），但**没明说在哪里**。

**正确处理**：
1. **多关键词搜**——`session_search` 用 ≥2 个相关 query
2. **不只搜当前 session**——可能跨多个 session
3. **搜不到**→ 直接问"我没找到你指的那份 skill，能给个提示吗"（**不**假装找到了瞎答）
4. **找到**→ 评估是否适用本次任务；如果适用就**当场用**

**反模式**（踩过的坑）：只用一个关键词搜一次，搜不到就放弃 → 错过用户的提示。

---

## Clarify 超时该收手（**实战 2026-06-05**）

`clarify` 工具默认 10 分钟无响应判定为 `[user did not respond within 10m]`。

**触发场景**：
- 你给 4 个选项，用户没有回
- 你给 yes/no 决策点，用户没回
- 你给"按 X 继续？"确认问，用户没回

**正确处理**：
1. **10 分钟没回** = 用户**没决策意愿**或**离开了**——**不要硬上**
2. 退到**最有把握 / 最保守**的方案
3. **报告**做了什么 + 没做什么 + 让用户回来时**知道从哪继续**
4. 把已沉淀的工件（文件、commit、报告）**列出来**——用户回来能直接接上

**反模式**（踩过的坑）：clarify 超时后再开一个新 clarify 问同样问题——**进入死循环**，浪费对话轮次且不解决问题。

**关键 skill 行为**：clarify 是**决策辅助**不是**决策锁**——10 分钟没回就视作用户说"**你来定**"。

---

## 验证阶段: 不在 Plan 文档里堆配置字段（**实战 2026-06-05**）

**反模式**：plan 写了一个最小化 cfg dict（`{out_root: ..., num_epochs: 1, batch_size: 2, ...}`），实施时一个一个补 cfg 字段（`accumulation`、`skill_init`、`split_mode`），**4 个连续 KeyError**才走 `load_config` 加载真实 yaml 模板。

**正确做法**（**plan 阶段就写**）：
- 凡是**调外部库 / 框架**的 plan，**先拉官方默认 config 模板**（curl raw github / PyPI / 内置）
- 在 plan 里**显式声明**用了哪个 config 模板 + 覆盖了哪些字段
- 实施时直接 `load_config(template_path)` 起步

**Plan 模板新增节**：
```markdown
## Config 来源

| 字段 | 来源 | 覆盖 |
|------|------|------|
| 全部 base 配置 | `<upstream>/configs/_base_/default.yaml` | — |
| 环境相关 | `<upstream>/configs/<env>/default.yaml` | 4 处 |
| 自定义 | 本 plan § 实施项 N | 2 处 |

**实施时**：`load_config("<plan_dir>/main.yaml")` 起步，不要从空 dict 拼。
```

**何时不适用**：完全自研的工具（无上游 config）—— 直接 inline dict 可以。

---

## Skip 阈值: 何时不再钻细节

**触发**：实施中某一步反复 fail（≥ 3 次同样错误族），但**核心目标已达成**（如"验证机制可行"+"发现关键 bug"）。

**判断标准**：
- 核心 deliverable **已经产生**（如 best_skill.md、PR、报告）→ 可以 skip 残余细节
- 核心 deliverable **还没产生**（如机制没跑通）→ 必须钻到底

**正确处理**：
- stop 钻细节
- 写**已发现 + 留给下个 PR**的清单
- 出报告 / 文档化发现
- 把**已沉淀的工件**列出来（**不**让用户回来时不知道发生了什么）

**反模式**（踩过的坑）：失败 3 次后还要"再试一次"——成本沉没 + 错失沉淀窗口。
