# Superpowers Applied: User Context

## User Profile

- **Name:** 二大爷（重庆人，说话直接）
- **Current focus:** 抖音儿童绘本账号运营
- **Tools:** OpenClaw（主力）、Hermes、CloudCode
- **Model:** GLM-5-Turbo（主力），MiniMax-M2.7（备用）
- **本月重点:** 抖音相关工作（运营两个儿童绘本视频账号）

## Douyin Content Creation → Superpowers Mapping

### Phase 1: Brainstorming (故事策划)

**Before generating anything, ask:**
- 目标观众是谁？（3-6岁？家长陪看？）
- 核心冲突/事件是什么？（货物触发、幽默惩罚）
- 希望观众记住什么？（不是"学到什么"）
- 时长控制在多少？（单集60-90秒？）
- 有没有参考图/风格要求？

**Design approval gate:** 用户说"可以"再进入下一步

### Phase 2: Writing Plans (任务分解)

**Example task breakdown for one episode:**
```
Task 1: 写故事梗概（3版本，500字内）
Task 2: 写分镜脚本（九宫格描述，每格动作+旁白）
Task 3: 生成角色图（托比+莱恩，face用参考图）
Task 4: 生成场景图（每个分镜一张）
Task 5: 生成视频（Seedance/即梦）
Task 6: 旁白+音乐
```

**Save to:** `docs/plans/YYYY-MM-DD-<episode-name>.md`

### Phase 3: Subagent Execution (per-task loop)

For each task:
1. Dispatch implementer subagent → produce draft
2. Dispatch reviewer subagent → confirm quality
3. Fix issues → next task

**For image/video generation tasks:** TDD spirit applies — define what "good" looks like before generating, not after.

### Phase 4: Systematic Debugging

**When video generation fails or looks wrong:**
1. Root cause: is it prompt? model? reference image? style mismatch?
2. Check recent changes: did the prompt change recently?
3. Test hypothesis: one change at a time
4. Fix at root: re-write prompt / switch model / adjust style params

### Phase 5: Finishing

- Verify all assets generated
- Present to user for final approval
- Archive plan file

## User Preferences (from memory)

- **输出格式:** 飞书表格用纯 Markdown（|分隔，无代码块），让飞书原生渲染
- **反馈风格:** 直接、"不行"、"太单调"、"继续" — 不要废话
- **绘本标准:** 拒绝单调，必须有具体细节/角色名字+性格+动机/完整因果链
- **教育目标:** 不能提前，必须先建立小社会让读者爱上角色，再自然呈现
- **幽默惩罚:** 百宝箱空了（幽默），不是帝国被毁（恐怖）

## Key Principles from User's Workflow

- **先建立小社会让读者爱上** → 在 brainstorm 阶段必须先问"观众/角色是谁"
- **货物触发具体夸张** → Brainstorm 设计部分的核心
- **不教育/不说教，结尾不问"故事告诉我们什么"** → 记得在 review 时检查

## Important Constraints

- 有参考图时：face用参考图图生图，outfit默认用参考图不重生成
- 不能一刀切种族锁定，先问创作者项目风格再决策
- 编排层(SKILL.md)×生成层(references/)×工具层(scripts/) 三层分离