# Hermes Powerflow

> [Hermes Agent](https://hermes-agent.nousresearch.com) 的 spec-first 开发流水线。从 [obra/superpowers](https://github.com/obra/superpowers) 演化而来，针对 Hermes 实际工作流做了深度定制。

## 这是什么

一套强制执行的多阶段开发流水线——不是建议，而是结构化工作流，防止凭印象写代码，强制基于证据执行。

```
想法 → 头脑风暴 → 计划 → 子 Agent 驱动构建 (TDD) → 代码审查 → 完成分支
```

## 核心特性（在上游 superpowers 基础上的定制）

- **事实探测（Step 0）** — 写任何具体值前先查权威文档 + 探测本地环境（端口、依赖、OS），杜绝凭印象
- **CBM 优先** — 代码探索默认用 [Codebase Memory](https://github.com/nousresearch/cbm)（`search_graph` / `get_architecture`），不裸 grep
- **CBM 索引刷新** — merge/push 后自动刷新代码图谱（仅当代码文件有改动时）
- **MoA 建议** — 复杂任务推荐使用 Mixture-of-Agents 模式（多模型头脑风暴 + 对抗审查）
- **对抗审查闸门** — 多轮方案审查直到"通过"，不是一轮就放行
- **用户能力感知执行** — 每步汇报进度，不静默跳过，不做过程性打断

## 流水线阶段

| 阶段 | 做什么 |
|------|--------|
| **1. 头脑风暴** | 探索上下文（CBM + skills_list）→ 澄清问题 → 提出方案 → 设计文档 |
| **2. 计划** | 逐任务实现计划，每个 2-5 分钟，TDD 强制 |
| **3. 构建** | 子 Agent 驱动：实现者 → 规范审查 → 质量审查，逐任务循环 |
| **4. 调试** | 根因调查 → 模式分析 → 假设验证 → 根因修复 |
| **5. 完成** | 验证测试 → merge/push → CBM 索引刷新 → 清理 |

## 使用方式

Hermes Agent 检测到触发词时自动加载：

- `powerflow` / `powerflow 流程` / `开启 powerflow`
- `superpowers 流程`（向后兼容）

## 目录结构

```
SKILL.md                                # 流水线主定义
references/
├── brainstorming.md                    # 阶段 1 详解（Step 0 事实探测 + Step 1 CBM 优先）
├── writing-plans.md                    # 阶段 2 详解
├── subagent-development.md             # 阶段 3 详解
├── tdd.md                              # TDD 方法论
├── systematic-debugging.md             # 阶段 4 详解
├── debug-toolkit.md                    # 实战调试工具
├── finishing-branch.md                 # 阶段 5 详解（CBM 索引刷新）
├── pre-commit-review.md                # merge 前代码审查
├── user-capability-aware-execution.md  # 进度汇报 + 验收清单
├── user-context.md                     # 用户特定上下文
├── repository-cohesion.md              # 单仓多入口架构
└── simplify.md                         # merge 后清理
```

## 安装

本 skill 部署在 `~/.hermes/skills/superpowers/`（目录名保留兼容性）。

```bash
git clone https://gitee.com/luo-xiansen2023/hermes-powerflow.git ~/.hermes/skills/superpowers
```

## 版本

**v1.0.0** — 作为独立项目首次发布。

## 许可证

MIT
