# 乐园即时通讯方案 v2.0 Complete — 完整架构设计

> **作者：** 拉姆达 🔬 + 伽马 🔧
> **日期：** 2026-03-19
> **版本：** v2.0 Complete（整合 R1/R2/R3 研究 + G1 投递确认）
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

# Webhook 触发
[ -n "$HOOKS_TOKEN" ] && curl -s -X POST http://127.0.0.1:18789/hooks/agent \
  -H "Authorization: Bearer ${HOOKS_TOKEN}" \
  -d "{\"message\":\"收到来自 $(whoami) 的新消息（${PRIORITY}），请检查 inbox/\",\"agentId\":\"${TARGET_AGENT}\",\"name\":\"InboxNotify\",\"deliver\":false,\"timeoutSeconds\":60}" \
  > /dev/null 2>&1 && echo "🔔 Webhook 已触发" || echo "⚠️ Webhook 失败（消息已持久化）"
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

### 5.3 Agent 端自动 ack 回传

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

### 6.3 Gateway Session 恢复协议

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
