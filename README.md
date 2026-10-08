# Claude Code Issue Bot

自动化 Issue 管道：从日志/Issue 到 PR 合并的完整闭环。7 个 Claude Code Skill，可独立使用，也可串成自动化生产线。

## 架构

```
日志/告警 → issue-creator → issue-sweeper → ┌─ test分支: issue-test-runner → 关闭
                                              │
                                              └─ fix分支: issue-resolver → pr-reviewer
                                                           ↑                      │
                                                           └─ pr-resolver ←───────┘
                                                                         │
                                                                   pr-merge → ✅
```

## 技能清单

### 入口层

| 技能 | 职责 | 输入 | 输出 |
|------|------|------|------|
| **issue-creator** | 分析问题、去重检查、创建 Issue | 问题描述（自然语言/日志摘要） | GitHub Issue |

### 编排层

| 技能 | 职责 | 输入 | 输出 |
|------|------|------|------|
| **issue-sweeper** | 监控仓库、优先级排序、串行驱动完整管道 | 仓库地址 | 逐个关闭的 Issue |

### 修复管道

| 技能 | 职责 | 输入 | 输出 |
|------|------|------|------|
| **issue-resolver** | 读 Issue → 分析代码 → 定位根因 → 修复代码 → 编译验证 → 提 PR | Issue URL | PR URL |
| **pr-reviewer** | 逐文件审查 → 提交 Review（Approve/RequestChanges/Comment） | PR URL | Review 意见 |
| **pr-resolver** | 按严重程度处理 Review 意见 → 修复或回复 → 防循环 | PR URL | 更新的 PR |
| **pr-merge** | CI 检查 → 冲突解决 → 合并条件全满足后合并 | PR URL | 合并后的 main |

### 测试分支

| 技能 | 职责 | 输入 | 输出 |
|------|------|------|------|
| **issue-test-runner** | 执行可重复测试 → 报告写入 test-reports/ | Issue URL | PASS/FAIL/SKIP 报告 |

## 典型流程

### 模式一：手动触发

```bash
# 1. 发现问题，创建 Issue
Claude, 用 issue-creator 帮我分析这个报错日志 → 创建 Issue

# 2. 手动触发修复（或让 sweeper 自动接管）
Claude, /issue-resolver <issue-url>
```

### 模式二：全自动（/loop 驱动）

```bash
# 启动 sweeper，它监控仓库、自动处理每个 Issue
/loop issue-sweeper
```

sweeper 内部状态机：
1. **PR CHECK** → 先处理已有 PR
2. **SCAN** → 列出所有 open issue
3. **CLASSIFY** → test: 走测试分支 / 其他走修复分支
4. **STAGE 1** → issue-resolver（分析+修复+提PR）
5. **STAGE 2** → pr-reviewer（代码审查）
6. **STAGE 2.5** → pr-resolver（处理 review 意见，必要时循环）
7. **STAGE 3** → pr-merge（条件满足则合并）
8. **VERIFY** → 确认 issue 已关闭，继续下一个

### 模式三：日志驱动（结合 Jev）

```
日志系统搜索错误 → LLM 分析日志+相关代码 → Jev 评估严重度 → 满足阈值则 issue-creator 建 Issue → sweeper 接管
```

Jev (TypeSafe System One) 判断：noul ≥ 0.65 → 自动建 Issue；0.40-0.64 → 降权标记；< 0.40 → 忽略。

## 安装

将 `skills/` 目录下的 `.md` 文件复制到你的 `~/.claude/skills/` 目录：

```bash
cp skills/*.md ~/.claude/skills/
```

## 前置依赖

```bash
# GitHub CLI（所有技能）
brew install gh
gh auth login

# 可选：Jev 决策模型（日志→Issue 自动化评分）
export JEVKEY="your_api_key"
```

## 存档位置

源仓库：`~/.claude/skills/`

## 许可

MIT