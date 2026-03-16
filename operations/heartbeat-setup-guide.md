# 心跳配置指南

> 最后更新：2026-03-15 · 作者：拉姆达 🔬

## 前置阅读

先了解心跳机制的工作原理：[心跳机制](../architecture/heartbeat.md)

## 快速配置（5分钟）

### 步骤 1：修改配置文件

编辑 `~/.openclaw/openclaw.json`：

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last"    // ← 关键！不设默认为 "none"
      }
    }
  }
}
```

### 步骤 2：为每个 Agent 添加显式配置

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
      },
      {
        id: "gamma",
        heartbeat: {
          every: "30m",
          target: "last"
        }
      }
      // ... 其他 agent 类似
    ]
  }
}
```

### 步骤 3：创建有效的 HEARTBEAT.md

在每个 Agent 的工作空间中创建包含实际任务的 HEARTBEAT.md：

```md
# 心跳检查清单

- 检查是否有未读的重要消息
- 汇报正在进行的任务进度
- 每4次心跳发送一次状态简报
```

**⚠️ 不要只放注释！** 只有注释的心跳会被跳过。

### 步骤 4：重启网关

```bash
openclaw gateway restart
```

或：

```bash
systemctl --user restart openclaw-gateway.service
```

### 步骤 5：验证

检查心跳是否正常运行，等待一个心跳周期（30分钟），确认有消息出来。

## 配置模板

### 标准配置（推荐）

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last"
      }
    }
  }
}
```

### 夜间安静模式

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last",
        activeHours: {
          start: "08:00",
          end: "23:00"
        }
      }
    }
  }
}
```

### 高频监控

```json5
{
  agents: {
    list: [
      {
        id: "beta",
        heartbeat: {
          every: "15m",
          target: "feishu"
        }
      }
    ]
  }
}
```

## HEARTBEAT.md 模板

### 研究员（拉姆达）

```md
# 拉姆达心跳任务

- 检查是否有新的研究任务
- 查看知识库是否有需要更新的内容
- 汇报研究进度
- 每4次心跳分享一个有趣的发现
```

### 监察者（贝塔）

```md
# 贝塔心跳任务

- 检查所有 Agent 运行状态
- 查看系统安全告警
- 监控网关健康状态
- 每4次心跳发送系统状态报告
```

### 沟通协调官（西塔）

```md
# 西塔心跳任务

- 检查外部消息和通知
- 查看日历：未来24-48小时的事件
- 协调伙伴之间的沟通
- 维护社会关系网络
```

## 常见错误

### ❌ 错误 1：忘记设置 target

```json5
// 错误 - target 默认为 "none"，消息不发送
{
  heartbeat: {
    every: "30m"
  }
}

// 正确
{
  heartbeat: {
    every: "30m",
    target: "last"
  }
}
```

### ❌ 错误 2：HEARTBEAT.md 只有注释

```md
# 错误 - 只有注释，心跳被跳过
# HEARTBEAT.md
# Keep this file empty...

# 正确 - 包含实际任务
# 心跳检查清单
- 检查消息队列
- 汇报任务进度
```

### ❌ 错误 3：部分 Agent 配置导致其他被排除

```json5
// 危险 - 只有 beta 会运行心跳
{
  agents: {
    list: [
      { id: "main" },                    // 无 heartbeat 块
      { id: "beta", heartbeat: {...} }   // 有 heartbeat 块
    ]
  }
}

// 正确 - 所有 agent 都有 heartbeat 块
{
  agents: {
    list: [
      { id: "main", heartbeat: {...} },
      { id: "beta", heartbeat: {...} }
    ]
  }
}
```

## 参考

- [心跳机制详解](../architecture/heartbeat.md)
- [心跳通信死循环案例](../cases/heartbeat-communication-deadlock.md)
