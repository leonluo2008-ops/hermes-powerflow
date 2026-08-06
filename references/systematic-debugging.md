# Systematic Debugging Reference

Source: obra/superpowers systematic-debugging skill

## The Iron Law

```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
```

Random fixes waste time and create new bugs. Symptom fixes are failure.

## When to Use

Any technical issue:
- Test failures
- Bugs in production
- Unexpected behaviour
- Performance problems
- Build failures
- Integration issues

Use this ESPECIALLY when under time pressure, when "one quick fix" seems obvious,
or when previous fixes didn't work. Rushing guarantees rework.

## The Four Phases

Complete each phase before proceeding to the next.

### Phase 1: Root Cause Investigation

**BEFORE attempting ANY fix:**

1. **Read error messages carefully**
   - Don't skim past errors or warnings
   - Read stack traces completely
   - Note line numbers, file paths, error codes

2. **Reproduce consistently**
   - Can you trigger it reliably?
   - What are the exact steps?
   - If not reproducible → gather more data, don't guess

3. **Check recent changes**
   - What changed that could cause this?
   - `git diff`, recent commits, new dependencies, config changes

4. **Gather evidence in multi-component systems**

   When the system has multiple layers (API → service → database, CI → build → signing):
   
   Before proposing fixes, add diagnostic instrumentation at each boundary:
   ```
   For each component boundary:
     - Log what data enters
     - Log what data exits
     - Verify config/env propagation
   Run once to see WHERE it breaks, THEN investigate that layer
   ```

5. **Trace data flow**
   - Where does the bad value originate?
   - What called this with the bad value?
   - Trace up the call stack until you find the source
   - Fix at source, not at symptom

### Phase 2: Pattern Analysis

**Find the pattern before fixing:**

1. Find working examples of similar code in the codebase
2. Compare against references — read completely, don't skim
3. List every difference between working and broken
4. Understand all dependencies and assumptions

### Phase 3: Hypothesis + Testing

1. Form one clear hypothesis: "I think X is the root cause because Y"
2. Write it down explicitly
3. Design a minimal test to prove/disprove it
4. Run the test
5. If disproved → form new hypothesis, repeat
6. If proved → proceed to fix

### Phase 4: Fix + Verification

1. Fix at root cause, not symptom
2. Write a test that would have caught this bug
3. Verify fix works
4. Verify no regressions (run full test suite)
5. Commit with clear message explaining root cause

## Anti-Patterns

- **"Just try this"** — random fixes without root cause
- **"It's probably X"** — guessing without evidence
- **"Quick patch"** — symptom fix leaving root cause intact
- **Fixing multiple things at once** — makes it impossible to know what worked
- **Skipping reproduction** — debugging something you can't trigger consistently

## Whack-a-Mole Trigger (2026-07-31 实战)

**当同一个功能连续出现 3+ 个 bug 时**，停止逐个打补丁，切换到全链路审查模式：

1. `delegate_task` 派子 agent 读所有相关文件（前端 + 后端 + 脚本 + 实际数据文件）
2. 追踪数据从源头到终端的完整路径，在每个阶段边界检查格式匹配
3. 列出所有断点（编号 + 文件:行号 + 修复方案），按严重程度排序
4. 一次性修复所有断点
5. 浏览器验证

**用户原话**：「你直接去改，你看你改了多少次了，都是失败，不是这里有问题，就是那里有问题」

**为什么有效**：全栈功能涉及多层（前端响应式 → 后端 JSON → AI 输出不确定性），单点修复几乎必然引入新问题。子 agent 在隔离 context 中不受"已经改了 5 次"的思维定势影响，能客观审查整个链路。

**信号识别**：如果第二次修复后用户又报告新问题，不要继续修——立即进入全链路审查。
