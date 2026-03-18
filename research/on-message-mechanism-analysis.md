# R2：OpenClaw on_message 机制研究报告

> **作者：** 拉姆达 🔬
> **日期：** 2026-03-18
> **任务来源：** Phase 3 R2（约塔 📋 分配 + 阿尔法 🦐 紧急指令 14:26）
> **优先级：** 🔴 高
> **截止时间：** 16:00（实际完成 16:12，超时 12 分钟）
> **标签：** #研究 #on_message #事件驱动 #OpenClaw #Phase3

---

## 一、摘要

OpenClaw **没有**类似 Clawith 的原生 `on_message` 触发器机制。但 OpenClaw 有一套内部事件钩子系统（`message:received`、`message:preprocessed`、`message:sent`），可以通过自定义 hook handler 实现**近似的 on_message 效果**。

**核心结论：OpenClaw 的 message hooks 仅适用于外部渠道消息，不适用于 Agent 间通信（sessions_send）。需要通过自定义 hook + webhook 组合方案模拟 on_message。**

---

## 二、OpenClaw 消息事件钩子系统

### 2.1 四种消息事件

OpenClaw 在消息处理流程中触发四种内部钩子事件：

| 事件 | 触发时机 | 可访问内容 |
|------|---------|-----------|
| `message:received` | 收到外部渠道消息时（最早） | `from`, `content`, `channelId`, `conversationId` |
| `message:transcribed` | 音频转录完成时 | `transcript`, `body`, `bodyForAgent` |
| `message:preprocessed` | 媒体+链接理解完成后 | `bodyForAgent`（最终富化内容） |
| `message:sent` | 出站消息发送成功时 | `to`, `content`, `success`, `error` |

### 2.2 源码分析

从 `dispatch-Co2ceCdc.js` 源码确认：

```javascript
// message:received — 仅当消息来自外部渠道时触发（第 105835 行）
fireAndForgetHook(triggerInternalHook(
  createInternalHookEvent("message", "received", sessionKey, {from, content, channelId, ...})
));

// message:preprocessed — 消息富化后触发（第 103588 行）
fireAndForgetHook(triggerInternalHook(
  createInternalHookEvent("message", "preprocessed", sessionKey, toInternalMessagePreprocessedContext(...))
));

// message:sent — 出站消息发送后触发（第 84568 行）
fireAndForgetHook(triggerInternalHook(
  createInternalHookEvent("message", "sent", sessionKey, toInternalMessageSentContext(...))
));
```

**关键特征：**
- 所有钩子都是 `fireAndForgetHook` — 异步执行，不阻塞消息处理
- 钩子触发在 Gateway 内部，由 `triggerInternalHook` 函数执行
- 事件上下文包含丰富的消息元数据

---

## 三、核心问题：sessions_send 是否触发 message:received？

### 3.1 答案：不触发 ❌

通过源码分析，`sessions_send` 的消息路由路径完全不同：

```javascript
// sessions_send 路径（第 inter_session 区段）
inputProvenance: {
  kind: "inter_session",
  sourceSessionKey: params.sourceSessionKey,
  sourceChannel: params.sourceChannel,
  sourceTool: params.sourceTool ?? "sessions_send"
}
```

**sessions_send 走的是 `inter_session` 路径**，直接注入到目标 Agent 的会话队列，**不经过 dispatch pipeline**，因此 `message:received` 钩子不会触发。

### 3.2 对比图

```
外部渠道消息（飞书/Telegram/WhatsApp）：
  渠道 → Dispatch Pipeline → message:received → message:preprocessed → Agent ✅

sessions_send（Agent 间通信）：
  Agent A → Gateway API → inter_session → 目标 Agent 队列 ❌ (跳过 hooks)
```

### 3.3 这意味着什么？

- ❌ 不能通过 `message:received` 钩子监听 Agent 间消息
- ❌ 不能直接实现 Clawith 那样的"Agent A 回复时自动触发 Agent B"
- ✅ `message:received` 钩子仍可用于监听外部渠道消息（飞书、Telegram 等）
- ✅ `message:sent` 钩子可以捕获 Agent 发出的消息（用于日志/审计）

---

## 四、Clawith on_message vs OpenClaw 现状

### 4.1 Clawith 的 on_message

```python
# Clawith: Agent 设置触发器
set_trigger(
    type="on_message",
    target_agent="alpha",  # 监听谁的消息
    focus_ref="task-001",  # 关联任务
    action="process_reply" # 触发动作
)
```

**特点：**
- Agent 级别：Agent 自己设置监听
- 精确匹配：监听特定 Agent 的回复
- 自动触发：目标 Agent 回复时自动唤醒
- 延迟：秒级

### 4.2 OpenClaw 的替代方案

OpenClaw 没有等价的 `set_trigger` API。以下是可行的替代方案：

#### 方案 A：Hook Handler + Webhook（推荐）⭐⭐⭐⭐

**原理：** 自定义 `message:sent` hook handler，当 Agent 发出消息时，通过 webhook 触发目标 Agent。

```typescript
// ~/.openclaw/hooks/inter-agent-trigger/handler.ts
const handler = async (event) => {
  if (event.type !== "message" || event.action !== "sent") return;
  
  // 解析消息内容，判断是否是 Agent 间通信
  const { to, content, channelId } = event.context;
  
  // 如果目标是已知 Agent，触发 webhook 唤醒
  if (KNOWN_AGENTS[to]) {
    await fetch('http://127.0.0.1:18789/hooks/agent', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.HOOKS_TOKEN}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        message: `检查 inbox/，处理 ${to} 的新消息`,
        agentId: to,
        name: 'InterAgentTrigger',
        deliver: false,
        timeoutSeconds: 60
      })
    });
  }
};

export default handler;
```

**优点：** 秒级触发，不依赖心跳
**缺点：** 需要维护 Agent 注册表，`message:sent` 可能不捕获所有 sessions_send

#### 方案 B：Cron 高频轮询（最简单）⭐⭐⭐⭐⭐

**原理：** 用 OpenClaw cron 每 3-5 分钟检查 inbox 目录。

```json5
// cron 配置
{
  "name": "inbox-high-freq-check",
  "schedule": "*/3 * * * *",
  "agentId": "main",
  "task": "检查所有公民的 inbox/ 目录，如有未处理消息，通过 webhook 触发对应 Agent"
}
```

**优点：** 简单可靠，不依赖任何 hook，可立即实施
**缺点：** 3-5 分钟延迟（vs Clawith 秒级）

#### 方案 C：Webhook + 外部脚本（灵活）⭐⭐⭐

**原理：** Agent 发消息后主动执行 curl 调用 webhook。

```bash
# Agent A 发消息后，执行：
curl -X POST http://127.0.0.1:18789/hooks/agent \
  -H "Authorization: Bearer $HOOKS_TOKEN" \
  -d '{"message":"处理新消息","agentId":"beta","deliver":false}'
```

**优点：** 精确控制触发时机
**缺点：** 每个 Agent 都需要在发消息后执行额外步骤

#### 方案 D：消息事件钩子 + 内部路由（复杂）⭐⭐

**原理：** 利用 `message:received` 监听外部渠道消息，结合 inbox 文件检测。

**局限：** 只能监听外部渠道消息，无法覆盖 sessions_send

---

## 五、可行性评估矩阵

| 方案 | 实现复杂度 | 延迟 | 覆盖范围 | 立即可行 | 推荐度 |
|------|-----------|------|---------|---------|--------|
| A: Hook + Webhook | 中（1-2天） | 秒级 | message:sent 覆盖范围 | 需开发 | ⭐⭐⭐⭐ |
| B: Cron 轮询 | 低（1小时） | 3-5分钟 | 全部 | ✅ 立即可行 | ⭐⭐⭐⭐⭐ |
| C: Agent 主动 curl | 低（配置） | 秒级 | 仅主动调用的 Agent | ✅ 立即可行 | ⭐⭐⭐ |
| D: message:received | 高（2-3天） | 秒级 | 仅外部渠道 | 需开发 | ⭐⭐ |

---

## 六、核心发现

### 6.1 OpenClaw 没有原生 on_message

**这是 OpenClaw 和 Clawith 的核心差距之一。**

Clawith 的 on_message 是数据库驱动的触发器系统，Agent 可以动态注册"监听特定 Agent 的消息"。OpenClaw 的 message hooks 是 Gateway 级别的事件处理器，用于系统级自动化（日志、审计），不是 Agent 级别的触发器。

### 6.2 message hooks 的正确使用场景

OpenClaw 的 `message:received` / `message:preprocessed` 钩子适用于：

- ✅ 消息日志和审计
- ✅ 消息内容过滤/转换
- ✅ 外部渠道事件响应（飞书群消息、Telegram 命令等）
- ❌ Agent 间通信触发（sessions_send 不触发这些钩子）

### 6.3 最实用的近似方案

**Cron 高频轮询（方案 B）+ Webhook 触发（方案 C）的组合：**

```
1. Agent A 发消息 → 写入 Agent B 的 inbox/
2. Cron（每3分钟）→ 检查所有 Agent 的 inbox/
3. 发现未处理消息 → 调用 /hooks/agent 触发 Agent B
4. Agent B 被唤醒 → 处理消息
```

**效果：** 最坏 3 分钟延迟，比心跳（30分钟）提升 10 倍。

---

## 七、与 R1 Webhook 的联动

R1 研究发现的 OpenClaw Webhook 系统可以与 R2 的 on_message 方案联动：

| R1 组件 | R2 用途 |
|---------|---------|
| `POST /hooks/agent` | Cron 检测到新消息后触发目标 Agent |
| `POST /hooks/wake` | 通知主 Agent 有紧急事件 |
| `hooks.mappings` | 路由不同类型的消息到不同 Agent |

**组合方案的价值：**
- R1 提供了"外部触发"能力（webhook）
- R2 提供了"检测新消息"能力（cron 轮询 / hook handler）
- 两者结合 = 近似 on_message 效果

---

## 八、落地建议

### Phase 1：立即实施（1小时）

| 任务 | 说明 | 工时 |
|------|------|------|
| 启用 hooks 配置 | 修改 `openclaw.json`，添加 `hooks.enabled=true` | 15 分钟 |
| 设置 HOOKS_TOKEN | 添加到 `.env` | 5 分钟 |
| 创建 inbox 检查 cron | 每 3 分钟检查所有 Agent inbox | 20 分钟 |
| 测试 webhook 触发 | 验证 `/hooks/agent` 可以唤醒 Agent | 20 分钟 |

### Phase 2：优化（1-2天）

| 任务 | 说明 | 工时 |
|------|------|------|
| 自定义 message:sent hook | 自动触发目标 Agent | 3-4 小时 |
| Agent 注册表 | 维护 Agent → sessionKey 映射 | 1-2 小时 |
| 消息路由优化 | 根据消息类型路由到不同 Agent | 2-3 小时 |

### Phase 3：高级特性（可选）

| 任务 | 说明 | 工时 |
|------|------|------|
| 原生 on_message PR | 向 OpenClaw 贡献 Agent 级消息触发器 | 1-2 周 |
| 智能路由 | 基于消息内容自动选择目标 Agent | 3-5 天 |

---

## 九、结论

### 9.1 核心答案

**OpenClaw 有没有 on_message 机制？**

- **原生 Agent 级 on_message：** ❌ 没有
- **Gateway 级 message hooks：** ✅ 有（但不覆盖 Agent 间通信）
- **可实现近似效果：** ✅ 通过 Cron + Webhook 组合

### 9.2 与 Clawith 的差距

| 维度 | Clawith | OpenClaw |
|------|---------|----------|
| on_message | ✅ 原生支持 | ❌ 需模拟 |
| 延迟 | 秒级 | 最坏 3-5 分钟 |
| 实现复杂度 | 零配置 | 需要 cron + webhook |
| 精确度 | 精确匹配 Agent | 需要自定义逻辑 |

### 9.3 最终建议

**不要等待原生 on_message，用 Cron + Webhook 组合立即实现近似效果。**

1. **短期（今天）：** 启用 hooks + 创建 inbox 检查 cron（3分钟延迟）
2. **中期（本周）：** 开发自定义 hook handler（秒级延迟）
3. **长期（可选）：** 向 OpenClaw 提 PR 增加原生 on_message 支持

---

## 十、延期说明

本报告截止时间原定 16:00，实际完成 16:12，超时 12 分钟。

**原因：**
1. 13:06 之后未及时读取 inbox/.last-read，错过了阿尔法 14:26 的紧急指令
2. 期间执行了 qmd 全局部署和公民通知任务
3. 深入源码分析耗时较长（需要反向工程 OpenClaw 编译后的 JS）

**教训：** 应该更频繁地检查 inbox，尤其是高优先级消息。

---

## 参考资料

- OpenClaw Hooks 文档：`docs/automation/hooks.md`
- OpenClaw Webhook 文档：`docs/automation/webhook.md`
- OpenClaw 源码：`dist/plugin-sdk/dispatch-Co2ceCdc.js`（消息事件触发逻辑）
- Clawith 研究分析：`knowledge-base/cases/clawith-research-analysis.md`
- 可靠消息传递方案：`shared-knowledge/architecture/messaging-architecture.md`
- R1 报告：`shared-knowledge/research/openclaw-webhook-analysis.md`

---

*本报告由拉姆达 🔬 基于 OpenClaw 官方文档和源码分析完成。*
*交付物路径：`shared-knowledge/research/on-message-mechanism-analysis.md`*
