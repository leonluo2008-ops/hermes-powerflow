# Repository 架构口径一致性（2026-06-11 spike 006 实测）

> 仓库是**一个工具集**，不是"多个并列入口"。多入口（CLI / MCP / shared lib）共享单真源，**统一仓库管理**。

## 核心原则

| 错误口径 | 正确口径 | 触发词 |
|---------|---------|--------|
| ❌ "3 个并列入口"（CLI / MCP / shared lib） | ✅ "**一个工具集**，3 种使用方式" | 写 README、SKILL、对外文档 |
| ❌ "CLI 是入口 / MCP 是另一个项目" | ✅ "**MCP 是 CLI 的工具子集**" | 设计产品 / 介绍新成员 |
| ❌ "跨仓库共享" | ✅ "**单仓库多入口**，改一处生效所有入口" | 决定要不要建新仓库 |
| ❌ "spike 改造" 当作独立产品 | ✅ "**spike 是仓库的演进历史**" | 写 commit message / 维护 README |

**关键判断（用户在 2026-06-11 spike 006 中明确）**：

> "你可以把 MCP 这个工具理解成集合在 seedance2.0 tool skill 里面的一个工具子集。他们实际上是一个仓库，我们本来就是设计的是一个仓库来管理的。"

**这条**是**设计哲学层**的口径——**不是**实现细节。任何仓库设计 / 文档撰写 / 重构决策前必过。

## 仓库结构模板（以 seedance2.0-tool 为例）

```
<project>/
├── README.md               ← 仓库总览（**口径：1 个工具集，N 种使用方式**）
├── INSTALL.md              ← 5 步安装
├── QUICKSTART.md           ← M 个最快跑通示例（覆盖所有入口）
├── TROUBLESHOOTING.md      ← K 常见错 + 修复
├── .env.example            ← 环境变量模板
├── SKILL.md                ← skill description（如该仓库装为 skill）
├── error-patterns.md       ← 踩坑积累
│
├── <entry-point-1>.py      ← 入口 1（CLI / MCP / shared lib / ...）
├── <entry-point-2>.py      ← 入口 2
├── <shared_lib>.py         ← 共享业务函数（**单真源**）
│
├── spikes/                 ← 演进历史（**仓库内的，不是项目外的**）
│   ├── 001-<topic>/
│   ├── 002-<topic>/
│   └── ...
│
└── references/             ← 实战沉淀（API 文档 / 范式 / Bug / 任务管理）
```

**关键设计点**：
1. **README 永远先讲"仓库是什么"**（不是"怎么用"）
2. **spikes/ 是仓库内子目录**（不是单独仓库、不是项目外的 blog post）
3. **3 入口（CLI/MCP/shared）** 共享**单真源**（改 `shared_lib.py` 一处，CLI + MCP 同时生效）
4. **INSTALL + QUICKSTART + TROUBLESHOOTING 是 3 件套**（**不**全堆在 README）

## 决策树：什么时候要拆仓库

| 触发 | 决策 |
|------|------|
| 一个仓库里 3+ 个独立产品（不共享代码）| ✅ 拆 |
| 多入口但共享业务函数（CLI/MCP/shared）| ❌ **不**拆（一个仓库）|
| Spike 是"演进历史"或"实验性 POC" | ❌ **不**拆（放 `spikes/`）|
| Spike 已经变正式功能 | ✅ 把它**吸收到主代码**（不是新仓库）|

## 文档撰写检查清单

写 README 前**必答**这 3 问：

1. **这个仓库是什么**（一句话）？  ← 第一行
2. **这个仓库提供几种使用方式**？  ← 顶部表格
3. **M 种使用方式共享单真源**（如果 1 个）或**完全独立**（如果 N 个）？

**错**（容易踩的坑）：
- ❌ README 写"我们提供 CLI / MCP / Python library 3 种入口"——**隐含 3 个并列**
- ❌ Spike README 写"接下来要做的清单"——**没注明"v1.0 时点的计划"**
- ❌ INSTALL.md 没指向 conductor / wrapper skill

**对**：
- ✅ README 写"CLI / MCP / shared lib = **3 种使用方式**，共享 `seedance_uploads.py` 单真源"
- ✅ Spike README 顶部加"⏰ 截至 YYYY-MM-DD 全部完成"banner
- ✅ INSTALL.md 含 conductor skill 安装指引

## 与"占位符思维"的区别

| 维度 | 占位符思维 | Repository 口径 |
|------|------------|-----------------|
| **层面** | 代码 / 文档 | 仓库设计 / 文档口径 |
| **关注** | 哪些值不进 skill（路径 / ID / 数字）| 仓库怎么组织 / 文档怎么写 |
| **触发** | "这值会因环境变吗" | "这仓库有几个产品" |
| **错误** | 硬编码 `huiben` / `3149...` | 把"工具集"写"3 个入口" |

**两者正交**——都必遵守。

## 实战模板（seedance2.0-tool README 第一段）

```markdown
# <Project Name>

> **仓库口径**：CLI + MCP + 共享业务函数 + 实战沉淀 = **一个工具集**

[一句话描述仓库做什么]

提供 N 种使用方式：

| 入口 | 用途 | 适合 |
|------|------|------|
| **CLI** | 命令行直接跑 | 本地开发、CI/CD 脚本 |
| **MCP server** | N 个 `<prefix>_*` 工具，LLM 调用 | Hermes / Claude Desktop / Cursor |
| **Shared lib** | Python import | 自建 agent / 工作流脚本 |

**N 种入口**共享**单真源**业务函数（`shared.py`）——改一处生效所有入口。
```

## 红线

- ❌ **不**为 MCP / spike 单独建仓库（**单仓库管理**）
- ❌ **不**把 spike 描述为"未来要做的独立项目"
- ❌ **不**写"3 个并列入口"等破坏口径的措辞
- ✅ spike 演进历史在仓库内 `spikes/NNN-*/` 子目录
- ✅ spike 转正 = 吸收到主代码（不是新仓库）
