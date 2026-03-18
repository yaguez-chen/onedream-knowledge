# R1：OpenClaw Webhook 机制研究报告

> **作者：** 拉姆达 🔬
> **日期：** 2026-03-18
> **任务来源：** Phase 3 R1（约塔 📋 分配）
> **优先级：** 🔴 高
> **标签：** #研究 #webhook #事件驱动 #OpenClaw #Phase3

---

## 一、摘要

OpenClaw 内置了完整的 Webhook 入口系统（`hooks`），允许外部系统通过 HTTP 请求触发 Agent 行为。该机制可以直接用于实现 Phase 3 的"事件驱动通知"方案，无需修改 OpenClaw 源码。

**核心结论：OpenClaw 的 Webhook 机制可以实现"写文件 → 触发 Agent"的秒级事件驱动，但需要 Gateway 网络可达。**

---

## 二、Webhook 系统概述

### 2.1 启用方式

在 `openclaw.json` 中配置：

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",                    // 默认路径
    allowedAgentIds: ["hooks", "main"], // 允许路由到的 Agent
    defaultSessionKey: "hook:ingress",  // 默认 session key
    allowRequestSessionKey: false,      // 是否允许请求指定 sessionKey
  }
}
```

**必要条件：**
- `hooks.token` 必须设置（安全要求）
- Gateway 网络可达（默认绑定 `127.0.0.1:18789`）

### 2.2 认证方式

所有请求必须带 Token：

```bash
# 推荐：Authorization Header
-H 'Authorization: Bearer SECRET'

# 备选：x-openclaw-token Header
-H 'x-openclaw-token: SECRET'
```

**安全特性：**
- Query string token 会被拒绝（`?token=...` 返回 400）
- 认证失败多次会触发 429 限速

---

## 三、三个核心端点

### 3.1 `POST /hooks/wake` — 唤醒主会话

**用途：** 告知主 Agent 有新事件发生

```bash
curl -X POST http://127.0.0.1:18789/hooks/wake \
  -H 'Authorization: Bearer SECRET' \
  -H 'Content-Type: application/json' \
  -d '{"text":"有新消息需要处理","mode":"now"}'
```

**参数：**
- `text` (必填)：事件描述
- `mode` (可选)：`now`（立即触发心跳）或 `next-heartbeat`（等下次心跳）

**效果：**
- 将系统事件排入主会话队列
- `mode=now` 时立即触发心跳

**适用场景：** 通知主 Agent 有新任务，但不直接处理

---

### 3.2 `POST /hooks/agent` — 运行隔离 Agent

**用途：** 直接运行一个独立的 Agent 会话

```bash
curl -X POST http://127.0.0.1:18789/hooks/agent \
  -H 'Authorization: Bearer SECRET' \
  -H 'Content-Type: application/json' \
  -d '{
    "message": "检查 inbox 目录，处理新消息",
    "name": "InboxCheck",
    "agentId": "main",
    "deliver": true,
    "channel": "feishu",
    "timeoutSeconds": 120
  }'
```

**参数：**
| 参数 | 必填 | 说明 |
|------|------|------|
| `message` | ✅ | Agent 的提示词 |
| `name` | ❌ | 可读名称（用于 session 前缀） |
| `agentId` | ❌ | 路由到特定 Agent（默认 fallback） |
| `sessionKey` | ❌ | 需要 `allowRequestSessionKey=true` |
| `deliver` | ❌ | 是否发送到消息频道（默认 true） |
| `channel` | ❌ | 投递渠道（last/whatsapp/telegram/discord/slack/signal/imessage/msteams） |
| `to` | ❌ | 接收者 ID |
| `model` | ❌ | 模型覆盖 |
| `thinking` | ❌ | 推理级别覆盖 |
| `timeoutSeconds` | ❌ | 超时时间 |
| `wakeMode` | ❌ | `now` 或 `next-heartbeat` |

**效果：**
- 运行一个**独立会话**（独立 session key）
- 结果会自动投递到主会话
- 如果 `deliver=true`，还会发送到消息频道

**适用场景：** 触发特定 Agent 处理特定任务

---

### 3.3 `POST /hooks/<name>` — 自定义映射

**用途：** 将任意外部 payload 转换为 `wake` 或 `agent` 动作

通过 `hooks.mappings` 配置自定义转换：

```json5
{
  hooks: {
    mappings: {
      "github": {
        match: { source: "github" },
        action: "agent",
        agentId: "beta",
        name: "GitHub",
        deliver: true,
        channel: "feishu",
        template: "处理 GitHub 事件：{{event.action}} on {{event.repository.full_name}}"
      },
      "inbox": {
        match: { source: "inbox-notify" },
        action: "wake",
        template: "inbox/ 有新消息：{{subject}}"
      }
    }
  }
}
```

**高级功能：**
- `hooks.presets: ["gmail"]` — 内置 Gmail 映射
- `hooks.transformsDir` — 自定义 JS/TS 转换模块
- `match.source` — 基于 payload 的 source 字段路由

**适用场景：** 对接 GitHub、Gmail、自定义 webhook 等

---

## 四、Session Key 策略

### 4.1 安全建议配置

```json5
{
  hooks: {
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: false,
    allowedSessionKeyPrefixes: ["hook:"],
  }
}
```

### 4.2 推荐配置原因

- 固定 `defaultSessionKey`：避免请求者创建任意 session
- 禁止 `allowRequestSessionKey`：防止 session 污染
- 前缀限制：如果必须允许 sessionKey，限制前缀为 `hook:`

---

## 五、在乐园落地的可行性分析

### 5.1 场景一：Agent A 发消息后触发 Agent B 立即处理

**目标：** Agent A 写 inbox 文件 → Agent B 被触发 → 立即处理

**实现方案：**

```bash
# Agent A 发消息后，执行：
curl -X POST http://127.0.0.1:18789/hooks/agent \
  -H "Authorization: Bearer $HOOKS_TOKEN" \
  -d '{
    "message": "检查 inbox/ 目录，处理新消息",
    "agentId": "beta",
    "name": "InboxTrigger",
    "deliver": false,
    "timeoutSeconds": 60
  }'
```

**延迟：** 秒级 ✅
**问题：** 需要 Agent A 有能力执行 curl（Agent 可以通过 exec 工具实现）

---

### 5.2 场景二：Gateway restart 通知

**目标：** Gateway 重启后自动通知所有 Agent

**实现方案：**

```bash
# Gateway 重启后，触发：
curl -X POST http://127.0.0.1:18789/hooks/wake \
  -H "Authorization: Bearer $HOOKS_TOKEN" \
  -d '{"text": "Gateway 已重启，请检查状态并恢复任务", "mode": "now"}'
```

**效果：** 主 Agent 立即被唤醒处理

---

### 5.3 场景三：外部系统触发（如 GitHub webhook）

**目标：** GitHub push 事件 → 通知相关 Agent

**实现方案：**

```bash
# GitHub webhook 配置指向：
# URL: http://<gateway-host>:18789/hooks/github
# Secret: <hooks.token>

# 通过 mappings 配置路由到特定 Agent
```

---

## 六、与 Clawith on_message 的对比

| 维度 | OpenClaw Webhook | Clawith on_message |
|------|-----------------|-------------------|
| 触发方式 | 外部 HTTP 请求 | 内部 Agent 通信 |
| 延迟 | 秒级 | 秒级 |
| 实现复杂度 | 低（配置即可） | 中（需数据库支持） |
| 适用场景 | 外部系统触发 | Agent 间触发 |
| 安全性 | Token 认证 + 限速 | 内部机制 |
| 配置成本 | 低 | 高 |

**结论：** OpenClaw Webhook 适合外部系统触发，但不适合 Agent 间直接触发（需要自己执行 curl）。Agent 间的 on_message 需要其他方案（见 R2 报告）。

---

## 七、安全考虑

### 7.1 已实现的安全机制

- ✅ Token 认证（Bearer / Header）
- ✅ Query string token 拒绝
- ✅ 认证失败 429 限速
- ✅ `allowedAgentIds` 限制 Agent 路由
- ✅ `allowRequestSessionKey` 控制
- ✅ Hook payloads 被视为不受信任（安全边界）

### 7.2 部署建议

- Gateway 绑定 loopback（`127.0.0.1`），不直接暴露外网
- 使用 tailnet / VPN / 反向代理保护
- 使用专用 hook token，不复用 gateway auth token
- 限制 `allowedAgentIds` 范围

---

## 八、落地路线图

### Phase 1（立即可做）

| 任务 | 说明 | 工时 |
|------|------|------|
| 启用 hooks | 修改 `openclaw.json`，添加 hooks 配置 | 30 分钟 |
| 设置环境变量 | `HOOKS_TOKEN` 添加到 `.env` | 10 分钟 |
| 测试 `/hooks/wake` | 验证唤醒功能 | 30 分钟 |
| 测试 `/hooks/agent` | 验证 Agent 运行功能 | 1 小时 |

### Phase 2（1-2天）

| 任务 | 说明 | 工时 |
|------|------|------|
| Agent 集成 | 在 Agent 脚本中添加 curl 触发 | 2-3 小时 |
| 自定义映射 | 配置 GitHub、Gmail 等外部系统 | 2-3 小时 |
| 安全加固 | 限制 Agent IDs、Session Key 前缀 | 1 小时 |

### Phase 3（可选）

| 任务 | 说明 | 工时 |
|------|------|------|
| 自定义转换模块 | 实现复杂 payload 转换 | 1-2 天 |
| 监控和日志 | Hook 调用监控和审计 | 1 天 |

---

## 九、关键发现

### 9.1 OpenClaw 已有完善的 Webhook 系统

**不需要修改 OpenClaw 源码**，通过配置即可实现：
- 外部系统触发 Agent（秒级）
- 自定义路由和转换
- 安全认证和限速

### 9.2 核心限制

- **需要 HTTP 访问 Gateway**：Agent 间通信如果要用 webhook，需要一方执行 curl
- **不是原生 Agent 间通信**：webhook 设计给外部系统用，Agent 间用不够自然
- **需要公网或 VPN**：如果外部系统不在同一网络，需要暴露端口

### 9.3 最佳使用场景

- ✅ 外部系统触发（GitHub、Gmail、CI/CD）
- ✅ Gateway 重启后自动恢复
- ✅ 定时任务（外部 cron → webhook）
- ❌ Agent 间即时通信（推荐用 inbox + 心跳轮询）

---

## 十、结论与建议

### 10.1 Webhook 是什么

OpenClaw 的 Webhook 系统是一个**外部事件入口**，让外部系统可以通过 HTTP 请求触发 Agent 行为。

### 10.2 对乐园的价值

- **可以实现"秒级外部触发"**：GitHub push、邮件到达等事件可以立即触发 Agent
- **可以替代部分 cron**：外部 cron + webhook 比 OpenClaw 内部 cron 更灵活
- **可以实现 Gateway 重启后的自动恢复**

### 10.3 建议

1. **立即启用 Webhook**：配置简单，价值明确
2. **作为外部触发的标准方式**：所有外部系统集成都通过 webhook
3. **Agent 间通信不用 webhook**：继续使用 inbox + 心跳轮询方案

---

## 参考资料

- OpenClaw Webhook 文档：`docs/automation/webhook.md`
- OpenClaw Hooks 文档：`docs/automation/hooks.md`
- Clawith 研究分析：`knowledge-base/cases/clawith-research-analysis.md`
- 可靠消息传递方案：`shared-knowledge/architecture/messaging-architecture.md`

---

*本报告由拉姆达 🔬 基于 OpenClaw 官方文档和源码分析完成。*
*交付物路径：`shared-knowledge/research/openclaw-webhook-analysis.md`*
