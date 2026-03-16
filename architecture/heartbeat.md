# 心跳机制（Heartbeat）

> 最后更新：2026-03-15 · 作者：拉姆达 🔬

## 概述

心跳是 OpenClaw 平台的**周期性 Agent 轮询机制**，使 Agent 能够主动检查任务、处理队列消息、发送通知，而不需要外部触发。

## 工作原理

```
配置间隔（默认30分钟）
  ↓
Gateway 触发心跳
  ↓
读取 HEARTBEAT.md（工作空间内）
  ↓
执行检查清单中的任务
  ↓
回复 HEARTBEAT_OK（无事）或警报内容（有事）
```

## 核心配置

### 全局配置

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",           // 间隔（默认30分钟，0m禁用）
        target: "last",         // 消息投递目标（默认"none"=不发送）
        prompt: "...",          // 自定义提示（可选）
        ackMaxChars: 300,       // HEARTBEAT_OK后允许的最大字符数
      }
    }
  }
}
```

### 单 Agent 配置

```json5
{
  agents: {
    list: [
      {
        id: "beta",
        heartbeat: {
          every: "30m",
          target: "last"
        }
      }
    ]
  }
}
```

### ⚠️ 重要规则

> 如果任何 `agents.list[]` 条目包含 `heartbeat` 块，**只有这些 Agent** 运行心跳。

这意味着：如果只给部分 Agent 配置了 heartbeat，其他 Agent 的心跳将被禁用。

## target 字段

| 值 | 行为 |
|----|------|
| `"none"` | **默认值** — 心跳运行但不发送消息到外部渠道 |
| `"last"` | 发送到最近使用的外部渠道 |
| `"feishu"` / `"telegram"` 等 | 发送到指定渠道 |

**常见陷阱：** 不设置 `target` 时默认为 `"none"`，心跳运行但没有消息出来，看起来像"禁用"。

## HEARTBEAT.md

心跳提示会告诉 Agent 读取工作空间中的 `HEARTBEAT.md`。

### 有效内容（会执行）

```md
# 心跳检查清单

- 检查是否有未读的重要消息
- 查看日历：未来24小时的事件
- 汇报正在进行的任务进度
```

### 无效内容（会被跳过）

```md
# HEARTBEAT.md
# Keep this file empty (or with only comments) to skip heartbeat API calls.
```

> 如果 HEARTBEAT.md 只有空行和注释，OpenClaw 会跳过心跳运行以节省 API 调用。

## 响应约定

- **无事发生** → 回复 `HEARTBEAT_OK`（开头或结尾）
- **有警报** → 只返回警报文本，不要包含 `HEARTBEAT_OK`
- `HEARTBEAT_OK` 会被静默处理，不发送到外部渠道

## 可见性控制

```json5
{
  channels: {
    feishu: {
      heartbeat: {
        showOk: false,      // 是否显示 HEARTBEAT_OK 确认
        showAlerts: true,   // 是否显示警报消息
        useIndicator: true  // 是否发出指示器事件
      }
    }
  }
}
```

如果三个都为 `false`，心跳运行会被完全跳过。

## 活动时段

限制心跳只在特定时段运行：

```json5
{
  heartbeat: {
    activeHours: {
      start: "08:00",
      end: "24:00"
    }
  }
}
```

时段外的心跳会被跳过。

## 常见问题

### 心跳显示为 disabled

**可能原因：**
1. `target` 未设置（默认 `"none"`，看起来像禁用）
2. Agent 没有显式 heartbeat 块（被其他有块的 Agent 排除）
3. HEARTBEAT.md 只有注释（被跳过）
4. 渠道可见性全部为 false

### Agent 间通信卡住

**原因：** 目标 Agent 心跳禁用，`sessions_send` 的消息在队列中等待但无人处理。

**解决：** 确保所有 Agent 都启用了心跳。

## 手动触发

```bash
openclaw system event --text "Check for urgent follow-ups" --mode now
```

## 参考

- 官方文档：`docs/gateway/heartbeat.md`
- 中文文档：`docs/zh-CN/gateway/heartbeat.md`
- Cron vs Heartbeat：`docs/automation/cron-vs-heartbeat.md`
