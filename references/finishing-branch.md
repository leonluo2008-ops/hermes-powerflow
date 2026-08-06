# Finishing a Development Branch Reference

Source: obra/superpowers finishing-a-development-branch skill

## Overview

Verify tests → Present options → Execute choice.

Announce at start: "I'm using the finishing-a-development-branch skill to complete this work."

## Step 1: Verify Tests

Run the project's test suite. If tests fail — stop. Do not proceed.

```bash
# Python
pytest -q

# Node/TS
pnpm test

# Rust
cargo test

# Go
go test ./...
```

Show failures clearly. Cannot merge/PR until tests pass.

## Step 2: Determine Base Branch

```bash
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
git branch --show-current
```

Or ask: "This branch split from main — is that correct?"

## Step 3: Present Options

Present exactly these 4 options — no extra explanation:

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

## Step 4: Execute Choice

### Option 1: Merge Locally

```bash
git checkout <base-branch>
git pull
git merge <feature-branch>
<run tests>
git branch -d <feature-branch>
```

### Option 2: Push + PR

```bash
git push -u origin <feature-branch>
gh pr create --title "<title>" --body "## Summary
- <bullet 1>
- <bullet 2>

## Test Plan
- [ ] <verification step>"
```

### Option 3: Keep As-Is

Report: "Keeping branch `<name>`. You can return to it later."

### Option 4: Discard

**Confirm first:**
```
This will permanently delete:
- Branch <name>
- All commits since <base-branch>

Type 'discard' to confirm.
```

Wait for exact word "discard". Then:
```bash
git checkout <base-branch>
git branch -D <feature-branch>
```

## Step 5: CBM 索引刷新（2026-08-06 新增）

**触发条件**：本次任务改了某个 git 仓库 / 独立项目（skill 仓库、app 代码仓库）的代码文件。

**只在 Step 4 选了 Option 1（Merge）或 Option 2（Push+PR）时执行**：
- Option 1/2 → 代码已合入主线或推送到远程，刷新索引让 CBM 图谱反映最新代码
- Option 3（Keep）→ 分支还在半成品状态，**不刷新**（避免索引未合入的代码）
- Option 4（Discard）→ 代码已删除，**不刷新**（避免 ghost nodes 污染图谱）

**动作**：
```
mcp__cbm__index_repository(repo_path="<改动仓库绝对路径>", mode="moderate")
```

**不触发**：纯文档改动、一次性脚本、非代码任务（内容创作类）。

**mode 说明**：一律 `moderate`（`full` 太慢含全部 similarity 边，`fast` 缺 semantic search 能力）。

## Quick Reference

| Option | Merge | Push | Keep Branch | Delete Branch |
|--------|-------|------|-------------|---------------|
| 1. Merge locally | ✓ | — | — | ✓ |
| 2. Create PR | — | ✓ | ✓ | — |
| 3. Keep as-is | — | — | ✓ | — |
| 4. Discard | — | — | — | ✓ (force) |

## Common Mistakes

- Skipping test verification before offering options → always verify first
- Merging without pulling latest base → always `git pull` before merge
- Deleting branch before confirming PR merged → wait for merge confirmation

## Long commit messages: 用 file 而非 inline（2026-06-11 spike 006 踩坑）

**反模式**：

```bash
# ❌ 长 commit message 含多行 / 引号 / 中文 → bash 转义崩
git commit -m "title: 含 \" 引号 / ' 单引号 / 中文段落 / *bold*"
# /usr/bin/bash: eval: 行 8: 寻找匹配的 '"' 时遇到了未预期的 EOF
```

**正确做法**：把 message 写到文件，`-F` 传文件：

```bash
# 1. 写 message 到文件
cat > /tmp/commit-msg.txt <<'EOF'
title: 简短标题

【变更内容】
- bullet 1
- bullet 2

【Co-Authored-By】
Co-Authored-By: Claude (noreply@anthropic.com)
EOF

# 2. commit 用 -F
git commit -F /tmp/commit-msg.txt
```

**触发条件**（**任一**满足就用 file 模式）：
- 消息 > 5 行
- 消息含 `"` / `'` / `*` / `\` / 反引号
- 消息含中文 + 任何标点
- 消息含反引号代码块

**快捷技巧**（hermes 里用 `write_file` 写）：

```python
write_file(path='/tmp/commit-msg.txt', content=message_body)
```

然后 `git commit -F /tmp/commit-msg.txt`。
