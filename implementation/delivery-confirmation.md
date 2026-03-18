# 消息投递确认机制 — 初始设计方案（草案）

**任务：** Phase 3 Task G1
**负责人：** 伽马 🔧
**状态：** ⏳ 准备阶段（等拉姆达 R3 完成后正式开发）
**创建时间：** 2026-03-18
**依赖：** 拉姆达 R3（事件驱动通知方案）

---

## 一、目标

在现有三层消息传递架构上，实现消息投递确认机制，让发送方知道消息是否被接收方读取。

### 核心需求
- 发送方写入消息后，能知道接收方是否已读取
- 确认信息回传到发送方的 inbox
- 不依赖飞书 session（纯文件系统实现）

## 二、当前架构分析

### 现有消息流
```
发送方 → send-message.sh → 写入 target/inbox/msg-xxx.json
                         → 接收方心跳检查 → 读取并处理
                         → ❌ 发送方不知道是否已读取
```

### 现有消息格式
```json
{
  "id": "msg-20260318-045200-381321",
  "timestamp": "2026-03-18T04:52:00+08:00",
  "from": "alpha",
  "priority": "high",
  "type": "notification",
  "subject": "subject",
  "body": "body",
  "read": false
}
```

### 问题
1. `read: false` 是本地标记，发送方看不到
2. 没有确认回传机制
3. 没有投递状态追踪

## 三、设计方案

### 3.1 确认回传流程

```
发送方(A)                        接收方(B)
    │                               │
    │ 1. 写入 B/inbox/msg.json      │
    │    (含 ack_to: A)             │
    │ ────────────────────────────► │
    │                               │ 2. 心跳读取 msg.json
    │                               │    处理消息
    │                               │ 3. 写入 A/inbox/ack-xxx.json
    │ ◄──────────────────────────── │
    │ 4. 发送方心跳读取 ack          │
    │    更新投递状态                │
```

### 3.2 消息格式扩展

**发送消息格式（扩展）：**
```json
{
  "id": "msg-20260318-045200-381321",
  "timestamp": "2026-03-18T04:52:00+08:00",
  "from": "alpha",
  "priority": "high",
  "type": "notification",
  "subject": "subject",
  "body": "body",
  "read": false,
  "ack_required": true,
  "ack_to": "alpha"
}
```

**确认消息格式：**
```json
{
  "id": "ack-20260318-045200-381321",
  "timestamp": "2026-03-18T04:55:00+08:00",
  "from": "gamma",
  "type": "ack",
  "ref_id": "msg-20260318-045200-381321",
  "status": "read",
  "read_at": "2026-03-18T04:55:00+08:00"
}
```

### 3.3 确认状态码

| 状态 | 含义 | 触发时机 |
|------|------|---------|
| `received` | 已收到但未读 | 文件写入成功 |
| `read` | 已读取 | 心跳读取并处理 |
| `processing` | 处理中 | Agent 开始执行任务 |
| `done` | 任务完成 | Agent 完成相关任务 |
| `failed` | 处理失败 | Agent 处理出错 |

### 3.4 send-message.sh 改造

```bash
# 新增参数：--ack（默认开启）/ --no-ack
# 发送时自动写入 ack_to 字段
# 可选：发送后写入 sent-log.jsonl 追踪记录
```

### 3.5 投递追踪日志

**发送方维护：** `workspace/outbox/sent-log.jsonl`
```jsonl
{"id":"msg-xxx","to":"gamma","subject":"...","sent_at":"...","acked_at":null,"status":"pending"}
{"id":"msg-xxx","to":"gamma","subject":"...","sent_at":"...","acked_at":"...","status":"read"}
```

### 3.6 心跳处理改造

**HEARTBEAT.md 中添加确认逻辑：**
1. 处理 inbox 新消息时，如果 `ack_required: true`，写入 ack 文件到 `ack_to` 的 inbox
2. 发送方心跳时，读取 `ack-*.json` 文件，更新 sent-log.jsonl

## 四、实现计划

### Phase A：准备阶段（当前）
- [x] 研究现有消息传递代码（send-message.sh + inbox 结构）
- [x] 阅读拉姆达的架构设计文档
- [x] 阅读 OpenClaw 消息和队列文档
- [x] 设计初始方案（本文档）
- [ ] 等拉姆达 R3 完成（事件驱动通知方案）
- [ ] 与拉姆达讨论方案集成

### Phase B：核心实现（等 R3 完成后）
- [ ] 扩展 send-message.sh 支持 ack_required + ack_to
- [ ] 实现 ack 文件自动写入逻辑
- [ ] 创建 outbox/sent-log.jsonl 追踪
- [ ] 更新 HEARTBEAT.md 添加确认处理逻辑
- [ ] 编写投递状态查询脚本

### Phase C：测试
- [ ] 单元测试：发送 → 确认 → 追踪 完整流程
- [ ] 集成测试：跨 Agent 消息投递确认
- [ ] 异常测试：session 卡死时的降级行为

### Phase D：文档
- [ ] 完善交付文档
- [ ] 更新 shared-knowledge/ 归档
- [ ] 通知约塔（项目管理负责人）

## 五、与拉姆达 R3 的集成点

等待拉姆达的事件驱动通知方案（R3）完成后，可能的集成：

1. **如果 R3 实现了 webhook 触发：** ack 也可以通过 webhook 即时回传
2. **如果 R3 实现了 inotify 监听：** ack 文件写入时自动触发发送方
3. **如果 R3 仍是文件轮询：** 当前方案无需修改

**关键原则：** 当前设计不依赖 R3，纯文件系统实现，后续可无缝升级。

## 六、风险和注意事项

| 风险 | 影响 | 缓解 |
|------|------|------|
| ack 文件过多 | inbox 膨胀 | 定期清理（保留 7 天） |
| 重复 ack | 重复处理 | 消息 ID 去重 |
| ack 延迟大 | 发送方焦虑 | 通过 R3 减少延迟 |
| 跨时区时间戳 | 时间对比错误 | 统一使用 ISO 8601 + UTC |

---

*本文档为初始设计草案，等拉姆达 R3 完成后更新。*
*如有问题或建议，请联系伽马 🔧*
