---
name: issue-creator
description: |
  Creates GitHub issues with analysis-first workflow. Triggers when the user wants to create an issue, file a bug, report a problem, suggest an improvement — in Chinese ("确认一下提个 issue" / "帮我提个 issue" / "要不要提个 issue") or English ("create an issue" / "file a bug" / "report this" / "open an issue" / "log a ticket").

  Automatically detects all Git repositories in the current workspace (main repo + submodules + nested repos), searches existing issues for duplicates, analyzes the codebase to validate the issue, determines which repo(s) are affected, presents the analysis for user confirmation, then creates the issue(s).

  Also trigger when the user describes something that should be tracked (a bug, missing feature, optimization) and asks what you think — suggest creating an issue and use this skill.

  Compatible with any project. Analyzes code regardless of language or framework.
compatibility:
  - gh (GitHub CLI) authenticated
  - git
---

# Issue Creator Skill

A structured workflow for creating GitHub issues. The workflow is: **check auth → detect repos → understand → search duplicates → analyze code → present → confirm → create**.

## Step 0: Pre-Check — GitHub CLI Authentication

Before anything else, verify `gh` is authenticated. All subsequent GitHub operations depend on this.

```bash
gh auth status 2>&1
```

If authentication fails or the exit code is non-zero, stop and tell the user:
```
gh CLI 未认证。请先运行 `gh auth login` 登录 GitHub。
```

Do not proceed to subsequent steps until authentication is confirmed.

## Step 1: Auto-Detect Repositories

Discover all Git repositories in the workspace:

```bash
# Find the root git repo
ROOT_REPO=$(git rev-parse --show-toplevel 2>/dev/null)

# Detect main repo remote
gh repo view --json nameWithOwner --jq '.nameWithOwner' 2>/dev/null

# Find all nested git repos (submodules or independent clones)
find "$ROOT_REPO" -name ".git" -type d -not -path "*/node_modules/*" -not -path "*/.dart_tool/*" -not -path "*/build/*" -not -path "*/.git/*" 2>/dev/null

# For each repo dir, get its remote
cd <repo-dir> && git remote get-url origin 2>/dev/null
```

Build a map of:
- Repo full name (e.g., `owner/repo`)
- Local path
- What the directory contains (by listing top-level files/dirs)

**Example output:**
```
📦 /Users/user/project (root)
  → github:owner/repo         → contains: src/, docs/, pubspec.yaml
  ├── 📦 packages/core/       → github:owner/core → contains: lib/, test/, pubspec.yaml
  └── 📦 extension/           → github:owner/ext  → contains: js/, manifest.json
```

## Step 2: Understand the Issue

Ask clarifying questions if the user's description is vague. Determine:

- What is the problem or desired improvement?
- Is it a bug, feature request, or optimization?
- What behavior do they expect?

## Step 3: Search Existing Issues (Duplicate Check)

Search ALL detected repos for similar open AND closed issues. Use multiple keyword phrasings:

```bash
gh issue list --repo <owner/repo> --state open --search "<keywords>" --limit 10
gh issue list --repo <owner/repo> --state closed --search "<keywords>" --limit 5
```

If you find potential duplicates, read them:
```bash
gh issue view <number> --repo <owner/repo>
```

## Step 4: Analyze Validity Against Codebase

Use the local codebase to evaluate the issue. This is where the analysis depth comes from:

### Search the code

**Detect the repo's language first, then use matching extensions** — do not hardcode a fixed list:

```bash
# Detect dominant language by file extension counts
find . -type f -not -path "*/node_modules/*" -not -path "*/build/*" -not -path "*/.git/*" \
  | sed 's/.*\.//' | sort | uniq -c | sort -rn | head -20

# Common extension sets per language:
# .NET/C#  → *.cs *.csproj *.csx *.razor *.json *.config *.yml
# TypeScript/JS → *.ts *.tsx *.js *.jsx *.json *.yml *.mjs *.cjs
# Python   → *.py *.pyi *.toml *.cfg *.yml
# Go       → *.go *.mod *.sum *.yml
# Java/Kotlin → *.java *.kt *.kts *.gradle *.xml *.yml
# Rust     → *.rs *.toml *.yml
# Dart     → *.dart *.yaml
# Ruby     → *.rb *.rake *.gemspec *.yml

# Search for code patterns (use the detected extensions)
grep -rn "<pattern>" --include="<ext1>" --include="<ext2>" . 2>/dev/null | head -30

# Find relevant files by name
find . -name "*<keyword>*" -type f 2>/dev/null

# Check imports/dependencies
grep -rn "import.*<package>" --include="<detected-ext>" . 2>/dev/null | head -20
```

### Read relevant files
Use the Read tool to examine:
- The specific widget/page/service mentioned
- Related models and controllers
- Any recent changes (git log for relevant files)

### Check repository metadata

```bash
# Check available labels (for use in issue creation)
gh label list --repo <owner/repo> --limit 50 2>/dev/null

# Check available milestones
gh api repos/<owner>/<repo>/milestones --jq '.[].title' 2>/dev/null
```

Look for labels that match the issue type (`bug`, `enhancement`, `documentation`, etc.) and suggest them in the analysis.

### Validate
- **Is it already implemented?** — check if the feature/fix already exists in the code
- **Is the user's description accurate?** — verify against actual code behavior, don't take at face value
- **Is it technically feasible?** — given the architecture and dependencies
- **What's the root cause?** — trace through the code path to find where the bug or missing feature originates

### Determine Affected Repos

Based on where the root cause and fix need to go:

- If the code lives in a subdirectory with its own git remote → that subdirectory's repo
- If the code is in the root repo → the root repo
- A single issue might affect multiple repos (e.g., backend changes + frontend changes)

## Step 5: Present Analysis to User

Format your findings clearly:

```
## 分析结果

### 检测到的仓库
- [ ] <owner/repo> — <description>

### 影响仓库
- [ ] <owner/repo> — <reason>

### 重复检查
- <owner/repo>: [#N 标题] — [相关/不重复/重复]
- ...

### 建议标签
- <owner/repo>: `bug` / `enhancement` / ... — <reason>

### 有效性评估
- [验证结果说明]

### Issue 草稿
[每个受影响仓库的 issue 内容，包含 title、body、建议 label]
```

Ask: "以上分析是否准确？确认后我将创建对应的 issue。"

### Multi-Repo Guidance

When issues span multiple repos:

1. **Create issues sequentially** — create the core/dependency issue first, then the dependent one
2. **Cross-reference** — in the second issue's body, link to the first: `Depends on: <owner/repo>#<number>`
3. **Present the order clearly** — tell the user which gets created first and why
4. **Confirm each** — user must confirm before each issue is created (not a single confirmation for all)

## Step 6: Create Issues (on User Confirmation)

Only proceed after the user explicitly confirms.

```bash
gh issue create \
  --repo <owner/repo> \
  --title "<issue title>" \
  --body "<issue body>" \
  --label "<label1>,<label2>"  # optional, if labels detected and user agreed
```

If the command fails (network error, rate limit, etc.), report the error and ask the user how to proceed (retry / skip / manual creation). Do not silently drop the failure.

## Step 7: Output Results

```
✅ <owner/repo>: https://github.com/<owner/repo>/issues/<number>
✅ <owner/repo>: https://github.com/<owner/repo>/issues/<number>
```

If any creation failed, report it explicitly with the error and what was already created.

## Important Notes

- **Never skip the analysis phase.** Always search existing issues and examine the code before telling the user whether the issue is valid.
- **Never create an issue without user confirmation.** The presentation step is mandatory.
- **If an issue is clearly a duplicate**, tell the user which existing issue covers it and suggest closing.
- **If an issue doesn't make sense** (already implemented, technically impossible, user misunderstood), explain why clearly and let the user decide whether to proceed.
- **Use `gh` CLI** for all GitHub operations. Do not use the GitHub API directly.
- **Analysis depth over speed.** Read the relevant files, understand the code path, and produce a thorough analysis. The skill should provide genuine value beyond what the user could do manually.
- **For imports and package references**, when you encounter `import 'package:...'` paths, use that to understand which package a file belongs to and how the modules are connected - this helps determine which repo is affected.
- **Always check `gh auth status` first** — if GitHub CLI is not authenticated, stop and instruct the user to log in. All subsequent steps depend on this.