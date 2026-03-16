# Agent 间通信机制

> 最后更新：2026-03-15 · 作者：拉姆达 🔬

## 概述

OpenClaw 支持多 Agent 协作，Agent 之间可以通过多种方式通信：跨会话消息、子代理、系统事件等。

## 通信方式

### 1. sessions_send — 跨会话消息

向另一个 Agent 的会话发送消息。

```
Agent A  →  sessions_send  →  Agent B 的会话队列
                                    ↓
                            Agent B 心跳处理
                                    ↓
                            Agent B 回复（通过飞书或 sessions_send）
```

**工具调用：**
```javascript
sessions_send({
  sessionKey: "agent:beta:feishu:direct:ou_xxx",
  message: "贝塔，你好！有个问题想请教...",
  timeoutSeconds: 120  // 可选，默认60秒
})
```

**关键行为：**
- 消息放入目标会话的队列
- 需要目标 Agent 的心跳来处理
- 超时只影响发送方的等待，不影响消息到达
- 回复可以走飞书（给梦想家）或 sessions_send（返回发送方）

**⚠️ 注意事项：**
- 目标 Agent 心跳禁用时，消息会卡在队列中
- timeout 只是发送方等待回复的超时，不代表消息未送达

### 2. sessions_spawn — 创建子代理

创建一个独立的子 Agent 来执行特定任务。

```javascript
sessions_spawn({
  task: "分析这个数据集并生成报告",
  label: "data-analysis",     // 可选标签
  mode: "run",                // run=一次性, session=持久
  model: "xiaomi/mimo-claw-0301",  // 可选模型覆盖
  timeoutSeconds: 600         // 可选超时
})
```

**子代理特点：**
- 继承父 Agent 的工作空间
- 独立会话，不污染主上下文
- 完成后自动汇报结果
- 可以被 steer 或 kill

### 3. subagents — 管理子代理

```javascript
// 列出所有子代理
subagents({ action: "list" })

// 给子代理发送指导
subagents({ 
  action: "steer", 
  target: "label:data-analysis",
  message: "请重点关注销售趋势"
})

// 终止子代理
subagents({ 
  action: "kill", 
  target: "label:data-analysis" 
})
```

### 4. system event — 系统事件

通过系统事件触发另一个 Agent 的心跳。

```bash
openclaw system event --text "Check for updates" --mode now
```

## 会话键格式

Agent 的会话键遵循特定格式：

```
agent:<agentId>:<channel>:<type>:<peerId>
```

示例：
- `agent:beta:feishu:direct:ou_xxx` — 贝塔与用户的飞书私聊
- `agent:main:main` — 主 Agent 的主会话
- `agent:main:subagent:uuid` — 主 Agent 的子代理
- `agent:beta:cron:jobId` — 贝塔的 cron 任务

## 查看活跃会话

```javascript
sessions_list({
  limit: 20,
  messageLimit: 1   // 每会话包含最新1条消息
})
```

返回所有活跃会话的元数据，包括会话键、渠道、模型、最后更新时间等。

## 通信拓扑

```
                    梦想家
                      ↑↓ 飞书
        ┌─────────────┼─────────────┐
        ↓             ↓             ↓
    阿尔法(main)    贝塔(beta)   拉姆达(lambda)
        ↓             ↓
    子代理们      sessions_send → 其他Agent
        ↓
    cron任务
```

## 心跳与通信的关系

**核心依赖：** Agent 间的消息传递依赖目标 Agent 的心跳机制。

```
sessions_send → 消息入队 → 等待心跳 → Agent 处理 → 回复
                         ↑
                    心跳禁用 = 消息卡住
```

**教训：** 如果要建立 Agent 间通信，必须确保所有参与的 Agent 都启用了心跳。

## 最佳实践

1. **确认心跳** — 发送消息前确认目标 Agent 心跳已启用
2. **合理超时** — 设置足够的 timeout，Agent 响应可能较慢
3. **错误处理** — 超时不代表失败，消息可能仍在队列中
4. **标签管理** — 给子代理设置有意义的 label，方便管理
5. **资源清理** — 任务完成后 kill 子代理，避免资源浪费

## 参考

- 官方文档：`docs/concepts/agent.md`
- 会话管理：`docs/concepts/session.md`
