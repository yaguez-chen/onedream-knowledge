# 消息投递确认机制 — 完整设计 + 实现

**任务：** Phase 3 Task G1
**负责人：** 伽马 🔧
**状态：** 🔴 实施中（R3 已完成，无阻塞）
**创建时间：** 2026-03-18
**更新时间：** 2026-03-18 17:32
**依赖：** 拉姆达 R3（事件驱动通知方案）✅ 已完成

---

## 一、目标

在拉姆达 v2.0 四级投递通道架构上，实现消息投递确认机制，让发送方知道消息是否被接收方读取和处理。

### 核心需求
- 发送方写入消息后，能知道接收方是否已读取
- 确认信息回传到发送方的 inbox
- 与 v2.0 webhook/cron/heartbeat 架构无缝集成
- 不依赖飞书 session（纯文件系统 + webhook 实现）

## 二、当前架构分析（v2.0 基础）

### v2.0 四级投递通道
```
Level 1: sessions_send（飞书，<1秒）         ← 发送方已有
Level 2: Webhook 触发（/hooks/agent，<3秒）   ← 发送方已有
Level 3: Cron 3分钟轮询                       ← 系统已有
Level 4: Heartbeat 30分钟兜底                 ← 系统已有
```

### 当前问题
```
发送方(A)                        接收方(B)
    │                               │
    │ 写入 B/inbox/msg.json         │
    │ ────────────────────────────► │
    │                               │ 处理消息
    │ ❌ 发送方不知道是否已读取        │
```

### v2.0 消息格式（现有）
```json
{
  "id": "msg-20260318-142605-491022",
  "timestamp": "2026-03-18T14:26:05+08:00",
  "from": "alpha",
  "priority": "high",
  "type": "notification",
  "subject": "...",
  "body": "...",
  "read": false
}
```

## 三、投递确认设计方案

### 3.1 确认回传流程（完整版）

```
发送方(A)                        接收方(B)
    │                               │
    │ 1. 写入 B/inbox/msg.json      │
    │    (含 ack_required + ack_to)  │
    │                               │
    │ 2. Webhook 触发 B             │
    │ ────────────────────────────► │
    │                               │ 3. B 心跳/webhook 处理消息
    │                               │ 4. B 写入 A/inbox/ack-xxx.json
    │                               │ 5. B Webhook 触发 A（ack 回传）
    │ ◄──────────────────────────── │
    │                               │
    │ 6. A 收到 ack                 │
    │ 7. 更新 outbox/sent-log.jsonl │
    │ 8. 标记消息为已确认            │
```

### 3.2 消息格式扩展

**发送消息格式（扩展）：**
```json
{
  "id": "msg-20260318-142605-491022",
  "timestamp": "2026-03-18T14:26:05+08:00",
  "from": "alpha",
  "to": "gamma",
  "priority": "high",
  "type": "notification",
  "subject": "Phase 3 G1 任务",
  "body": "...",
  "read": false,
  "ack_required": true,
  "ack_to": "alpha"
}
```

**确认消息格式：**
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

### 3.3 确认状态码

| 状态 | 含义 | 触发时机 |
|------|------|---------|
| `received` | 已收到但未读 | 文件写入成功（可选，由发送方自己标记） |
| `read` | 已读取 | Agent 心跳/webhook 读取并处理 |
| `processing` | 处理中 | Agent 开始执行任务（可选） |
| `done` | 任务完成 | Agent 完成相关任务 |
| `failed` | 处理失败 | Agent 处理出错 |

### 3.4 投递追踪日志

**发送方维护：** `workspace/outbox/sent-log.jsonl`
```jsonl
{"id":"msg-xxx","to":"gamma","subject":"...","sent_at":"...","acked_at":null,"status":"pending"}
{"id":"msg-xxx","to":"gamma","subject":"...","sent_at":"...","acked_at":"...","status":"read"}
```

### 3.5 与 v2.0 架构集成

| v2.0 组件 | 投递确认集成 |
|-----------|-------------|
| Webhook（Level 2）| 接收方处理后通过 webhook 触发发送方 ack |
| Cron 3分钟（Level 3）| 发送方可通过 cron 扫描未确认消息，重试 |
| Heartbeat（Level 4）| 发送方 heartbeat 处理 ack-*.json 文件 |

## 四、实现代码

### 4.1 send-and-notify.sh（带确认支持的发送脚本）

```bash
#!/bin/bash
# send-and-notify.sh — 带投递确认支持的消息发送脚本
# 用法: ./send-and-notify.sh <target-agent> <priority> <subject> <body> [--no-ack]
# 示例: ./send-and-notify.sh gamma high "Phase 3 启动" "请查看 PHASE4-ALERT.md"

set -euo pipefail

TARGET_AGENT="${1:-}"
PRIORITY="${2:-normal}"
SUBJECT="${3:-}"
BODY="${4:-}"
NO_ACK="${5:-}"

if [ -z "$TARGET_AGENT" ] || [ -z "$SUBJECT" ] || [ -z "$BODY" ]; then
  echo "用法: $0 <target-agent> <priority> <subject> <body> [--no-ack]"
  echo "优先级: urgent | high | normal | low"
  echo "添加 --no-ack 可跳过投递确认"
  exit 1
fi

WORKSPACE="/home/gang/.openclaw/workspace-${TARGET_AGENT}"
INBOX="${WORKSPACE}/inbox"
OUTBOX="/home/gang/.openclaw/workspace-gamma/outbox"
MSG_ID="msg-$(date +%Y%m%d-%H%M%S)-$$"
TIMESTAMP=$(date -Iseconds)
HOOKS_TOKEN=$(grep OPENCLAW_HOOKS_TOKEN ~/.openclaw/.env 2>/dev/null | cut -d= -f2 || echo "")

# 检查目标工作空间
if [ ! -d "$WORKSPACE" ]; then
  echo "❌ 目标工作空间不存在: $WORKSPACE"
  exit 1
fi

# 确保目录存在
mkdir -p "$INBOX"
mkdir -p "$OUTBOX"

# 确定是否需要确认
ACK_REQUIRED="true"
if [ "$NO_ACK" = "--no-ack" ]; then
  ACK_REQUIRED="false"
fi

# 写入消息文件
cat > "${INBOX}/${MSG_ID}.json" << EOF
{
  "id": "${MSG_ID}",
  "timestamp": "${TIMESTAMP}",
  "from": "gamma",
  "to": "${TARGET_AGENT}",
  "priority": "${PRIORITY}",
  "type": "notification",
  "subject": "${SUBJECT}",
  "body": "${BODY}",
  "read": false,
  "ack_required": ${ACK_REQUIRED},
  "ack_to": "gamma"
}
EOF

echo "✅ 消息已写入: ${INBOX}/${MSG_ID}.json"

# 记录到 outbox 追踪日志
if [ "$ACK_REQUIRED" = "true" ]; then
  echo "{\"id\":\"${MSG_ID}\",\"to\":\"${TARGET_AGENT}\",\"subject\":\"${SUBJECT}\",\"sent_at\":\"${TIMESTAMP}\",\"acked_at\":null,\"status\":\"pending\"}" \
    >> "${OUTBOX}/sent-log.jsonl"
  echo "📝 已记录到 outbox 追踪日志"
fi

# Webhook 触发接收方（如果 token 可用）
if [ -n "$HOOKS_TOKEN" ]; then
  curl -s -X POST http://127.0.0.1:18789/hooks/agent \
    -H "Authorization: Bearer ${HOOKS_TOKEN}" \
    -H "Content-Type: application/json" \
    -d "{\"message\":\"你有来自 伽马 的新消息（${PRIORITY}），请立即检查 inbox/ 目录并处理。处理后请回复确认。\",\"agentId\":\"${TARGET_AGENT}\",\"name\":\"InboxNotify\",\"deliver\":false,\"timeoutSeconds\":60}" \
    > /dev/null 2>&1 && echo "🔔 Webhook 已触发 ${TARGET_AGENT}" || echo "⚠️ Webhook 触发失败（消息已持久化，等待 cron/heartbeat 兜底）"
else
  echo "⚠️ OPENCLAW_HOOKS_TOKEN 未配置，跳过 webhook 触发（消息已持久化）"
fi

echo ""
echo "📬 消息投递详情："
echo "  目标: ${TARGET_AGENT}"
echo "  优先级: ${PRIORITY}"
echo "  主题: ${SUBJECT}"
echo "  ID: ${MSG_ID}"
echo "  需要确认: ${ACK_REQUIRED}"
```

### 4.2 ack-processor.sh（确认消息处理器）

```bash
#!/bin/bash
# ack-processor.sh — 处理收到的确认消息，更新追踪日志
# 用法: ./ack-processor.sh
# 由 heartbeat 或 cron 调用，扫描 inbox/ 中的 ack-*.json 文件

set -euo pipefail

INBOX="/home/gang/.openclaw/workspace-gamma/inbox"
OUTBOX="/home/gang/.openclaw/workspace-gamma/outbox"
SENT_LOG="${OUTBOX}/sent-log.jsonl"

# 确保目录存在
mkdir -p "$OUTBOX"

# 扫描所有 ack 文件
ACK_FILES=$(ls "${INBOX}"/ack-*.json 2>/dev/null | sort || true)

if [ -z "$ACK_FILES" ]; then
  # 无确认文件
  exit 0
fi

echo "📬 发现确认消息："
echo ""

for ACK_FILE in $ACK_FILES; do
  ACK_ID=$(basename "$ACK_FILE" .json)
  
  # 解析确认消息
  REF_ID=$(grep -o '"ref_id":"[^"]*"' "$ACK_FILE" | cut -d'"' -f4)
  STATUS=$(grep -o '"status":"[^"]*"' "$ACK_FILE" | cut -d'"' -f4)
  READ_AT=$(grep -o '"read_at":"[^"]*"' "$ACK_FILE" | cut -d'"' -f4)
  FROM=$(grep -o '"from":"[^"]*"' "$ACK_FILE" | cut -d'"' -f4)
  
  echo "  ✅ ${REF_ID}"
  echo "     来自: ${FROM}"
  echo "     状态: ${STATUS}"
  echo "     读取时间: ${READ_AT}"
  
  # 更新 sent-log.jsonl
  if [ -f "$SENT_LOG" ]; then
    # 读取原记录，更新状态
    OLD_ENTRY=$(grep "\"id\":\"${REF_ID}\"" "$SENT_LOG" 2>/dev/null || true)
    if [ -n "$OLD_ENTRY" ]; then
      # 生成更新后的条目
      NEW_ENTRY=$(echo "$OLD_ENTRY" | sed "s/\"acked_at\":null/\"acked_at\":\"${READ_AT}\"/g" | sed "s/\"status\":\"pending\"/\"status\":\"${STATUS}\"/g")
      
      # 替换原条目
      sed -i "s|${OLD_ENTRY}|${NEW_ENTRY}|g" "$SENT_LOG"
      echo "     📝 追踪日志已更新"
    else
      echo "     ⚠️ 追踪日志中未找到 ${REF_ID}（可能是在其他 session 发送的）"
    fi
  fi
  
  # 移动已处理的 ack 文件到 archive
  ARCHIVE_DIR="${INBOX}/archive"
  mkdir -p "$ARCHIVE_DIR"
  mv "$ACK_FILE" "$ARCHIVE_DIR/"
  echo "     📦 已归档到 archive/"
  echo ""
done

echo "✅ 确认处理完成"
```

### 4.3 inbox-check.sh（统一 inbox 检查脚本）

```bash
#!/bin/bash
# inbox-check.sh — 统一的 inbox 检查脚本
# 用法: ./inbox-check.sh [agent-name]
# 不带参数时检查自己的 inbox，带参数时检查指定 agent 的 inbox
# 用于 heartbeat 和 cron 集成

set -euo pipefail

AGENT="${1:-gamma}"
INBOX="/home/gang/.openclaw/workspace-${AGENT}/inbox"
LAST_READ_FILE="${INBOX}/.last-read"

if [ ! -d "$INBOX" ]; then
  echo "📭 无 inbox 目录"
  exit 0
fi

# 读取 last-read
LAST_READ=""
if [ -f "$LAST_READ_FILE" ]; then
  LAST_READ=$(cat "$LAST_READ_FILE")
fi

# 列出新消息（排除 ack 和 archive）
NEW_MSGS=$(ls "${INBOX}"/msg-*.json 2>/dev/null | while read f; do
  basename "$f" .json
done | sort | awk -v last="$LAST_READ" 'last == "" || $0 > last' || true)

ACK_FILES=$(ls "${INBOX}"/ack-*.json 2>/dev/null | while read f; do
  basename "$f" .json
done | sort || true)

if [ -z "$NEW_MSGS" ] && [ -z "$ACK_FILES" ]; then
  exit 0
fi

# 输出新消息列表
if [ -n "$NEW_MSGS" ]; then
  echo "📬 新消息："
  for MSG_ID in $NEW_MSGS; do
    MSG_FILE="${INBOX}/${MSG_ID}.json"
    SUBJECT=$(grep -o '"subject":"[^"]*"' "$MSG_FILE" | cut -d'"' -f4)
    PRIORITY=$(grep -o '"priority":"[^"]*"' "$MSG_FILE" | cut -d'"' -f4)
    FROM=$(grep -o '"from":"[^"]*"' "$MSG_FILE" | cut -d'"' -f4)
    ACK_REQ=$(grep -o '"ack_required":[^,}]*' "$MSG_FILE" | cut -d: -f2 | tr -d ' ')
    
    echo "  [${PRIORITY}] ${FROM}: ${SUBJECT}"
    if [ "$ACK_REQ" = "true" ]; then
      echo "    ↳ 需要确认回传"
    fi
  done
fi

# 输出待处理的确认
if [ -n "$ACK_FILES" ]; then
  echo ""
  echo "📨 待处理确认："
  for ACK_ID in $ACK_FILES; do
    ACK_FILE="${INBOX}/${ACK_ID}.json"
    REF_ID=$(grep -o '"ref_id":"[^"]*"' "$ACK_FILE" | cut -d'"' -f4)
    STATUS=$(grep -o '"status":"[^"]*"' "$ACK_FILE" | cut -d'"' -f4)
    echo "  ${REF_ID} → ${STATUS}"
  done
fi
```

### 4.4 HEARTBEAT.md 确认处理逻辑

在 HEARTBEAT.md 中添加（或更新）以下处理步骤：

```markdown
### 投递确认处理
处理 inbox 中的 ack-*.json 文件：
1. 运行 `./ack-processor.sh` 更新追踪日志
2. 如有重要确认（如任务完成），记录到 memory/
```

### 4.5 发送方 heartbeat 集成

在发送方（如阿尔法）的 HEARTBEAT.md 中添加：

```markdown
### 待确认消息追踪
检查 outbox/sent-log.jsonl 中 status=pending 的消息：
1. 超过 30 分钟未确认 → 重试发送（最多 3 次）
2. 超过 2 小时未确认 → 标记为超时，上报梦想家
```

## 五、实施计划

### Phase A：核心实现（今天）
- [x] 研究现有消息传递代码（send-message.sh + inbox 结构）
- [x] 阅读拉姆达的 v2.0 架构设计（R3）
- [x] 设计投递确认方案（本文档）
- [x] 编写 send-and-notify.sh（带 ack 支持）
- [x] 编写 ack-processor.sh（确认处理）
- [x] 编写 inbox-check.sh（统一检查）
- [ ] 部署脚本到 workspace/scripts/
- [ ] 更新 HEARTBEAT.md 添加确认处理逻辑

### Phase B：测试验证
- [ ] 单元测试：发送 → 确认 → 追踪 完整流程
- [ ] 集成测试：跨 Agent 消息投递确认
- [ ] 异常测试：session 卡死时的降级行为

### Phase C：文档归档
- [ ] 完善交付文档
- [ ] 归档到 shared-knowledge/implementation/
- [ ] 通知约塔（项目管理负责人）

## 六、与拉姆达 R3 的集成

### 6.1 Webhook 集成
- 发送方发消息后调用 webhook 触发接收方（已有）
- 接收方处理后调用 webhook 触发发送方回传 ack（新增）

### 6.2 Cron 集成
- 发送方可通过 cron 扫描未确认消息，超时自动重试
- 接收方可通过 cron 扫描未处理消息，自动触发 ack 回传

### 6.3 Heartbeat 集成
- 发送方 heartbeat 运行 ack-processor.sh 处理确认
- 接收方 heartbeat 运行 inbox-check.sh 检查新消息

## 七、风险和注意事项

| 风险 | 影响 | 缓解 |
|------|------|------|
| ack 文件过多 | inbox 膨胀 | archive 归档 + 定期清理（保留 7 天） |
| 重复 ack | 重复处理 | 消息 ID 去重 + archive 移动 |
| ack 延迟大 | 发送方焦虑 | 通过 v2.0 webhook 减少延迟到 <3 秒 |
| 跨时区时间戳 | 时间对比错误 | 统一使用 ISO 8601 + UTC |
| Gateway 重启丢失追踪 | 状态不一致 | outbox/sent-log.jsonl 持久化在磁盘 |

---

*本文档由伽马 🔧 基于拉姆达 R3 v2.0 架构设计完成。*
*如有问题或建议，请联系伽马 🔧*
