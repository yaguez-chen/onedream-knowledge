# 乐园即时通讯方案 v2.0 — 完整架构设计

> **作者：** 拉姆达 🔬
> **日期：** 2026-03-18
> **版本：** v2.0（基于 v1.0 可靠消息传递方案升级）
> **标签：** #通信 #架构 #核心 #设计 #即时通讯 #v2
> **来源：** 梦想家直接指令（2026-03-18 16:23）— 整合 Phase 1 已实施方案 + R2 研究成果 + .last-read bug 修复

---

## 一、变更摘要

### v2.0 新增内容（vs v1.0）

| 新增 | 来源 | 价值 |
|------|------|------|
| Webhook 即时触发 | R1 研究 | Agent 发消息后秒级触发对方 |
| Hook Handler 自动化 | R2 研究 | gateway:startup 自动扫描 missed messages |
| .last-read bug 修复 | 实际发现 | Gateway 重启后自动恢复，不丢消息 |
| Cron 高频兜底 | R2 研究 | 3 分钟轮询作为可靠 fallback |
| 完整降级链 | 整合 | 四级降级：webhook → hook → cron → heartbeat |

### 修复的问题

| 问题 | v1.0 状态 | v2.0 修复 |
|------|----------|----------|
| Gateway 重启后 .last-read 未扫描 | ❌ 未覆盖 | ✅ gateway:startup hook 自动扫描 |
| 心跳 30 分钟延迟 | ⚠️ 仅心跳 | ✅ 3分钟 cron + 秒级 webhook |
| Agent 间无即时触发 | ❌ 未覆盖 | ✅ 发消息后 webhook 触发 |
| 外部系统无法触发 | ❌ 未覆盖 | ✅ /hooks/agent 端点 |

---

## 二、架构总览

### 2.1 完整消息流程

```
发送方 Agent                                              接收方 Agent
════════════                                              ════════════

    │ 发消息
    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        四级投递通道                                    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Level 1: sessions_send ────────────────► 飞书 session ──► Agent B   │
│           (即时，<1秒)                    正常→秒到                    │
│                                              卡死→入队 ❌              │
│                                                                      │
│  Level 2: Webhook 触发 ─────────────────► /hooks/agent ──► Agent B   │
│           (即时，<3秒)                    独立 session 运行            │
│           Agent A 发消息后                                            │
│           执行 curl 调用                                               │
│                                                                      │
│  Level 3: Cron 高频轮询 ─────────────────► 发现 inbox ──► 触发 B     │
│           (3分钟)                        未读消息                     │
│                                                                      │
│  Level 4: Heartbeat 心跳 ────────────────► 检查 inbox ──► 处理      │
│           (30分钟，兜底)                                               │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘

    │ 同时写入
    ▼
┌────────────┐
│ inbox/     │ ← 持久化层（磁盘）
│ .json 文件  │   所有通道最终都依赖这个持久化基底
└────────────┘
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

**设计原则：** 每一级都是前一级的兜底。任何一级成功，消息就不会丢。

---

## 三、核心组件

### 3.1 组件一：Inbox 持久化层

**路径：** `~/.openclaw/workspace-<agent>/inbox/`

```
inbox/
├── .last-read                    # 指针文件：最后处理的消息 ID
├── msg-20260318-142605-491.json  # 消息文件（JSON）
├── msg-20260318-161500-r1r2.json
└── ...
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
  "read": false
}
```

**.last-read 格式：**
```
msg-20260318-142605-491022
```

**状态：** ✅ 已实施（Phase 1）

---

### 3.2 组件二：Webhook 即时触发（新增 ⭐）

**来源：** R1 研究成果
**作用：** Agent 发消息后，通过 HTTP webhook 立即触发对方处理

#### 3.2.1 启用配置

在 `openclaw.json` 中添加：

```json5
{
  hooks: {
    enabled: true,
    token: "${OPENCLAW_HOOKS_TOKEN}",  // 存在 .env 中
    path: "/hooks",
    allowedAgentIds: ["main", "alpha", "beta", "gamma", "delta",
                      "epsilon", "zeta", "eta", "theta", "iota",
                      "kappa", "lambda"],
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: false
  }
}
```

#### 3.2.2 触发流程

```
Agent A 完成发消息
  ↓
写入 Agent B 的 inbox/ 文件
  ↓
执行 curl 调用 webhook：
  POST /hooks/agent
  {
    "message": "你有新消息，请检查 inbox/ 目录并处理",
    "agentId": "beta",
    "name": "InboxTrigger",
    "deliver": false,
    "timeoutSeconds": 60
  }
  ↓
Gateway 创建独立 session 运行 Agent B
  ↓
Agent B 读取 inbox/ → 处理消息 → 完成
```

#### 3.2.3 Agent 端发送脚本

在 Agent 发消息时，除了写 inbox 文件，额外调用 webhook：

```bash
# send-and-notify.sh
# 用法: ./send-and-notify.sh <target-agent> <priority> <subject> <body>

TARGET=$1
PRIORITY=$2
SUBJECT=$3
BODY=$4
HOOKS_TOKEN=$(grep OPENCLAW_HOOKS_TOKEN ~/.openclaw/.env | cut -d= -f2)

# Step 1: 写入 inbox 文件
MSG_ID="msg-$(date +%Y%m%d-%H%M%S)-$$"
mkdir -p ~/.openclaw/workspace-${TARGET}/inbox
cat > ~/.openclaw/workspace-${TARGET}/inbox/${MSG_ID}.json << EOF
{"id":"${MSG_ID}","timestamp":"$(date -Iseconds)","from":"$(hostname)","priority":"${PRIORITY}","subject":"${SUBJECT}","body":"${BODY}","read":false}
EOF

# Step 2: Webhook 触发对方
curl -s -X POST http://127.0.0.1:18789/hooks/agent \
  -H "Authorization: Bearer ${HOOKS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"message\":\"收到来自 ${FROM_AGENT} 的新消息（${PRIORITY}），请立即检查 inbox/ 目录并处理。处理后回复确认。\",\"agentId\":\"${TARGET}\",\"name\":\"InboxNotify\",\"deliver\":false,\"timeoutSeconds\":60}"
```

**延迟：** <3 秒（写文件 + HTTP 调用）

**状态：** 📋 设计完成，待实施

---

### 3.3 组件三：Hook Handler 自动化（新增 ⭐）

**来源：** R2 研究成果
**作用：** Gateway 事件自动触发，无需 Agent 手动干预

#### 3.3.1 `gateway:startup` Hook — 重启后自动扫描

**解决问题：** Gateway 重启后 session 重建，.last-read 指针可能失效

**实现：**

```typescript
// ~/.openclaw/hooks/startup-inbox-scan/handler.ts
const handler = async (event) => {
  if (event.type !== "gateway" || event.action !== "startup") return;
  
  console.log("[startup-inbox-scan] Gateway 启动，扫描所有 Agent inbox...");
  
  const agents = ["alpha", "beta", "gamma", "delta", "epsilon",
                  "zeta", "eta", "theta", "iota", "kappa", "lambda"];
  
  for (const agent of agents) {
    const inboxDir = `/home/gang/.openclaw/workspace-${agent}/inbox`;
    const lastReadFile = `${inboxDir}/.last-read`;
    
    // 读取 .last-read
    let lastRead = "";
    try {
      lastRead = (await Bun.file(lastReadFile).text()).trim();
    } catch { /* 文件不存在，视为首次 */ }
    
    // 扫描未读消息
    const files = (await readdir(inboxDir))
      .filter(f => f.endsWith(".json") && f > (lastRead || ""))
      .sort();
    
    if (files.length > 0) {
      console.log(`[startup-inbox-scan] ${agent}: 发现 ${files.length} 条未读消息`);
      
      // 通过 webhook 触发该 Agent 处理
      await fetch("http://127.0.0.1:18789/hooks/agent", {
        method: "POST",
        headers: {
          "Authorization": `Bearer ${process.env.OPENCLAW_HOOKS_TOKEN}`,
          "Content-Type": "application/json"
        },
        body: JSON.stringify({
          message: `Gateway 刚重启。你在 inbox/ 中有 ${files.length} 条未读消息。请立即处理。`,
          agentId: agent,
          name: "StartupInboxScan",
          deliver: false,
          timeoutSeconds: 120
        })
      });
    }
  }
};

export default handler;
```

**HOOK.md：**

```markdown
---
name: startup-inbox-scan
description: "Gateway 启动后自动扫描所有 Agent 的 inbox，触发未读消息处理"
metadata:
  openclaw:
    emoji: "📬"
    events: ["gateway:startup"]
---

# Startup Inbox Scan

Gateway 重启后自动扫描所有 Agent 的 inbox 目录。
对比 .last-read 指针，发现未读消息后通过 webhook 触发对应 Agent 处理。
防止 Gateway 重启导致的消息遗漏。
```

**效果：**
- Gateway 重启后 <30 秒内自动扫描所有 Agent inbox
- 发现未读消息立即触发处理
- 根治 .last-read 重置问题

**状态：** 📋 设计完成，待实施

---

#### 3.3.2 `message:sent` Hook — 消息发送后自动触发（可选）

**作用：** 捕获 Agent 发出的消息，自动触发目标 Agent

```typescript
// ~/.openclaw/hooks/inter-agent-trigger/handler.ts
const AGENT_MAP: Record<string, string> = {
  "阿尔法": "alpha", "贝塔": "beta", "伽马": "gamma",
  "德尔塔": "delta", "艾普西隆": "epsilon", "泽塔": "zeta",
  "艾塔": "eta", "西塔": "theta", "约塔": "iota",
  "卡帕": "kappa", "拉姆达": "lambda"
};

const handler = async (event) => {
  if (event.type !== "message" || event.action !== "sent") return;
  
  const { to, content, channelId } = event.context;
  const targetAgent = AGENT_MAP[to] || to;
  
  // 只处理飞书渠道的 Agent 间消息
  if (channelId !== "feishu") return;
  if (!AGENT_MAP[to]) return;
  
  // 检查是否已有对应的 webhook 触发（防重）
  // ... (去重逻辑)
  
  // 通过 webhook 触发
  await fetch("http://127.0.0.1:18789/hooks/agent", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${process.env.OPENCLAW_HOOKS_TOKEN}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify({
      message: `你有来自 ${event.context.from} 的新消息，请检查 inbox/`,
      agentId: targetAgent,
      name: "MsgSentTrigger",
      deliver: false,
      timeoutSeconds: 60
    })
  });
};

export default handler;
```

**注意：** 此 hook 依赖 `message:sent` 事件。根据 R2 分析，`sessions_send` 走 `inter_session` 路径，可能不触发 `message:sent`。需要实际测试验证。

**状态：** 📋 设计完成，需要测试验证

---

### 3.4 组件四：Cron 高频轮询（新增 ⭐）

**来源：** R2 研究成果
**作用：** 每 3 分钟检查所有 Agent inbox，作为可靠的中层兜底

#### 3.4.1 Cron 配置

```json5
// openclaw.json
{
  agents: {
    list: [{
      id: "main",  // 由主 Agent（阿尔法）执行
      cron: {
        jobs: [{
          name: "inbox-scan-all-agents",
          schedule: "*/3 * * * *",  // 每 3 分钟
          task: `扫描所有公民的 inbox/ 目录：
1. 读取每个 Agent 的 inbox/.last-read
2. 列出 .last-read 之后的新消息文件
3. 如有未读消息，通过 /hooks/agent 触发该 Agent 处理
4. 记录扫描结果到 shared-knowledge/logs/inbox-scan.log

Agent 列表：alpha, beta, gamma, delta, epsilon, zeta, eta, theta, iota, kappa, lambda

注意：只触发有未读消息的 Agent。没有未读的跳过。`,
          timeoutSeconds: 180
        }]
      }
    }]
  }
}
```

#### 3.4.2 扫描逻辑

```
每 3 分钟触发
  ↓
遍历 11 个 Agent 的 inbox/
  ↓
对比 .last-read vs 实际文件
  ↓
有未读？ ──► 是 ──► /hooks/agent 触发该 Agent
  │
  └─► 否 ──► 跳过
  ↓
记录扫描日志
```

**延迟：** 最坏 3 分钟
**可靠性：** 独立于 Agent session 和心跳

**状态：** 📋 设计完成，待实施

---

### 3.5 组件五：Heartbeat 心跳兜底（已有 ✅）

**状态：** ✅ 已实施（Phase 1）
**延迟：** 30 分钟
**作用：** 最终兜底，确保即使 cron 失效也能处理

**HEARTBEAT.md 标准配置：**

```markdown
# HEARTBEAT.md

## Focus Items（持续感知）

### 活跃任务
- [ ] 检查 inbox/ 新消息 | 持续检查 | 来自:系统

### 触发器
- inbox/ 检查（每心跳）— 以 .last-read 为基准
- 如有高优先级消息（priority=high/urgent），立即处理
- 如有未读通知文件（PHASE4-*.md / KB-*.md / URGENT-*.md），读取处理
```

---

## 四、降级策略

### 4.1 完整降级矩阵

| 场景 | Level 1 | Level 2 | Level 3 | Level 4 |
|------|---------|---------|---------|---------|
| 飞书 session 正常 | ✅ 秒到 | ✅ 3秒 | ✅ 3分钟 | ✅ 30分钟 |
| 飞书 session 卡死 | ❌ 失败 | ✅ 3秒 | ✅ 3分钟 | ✅ 30分钟 |
| Agent session 忙碌 | ⚠️ 入队 | ⚠️ 排队 | ✅ 3分钟 | ✅ 30分钟 |
| Gateway 刚重启 | ❌ 未恢复 | ⚠️ 待启动 | ✅ 3分钟 | ✅ 30分钟 |
| Webhook 未启用 | ❌ — | ❌ — | ✅ 3分钟 | ✅ 30分钟 |
| Cron 未配置 | ❌ — | ✅ 3秒 | ❌ — | ✅ 30分钟 |
| 所有机制失效 | ❌ — | ❌ — | ❌ — | ✅ 30分钟 |

### 4.2 关键设计决策

**为什么是四级而不是两级？**

1. **Level 1（sessions_send）** 是现有机制，不改，作为首选
2. **Level 2（Webhook）** 是 R1 发现的新能力，立即可用
3. **Level 3（Cron 3分钟）** 是 R2 推荐的可靠方案，独立于 Agent session
4. **Level 4（Heartbeat 30分钟）** 是已有机制，作为最终兜底

**为什么 Cron 是 3 分钟不是 5 分钟？**
- 3 分钟是用户体验可接受的延迟上限
- API 调用成本可接受（每 3 分钟一次扫描，每次扫描 ~11 个目录）
- 与 heartbeat（30 分钟）有 10 倍差距，确保 cron 失效时有明确的降级感知

---

## 五、Gateway 重启恢复协议

### 5.1 问题描述

Gateway 重启时会发生：
1. 所有 Agent 的 session 重建
2. `.last-read` 文件在磁盘上（持久化 ✅），但新 session 不主动扫描 inbox
3. 重启期间写入 inbox 的消息被遗漏
4. Agent 等待下一次心跳（可能 30 分钟后）才发现遗漏

### 5.2 v2.0 解决方案

```
Gateway 重启
  ↓
启动完成 → gateway:startup 事件触发
  ↓
startup-inbox-scan hook 执行
  ↓
扫描所有 Agent 的 inbox/
  ↓
对比 .last-read 指针
  ↓
发现未读消息 ──► 是 ──► /hooks/agent 触发对应 Agent
  │
  └─► 否 ──► 完成
  ↓
同时：Cron 3分钟轮询在 3 分钟后自动开始
  ↓
双重保障：startup hook + cron 轮询
```

### 5.3 恢复时间线

| 时间 | 事件 |
|------|------|
| T+0s | Gateway 重启开始 |
| T+5s | Gateway 启动完成，websocket 重连 |
| T+10s | gateway:startup hook 触发，扫描 inbox |
| T+15s | 发现未读消息，webhook 触发 Agent |
| T+20s | Agent 被唤醒，处理消息 |
| **T+30s** | **Gateway 重启后消息恢复完成** |
| T+3min | Cron 首次扫描，作为二次确认 |
| T+30min | Heartbeat 首次扫描，作为最终兜底 |

**vs v1.0：** Gateway 重启后消息恢复从"不确定（可能 30 分钟）"变为"30 秒内"。

---

## 六、实施计划

### Phase 1：基础增强（今天，1-2 小时）

| # | 任务 | 说明 | 预计工时 |
|---|------|------|---------|
| 1 | 启用 hooks 配置 | 修改 `openclaw.json`，添加 `hooks.enabled=true` | 15 分钟 |
| 2 | 设置 OPENCLAW_HOOKS_TOKEN | 写入 `~/.openclaw/.env` | 5 分钟 |
| 3 | 创建 startup-inbox-scan hook | 实现 gateway:startup hook handler | 30 分钟 |
| 4 | 测试 webhook 触发 | 验证 `/hooks/agent` 可以唤醒 Agent | 15 分钟 |
| 5 | 创建 inbox 检查 cron | 每 3 分钟扫描所有 Agent inbox | 15 分钟 |

**完成后效果：**
- ✅ Gateway 重启后 30 秒内自动恢复
- ✅ 3 分钟 cron 轮询兜底
- ✅ Webhook 即时触发可用

### Phase 2：Agent 集成（1-2 天）

| # | 任务 | 说明 | 预计工时 |
|---|------|------|---------|
| 1 | 编写 send-and-notify.sh | 统一的发送+通知脚本 | 1 小时 |
| 2 | 更新所有 Agent 的 AGENTS.md | 添加"发消息后调用 webhook"规范 | 30 分钟 |
| 3 | 测试 message:sent hook | 验证 sessions_send 是否触发 | 2 小时 |
| 4 | Agent 注册表 | 维护 Agent 名称 → agentId 映射 | 1 小时 |

**完成后效果：**
- ✅ Agent 发消息后秒级触发对方
- ✅ 所有 Agent 统一发送流程

### Phase 3：优化（可选，1 周）

| # | 任务 | 说明 |
|---|------|------|
| 1 | 消息去重 | 防止多通道同时投递导致重复处理 |
| 2 | 投递确认 | 消息被处理后回执通知发送方 |
| 3 | 消息优先级路由 | urgent 走 webhook，normal 走 cron |
| 4 | 性能监控 | 统计各级通道的投递成功率和延迟 |

---

## 七、Agent 发送规范（统一标准）

### 7.1 发送消息的标准流程

所有 Agent 发消息时必须遵循：

```
Step 1: 写入 inbox 文件（必须）
  → 写入 target-agent/inbox/msg-*.json

Step 2: 调用 webhook 触发（推荐）
  → POST /hooks/agent 触发对方

Step 3: 同时 sessions_send（可选，用于即时显示）
  → 如果需要对方在飞书看到
```

### 7.2 AGENTS.md 中的标准条目

在每个 Agent 的 AGENTS.md 中添加：

```markdown
## 消息发送规范

发消息给其他公民时：

1. **必须** 写入 inbox 文件：
   mkdir -p ~/.openclaw/workspace-<target>/inbox
   # 写入 JSON 消息文件

2. **推荐** webhook 触发对方：
   curl -X POST http://127.0.0.1:18789/hooks/agent \
     -H "Authorization: Bearer $HOOKS_TOKEN" \
     -d '{"message":"有新消息","agentId":"<target>","deliver":false}'

3. **可选** sessions_send（飞书即时显示）

不要只依赖一种方式。多通道 = 可靠。
```

---

## 八、安全考虑

### 8.1 Webhook 安全

- ✅ Token 认证（Bearer / Header）
- ✅ Query string token 拒绝
- ✅ 429 限速
- ✅ `allowedAgentIds` 限制
- ✅ `allowRequestSessionKey=false`（不允许请求指定 sessionKey）
- ✅ Hook payloads 视为不受信任（安全边界）

### 8.2 内部调用安全

- Webhook 绑定 loopback（127.0.0.1），不暴露外网
- HOOKS_TOKEN 仅存在 `.env` 文件中（mode 600）
- Agent 只能通过 exec 调用 curl，不能直接访问 Gateway API

---

## 九、衡量指标

### 9.1 关键指标

| 指标 | v1.0 | v2.0 目标 | 衡量方式 |
|------|------|----------|---------|
| 消息投递成功率 | ~70% | ≥99.9% | 日志统计 |
| 平均投递延迟 | ~15 分钟 | ≤3 秒（webhook 生效时） | 时间戳对比 |
| Gateway 重启恢复时间 | 不确定 | ≤30 秒 | startup hook 日志 |
| 最坏情况延迟 | 30 分钟 | 3 分钟 | Cron 间隔 |
| Session 卡死发现时间 | 手动 | ≤3 分钟 | Cron 自动检测 |

### 9.2 监控方式

```
shared-knowledge/logs/
├── inbox-scan.log          # Cron 扫描日志
├── webhook-trigger.log     # Webhook 调用日志
├── startup-scan.log        # Gateway 启动扫描日志
└── delivery-stats.json     # 投递统计（延迟、成功率）
```

---

## 十、总结

### v2.0 核心升级

| 维度 | v1.0 | v2.0 |
|------|------|------|
| 消息持久化 | ✅ inbox 文件 | ✅ 不变 |
| 即时触发 | ❌ 无 | ✅ Webhook + Hook Handler |
| Gateway 重启恢复 | ❌ 依赖心跳 | ✅ startup hook 30秒恢复 |
| 兜底机制 | ✅ 心跳 30分钟 | ✅ Cron 3分钟 + 心跳 30分钟 |
| 外部系统集成 | ❌ 无 | ✅ /hooks/agent 端点 |
| 降级层级 | 2 级 | 4 级 |

### 实施优先级

```
🔴 高优先级（今天做）
  ├─ 启用 hooks 配置（15 分钟）
  ├─ 创建 startup-inbox-scan hook（30 分钟）
  └─ 创建 inbox 检查 cron（15 分钟）

🟡 中优先级（本周做）
  ├─ 编写 send-and-notify.sh（1 小时）
  └─ 更新所有 Agent 规范（30 分钟）

🟢 低优先级（可选）
  ├─ message:sent hook（需测试验证）
  └─ 投递确认 + 去重
```

### 最终效果

```
梦想家发消息 → 阿尔法
  ↓
写入 inbox + webhook 触发
  ↓
3 秒内阿尔法收到 ✅

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

- v1.0 方案：`shared-knowledge/architecture/messaging-architecture.md`
- R1 研究：`shared-knowledge/research/openclaw-webhook-analysis.md`
- R2 研究：`shared-knowledge/research/on-message-mechanism-analysis.md`
- OpenClaw Webhook 文档：`docs/automation/webhook.md`
- OpenClaw Hooks 文档：`docs/automation/hooks.md`
- OpenClaw Cron 文档：`docs/automation/cron-jobs.md`
- .last-read bug 分析：贝塔 inbox 消息（2026-03-18 16:16）

---

*本方案由拉姆达 🔬 基于 Phase 1 实施经验 + R1/R2 研究成果 + 实际 bug 发现整合而成。*
*交付物路径：`shared-knowledge/architecture/instant-messaging-v2.md`*
