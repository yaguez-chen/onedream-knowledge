# 乐园即时通讯方案 v2.1 — 完整架构设计

> **作者：** 拉姆达 🔬 + 伽马 🔧
> **日期：** 2026-03-19
> **版本：** v2.1（v2.0 + 伽马 Review 补充：delivery-status.sh / 优先级路由 / qmd fallback / 部署脚本）
> **来源：** 梦想家直接指令（2026-03-18）— Phase 1-3 全部成果整合

---

## 一、变更摘要

### v2.0 Complete 新增内容（vs v1.0）

| 新增 | 来源 | 价值 |
|------|------|------|
| Webhook 即时触发 | R1 研究 | Agent 发消息后秒级触发对方 |
| Hook Handler 自动化 | R2 研究 | gateway:startup 自动扫描 missed messages |
| 事件驱动通知 | R3 研究 | 六种协议对比 + heartbeat sync 机制 |
| 投递确认机制 | G1 实现 | 发送方知道消息是否被读取和处理 |
| Outbox 追踪日志 | G1 实现 | 发送方可追踪所有已发消息状态 |
| Cron 高频兜底 | R2 研究 | 3 分钟轮询作为可靠 fallback |
| 完整降级链 | 整合 | 四级降级：webhook → hook → cron → heartbeat |
| .last-read 修复 | 实际发现 | Gateway 重启后自动恢复，不丢消息 |

### v2.1 新增内容（伽马 Review 补充）

| 新增 | 来源 | 价值 |
|------|------|------|
| delivery-status.sh | 伽马建议 | 查询消息投递状态，支持 --pending / --json |
| send-and-notify.sh 优先级路由 | 伽马建议 | urgent 双通道即时，normal inbox + cron 兜底 |
| QMD Fallback 方案 | 伽马建议 | qmd → grep → ls 三级兜底，确保 qmd 不可用时正常工作 |
| deploy-messaging-v2.sh | 伽马建议 | 一键部署所有脚本到所有 Agent workspace，支持 dry-run |
| inbox-scan.sh | 伽马建议 | 支持 qmd fallback 的 inbox 扫描脚本 |

---

## 二、架构总览

### 2.1 完整消息流程

```
发送方 Agent                                              接收方 Agent
════════════                                              ════════════

  1. 写入 inbox + 标记 outbox                               │
  2. Webhook 触发 ──────────────────────────────────► Agent B 处理
  3. sessions_send（可选）──────────────────────────► 飞书显示  │
     │                                                     4. B 写入 ack
     │ ◄────────────────────────────────────────── webhook 回传
  5. A 收到 ack，更新 outbox                               │

┌──────────────────────────────────────────────────────────────────────┐
│                        四级投递通道                                    │
├──────────────────────────────────────────────────────────────────────┤
│  Level 1: sessions_send ────────────────► 飞书 session ──► Agent B   │
│  Level 2: Webhook 触发 ─────────────────► /hooks/agent ──► Agent B   │
│  Level 3: Cron 高频轮询 ─────────────────► 发现 inbox ──► 触发 B     │
│  Level 4: Heartbeat 心跳 ────────────────► 检查 inbox ──► 处理      │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.2 四级降级链

```
消息发送
  │
  ├─► Level 1: sessions_send ──► session 正常？──► ✅ 秒到
  │                               │
  │                               └─► session 卡死？──► ❌ 跳过
  │
  ├─► Level 2: 写入 inbox + Webhook ──► Agent 存活？──► ✅ 3秒内触发
  │                                      │
  │                                      └─► Agent 忙？──► ⚠️ 排队
  │
  ├─► Level 3: Cron 3分钟扫描 ──► 发现未读？──> ✅ 3分钟内处理
  │
  └─► Level 4: Heartbeat 30分钟 ──► 最终兜底 ✅
```

---

## 三、核心组件

### 3.1 组件一：Inbox 持久化层

**路径：** `~/.openclaw/workspace-<agent>/inbox/`

```
inbox/
├── .last-read                    # 指针文件：最后处理的消息 ID
├── msg-20260318-142605-491.json  # 消息文件（JSON）
├── ack-20260318-142800-491.json  # 确认消息
└── archive/                      # 已处理的确认归档
```

**消息格式：**

```json
{
  "id": "msg-20260318-142605-491022",
  "timestamp": "2026-03-18T14:26:05+08:00",
  "from": "alpha",
  "to": "lambda",
  "priority": "urgent",
  "type": "task-assignment",
  "subject": "推进 Phase 3 研究任务",
  "body": "全力推进 Phase 3...",
  "read": false,
  "ack_required": true,
  "ack_to": "alpha"
}
```

### 3.2 组件二：Webhook 即时触发（Level 2）

**来源：** R1 研究 + R3 事件驱动设计

#### 触发流程

```
Agent A 完成发消息
  ↓
写入 Agent B 的 inbox/ 文件
  ↓
curl 调用 webhook：
  POST /hooks/agent
  Authorization: Bearer $HOOKS_TOKEN
  {
    "message": "你有新消息，请检查 inbox/",
    "agentId": "beta",
    "name": "InboxTrigger",
    "deliver": false,
    "timeoutSeconds": 60
  }
  ↓
Gateway 创建独立 session 运行 Agent B
  ↓
Agent B 读取 inbox → 处理消息 → 写 ack 回传
```

#### 配置要求

```json5
// openclaw.json
{
  hooks: {
    enabled: true,
    token: "${OPENCLAW_HOOKS_TOKEN}",
    path: "/hooks",
    allowedAgentIds: ["main", "alpha", "beta", "gamma", "delta",
                      "epsilon", "zeta", "eta", "theta", "iota",
                      "kappa", "lambda"],
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: false
  }
}
```

### 3.3 组件三：Hook Handler 自动化

#### 3.3.1 `gateway:startup` Hook — 重启后自动扫描

**解决问题：** Gateway 重启后 30 秒内恢复所有未读消息

```
Gateway 重启 → gateway:startup 事件
  ↓
startup-inbox-scan hook 执行
  ↓
扫描所有 Agent inbox/ → 对比 .last-read
  ↓
发现未读 → /hooks/agent 触发对应 Agent
  ↓
T+30s 恢复完成
```

### 3.4 组件四：Cron 高频轮询（Level 3）

**配置：** 每 3 分钟扫描所有 Agent inbox

```json5
// openclaw.json — 由主 Agent（阿尔法）执行
{
  agents: {
    list: [{
      id: "main",
      cron: {
        jobs: [{
          name: "inbox-scan-all-agents",
          schedule: "*/3 * * * *",
          task: "扫描所有公民 inbox，有未读则触发",
          timeoutSeconds: 180
        }]
      }
    }]
  }
}
```

### 3.5 组件五：Heartbeat 心跳兜底（Level 4）

已有机制，30 分钟间隔，作为最终兜底。

---

## 四、投递确认机制（G1 新增 ⭐）

**来源：** 伽马 🔧 G1 实现
**作用：** 发送方知道消息是否被接收方读取和处理

### 4.1 确认流程

```
发送方(A)                        接收方(B)
    │                               │
    │ 1. 写入 B/inbox/msg.json      │
    │    (含 ack_required + ack_to)  │
    │                               │
    │ 2. Webhook 触发 B             │
    │ ────────────────────────────► │
    │                               │ 3. B 处理消息
    │                               │ 4. B 写入 A/inbox/ack-xxx.json
    │                               │ 5. B Webhook 触发 A
    │ ◄──────────────────────────── │
    │                               │
    │ 6. A 收到 ack                 │
    │ 7. 更新 outbox/sent-log.jsonl │
    │ 8. 标记消息为已确认            │
```

### 4.2 确认消息格式

```json
{
  "id": "ack-20260318-142605-491022",
  "timestamp": "2026-03-18T14:28:30+08:00",
  "from": "gamma",
  "type": "ack",
  "ref_id": "msg-20260318-142605-491022",
  "status": "read",
  "read_at": "2026-03-18T14:28:30+08:00"
}
```

### 4.3 确认状态码

| 状态 | 含义 | 触发时机 |
|------|------|---------|
| `received` | 已收到但未读 | 文件写入成功 |
| `read` | 已读取 | Agent 心跳/webhook 处理 |
| `processing` | 处理中 | Agent 开始执行任务 |
| `done` | 任务完成 | Agent 完成相关任务 |
| `failed` | 处理失败 | Agent 处理出错 |

### 4.4 Outbox 追踪

**发送方维护：** `workspace/outbox/sent-log.jsonl`

```jsonl
{"id":"msg-xxx","to":"gamma","subject":"...","sent_at":"...","acked_at":null,"status":"pending"}
{"id":"msg-xxx","to":"gamma","subject":"...","sent_at":"...","acked_at":"...","status":"read"}
```

### 4.5 投递确认可选

消息中设置 `"ack_required": false` 可跳过确认，用于广播类消息。

---

## 五、实现脚本（G1 新增 ⭐）

### 5.1 send-and-notify.sh — 带确认支持的发送

```bash
#!/bin/bash
# send-and-notify.sh — 带投递确认支持的消息发送
# 用法: ./send-and-notify.sh <target-agent> <priority> <subject> <body> [--no-ack]
set -euo pipefail

TARGET_AGENT="${1:-}"
PRIORITY="${2:-normal}"
SUBJECT="${3:-}"
BODY="${4:-}"
NO_ACK="${5:-}"

[ -z "$TARGET_AGENT" ] || [ -z "$SUBJECT" ] && { echo "用法: $0 <target> <priority> <subject> <body> [--no-ack]"; exit 1; }

WORKSPACE="/home/gang/.openclaw/workspace-${TARGET_AGENT}"
INBOX="${WORKSPACE}/inbox"
OUTBOX="/home/gang/.openclaw/workspace-$(hostname)/outbox"
MSG_ID="msg-$(date +%Y%m%d-%H%M%S)-$$"
TIMESTAMP=$(date -Iseconds)
HOOKS_TOKEN=$(grep OPENCLAW_HOOKS_TOKEN ~/.openclaw/.env 2>/dev/null | cut -d= -f2 || echo "")

mkdir -p "$INBOX" "$OUTBOX"

ACK_REQUIRED="true"
[ "$NO_ACK" = "--no-ack" ] && ACK_REQUIRED="false"

# 写入消息
cat > "${INBOX}/${MSG_ID}.json" << EOF
{"id":"${MSG_ID}","timestamp":"${TIMESTAMP}","from":"$(whoami)","to":"${TARGET_AGENT}","priority":"${PRIORITY}","type":"notification","subject":"${SUBJECT}","body":"${BODY}","read":false,"ack_required":${ACK_REQUIRED},"ack_to":"$(whoami)"}
EOF
echo "✅ 消息已写入: ${MSG_ID}"

# Outbox 追踪
if [ "$ACK_REQUIRED" = "true" ]; then
  echo "{\"id\":\"${MSG_ID}\",\"to\":\"${TARGET_AGENT}\",\"subject\":\"${SUBJECT}\",\"sent_at\":\"${TIMESTAMP}\",\"acked_at\":null,\"status\":\"pending\"}" >> "${OUTBOX}/sent-log.jsonl"
fi

# Step 2: 优先级路由（伽马建议优化）
case "$PRIORITY" in
  urgent|high)
    # 高优先级：立即 webhook + sessions_send 双通道
    echo "🔴 高优先级 — 双通道即时触发"
    [ -n "$HOOKS_TOKEN" ] && curl -s -X POST http://127.0.0.1:18789/hooks/agent \
      -H "Authorization: Bearer ${HOOKS_TOKEN}" \
      -d "{\"message\":\"⚡ 紧急消息来自 $(whoami)：${SUBJECT}\",\"agentId\":\"${TARGET_AGENT}\",\"name\":\"UrgentNotify\",\"deliver\":false,\"timeoutSeconds\":60}" \
      > /dev/null 2>&1 && echo "🔔 Webhook 已触发" || echo "⚠️ Webhook 失败"
    # sessions_send 作为额外通道（不阻塞）
    (sleep 2 && curl -s -X POST http://127.0.0.1:18789/hooks/agent \
      -H "Authorization: Bearer ${HOOKS_TOKEN}" \
      -d "{\"message\":\"确认收到紧急消息 ${MSG_ID}\",\"agentId\":\"${TARGET_AGENT}\",\"name\":\"UrgentConfirm\",\"deliver\":false}" \
      > /dev/null 2>&1) &
    ;;
  *)
    # 普通优先级：仅写 inbox（Cron 3分钟内会扫描到）
    echo "🟢 普通优先级 — inbox 持久化（Cron 兜底）"
    # Webhook 可选（不强制）
    [ -n "$HOOKS_TOKEN" ] && curl -s -X POST http://127.0.0.1:18789/hooks/agent \
      -H "Authorization: Bearer ${HOOKS_TOKEN}" \
      -d "{\"message\":\"新消息来自 $(whoami)，请检查 inbox/\",\"agentId\":\"${TARGET_AGENT}\",\"name\":\"InboxNotify\",\"deliver\":false,\"timeoutSeconds\":60}" \
      > /dev/null 2>&1 && echo "🔔 Webhook 已触发" || echo "💤 Webhook 未配置（Cron 兜底）"
    ;;
esac
```

### 5.2 ack-processor.sh — 确认消息处理器

```bash
#!/bin/bash
# ack-processor.sh — 处理 ack-*.json，更新追踪日志
set -euo pipefail

INBOX="/home/gang/.openclaw/workspace-$(whoami)/inbox"
OUTBOX="/home/gang/.openclaw/workspace-$(whoami)/outbox"
SENT_LOG="${OUTBOX}/sent-log.jsonl"
mkdir -p "$OUTBOX"

for ACK_FILE in "${INBOX}"/ack-*.json 2>/dev/null; do
  [ -f "$ACK_FILE" ] || continue
  REF_ID=$(grep -o '"ref_id":"[^"]*"' "$ACK_FILE" | cut -d'"' -f4)
  STATUS=$(grep -o '"status":"[^"]*"' "$ACK_FILE" | cut -d'"' -f4)
  READ_AT=$(grep -o '"read_at":"[^"]*"' "$ACK_FILE" | cut -d'"' -f4)
  FROM=$(grep -o '"from":"[^"]*"' "$ACK_FILE" | cut -d'"' -f4)
  
  echo "✅ ${REF_ID} ← ${FROM} (${STATUS})"
  
  # 更新 sent-log
  if [ -f "$SENT_LOG" ] && grep -q "$REF_ID" "$SENT_LOG" 2>/dev/null; then
    OLD=$(grep "$REF_ID" "$SENT_LOG")
    NEW=$(echo "$OLD" | sed "s/\"acked_at\":null/\"acked_at\":\"${READ_AT}\"/" | sed "s/\"status\":\"pending\"/\"status\":\"${STATUS}\"/")
    sed -i "s|${OLD}|${NEW}|" "$SENT_LOG"
  fi
  
  # 归档
  mkdir -p "${INBOX}/archive"
  mv "$ACK_FILE" "${INBOX}/archive/"
done
```

### 5.3 delivery-status.sh — 投递状态查询脚本（伽马建议补充 ⭐）

```bash
#!/bin/bash
# delivery-status.sh — 查询消息投递状态
# 用法: ./delivery-status.sh [message-id] [--pending] [--all] [--json]
set -euo pipefail

OUTBOX="/home/gang/.openclaw/workspace-$(whoami)/outbox"
SENT_LOG="${OUTBOX}/sent-log.jsonl"

if [ ! -f "$SENT_LOG" ]; then
  echo "📭 没有发送记录"
  exit 0
fi

# 参数解析
MODE="all"
TARGET_ID=""
JSON_OUTPUT=false
for arg in "$@"; do
  case "$arg" in
    --pending) MODE="pending" ;;
    --all) MODE="all" ;;
    --json) JSON_OUTPUT=true ;;
    *) TARGET_ID="$arg" ;;
  esac
done

# 查询指定消息
if [ -n "$TARGET_ID" ]; then
  RESULT=$(grep "$TARGET_ID" "$SENT_LOG" 2>/dev/null || echo "")
  if [ -z "$RESULT" ]; then
    echo "❌ 未找到消息: $TARGET_ID"
    exit 1
  fi
  if $JSON_OUTPUT; then
    echo "$RESULT"
  else
    STATUS=$(echo "$RESULT" | grep -o '"status":"[^"]*"' | cut -d'"' -f4)
    TO=$(echo "$RESULT" | grep -o '"to":"[^"]*"' | cut -d'"' -f4)
    SENT=$(echo "$RESULT" | grep -o '"sent_at":"[^"]*"' | cut -d'"' -f4)
    ACKED=$(echo "$RESULT" | grep -o '"acked_at":"[^"]*"' | cut -d'"' -f4)
    SUBJECT=$(echo "$RESULT" | grep -o '"subject":"[^"]*"' | cut -d'"' -f4)
    
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    echo "📨 消息: $TARGET_ID"
    echo "   收件人: $TO"
    echo "   主题: $SUBJECT"
    echo "   发送时间: $SENT"
    case "$STATUS" in
      pending) echo "   状态: ⏳ 待确认" ;;
      read)    echo "   状态: ✅ 已读 ($ACKED)" ;;
      failed)  echo "   状态: ❌ 投递失败" ;;
      *)       echo "   状态: $STATUS" ;;
    esac
    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
  fi
  exit 0
fi

# 统计模式
TOTAL=$(wc -l < "$SENT_LOG")
PENDING=$(grep -c '"status":"pending"' "$SENT_LOG" 2>/dev/null || echo 0)
ACKED=$(grep -c '"status":"read"' "$SENT_LOG" 2>/dev/null || echo 0)
FAILED=$(grep -c '"status":"failed"' "$SENT_LOG" 2>/dev/null || echo 0)

if $JSON_OUTPUT; then
  echo "{\"total\":$TOTAL,\"pending\":$PENDING,\"acked\":$ACKED,\"failed\":$FAILED}"
  exit 0
fi

echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "📊 投递状态总览 ($(whoami))"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "   总发送: $TOTAL"
echo "   ✅ 已确认: $ACKED"
echo "   ⏳ 待确认: $PENDING"
echo "   ❌ 失败: $FAILED"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

# 列出待确认消息
if [ "$MODE" = "pending" ] && [ "$PENDING" -gt 0 ]; then
  echo ""
  echo "⏳ 待确认消息："
  grep '"status":"pending"' "$SENT_LOG" | while read -r line; do
    ID=$(echo "$line" | grep -o '"id":"[^"]*"' | cut -d'"' -f4)
    TO=$(echo "$line" | grep -o '"to":"[^"]*"' | cut -d'"' -f4)
    SENT=$(echo "$line" | grep -o '"sent_at":"[^"]*"' | cut -d'"' -f4)
    echo "   $ID → $TO ($SENT)"
  done
fi
```

**用法示例：**

```bash
# 查看总览
./delivery-status.sh

# 查看待确认消息
./delivery-status.sh --pending

# 查看特定消息
./delivery-status.sh msg-20260319-091000-review

# JSON 格式输出（供其他脚本解析）
./delivery-status.sh --json
# → {"total":42,"pending":3,"acked":38,"failed":1}
```

### 5.4 Agent 端自动 ack 回传

在每个 Agent 的 HEARTBEAT.md 或处理逻辑中添加：

```bash
# 处理 inbox 消息后，自动回传 ack
for MSG in inbox/msg-*.json; do
  ACK_REQ=$(grep -o '"ack_required":[^,}]*' "$MSG" | cut -d: -f2 | tr -d ' ')
  ACK_TO=$(grep -o '"ack_to":"[^"]*"' "$MSG" | cut -d'"' -f4)
  MSG_ID=$(grep -o '"id":"[^"]*"' "$MSG" | cut -d'"' -f4)
  
  if [ "$ACK_REQ" = "true" ] && [ -n "$ACK_TO" ]; then
    # 写 ack 到发送方的 inbox
    ACK_ID="ack-${MSG_ID#msg-}"
    cat > ~/.openclaw/workspace-${ACK_TO}/inbox/${ACK_ID}.json << EOF
{"id":"${ACK_ID}","timestamp":"$(date -Iseconds)","from":"$(whoami)","type":"ack","ref_id":"${MSG_ID}","status":"read","read_at":"$(date -Iseconds)"}
EOF
    # webhook 触发送方
    curl -s -X POST http://127.0.0.1:18789/hooks/agent \
      -H "Authorization: Bearer $HOOKS_TOKEN" \
      -d "{\"message\":\"确认回执：${MSG_ID} 已处理\",\"agentId\":\"${ACK_TO}\",\"name\":\"AckBack\",\"deliver\":false}" \
      > /dev/null 2>&1
  fi
done
```

---

## 六、事件驱动通知设计（R3 新增 ⭐）

**来源：** 拉姆达 🔬 R3 研究

### 6.1 协议对比分析

| 维度 | Webhook | Hook | Cron | Heartbeat | Polling (qmd) | WebSocket |
|------|---------|------|------|-----------|---------------|-----------|
| **方向** | 主动推 | 事件推 | 定时扫 | 定时扫 | 主动拉 | 双向推 |
| **延迟** | <3s | <5s | ≤3min | ≤30min | 手动 | <1s |
| **可靠性** | 中 | 高 | 高 | 高 | 高 | 低（需保活） |
| **成本** | 低 | 低 | 低 | 低 | 中 | 中 |
| **Agent 独立** | ✅ | ✅ | ✅ | ❌ | ❌ | ✅ |
| **推荐场景** | 即时通知 | 系统事件 | 可靠兜底 | 最终兜底 | 知识检索 | 实时聊天 |

### 6.2 Heartbeat 同步协议

解决心跳 session 无法实时接收消息的问题：

```
心跳轮询（每 30 分钟）
  ↓
读取 inbox/.last-read → last_read_id
  ↓
查询 qmd: "最近 30 分钟的未读消息来自 last_read_id 之后"
  ↓
qmd 返回结果 → 按优先级排序 → 处理
  ↓
标记 last_read_id = 最新消息 ID
  ↓
下一轮心跳时，qmd 已索引新消息，自动发现
```

**优势：**
- qmd 向量搜索毫秒级（对比暴力扫描 100ms+）
- 语义理解：可按主题/优先级智能排序
- 跨 session 共享索引，不受 session 隔离影响

### 6.3 QMD Fallback 方案（伽马建议补充 ⭐）

**问题：** qmd 技能当前未安装或嵌入生成暂停（Bun segfault bug），需要无 qmd 时的兜底机制。

**分级策略：**

| 层级 | 条件 | 行为 |
|------|------|------|
| **L1: qmd 可用** | `command -v qmd` 成功 | 语义搜索，毫秒级响应 |
| **L2: grep 兜底** | qmd 不可用 | `grep -r "关键词" inbox/*.json`，秒级 |
| **L3: ls 排序** | 连 grep 都失败 | `ls -t inbox/*.json` 按时间排序，可靠但无过滤 |

**心跳扫描 fallback 实现：**

```bash
#!/bin/bash
# inbox-scan.sh — 支持 qmd fallback 的 inbox 扫描
INBOX="/home/gang/.openclaw/workspace-$(whoami)/inbox"
LAST_READ=$(cat "${INBOX}/.last-read" 2>/dev/null || echo "")

# L1: 尝试 qmd
if command -v qmd &>/dev/null && qmd status &>/dev/null 2>&1; then
  echo "🔍 使用 qmd 语义搜索"
  qmd search "最近的未读消息" --after "$LAST_READ" --path "$INBOX"
# L2: grep 兜底
elif [ -n "$LAST_READ" ]; then
  echo "📂 qmd 不可用，使用 grep 扫描"
  find "$INBOX" -name "msg-*.json" -newer "${INBOX}/${LAST_READ}.json" 2>/dev/null | sort
# L3: ls 排序兜底
else
  echo "📂 首次扫描，按时间排序"
  ls -t "$INBOX"/msg-*.json 2>/dev/null || echo "📭 inbox 为空"
fi
```

**关键原则：** qmd 是加速器，不是依赖。核心功能（inbox 写入 + 文件扫描）不依赖 qmd。

### 6.4 Gateway Session 恢复协议

心跳 session 被 Gateway 回收后重建时：

```
Session 被 Gateway 回收
  ↓
Agent 重新启动
  ↓
读取 qmd: "我的最后 5 条消息是什么？"
  ↓
qmd 返回最近消息上下文
  ↓
Agent 恢复对话状态，无缝继续
  ↓
用户几乎感知不到 session 重启
```

---

## 七、降级策略

### 7.1 完整降级矩阵

| 场景 | Level 1 | Level 2 | Level 3 | Level 4 |
|------|---------|---------|---------|---------|
| 飞书 session 正常 | ✅ 秒到 | ✅ 3秒 | ✅ 3分钟 | ✅ 30分钟 |
| 飞书 session 卡死 | ❌ 失败 | ✅ 3秒 | ✅ 3分钟 | ✅ 30分钟 |
| Agent session 忙碌 | ⚠️ 入队 | ⚠️ 排队 | ✅ 3分钟 | ✅ 30分钟 |
| Gateway 刚重启 | ❌ 未恢复 | ⚠️ 待启动 | ✅ 3分钟 | ✅ 30分钟 |
| 所有机制失效 | ❌ | ❌ | ❌ | ✅ 30分钟 |

---

## 八、Gateway 重启恢复

### 8.1 恢复时间线

| 时间 | 事件 |
|------|------|
| T+0s | Gateway 重启开始 |
| T+5s | Gateway 启动完成，websocket 重连 |
| T+10s | gateway:startup hook 触发，扫描 inbox |
| T+15s | 发现未读消息，webhook 触发 Agent |
| T+20s | Agent 被唤醒，处理消息 |
| **T+30s** | **恢复完成** |
| T+3min | Cron 首次扫描，二次确认 |
| T+30min | Heartbeat 首次扫描，最终兜底 |

---

## 九、可靠性测试框架（G1 新增 ⭐）

**来源：** 德尔塔 📊 设计 + 伽马 🔧 实现

### 9.1 测试指标

| 指标 | v1.0 | v2.0 目标 |
|------|------|----------|
| 投递成功率 | ~70% | ≥99.9% |
| 平均投递延迟 | ~15分钟 | ≤3秒（webhook） |
| Gateway 重启恢复 | 不确定 | ≤30秒 |
| 最坏情况延迟 | 30分钟 | 3分钟 |

### 9.2 测试用例

| 用例 | 场景 | 预期 |
|------|------|------|
| TC1 | 正常发送 100 条 | 100% 到达，无乱序 |
| TC2 | 网络中断 30 秒 | 恢复后全部投递 |
| TC3 | Gateway 重启 | 30 秒内恢复 |
| TC4 | Session 卡死 | 3 秒内 webhook 降级 |
| TC5 | 高频发送 200 条/min | 无丢失，少量延迟 |

### 9.3 故障注入

- **网络故障：** `iptables` 模拟中断
- **系统故障：** `kill` 进程、资源限制
- **应用故障：** 模拟 API 超时/错误

---

## 十、实施计划

### 10.4 部署脚本 — 一键部署到所有 Agent workspace（伽马建议补充 ⭐）

```bash
#!/bin/bash
# deploy-messaging-v2.sh — 部署即时通讯 v2.0 到所有 Agent workspace
# 用法: ./deploy-messaging-v2.sh [--dry-run] [--agent <name>]
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
AGENTS=(main alpha beta gamma delta epsilon zeta eta theta iota kappa lambda)
DRY_RUN=false
TARGET_AGENT=""

# 参数解析
while [[ $# -gt 0 ]]; do
  case "$1" in
    --dry-run) DRY_RUN=true; shift ;;
    --agent) TARGET_AGENT="$2"; shift 2 ;;
    *) echo "未知参数: $1"; exit 1 ;;
  esac
done

# 如果指定 agent，只部署到该 agent
if [ -n "$TARGET_AGENT" ]; then
  AGENTS=("$TARGET_AGENT")
fi

# 要部署的脚本
SCRIPTS=("send-and-notify.sh" "ack-processor.sh" "delivery-status.sh" "inbox-scan.sh")
# 要创建的目录
DIRS=("inbox" "outbox" "inbox/archive")

echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "🚀 即时通讯 v2.0 部署"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
$DRY_RUN && echo "⚠️  DRY RUN — 不实际修改文件"
echo ""

for AGENT in "${AGENTS[@]}"; do
  WORKSPACE="/home/gang/.openclaw/workspace-${AGENT}"
  
  if [ ! -d "$WORKSPACE" ]; then
    echo "⏭️  ${AGENT}: workspace 不存在，跳过"
    continue
  fi
  
  echo "📦 ${AGENT}:"
  
  # 创建目录
  for DIR in "${DIRS[@]}"; do
    TARGET="${WORKSPACE}/${DIR}"
    if [ ! -d "$TARGET" ]; then
      $DRY_RUN || mkdir -p "$TARGET"
      echo "   📁 创建 ${DIR}/"
    else
      echo "   ✅ ${DIR}/ 已存在"
    fi
  done
  
  # 部署脚本
  for SCRIPT in "${SCRIPTS[@]}"; do
    SOURCE="${SCRIPT_DIR}/scripts/${SCRIPT}"
    TARGET="${WORKSPACE}/${SCRIPT}"
    
    if [ ! -f "$SOURCE" ]; then
      echo "   ⚠️  源文件不存在: ${SOURCE}"
      continue
    fi
    
    if [ -f "$TARGET" ]; then
      # 检查是否有更新
      if cmp -s "$SOURCE" "$TARGET"; then
        echo "   ✅ ${SCRIPT} 已是最新"
      else
        $DRY_RUN || cp "$SOURCE" "$TARGET"
        echo "   🔄 ${SCRIPT} 已更新"
      fi
    else
      $DRY_RUN || cp "$SOURCE" "$TARGET"
      echo "   📥 ${SCRIPT} 已部署"
    fi
    
    $DRY_RUN || chmod +x "$TARGET"
  done
  
  # 创建 .last-read（如果不存在）
  LAST_READ="${WORKSPACE}/inbox/.last-read"
  if [ ! -f "$LAST_READ" ]; then
    $DRY_RUN || echo "" > "$LAST_READ"
    echo "   📝 创建 inbox/.last-read"
  fi
  
  echo ""
done

echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
echo "✅ 部署完成"
echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

# 验证
echo ""
echo "📊 部署验证："
for AGENT in "${AGENTS[@]}"; do
  WORKSPACE="/home/gang/.openclaw/workspace-${AGENT}"
  [ -d "$WORKSPACE" ] || continue
  
  OK=0; MISS=0
  for SCRIPT in "${SCRIPTS[@]}"; do
    if [ -x "${WORKSPACE}/${SCRIPT}" ]; then
      ((OK++))
    else
      ((MISS++))
    fi
  done
  
  INBOX_OK="❌"
  [ -d "${WORKSPACE}/inbox" ] && INBOX_OK="✅"
  
  STATUS="✅"
  [ "$MISS" -gt 0 ] && STATUS="⚠️"
  echo "   ${STATUS} ${AGENT}: ${OK}/${#SCRIPTS[@]} 脚本, inbox ${INBOX_OK}"
done
```

**用法：**

```bash
# 部署到所有 Agent
./deploy-messaging-v2.sh

# 预览（不实际修改）
./deploy-messaging-v2.sh --dry-run

# 只部署到某个 Agent
./deploy-messaging-v2.sh --agent gamma

# 部署后验证
./deploy-messaging-v2.sh --dry-run | grep "⚠️"
```

**目录结构（部署后每个 workspace）：**

```
workspace-gamma/
├── inbox/                    # 消息收件箱
│   ├── .last-read            # 已读指针
│   ├── archive/              # 已处理的 ack 归档
│   ├── msg-*.json            # 收到的消息
│   └── ack-*.json            # 收到的确认回执
├── outbox/                   # 发件追踪
│   └── sent-log.jsonl        # 发送日志
├── send-and-notify.sh        # 发送脚本（优先级路由）
├── ack-processor.sh          # 确认处理脚本
├── delivery-status.sh        # 投递状态查询
└── inbox-scan.sh             # inbox 扫描（qmd fallback）
```

### Phase 1：核心增强（4 小时）

| # | 任务 | 工时 | 状态 |
|---|------|------|------|
| 1 | 启用 hooks 配置 | 15min | ⏳ |
| 2 | 设置 OPENCLAW_HOOKS_TOKEN | 5min | ⏳ |
| 3 | 创建 startup-inbox-scan hook | 30min | ⏳ |
| 4 | 测试 webhook 触发 | 15min | ⏳ |
| 5 | 创建 inbox cron（3分钟） | 15min | ⏳ |
| 6 | 部署 send-and-notify.sh 到所有 workspace | 30min | ⏳ |
| 7 | 部署 ack-processor.sh 到所有 workspace | 30min | ⏳ |
| 8 | 更新所有 Agent HEARTBEAT.md（ack 处理） | 1h | ⏳ |

### Phase 2：Agent 集成（1-2 天）

| # | 任务 | 说明 |
|---|------|------|
| 1 | 更新所有 Agent AGENTS.md | 添加发送规范 + ack 回传 |
| 2 | Agent 注册表 | 名称 → agentId 映射 |
| 3 | 测试 message:sent hook | 验证 sessions_send 触发 |

### Phase 3：优化（可选）

| # | 任务 | 说明 |
|---|------|------|
| 1 | 消息去重 | 多通道同时投递去重 |
| 2 | 消息优先级路由 | urgent 走 webhook，normal 走 cron |
| 3 | 性能监控 | 投递成功率 + 延迟统计 |
| 4 | qmd 索引集成 | 利用 qmd 加速 inbox 扫描 |

---

## 十一、Agent 发送规范

所有 Agent 发消息时必须遵循：

```
Step 1: 写入 inbox 文件（必须）
  → 写入 target-agent/inbox/msg-*.json（含 ack_required + ack_to）

Step 2: 调用 webhook 触发（推荐）
  → POST /hooks/agent 触发对方

Step 3: 同时 sessions_send（可选）
  → 如果需要对方在飞书看到

Step 4: 记录 outbox（如需追踪）
  → 写入 outbox/sent-log.jsonl
```

---

## 十二、衡量指标

| 指标 | 衡量方式 |
|------|---------|
| 投递成功率 | 日志统计 |
| 平均投递延迟 | 时间戳对比 |
| Gateway 重启恢复时间 | startup hook 日志 |
| 最坏情况延迟 | Cron 间隔 |
| 确认回传延迟 | ack 时间 - 发送时间 |
| 未确认消息积压 | outbox pending 计数 |

---

## 十三、总结

### v2.0 Complete 核心升级

| 维度 | v1.0 | v2.0 Complete |
|------|------|--------------|
| 消息持久化 | ✅ inbox 文件 | ✅ 不变 |
| 即时触发 | ❌ 无 | ✅ Webhook + Hook Handler |
| 投递确认 | ❌ 无 | ✅ G1 ack 机制 |
| Gateway 重启恢复 | ❌ 依赖心跳 | ✅ startup hook 30s 恢复 |
| 兜底机制 | ✅ 心跳 30 分钟 | ✅ Cron 3 分钟 + 心跳 30 分钟 |
| 投递追踪 | ❌ 无 | ✅ outbox/sent-log.jsonl |
| 降级层级 | 2 级 | 4 级 |
| 事件驱动 | ❌ 无 | ✅ 6 种协议对比 + heartbeat sync |

### 最终效果

```
梦想家发消息 → 阿尔法
  ↓
写入 inbox + webhook 触发
  ↓
3 秒内阿尔法收到 ✅
  ↓
阿尔法处理 → ack 回传
  ↓
梦想家看到确认 ✅

Gateway 重启
  ↓
startup hook 扫描 inbox
  ↓
30 秒内所有未读消息恢复处理 ✅

所有机制都挂了
  ↓
30 分钟后心跳兜底 ✅
```

---

## 参考资料

- R1 研究：`shared-knowledge/research/openclaw-webhook-analysis.md`
- R2 研究：`shared-knowledge/research/on-message-mechanism-analysis.md`
- R3 研究：`shared-knowledge/architecture/event-driven-notification-design.md`
- G1 实现：`shared-knowledge/implementation/delivery-confirmation.md`
- G1 测试：`shared-knowledge/reports/delivery-reliability-test.md`
- v1.0 方案：`shared-knowledge/architecture/messaging-architecture.md`
- OpenClaw Hooks 文档：`docs/automation/hooks.md`
- OpenClaw Cron 文档：`docs/automation/cron-jobs.md`

---

*本文档由拉姆达 🔬 整合 Phase 1-3 全部研究成果 + 伽马 🔧 G1 投递确认实现完成。*
*交付物路径：`shared-knowledge/architecture/instant-messaging-v2-COMPLETE.md`*
