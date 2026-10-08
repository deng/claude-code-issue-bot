---
name: pr-merge
description: |
  有条件地自动合并 PR。检查 CI、merge conflict、change requests、review 记录等条件全部满足后执行合并。
  触发语境：用户要求合并 PR、"merge this PR"、"可以合了"、"条件满足的话帮我合掉"、"auto merge"等。
compatibility:
  - gh (GitHub CLI) authenticated
  - git
---

# PR Merge Skill

有条件地自动合并 PR。流程：**解析 PR → 逐项检查条件 → 全部通过则合并**。

## 合并流程

检查顺序：先检前四项条件（CI、Change Requests、Comment、Review 修复 Commit），最后检查 Mergeable —— 因为解决冲突后 CI 需要重新跑。

### 检查前四项条件

#### 条件 1: CI Checks 全部通过

```bash
gh pr view {number} --repo {owner}/{repo} --json statusCheckRollup --jq '.statusCheckRollup[] | {name, status, conclusion}'
```

- 所有 check 的 `conclusion` 为 `SUCCESS` → 通过
- 存在 check `conclusion` 为 `FAILURE` 或 `TIMED_OUT` → 不通过，列出失败项
- 存在 check `status` 为 `IN_PROGRESS` 或 `PENDING` → 不通过，列出还在跑的项

#### 条件 2: 无 Pending Change Requests

```bash
gh api "/repos/{owner}/{repo}/pulls/{number}/reviews" --jq '
  [.[] | select(.state == "CHANGES_REQUESTED")] | length'
```

- 返回 0 → 通过
- 返回 > 0 → 不通过，列出 change request 数量和来源

注意：只检查 `CHANGES_REQUESTED` 状态的 review，`COMMENTED` 和 `APPROVED` 不计入。

#### 条件 3: 至少一条 PR Comment

```bash
gh api "/repos/{owner}/{repo}/issues/{number}/comments" --jq 'length'
```

- 返回 >= 1 → 通过
- 返回 0 → 不通过，提示 PR 尚未被 review

注意：检查的是 issue comments（PR 主评论区），不含 inline review comments。

#### 条件 4: Review 反馈已有修复 Commit

检查 PR 上是否存在 review 之后的修复 commit，确保 review 反馈已被处理。

```bash
# 获取 latest review 时间
gh api "/repos/{owner}/{repo}/pulls/{number}/reviews" --jq '
  [.[] | select(.state != "PENDING" and .state != "DISMISSED")] | max_by(.submitted_at).submitted_at'

# 获取 PR latest commit 时间
gh pr view {number} --repo {owner}/{repo} --json commits --jq '
  .commits | last | .committedDate'
```

- 无有效 review（无 review 或所有 review 均为 `DISMISSED`/`PENDING`）→ 通过（无需修复反馈）
- 有有效 review 且 `latest commit > latest review`（review 之后有新 commit） → 通过
- 有有效 review 且 `latest commit <= latest review`（review 之后无新 commit） → **不通过**，提示：review 反馈尚未处理，请先推送修复 commit

注意：此条件不因 self-review 豁免。所有合并条件一视同仁。

### 检查 Mergeable 状态 + 冲突解决

前四项条件通过后，检查 mergeable 状态：

```bash
gh pr view {number} --repo {owner}/{repo} --json mergeable,headRefName,baseRefName --jq '{mergeable, headRefName, baseRefName}'
```

- `MERGEABLE` → 跳到合并执行
- `UNKNOWN` → 等待 10 秒后重试，最多 3 次
- `CONFLICTING` → 进入冲突解决

#### 冲突解决流程

1. **创建 worktree 并合并 base 分支：**

```bash
PR_NUMBER={number}
HEAD_REF=<headRefName>
BASE_REF=<baseRefName>
WORKTREE_DIR="../merge-conflict-${PR_NUMBER}"

git worktree add "$WORKTREE_DIR" "origin/${HEAD_REF}"
cd "$WORKTREE_DIR"
git fetch origin "${BASE_REF}"
git merge "origin/${BASE_REF}"  # 此时会报告冲突
```

2. **逐个解决冲突：**

对于每个冲突文件，阅读并理解双方改动：

```
<<<<<<< HEAD
PR 分支的代码
=======
base 分支的代码
>>>>>>> origin/main
```

**解决原则：**
- 以 PR 的修改意图为主，同时保留 base 分支的新变化
- 双方改同一行时，理解两个改动的意图后按需融合
- base 分支有重构时，调整 PR 代码适配新结构
- 解决后确保代码语法正确、逻辑合理

```bash
# 解决后标记已解决
git add <resolved-file>

# 确认无残留冲突标记
git diff --check
```

3. **完成合并并推送：**

```bash
# 无冲突标记残留后
git commit -m "Merge branch '${BASE_REF}' into '${HEAD_REF}' (resolve conflicts)

自动解决 PR #${PR_NUMBER} 的合并冲突。"
git push origin "${HEAD_REF}"
```

4. **清理 worktree：**

```bash
cd ..
git worktree remove "$WORKTREE_DIR"
```

5. **重新检查条件 1（CI）：**

冲突解决后 CI 需要重新跑，等待 CI 通过后再继续。如果 CI 不通过，停在当前状态并报告。

### 合并执行

```bash
gh pr merge {number} --repo {owner}/{repo} --merge --subject "<合并信息>"
```

合并成功后在 PR 中回复：

```
PR #${number} 已合并 ✅

合并条件检查：
- CI Checks 全部通过                    ✅
- 无 Pending Change Requests            ✅
- 至少一条 PR Comment                   ✅
- Review 反馈已有修复 Commit            ✅
- 冲突已自动解决                        ✅/-
```

## 任一条件不满足时

输出检查结果报告，不执行合并：

```
PR #${number} 合并条件未满足 ❌

- CI Checks 全部通过                    ✅/❌ <原因>
- 无 Pending Change Requests            ✅/❌ <原因>
- 至少一条 PR Comment                   ✅/❌ <原因>
- Review 反馈已有修复 Commit            ✅/❌ <原因>
- 冲突已解决                            ✅/❌ <原因>
```

## 依赖

- `gh` (GitHub CLI) — 需要 `repo` 权限
- `git` — 基础检出和合并操作