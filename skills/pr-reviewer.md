---
name: pr-reviewer
description: |
  PR Code Review 技能。当用户提交 PR 后需要 Code Review 时触发。客观、全面地审查 PR 代码变更，以 Reviewer 身份提交审核意见。

  触发方式：用户提供 PR 链接（如 https://github.com/owner/repo/pull/N）、输入 `/review-pr <url>`、或说"帮我 review 这个 PR"、"审核一下 PR"、"code review"、"看看这个 PR"、"review this PR"、"review please"等。
compatibility:
  - gh (GitHub CLI) authenticated
  - git
---

# PR Reviewer Skill

以 Reviewer 身份对 PR 进行 Code Review。流程：**解析 PR → 预检 → 获取变更 → 逐文件审查 → 输出审核报告 → 提交 Review**。

## Step 0: 解析 PR URL

从用户消息中提取 PR URL，格式为 `https://github.com/<owner>/<repo>/pull/<number>`。

```bash
# URL: https://github.com/deng/OneTeamAI/pull/5
# owner=deng, repo=OneTeamAI, number=5
```

如果用户只说"PR #5"，确认是哪个仓库后补充完整 URL。

## Step 0.5: Pre-Check — GitHub CLI Authentication

```bash
gh auth status 2>&1
```

如果认证失败或退出码非零，停止并告诉用户：
```
gh CLI 未认证。请先运行 `gh auth login` 登录 GitHub。
```

## Step 1: 获取 PR 详情

```bash
# PR 基本信息
gh pr view <number> --repo <owner>/<repo> --json title,body,state,author,additions,deletions,files,createdAt,labels,headRefName,baseRefName

# PR 关联的 issue（用于验证 PR 是否实现了它声称要解决的问题）
gh pr view <number> --repo <owner>/<repo> --json body | grep -oiE 'https://github.com/[^ )]+/(issues|pull)/[0-9]+|#[0-9]+' || echo "无关联 issue 链接"
```

读取以下信息：
- **标题和描述**：PR 要解决什么问题？
- **变更文件列表**：改了哪些文件，增删行数
- **基础分支和目标分支**：从哪里合并到哪里
- **关联 issue**：PR 是否链接了 issue？变更是否确实解决该 issue？

### 处理大 PR（防上下文溢出）

**禁止一次性 `gh pr diff` 全量输出。** 先看文件列表，判断规模：

- `additions + deletions <= 500` → 可以一次获取完整 diff
- `additions + deletions > 500` → **分文件获取 diff**，按文件逐个审查

```bash
# 大 PR：逐个文件获取 diff（不要一次全量）
gh pr view <number> --repo <owner>/<repo> --json files --jq '.files[].path' | while read f; do
  echo "===== $f ====="
  gh pr diff <number> --repo <owner>/<repo> --name-only 2>/dev/null
  # 或针对单个文件的 diff
  gh api "repos/<owner>/<repo>/pulls/<number>/files?per_page=100" --jq '.[] | select(.filename=="'$f'") | .patch'
done
```

优先审查：
1. 手写的业务逻辑文件（非自动生成、非锁文件）
2. issue/PR 描述中明确提到的文件
3. 高风险文件（涉及资金、权限、认证、数据迁移）

## Step 2: 逐文件审查

### 2a 先获取完整文件上下文（可选但推荐）

diff 只显示变更行，缺少前后文。对于核心变更文件，将 PR 分支 fetch 到本地 worktree，用 Read 阅读完整文件：

```bash
# 在临时 worktree 中阅读完整文件（read-only，不改任何代码）
git fetch origin "pull/<number>/head:pr-<number>" 2>/dev/null || git fetch origin "refs/pull/<number>/head"
# 用 git show 或直接 read 文件内容
```

如果不想创建 worktree，也可以用 `gh api` 拉取单个文件的完整内容：

```bash
gh api "repos/<owner>/<repo>/contents/<file-path>?ref=<headRefName>" --jq '.content' | base64 -d
```

### 2b 审查维度

对每个变更文件，按照以下维度审查：

#### 功能正确性
- 代码逻辑是否正确？是否实现了 PR 描述的功能？
- 是否真正解决了关联 issue 描述的问题？
- 边界条件和异常路径是否处理？
- 是否存在竞态条件或并发问题？

#### 安全性
- 是否存在注入风险（SQL、XSS、命令注入）？
- 敏感信息是否泄露（密钥、token、内部 URL）？
- 输入验证是否充分？
- 权限检查是否到位？

#### 性能
- 是否有不必要的重复计算或查询？
- 循环/递归是否存在性能风险？
- 是否有合适的缓存或延迟加载策略？

#### 代码质量
- 代码是否清晰可读？命名是否有意义？
- 是否遵循项目现有的代码风格和约定？
- 是否有重复代码可抽取复用？
- 错误处理是否恰当？

#### 可维护性
- 是否添加了必要的注释（为什么这么做而非做了什么）？
- 是否更新了相关配置、文档？
- 测试是否覆盖了主要变更路径？

### 2c 跳过审查的内容

以下文件类型**不需要逐行审查**，在报告中一句话说明跳过原因即可：
- 自动生成的文件（`*.generated.cs`、`*.g.cs`、`*.pb.go`、`*_pb2.py`、`dist/`、`build/` 产物）
- 锁文件（`package-lock.json`、`yarn.lock`、`go.sum`、`Cargo.lock`）— 仅确认依赖变更是否合理
- 纯格式化的变更（大量空行/缩进调整，无逻辑变化）
- 仅 bump 版本号的变更

## Step 3: 输出审核报告

采用以下格式输出报告：

```
## Code Review: #<number> <title>

### 总体评价
[Comment / Request Changes]

### 审查总结
[一段话概括本次 Review 的结论]

### 审查详情

#### 文件: <path/to/file>

**✅ 好的方面**
- ...

**⚠️ 问题/建议**
- [Critical/Major/Minor] 问题描述 — 建议修复方式

#### 文件: <path/to/file2>
...

### 综合建议
- [可选] 整体改进建议
- [可选] 后续需关注的方面
```

### 严重级别定义

| 级别 | 含义 | 行动 |
|------|------|------|
| **Critical** | 导致 Bug、安全漏洞或功能错误的 | 必须修复 |
| **Major** | 可能导致问题或严重违反最佳实践的 | 建议修复 |
| **Minor** | 风格、命名等非功能性建议 | 可选修复 |

## Step 4: 提交 Review

将报告写入临时文件再提交，**不要**假设报告内容存在于 shell 变量中：

```bash
# 将报告写入临时文件
cat > /tmp/review-body-<number>.md <<'EOF'
[完整的审核报告内容]
EOF
```

根据审查结果选择提交方式：

**存在 Critical/Major 问题** → `--request-changes`：

```bash
gh pr review <number> --repo <owner>/<repo> \
  --body "$(cat /tmp/review-body-<number>.md)" \
  --request-changes
```

**仅 Minor 建议或无问题** → `--comment`：

```bash
gh pr review <number> --repo <owner>/<repo> \
  --body "$(cat /tmp/review-body-<number>.md)" \
  --comment
```

提交后清理临时文件：

```bash
rm /tmp/review-body-<number>.md
```

如果 `gh pr review` 提交失败，报告错误给用户，不要静默丢弃审核结果。

## 重要原则

1. **客观公正**：基于代码事实，不要因人而异。好的要说好，有问题要指出
2. **有据可依**：每个问题都要说明为什么这是个问题，而不是主观偏好
3. **分清主次**：Critical/Major/Minor 要清晰区分，不要把所有意见混为一谈
4. **肯定优点**：好的设计、清晰的代码也要指出来，不要只报问题
5. **先整体后局部**：先理解 PR 的整体目标，再看具体实现细节
6. **上下文优先**：注意代码所在项目的约定和风格，不要用外部标准生搬硬套
7. **验证关联 issue**：PR 声称解决了某 issue，就要验证变更是否确实覆盖了该 issue 的全部要点
8. **不审查噪声**：自动生成文件、锁文件、纯格式化变更跳过逐行审查，把精力放在业务逻辑上