# 消息防抖机制（Message Debouncing）

> 最后更新：2026-03-15 · 作者：拉姆达 🔬

## 概述

OpenClaw 有三层消息处理保护机制：**去重**、**防抖**和**队列合并**，共同防止重复处理、资源竞争和消息丢失。

## 一、入站去重（Inbound Dedupe）

### 问题

渠道重连后可能重复投递同一条消息。

### 解决

OpenClaw 维护一个短期缓存，按 `channel/account/peer/session/message id` 键存储。

```
消息到达 → 缓存检查
  ↓
命中 → 丢弃（重复）
未命中 → 正常处理 + 写入缓存
```

## 二、入站防抖（Inbound Debouncing）

### 问题

用户快速连续发送多条消息，每条触发一次 Agent run，浪费资源且可能产生冲突回复。

### 解决

等待安静期后再处理，将防抖窗口内的所有消息合并为一次 Agent turn。

### 配置

```json5
{
  messages: {
    inbound: {
      debounceMs: 2000,      // 全局默认 2秒
      byChannel: {
        whatsapp: 5000,      // WhatsApp 5秒
        slack: 1500,         // Slack 1.5秒
        discord: 1500,       // Discord 1.5秒
        feishu: 2000         // 飞书 2秒（如未设置则用全局默认）
      }
    }
  }
}
```

### 工作流程

```
t=0s    消息A到达 → 开始防抖计时
t=0.5s  消息B到达 → 重置计时器
t=1s    消息C到达 → 重置计时器
t=3s    安静期结束（无新消息）
        ↓
合并 A+B+C → 一次 Agent turn（使用C的回复ID）
```

### 特殊规则

| 消息类型 | 防抖行为 |
|----------|----------|
| 纯文本 | 参与防抖合并 |
| 媒体/附件 | 立即刷新（不防抖） |
| 控制命令 | 绕过防抖，独立执行 |

## 三、队列机制（Queue）

### 问题

Agent 正在处理时，新消息到达，如何处理？

### 队列模式

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| `collect` | **默认** — 合并所有排队消息为一次 followup | 日常对话 |
| `steer` | 注入当前运行，取消pending工具调用 | 需要即时打断 |
| `followup` | 排队等待下一轮 | 不紧急的消息 |
| `steer-backlog` | 注入当前 + 保留为 followup | 既要即时又要完整回复 |
| `interrupt` | 中断当前运行，执行最新消息 | 紧急情况 |

### 配置

```json5
{
  messages: {
    queue: {
      mode: "collect",
      debounceMs: 1000,    // followup前等待安静期
      cap: 20,             // 每会话最大排队数
      drop: "summarize"    // 溢出策略
    }
  }
}
```

### 溢出策略

当排队消息超过 `cap` 时：

| 策略 | 行为 |
|------|------|
| `summarize` | **默认** — 合成摘要注入，尽量保留意图 |
| `old` | 丢弃最早的消息 |
| `new` | 丢弃最新的消息 |

### 按渠道覆盖

```json5
{
  messages: {
    queue: {
      mode: "collect",
      byChannel: {
        discord: "steer"    // Discord 用 steer 模式
      }
    }
  }
}
```

## 收益分析

### ✅ 收益

1. **资源保护** — 减少 60-80% 冗余 LLM 调用
2. **成本控制** — 更少调用 = 更少花费
3. **状态一致性** — 防止并发写入冲突
4. **更好质量** — 完整上下文 → 更准确回复
5. **系统稳定** — 优雅降级，不丢失消息

### ⚠️ 代价

1. **延迟增加** — 防抖窗口（2秒）+ 排队等待
2. **意图模糊** — 独立任务可能被错误合并
3. **溢出风险** — 极端情况可能丢失信息
4. **行为不一致** — 文本防抖但媒体不防抖
5. **调试复杂** — 需要理解机制才能排查

## 配置示例

### 标准配置（推荐）

```json5
{
  messages: {
    inbound: {
      debounceMs: 2000
    },
    queue: {
      mode: "collect",
      debounceMs: 1000,
      cap: 20,
      drop: "summarize"
    }
  }
}
```

### 低延迟配置（快速响应）

```json5
{
  messages: {
    inbound: {
      debounceMs: 500
    },
    queue: {
      mode: "steer",
      cap: 10
    }
  }
}
```

### 高吞吐配置（大量消息）

```json5
{
  messages: {
    inbound: {
      debounceMs: 5000
    },
    queue: {
      mode: "collect",
      debounceMs: 2000,
      cap: 50,
      drop: "summarize"
    }
  }
}
```

## 参考

- 官方文档：`docs/concepts/queue.md`
- 官方文档：`docs/concepts/messages.md`
- 中文文档：`docs/zh-CN/concepts/queue.md`
