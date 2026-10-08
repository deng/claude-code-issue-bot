# Claude Code Issue Bot

自动化 Issue 管道：从日志/Issue 到 PR 合并的完整闭环。8 个 Claude Code Skill，可独立使用，也可串成自动化生产线。

## 架构

```
ELK 日志系统
     │
     ▼
┌─────────────┐
│ log-catcher  │  ← 搜错 + Jev 决策（新增）
│ 搜错误日志   │
│ Jev 评估严重度│
│ noul≥0.65 → │
└──────┬──────┘
       │ 错误摘要
       ▼
┌─────────────┐
│issue-creator │  ← 分析 + 去重 + 建 Issue
│ LLM 分析堆栈 │
│ 代码验证     │
│ 创建 Issue   │
└──────┬──────┘
       │ Issue
       ▼
┌─────────────┐
│issue-sweeper │  ← 编排调度
│ 依赖排序     │
│ 串行驱动     │
└──┬──────┬───┘
   │      │
   │      └──────────────────┐
   ▼ fix分支                  ▼ test分支
┌──────────┐          ┌────────────────┐
│  resolver │          │test-runner     │
│  修复+提PR│          │ 运行测试+报告   │
└────┬─────┘          └────────┬───────┘
     ▼                         ▼
┌──────────┐              关闭 Issue
│ reviewer │
│ 代码审查  │
└────┬─────┘
     ▼
┌──────────┐
│ resolver │
│ 处理反馈  │
└────┬─────┘
     ▼
┌──────────┐
│  merge   │
│ 自动合并  │
└──────────┘
```

## 技能清单

### 链路层（日志→Issue）

| 技能 | 职责 | 输入 | 输出 |
|------|------|------|------|
| **log-catcher** 🆕 | 连接ELK搜错误日志、Jev评估严重度、达标调用issue-creator | ELK查询结果 | 错误摘要 + Issue URL |
| **issue-creator** | LLM分析堆栈+代码、去重检查、验证有效性、创建Issue | 错误摘要/问题描述 | GitHub Issue |

### 编排层

| 技能 | 职责 | 输入 | 输出 |
|------|------|------|------|
| **issue-sweeper** | 监控仓库、依赖排序、串行驱动完整管道 | 仓库地址 | 逐个关闭的 Issue |

### 修复管道

| 技能 | 职责 | 输入 | 输出 |
|------|------|------|------|
| **issue-resolver** | 读Issue→分析代码→定位根因→修复→编译验证→提PR | Issue URL | PR URL |
| **pr-reviewer** | 逐文件审查→提交Review（Approve/RequestChanges/Comment） | PR URL | Review 意见 |
| **pr-resolver** | 处理Review意见→按严重度修复或回复→防循环检测 | PR URL | 更新的 PR |
| **pr-merge** | CI检查→冲突解决→合并条件全满足后合并 | PR URL | 合并后的 main |

### 测试分支

| 技能 | 职责 | 输入 | 输出 |
|------|------|------|------|
| **issue-test-runner** | 执行可重复测试→报告写入 test-reports/ | Issue URL | PASS/FAIL/SKIP 报告 |

## 典型流程

### 全自动（两条 /loop 并行）

```bash
# 终端1：每10分钟搜ELK错误 → Jev评估 → 建Issue
/loop log-catcher --interval 10m

# 终端2：后台监控Issue → 自动修复合并
/loop issue-sweeper
```

```
终端1: log-catcher         终端2: issue-sweeper
  │                            │
  ├─ ELK搜错 ●                 ├─ 监控issues ●
  ├─ Jev评估 ●                 ├─ 发现#42 (新的!)
  ├─ noul=0.82 → 建Issue ────→├─ 依赖排序
  ├─ 记录状态                  ├─ STAGE1: resolver ●
  └─ 等10分钟...               ├─ STAGE2: reviewer ●
                               ├─ STAGE3: resolver ●
                               ├─ STAGE4: merge ●
                               ├─ #42 CLOSED ✅
                               └─ 扫描下一个...
```

### 手动触发

```bash
# 直接搜错+建Issue
Claude, /log-catcher 搜一下ELK最近1小时的ERROR

# 手动触发修复
Claude, /issue-resolver <issue-url>
```

## 安装

```bash
git clone https://github.com/deng/claude-code-issue-bot
cp claude-code-issue-bot/skills/*.md ~/.claude/skills/
```

## 前置依赖

```bash
# GitHub CLI
brew install gh && gh auth login

# Jev 决策模型（log-catcher 需要）
export JEVKEY="your_api_key"

# ELK 访问（log-catcher 需要）
export ES_HOST="https://elasticsearch.internal:9200"
```

## 许可

MIT