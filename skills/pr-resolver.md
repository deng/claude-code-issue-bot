---
name: pr-resolver
description: |
  处理 PR 中的 Comment 和 Review 意见，按严重程度自动修复或回复。
  触发语境：用户要求处理 PR 评论、review 意见、fix comment、修改 review 指出的问题、
  在 PR 上做修改、根据 feedback 改代码等。包含循环修复检测。
compatibility:
  - gh (GitHub CLI) authenticated
  - git
---

# PR Resolver Skill

处理 PR Comment 和 Review 意见。流程：**解析 Comment → 分类严重程度 → 检查循环 → 修复/回复 → 更新约定文档**。

## Step 0: Pre-Check — GitHub CLI Authentication

```bash
gh auth status 2>&1
```

如果认证失败或退出码非零，停止并告诉用户：
```
gh CLI 未认证。请先运行 `gh auth login` 登录 GitHub。
```

## 严重程度分类

| 级别 | 含义 | 处理方式 |
|------|------|---------|
| **Critical** | 功能错误、安全漏洞、数据正确性问题 | 自动修复，无需确认 |
| **Improvement** | 代码质量、性能优化、健壮性、设计改进 | 自动修复。回复 comment 说明已修复 |
| **Style (Convention)** | 命名风格、格式、文件组织等可统一规范的问题 | 修复 + 写入宪法文档。若有争议则先回复讨论 |
| **Question** | 对实现方式的疑问、请求解释 | 评论回复解释原因，不改代码 |

### Style (Convention) 处理细则

- **可统一的规范**（命名风格、错误处理模式、文件组织、导入顺序等）→ 自动修复，同时更新对应的宪法文档
- **主观偏好**（"我觉得这样更好看"）→ 评论回复解释当前设计的原因和考虑
- **不确定的** → 按争议最小的方式修复，并在 commit message 和回复评论中标注为候选（`[Convention Candidate]`），让 PR author 确认

## Step 1: 解析 PR Comment

```bash
# 获取 comment 详情（使用全量路径，避免 :owner 花括号混用）
gh api "repos/<owner>/<repo>/pulls/<pr_number>/comments" --jq '.[] | {id, path, line, body, user: .user.login}'

# 获取 PR 基本信息
gh pr view <pr_number> --repo <owner>/<repo> --json headRefName,baseRefName,body,state,files

# 获取 PR 最新 diff
gh pr diff <pr_number> --repo <owner>/<repo>
```

确认以下信息：
- Comment 指向哪个文件/哪段代码？
- Comment 的内容和意图是什么？
- Comment 作者是谁？（是否是 bot 自身？如果是，跳过）
- 此前的 comment 是否有未解决的讨论链？

## Step 2: 循环修复检测

修复前必须执行以下检查，防止 A→B→A→B 的来回修改：

### 2a Cycle Counter 检查

```bash
# 获取当前修复次数
gh pr view <pr_number> --repo <owner>/<repo> --json body | grep -o 'fix-cycle: [0-9]*' || echo "0"
```

或者维护 PR body 中的标记：
```
fix-cycle: 3
```

- 当前 cycle >= 3 → **停止自动修复**，在 PR 中评论：
  > 已检测到多次循环修复（{N} 次），建议人工介入评估根本原因。
  >
  > 循环记录：
  > 1. `<时间>` — `<改动概要>`（commit: `<sha>`）
  > 2. ...
- cycle < 3 → 继续

### 2b Git Diff 回环检测

用 git 检查当前修改是否与历史 commit 形成回环：

```bash
# 获取最近 N 次 commit 的 diff
git log --oneline -5
git diff HEAD~1 HEAD | head -200  # 获取上次 commit 的变更
```

**回环检测逻辑：**
1. 获取最近 3 次 commit 的变更摘要（改了哪些文件的哪些行）
2. 如果本次待修改的内容，与最近某次 commit 的变更方向相反（把 A 改回 B，而前一次 commit 恰好是 B 改 A）→ **判定为回环**
3. 回环时：停止修复，回复 comment，建议人工介入

### 2c Session 上下文

如果通过 SDK 调用（`sessionId` 续传），Agent 应感知到：
- "当前代码就是我之前改的"
- "这次的要求是把我的改动改回去"
- 结合前面的 diff 检查判断是否合理

## Step 3: 实施修复或回复

### 3a 环境准备

使用 `EnterWorktree` 工具创建隔离工作树，**与 issue-resolver 保持一致**，不要用原生 `git worktree add`：

- 调用 `EnterWorktree` 传入 `name` 参数（如 `pr-${PR_NUMBER}-fix`），并切换会话工作目录到 worktree
- 进入 worktree 后，checkout PR 分支：
  ```bash
  git fetch origin "pull/${PR_NUMBER}/head:${HEAD_REF}" 2>/dev/null || git fetch origin "refs/pull/${PR_NUMBER}/head"
  git checkout "${HEAD_REF}"
  ```
- **修复后必须推送到原 PR 分支（HEAD_REF），不要创建新分支推送**

**⚠️ `EnterWorktree` 之后，所有代码修改必须只在 worktree 中进行。禁止修改主仓库的任何文件。**
- 后续所有 Read/Edit/Write 操作的文件路径必须是 worktree 内的路径
- 所有 git 操作（add、commit、push）必须在 worktree 目录中执行
- 如需对比主仓库代码做参考，read-only 可以，但绝不能 write

### 3b 修复（Critical / Improvement / Style）

1. **修复前先确认编译基线**：修改任何代码前，先在 worktree 中编译一次，确认当前 PR 分支本身能编译通过。如果基线就编译失败，停止修复并告知用户——否则无法区分是修复引入的新错误还是已有问题。

   ```bash
   # .NET：dotnet build / Node：npm run build 或 npx tsc --noEmit / Go：go build ./...
   ```

2. **修改范围控制：**
   - Critical：不限范围，必须解决问题
   - Improvement：限定在 comment 指出的文件范围内，不额外改动
   - Style：仅限于 style 相关的改动，不改逻辑

3. **修改后验证：**
   - 在 worktree 内执行项目编译通过（`dotnet build` / `tsc --noEmit` 等，视项目类型而定；只编译相关项目，不要全量编译无关项目）
   - 代码风格符合项目规范（参考项目 CLAUDE.md 或已有代码风格）

4. **Commit（在 worktree 中执行）：**
   ```bash
   git add <files>
   git commit -m "fix: <comment 内容概要>

   依据 PR #${PR_NUMBER} Review 意见。
   类型: Critical/Improvement/Style

   Co-Authored-By: Claude <noreply@anthropic.com>"
   ```

### 3c 回复（Question / 主观偏好 / 需要讨论）

回复 comment，说明理由：

```bash
gh pr comment <pr_number> --repo <owner>/<repo> --body "$(cat /tmp/reply-<pr_number>.md)"
```

回复内容写入临时文件再传入，避免 shell 变量不可靠：

```bash
cat > /tmp/reply-<pr_number>.md <<'EOF'
@author

关于 `<文件名>:<行号>`：

[解释当前实现的选择原因/为什么要这样设计]

[如果建议合理但不适合在此 PR 中改动 → "[Convention Candidate] 这个建议值得采纳，我已记录到约定文档待讨论。"]

[如果是在问问题 → 按实际情况回答]
EOF
```

### 3d Style (Convention) — 写入宪法（⚠️ 需满足条件才写）

写入宪法文档是**可选且受限**的操作，必须满足以下**全部**条件：

1. 该规范是**跨文件可统一**的规则（不是某处一次性判断）
2. 该 PR 的目标分支是 **main / master**（不是 feature 分支）
3. 已向用户说明并**得到用户确认**（不要自动写入）

不满足以上条件时，仅在 commit message 和回复 comment 中标注为 `[Convention Candidate]`，不写入宪法文档。

满足条件时，写入 `.claude/conventions/`：

```markdown
<!-- .claude/conventions/<category>.md -->

## <规则名>

- **发现场景**: PR #{pr_number} — <comment 概要>
- **规则**: <具体的规范描述>
- **依据**: <为什么这个规范有助于项目>
```

宪法文档按类别组织：

```
.claude/conventions/
├── naming.md           # 命名规范
├── error-handling.md   # 错误处理模式
├── project-structure.md  # 文件组织
├── style-guide.md      # 代码风格
└── patterns.md         # 常用设计模式
```

如果对应类别的文档已存在，追加新规则。如果不存在，创建新文档。

## Step 4: 提交修改（在 worktree 中执行）

**所有 git 操作必须在 worktree 目录中执行。不要切换到主仓库执行 git 命令。**

```bash
# 推送到 PR 原分支，不要创建新分支
git push origin HEAD:${HEAD_REF}
```

### 更新循环计数器（用临时文件，不用嵌入 Python 的管道）

```bash
# 读取当前 body 到临时文件
gh pr view ${PR_NUMBER} --repo ${owner}/${repo} --json body --jq '.body' > /tmp/pr-body-current-${PR_NUMBER}.md

# 用 perl 安全替换（兼容 macOS/GNU）
CURRENT_CYCLE=$(grep -o 'fix-cycle: [0-9]*' /tmp/pr-body-current-${PR_NUMBER}.md | grep -o '[0-9]*' || echo "0")
NEXT_CYCLE=$((CURRENT_CYCLE + 1))

if grep -q 'fix-cycle:' /tmp/pr-body-current-${PR_NUMBER}.md; then
  perl -pi -e "s/fix-cycle: [0-9]+/fix-cycle: ${NEXT_CYCLE}/" /tmp/pr-body-current-${PR_NUMBER}.md
else
  printf '\nfix-cycle: %s\n' "${NEXT_CYCLE}" >> /tmp/pr-body-current-${PR_NUMBER}.md
fi

# 写回 PR body
gh pr edit ${PR_NUMBER} --repo ${owner}/${repo} --body-file /tmp/pr-body-current-${PR_NUMBER}.md

# 清理临时文件
rm /tmp/pr-body-current-${PR_NUMBER}.md
```

回复 comment 通知已修复：

```bash
cat > /tmp/fixed-reply-${PR_NUMBER}.md <<'EOF'
@author 已修复 ✓

改动：<简述改了什么>

类型: ${severity}
EOF
gh pr comment ${PR_NUMBER} --repo ${owner}/${repo} --body "$(cat /tmp/fixed-reply-${PR_NUMBER}.md)"
rm /tmp/fixed-reply-${PR_NUMBER}.md
```

## 循环防护总结

| 防护层 | 检测方式 | 触发动做 |
|--------|----------|----------|
| Cycle Counter | PR body 中 `fix-cycle` 标记 | >= 3 次停止，通知人工介入 |
| Git Diff 回环 | 对比最近 commit 与当前修改方向 | 检测到回环时停止 |
| Session 上下文 | Agent 感知自身历史改动 | 配合 diff 检查做综合判断 |