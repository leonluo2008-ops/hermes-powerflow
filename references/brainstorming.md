# Brainstorming Reference

Source: obra/superpowers brainstorming skill

## Checklist (in order)

### Step 0: 事实探测（2026-08-06 新增 — 避免凭印象猜测）

> **触发条件**：所有 brainstorm 任务。superpowers 处理的是复杂任务，任何基于错误假设的方案都会导致后续方向全废。

**0a 权威文档**（涉及外部库/平台/工具/API 时）：
- **触发条件**：涉及具体版本号、API 签名、配置字段名、CLI 参数语法、环境变量名等"凭印象会写错"的值时
- **不触发**：标准库的常规用法（`requests.get()` / `os.path.join()` 这种不需要拉文档）
- **动作**：`web_search` 找官方文档 → `web_extract` 拉全文 → 关键值写进 plan 事实段，标来源（如"官方文档 §3.2"）
- **反模式**：凭印象写 `api_mode: openai_chat`（实际是 `chat_completions`）、凭印象写端口号、凭印象写 CLI 子命令名

**0b 本地环境探测**（涉及部署/基础设施/依赖安装/端口时）：
- **探测维度**（按任务相关性选，不是全跑）：
  - 系统层：`uname -a` / `cat /etc/os-release` / `free -h` / `df -h`
  - 依赖层：目标语言版本（`python3 --version` / `node --version` / `docker --version`）+ 已装包（`pip list | grep <pkg>` / `npm ls <pkg>`）
  - 端口服务层：`ss -tlnp | grep <port>` / `systemctl --user status <svc>` / `docker ps`
- **降级规则**：
  - 每条命令带 `timeout 5`（防 hang）
  - 非 root 权限 `ss -tlnp` 看不到进程名 → 降级到 `ss -tln`（至少看到端口占用）
  - `systemctl status` 查 system service 失败 → 试 `systemctl --user status`（路径不同）
  - 命令不存在 → 记录"X 未安装"，列为依赖项
- **冲突处理**：探测结果跟预期冲突（如端口已被占用、依赖版本不符）→ **列为拍板项问用户**，不自动决策
- **产出**：探测结果写进 plan 文档的事实段（`## 环境事实`），后续所有方案基于探测到的事实

### Step 1: 探索项目上下文 + CBM 优先（2026-08-06 修订）

**1a 代码探索 — CBM 优先**（涉及已有代码仓时）：
- **目标在已索引的 cbm 项目内** → 第一动作是 `mcp__cbm__search_graph`（找定义/实现）或 `mcp__cbm__get_architecture`（架构概览），**不是 grep**
- **不确定是否已索引** → 先 `mcp__cbm__list_projects` 确认；没索引的项目首次走 `mcp__cbm__index_repository(repo_path, mode="moderate")`
- **索引过期检测**：`list_projects` 后查 `mcp__cbm__index_status`，如果项目有新 git commit 或索引超过 7 天 → 先 `index_repository` 刷新再查
- **CBM 适合的查询**：架构/依赖/调用链/跨文件关系（`search_graph` / `trace_path` / `get_architecture` / `query_graph`）
- **CBM 不适合的查询**：某文件第几行有 bug、纯文本搜索 → 仍用 `read_file` / `search_files`
- **Fallback 规则**：
  - CBM 查询返回空 → fallback 到 `search_files`（ripgrep），不死磕 CBM
  - CBM 进入 cooldown（连续失败报 `unreachable`）→ 立刻切 grep，等 cooldown 结束再回来
  - CBM `fast` 模式索引不含 semantic edges → semantic 查询返回空时切 grep
- **反模式**：裸 grep 找函数定义（该用 `search_graph`）、裸 grep 评估改动影响面（该用 `detect_changes`）

**1b 查 skill 库已有沉淀**（2026-06-11 实战）：
- 当前 skill 的 `references/` 目录（你之前的实战沉淀、官方 API 调研笔记、踩坑记录）
- 关联 skill（用 `skills_list` + `skill_view` 找 class-level 对应 skill）
- git log（最近 N 个 commit，看历史改了什么、为什么改）
- **反模式**：不查直接写 = 必然重踩历史 bug（seedance2.0-tool 06-04 修过的 Bug 4 在 06-11 spike 重新踩中）

### Step 2-7: 原有流程

2. **Ask clarifying questions** — one at a time, understand purpose/constraints/success criteria
3. **Propose 2–3 approaches** — with trade-offs and recommendation
4. **Present design** — in sections scaled to complexity, get approval after each section
5. **Write design doc** — `docs/plans/YYYY-MM-DD-<topic>-design.md` → commit
6. **Transition** — invoke writing-plans phase

## Rules

- One question per message only
- Multiple choice preferred over open-ended
- Every project goes through this process — no exceptions for "simple" ones
- HARD GATE: Do NOT write code, scaffold, or implement anything until design is approved
- Propose 2–3 approaches before settling, lead with recommendation

## Questions to Ask

- What are you really trying to do? (purpose)
- What constraints exist? (time, tech stack, dependencies)
- What does success look like? (success criteria)
- What should this NOT do? (scope boundaries)

## Approach Selection 偏好（2026-06-11 用户信号）

> **用户偏好**：当前方案能用就干活，**不**要主动提议优化（"先不折腾"是常态）。

**翻译成 superpowers 流程的调整**：

| 场景 | 默认行为 |
|---|---|
| Step 4 列 2-3 个 approaches | 优先"能跑通的最小方案"，**不**主动推"理论更好但要重构"的方案 |
| 用户已经在用某个方案 | **不**主动建议替换（除非用户明确说"想换"）|
| 优化建议 | 留在"未来 spike 范围"，**不**在当前 plan 推进 |
| 抽象层（拆 MCP / 拆工具 / 拆模块）| 只在**当下能解决具体问题**时做，**不**为"未来可能用到"提前抽象 |

**反模式**：
- ❌ "既然在写 MCP 了，顺便把 X 也重构了吧"（scope creep）
- ❌ 列 3 个 approach 时把"更好的架构"放在推荐位（用户选了方案 1 后才 0 摩擦）
- ❌ 主动提"这可以更通用" / "未来可以..." / "理想情况下..."
- ❌ **用多行 Markdown 对比表呈现 approaches** — SOUL.md 禁 markdown 表格。approaches 用纯文字段落（推荐 + 一句话理由 + 其他选项简述），不用 `| 列1 | 列2 |` 表格格式

**正确做法**：
- ✅ 推荐位 = "现在能跑、改动最小、跟现有约束一致"
- ✅ 其它方案 = "可行但有 trade-off，用 1-2 句话描述供选，**不**用表格"
- ✅ 优化建议 → spike/README 里写"未来可以..."，**不**进当前 plan

## Design Sections to Cover

- Architecture overview
- Components and their responsibilities
- Data flow
- Error handling approach
- Testing strategy

## Pitfall: 设计文档中的数据结构假设（2026-07-28 实战）

**问题**：写设计文档时凭印象描述数据结构（"每个 section 有 judge 评分 badge"），但实际数据可能在不同层级（judge 在 module 级别不是 section 级别）。对抗审查必须**读实际 JSON 文件**验证数据层级，不能只看设计文档自述。

**具体踩坑**（course-builder Web Console 设计）：
- 假设 `00-pipeline-status.json` 有 7 个 stage → 实际只有 5 个（Prelude 不走 PipelineHeartbeat）
- 假设 judge 评分在 section 级别 → 实际在 module 级别
- 假设 Writer 是单 provider → 实际有 write_provider + judge_provider 双 provider
- 假设 `build_command()` 是已有能力 → 实际需要自己实现参数映射（下划线→连字符、布尔 flag、空值省略）

**教训**：设计文档涉及已有数据结构时，Step 1（探索项目上下文）必须包含**读实际 JSON 产出物的完整结构**（不只是 `ls` 看文件名），确认数据在哪个层级、有哪些字段、嵌套关系如何。

## Pitfall: 新 Web 服务端口复用冲突（2026-07-28 实战）

**问题**：设计新 Web 服务时复用已有 skill 的默认端口，两个服务同时运行时第二个崩溃。

**实战**：course-builder Web Console 设计端口 8765 = web-reader-local-panel 默认端口。决策专员发现两者会同时运行 → `Address already in use`。

**铁律**：新 Web 服务不复用已有服务的默认端口，推荐连号（8765→8766）。对抗审查/决策专员审查时必查端口冲突这一项。

## After Design Approval

- Write design to `docs/plans/YYYY-MM-DD-<topic>-design.md`
- Commit the design doc
- Hand off to writing-plans — no other skill, no implementation
