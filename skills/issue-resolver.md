---
name: issue-resolver
description: |
  根据 GitHub Issue 自动分析代码并实现修复。当用户输入 GitHub Issue 链接（如 https://github.com/owner/repo/issues/N）、或输入 `/resolve-issue <url>`、或说"解决这个 issue"、"处理这个 issue"、"把这个 issue 修一下"时触发。

  工作流程：检查仓库匹配 → 读取 issue 详情 → 分析代码定位根因 → 创建 worktree 和分支 → 实施修复 → 编译验证 → 提交代码并创建 PR。
compatibility:
  - gh (GitHub CLI) authenticated
  - git
---

# Issue Resolver Skill

根据 GitHub Issue 自动创建分支并实施修复。流程：**解析URL → 匹配仓库 → 分析代码 → 创建工作树 → 实施修复 → 编译验证 → 提交确认**。

## Step 0: 解析 Issue URL

从用户消息中提取 GitHub Issue URL，格式为 `https://github.com/<owner>/<repo>/issues/<number>`。

```bash
# 示例
# URL: https://github.com/deng/OneTeamAI/issues/4
# owner=deng, repo=OneTeamAI, number=4
```

如果消息中同时包含多个 issue URL，让用户选择先处理哪一个。

## Step 1: 匹配仓库

### 1a 检测当前仓库

```bash
ROOT_REPO=$(git rev-parse --show-toplevel 2>/dev/null)
ROOT_OWNER_REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner' 2>/dev/null)
```

### 1b 检测子目录仓库

如果 issue 的 owner/repo 与当前仓库不匹配，查找所有子目录中的独立 git 仓库：

```bash
find "$ROOT_REPO" -name ".git" -type d -not -path "*/node_modules/*" -not -path "*/.git/*" 2>/dev/null | while read d; do
  dir=$(dirname "$d")
  remote=$(cd "$dir" && git remote get-url origin 2>/dev/null || echo "")
  if echo "$remote" | grep -qi "${owner}/${repo}"; then
    echo "MATCH:$dir"
  fi
done
```

- 如果匹配到子目录仓库，后续所有操作在该子目录中执行（`cd <subdir>`）
- 如果未匹配到，报告用户无法找到对应仓库并停止

### 1c 确认匹配

告知用户匹配到的仓库路径，让用户确认后继续。

## Step 2: 读取 Issue 详情

```bash
gh issue view <number> --repo <owner/repo> --json title,body,labels,state,assignees
```

读取并理解 issue 的内容，包括：
- **标题和描述**：问题是什么？
- **复现步骤**：如何复现？
- **根因分析**：issue 中是否已经分析了根因？
- **修复方案**：issue 中是否已建议修复方案？

## Step 3: 分析代码定位根因

根据 issue 的描述分析代码库。不要完全信任 issue 中的根因分析——要亲自验证。

### 3a 搜索相关文件

**根据仓库实际文件类型动态构造搜索范围**，不要使用固定的扩展名列表。

```bash
# 先检测仓库主要语言
# .NET/C# 项目：搜索 *.cs *.csproj *.json *.config *.yml
# TypeScript/JavaScript 项目：搜索 *.ts *.tsx *.js *.jsx *.json *.yml
# Python 项目：搜索 *.py *.pyi *.toml *.cfg *.yml
# Go 项目：搜索 *.go *.mod *.sum *.yml
# 通用模式：优先搜索 issue 中提到的关键词

# 按文件名搜索
find . -name "*<keyword>*" -type f 2>/dev/null

# 搜索代码模式（根据仓库类型选择扩展名）
grep -rn "<pattern>" --include="*.cs" --include="*.csproj" . 2>/dev/null | head -30

# 查看相关文件的 git 历史
git log --oneline -10 -- <file-path>
```

### 3b 阅读关键文件

用 Read 工具阅读：
- issue 中提到的具体文件
- 相关的配置文件、调用链
- 最近的 git 提交记录（了解上下文）

### 3c 验证根因

亲自验证根因：
- 复现步骤是否有效？
- issue 中的根因分析是否正确？
- 修复方案是否完整？有没有遗漏的调用点或配置？

## Step 4: 创建工作树并创建分支

使用 `EnterWorktree` 工具创建工作树，**禁止**在原始工作区直接操作分支，避免干扰当前工作。

### 4a 命名分支

确定分支名用于 worktree。分支名基于 issue 标题关键字生成，使用 `perl` 兼容 macOS 和 Linux：

```bash
# 命名规则：fix/issue-<number>-<简短英文描述>
# 描述从 issue 标题中提取关键字，不超过 40 字符
# 使用 perl -pe 代替 sed，因为 macOS BSD sed 与 GNU sed 不兼容
ISSUE_NUMBER="<number>"
ISSUE_TITLE="<title>"
SAFE_TITLE=$(echo "$ISSUE_TITLE" | perl -pe 's/[^a-zA-Z0-9一-鿿]/-/g; s/-{2,}/-/g; s/^-//; s/-$//; $_=lc($_); $_=substr($_,0,40)')
BRANCH_NAME="fix/issue-${ISSUE_NUMBER}-${SAFE_TITLE}"
```

如果生成的分支名以 `-` 结尾，再去除一次。

### 4b 同步 main 分支

创建工作树前，先确保本地 main 引用是最新的，避免从过期代码创建分支：

```bash
git fetch origin main && git update-ref refs/heads/main origin/main
```

> **为什么用 `update-ref` 而不是 `git checkout main && git pull`**：如果 main 已被其他 worktree 占用，`git checkout main` 会失败（`fatal: 'main' is already used by worktree at '...'`）。`git update-ref` 直接更新本地引用指针，不受 worktree 占用限制，效果等价。

### 4c 创建工作树

调用 `EnterWorktree` 工具创建隔离工作树，传入 `name` 参数为分支名：

- `EnterWorktree` 会自动在 `.claude/worktrees/` 下创建隔离工作目录
- 基于远程默认分支（通常是 `origin/main` 或 `origin/master`，由 `worktree.baseRef` 配置决定）创建新分支
- **不要使用** `git stash` / `git checkout main` / `git pull` 等传统方式切换分支
- 工作树与原始工作区完全隔离，不干扰用户当前工作

**错误处理**：如果 `EnterWorktree` 失败（如分支名冲突），检查失败原因并修复后再重试，不得跳过此步骤直接在主仓库修改。

### 4d 确认分支

创建完成后，告知用户 worktree 的路径和分支名，让用户确认后继续。

## Step 5: 实施修复（⚠️ 只在 worktree 中修改）

**硬规则：`EnterWorktree` 之后，所有代码修改必须只在 worktree 中进行。禁止修改主仓库的任何文件。**

- 所有 Read/Edit/Write 操作的文件路径必须是 worktree 内的路径，不是主仓库路径
- 如果启动子 agent 实施修复，必须明确告知其工作目录为 worktree，不得接触主仓库
- 如需对比主仓库代码做参考，read-only 可以，但绝不能 write
- 验证修复效果（build/run）也必须在 worktree 中执行

实施修复，遵循以下原则：
- **最小改动原则**：只修改必要文件，不要引入无关重构
- **保持代码风格**：与项目现有风格一致
- **更新所有相关位置**：如果问题是配置遗漏，检查是否有其他类似配置也需要同步更新

### 推荐工作方式

1. 先规划要修改的文件列表，一次性呈现给用户
2. 得到确认后再逐个修改
3. 每次修改后简单说明改了哪里

## Step 6: 编译验证（在 worktree 中执行）

提交前必须在 worktree 中验证修改可以成功编译，避免提交无法构建的代码。

```bash
# .NET 项目
dotnet build

# Node.js / TypeScript 项目
npm run build  # 或 npx tsc --noEmit

# Go 项目
go build ./...

# Python 项目（至少语法检查）
python -m py_compile <changed_files>
```

如果编译失败，修正编译错误后重新验证，直到通过构建。

## Step 7: 提交并创建 PR（在 worktree 中执行）

实施修复并验证编译通过后，**提交前需让用户确认修改内容，确认后**在 worktree 中提交代码、推送并创建 PR。

**所有 git 操作必须在 worktree 目录中执行。不要切换到主仓库执行 git 命令。**

```bash
# 将 PR body 写入临时文件，避免 heredoc 在 $() 中的兼容性问题
cat > /tmp/pr-body-<number>.md <<'EOF'
## 相关 Issue
Closes #<number>

## 修改内容
<修改描述>

## 测试方式
<验证步骤>
EOF

# 提交代码
git add <files>
git commit -m "fix: <issue-title>

Closes #<number>"

# 推送并创建 PR
git push -u origin "$BRANCH_NAME"
gh pr create \
  --repo <owner/repo> \
  --title "fix: <issue-title>" \
  --body "$(cat /tmp/pr-body-<number>.md)"

# 清理临时文件
rm /tmp/pr-body-<number>.md
```

提交后输出修复总结：

```
## 修复总结

### Issue
#<number> <title>

### 修改文件
- <path/to/file> — 改了什么
- ...

### PR
#<pr-number> <pr-url>

### 分支
<branch-name>

### 工作树
<worktree-path>
```

### 退出工作树

PR 创建完成后，使用 `ExitWorktree` 工具退出工作树：
- 如果用户还需要保留工作目录（例如后面可能继续在该分支上工作），使用 `action: "keep"`
- 如果确认后续不需要了，使用 `action: "remove"`，若工作树有未提交变更需先 `discard_changes: true`

退出前询问用户是保留还是删除工作树。

## 重要原则

1. **先分析再动手**：不要跳过代码分析直接改代码
2. **不信任 issue 中的根因**：亲自验证，issue 可能写错
3. **必须使用 worktree**：禁止在原始工作区直接操作分支，必须通过 `EnterWorktree` 创建隔离工作树进行修复，避免干扰用户当前工作
4. **提交前确认**：修改代码前让用户确认方案，提交前让用户确认改动
5. **编译验证**：提交前必须编译通过，不提交有编译错误的代码
6. **最小改动**：只改 issue 相关代码，不做额外重构或格式化
7. **如果发现更深层问题**：如实告知用户，让用户决定是否在当前修复中处理

## 后续操作

修复完成后，常见后续流程：
- 运行 `/pr-reviewer` 对修改进行代码审查
- 审查通过后运行 `/pr-resolver` 合并 PR（需要 CI 通过、无冲突）
- 如果是回归类型修复，可运行 `/issue-test-runner` 验证修复效果