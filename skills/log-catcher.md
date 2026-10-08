---
name: log-catcher
description: |
  从 ELK（Elasticsearch）等日志系统搜索错误日志，Jev 评估严重度，满足条件时调用 issue-creator 自动建 Issue。
  触发词：日志监控、ELK搜错、log catcher、捕获错误、错误日志分析、log to issue。
compatibility:
  - curl (HTTP 请求 ELK/ES API)
  - python3 + jev_call.py (Jev 决策)
  - gh (GitHub CLI)
  - issue-creator skill
---

# Log Catcher — 日志捕获器

从分布式日志系统搜索错误，Jev 评估严重度，达标后自动喂入 issue-creator 建 Issue。
这是整个自动化排障管道的**第一条链路**。

## 架构定位

```
log-catcher → issue-creator → issue-sweeper → 修复管道
  (搜错+决策)   (分析+建Issue)   (编排调度)    (resolver→reviewer→resolver→merge)
```

log-catcher 只管两件事：**搜日志** + **评估要不要建 Issue**。分析和修复是下游 Skill 的事。

## 工作流程

### Step 1: 连接日志源搜索错误

支持多种日志源，通过配置切换：

```bash
# ELK (Elasticsearch) — 默认
curl -s -u "${ES_USER}:${ES_PASS}" \
  "${ES_HOST}/logs-*/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "bool": {
        "must": [
          {"match": {"level": "ERROR"}},
          {"range": {"@timestamp": {"gte": "now-1h"}}}
        ]
      }
    },
    "sort": [{"@timestamp": "desc"}],
    "size": 20
  }'

# Grafana Loki
curl -s -u "${LOKI_USER}:${LOKI_TOKEN}" \
  "${LOKI_HOST}/loki/api/v1/query_range" \
  --data-urlencode 'query={level="error"}' \
  --data-urlencode "start=$(date -u -v-1H +%s)000000000"

# 阿里云 SLS
aliyunlog get_logs --project=${PROJECT} --logstore=${LOGSTORE} \
  --query="level:ERROR" --from_time="1h ago"
```

### Step 2: 提取错误摘要

从日志搜索结果中提取结构化错误摘要：

```python
# 每条错误提取为：
{
  "id": "日志唯一ID",
  "timestamp": "2026-10-09T02:00:00Z",
  "level": "ERROR",
  "service": "user-service",
  "message": "NullPointerException at UserService.java:83",
  "stack_trace": "完整堆栈（如有）",
  "count": 15,             # 同一错误出现次数
  "first_seen": "...",     # 首次出现时间
  "host": "prod-node-3"
}
```

### Step 3: Jev 严重度评估

对每条唯一错误，调用 Jev 评估是否值得建 Issue：

```bash
python3 ~/.claude/skills/ai-jev/scripts/jev_call.py \
  --state "{\"error\":\"${MESSAGE}\",\"service\":\"${SERVICE}\",\"count\":${COUNT},\"stack\":\"${STACK_SUMMARY}\"}" \
  --questions '{"severity":{"type":"noul","instructions":"Should this error trigger an automated issue? Consider: error severity, frequency, user impact, data risk.","criteria":{"true":"Serious error — crash/NPE/data loss, high frequency, or user-facing impact. Worth investigating.","false":"Transient/benign — network timeout/rate limit, expected 4xx, or already handled by retry logic."}}}'
```

**决策规则**：
- `noul >= 0.65` → 自动创建 Issue（直接调用 issue-creator）
- `0.40 <= noul < 0.65` → 降权记录到 `.log-catcher/low_priority.json`，每天汇总一次人工确认
- `noul < 0.40` → 忽略，记录到 `ignored` 计数
- Jev 不可用 → 降级规则：按 `level:ERROR + count >= 3` 兜底判断

### Step 4: 调用 issue-creator 建 Issue

满足阈值后，将错误摘要喂入 issue-creator：

```
Claude, 用 issue-creator 处理以下日志错误：

服务: {service}
错误: {message}
堆栈: {stack_trace}
出现次数: {count}
首次出现: {first_seen}
Jev 评分: {noul_score}

请分析代码库验证后创建 Issue。
```

issue-creator 负责后续的：读代码 → 分析根因 → 去重 → 建 Issue。

### Step 5: 记录状态

```bash
echo "$(date -Iseconds) | ${ERROR_ID} | noul=${SCORE} | issue=${ISSUE_URL:-skipped}" \
  >> .log-catcher/history.log
```

### 完整示例

```bash
# 一次性运行：搜ELK错误 → 评估 → 建Issue
# 适合 cron 定时触发

export ES_HOST="https://elasticsearch.internal:9200"
export ES_USER="reader"
export ES_PASS="${ES_PASSWORD}"

# 搜错误
ERRORS=$(curl -s -u "$ES_USER:$ES_PASS" \
  "$ES_HOST/logs-*/_search" \
  -H "Content-Type: application/json" \
  -d '{"query":{"bool":{"must":[{"match":{"level":"ERROR"}},{"range":{"@timestamp":{"gte":"now-1h"}}}]}},"size":10}')

# 逐条评估
echo "$ERRORS" | jq -c '.hits.hits[]._source' | while read -r log; do
  MSG=$(echo "$log" | jq -r '.message')
  SVC=$(echo "$log" | jq -r '.service // "unknown"')
  CNT=$(echo "$log" | jq -r '.count // 1')

  # Jev 评估
  RESULT=$(python3 ~/.claude/skills/ai-jev/scripts/jev_call.py \
    --state "{\"error\":\"$MSG\",\"service\":\"$SVC\",\"count\":$CNT}" \
    --questions '{"severity":{"type":"noul","instructions":"Should this error trigger an automated issue?","criteria":{"true":"Crash/NPE/data loss/high frequency/user-facing","false":"Transient/benign/timeout/expected"}}}')

  NOUL=$(echo "$RESULT" | jq -r '.answers.severity.noul')

  if (( $(echo "$NOUL >= 0.65" | bc -l) )); then
    echo "[ISSUE] $SVC: $MSG (noul=$NOUL)"
    # 喂入 issue-creator
  elif (( $(echo "$NOUL >= 0.40" | bc -l) )); then
    echo "[PENDING] $SVC: $MSG (noul=$NOUL)"
  else
    echo "[IGNORE] $SVC: $MSG (noul=$NOUL)"
  fi
done
```

## 配置

环境变量（或 `.log-catcher/config` 文件）：

```bash
# 日志源
LOG_SOURCE=elk                # elk | loki | sls | cls
ES_HOST=https://...           # ELK 地址
ES_USER=reader
ES_PASS=xxx

# Jev 阈值
JEV_SEVERITY_MIN=0.65         # 自动建 Issue 阈值
JEV_PENDING_MIN=0.40          # 降权记录阈值

# 时间窗口
LOOKBACK_MINUTES=60           # 每次搜索的日志回溯时间

# 频率控制
MIN_COUNT=3                   # 同一错误最少出现次数才评估（降噪）
```

## /loop 模式

log-catcher 支持 `/loop` 后台运行，定时扫描日志：

```bash
/loop log-catcher --interval 10m
```

每 10 分钟：搜最近 1 小时 ERROR → 去重 → Jev 评估 → 建 Issue。
配合 issue-sweeper 的 `/loop`，形成完整无人值守管道：

```
log-catcher (/loop 10m)          issue-sweeper (/loop)
     │                                  │
     ├─ ELK搜错                         ├─ 监控 open issues
     ├─ Jev评估                         ├─ 依赖排序
     ├─ 达标 → issue-creator 建Issue ──→├─ 逐Issue串行修复
     └─ 记录状态                        └─ reviewer→resolver→merge
```

## 去重策略

避免同一错误重复建 Issue：

1. **基于错误指纹**：对 `service + exception_type + 堆栈前3帧` 做 hash
2. **检查已有 Issue**：`gh issue list --search "hash:${FINGERPRINT}"` 
3. **检查历史记录**：`.log-catcher/history.log` 中是否已处理
4. **同一错误频次聚合**：同一小时内的重复错误合并为一个评估，记录 `count`

## 底线规则

- Jev 不可用时降级为规则判断，不阻塞管道
- ELK 不可达时记录错误并退出，不等不重试
- 同一指纹的错误 24 小时内不重复建 Issue
- 已建 Issue 的错误，后续再次出现时更新 Issue 评论追加次数
- 所有决策记录到 `.log-catcher/history.log`，可审计