# 可靠消息传递机制 — 架构设计方案

**创建日期：** 2026-03-18
**作者：** 拉姆达 🔬
**标签：** #通信 #架构 #核心 #设计
**来源：** 阿尔法研究任务（2026-03-18 01:00，梦想家直接指令）

---

## 一、问题分析

### 1.1 现状回顾

当前通信机制存在三个关键环节，其中两个已知有缺陷：

```
发送方 (阿尔法)
  ↓ 方式1: sessions_send ✅ 可靠（当目标 session 正常时）
  ↓ 方式2: 写工作空间文件 ❌ 对方不知道文件存在
  ↓
目标 Agent
  ↓ 飞书 session 活跃 → 秒到 ✅
  ↓ 飞书 session 卡死 → 消息入队但无法处理 ❌
  ↓ 心跳检查 → 大部分 Agent 的 HEARTBEAT.md 为空 → 心跳被跳过 ❌
  ↓
最终结果：消息丢失（对方永远不知道存在）
```

### 1.2 根因分析

**根因 1：飞书 session 不稳定**

- 流式错误（`"message_delta before message_start"`、`"request ended without sending any chunks"`）导致 session 卡死
- 卡死后，`sessions_send` 的消息入队但无法处理
- 受影响 Agent：阿尔法（3次）、贝塔（1次）、泽塔（1次）、伽马（从 3/16 起卡死）

**根因 2：工作空间文件不等于通知**

- 写文件到 `/home/gang/.openclaw/workspace-xxx/` 只是磁盘操作
- Agent 没有内置的文件监听机制
- Agent 只在以下情况才会看到文件：
  - 被飞书 ping 激活后读取工作空间
  - 心跳轮询检查（如果 HEARTBEAT.md 配置了检查任务）
  - 新 session 启动时的 bootstrap 读取

**根因 3：心跳机制未正确配置**

- 所有 Agent 的 HEARTBEAT.md 只有注释（空内容）
- OpenClaw 心跳设计：如果 HEARTBEAT.md 为空（只有注释/标题），**跳过心跳运行以节省 API 调用**
- 结论：大部分 Agent 的心跳形同虚设

**根因 4：无降级策略**

- 飞书 session 卡死后，没有自动切换到其他通信通道
- 没有 session 健康检测机制
- 没有消息投递确认机制

### 1.3 需求定义

| 需求 | 指标 | 说明 |
|------|------|------|
| 即时性 | ≤15 分钟 | 消息发出后对方能在 15 分钟内看到 |
| 可靠性 | 100% | 消息不会丢失（即使飞书 session 卡死） |
| 可达性 | 被动接收 | 对方不需要主动检查，系统自动推送或轮询 |
| 降级 | 自动 | 飞书不通时自动降级到其他通道 |

---

## 二、架构方案对比

### 方案 A：心跳文件轮询（Heartbeat File Polling）

**原理：** 在所有 Agent 的 HEARTBEAT.md 中添加"检查 inbox 目录"任务，每次心跳时扫描新文件。

```
发送方 → 写文件到 target-agent/inbox/ → 目标 Agent 心跳轮询 → 读取并处理
```

| 维度 | 评估 |
|------|------|
| 即时性 | ⚠️ 差 — 心跳间隔 30 分钟，最坏等待 30 分钟 |
| 可靠性 | ✅ 好 — 文件持久化在磁盘，不会丢失 |
| 可达性 | ✅ 好 — 心跳自动触发，无需主动检查 |
| 降级 | ⚠️ 部分 — 依赖心跳正常运行 |
| 实施成本 | 🟢 低 — 只需修改 HEARTBEAT.md |

**优点：** 利用现有心跳机制，改动最小
**缺点：** 30 分钟间隔太长，且心跳可能被跳过

---

### 方案 B：高频 Cron 轮询（Cron-Based Polling）

**原理：** 使用 OpenClaw cron 机制，每 5 分钟运行一次"检查 inbox"任务。

```
发送方 → 写文件到 inbox/ → Cron 任务每 5 分钟触发 → 检查并通知
```

| 维度 | 评估 |
|------|------|
| 即时性 | ⚠️ 中等 — 最坏等待 5 分钟 |
| 可靠性 | ✅ 好 — 文件持久化 |
| 可达性 | ✅ 好 — Cron 自动触发 |
| 降级 | ⚠️ 部分 — 依赖 cron 正常运行 |
| 实施成本 | 🟢 低 — 配置 cron 任务 |

**优点：** 比心跳更快，独立于 Agent session
**缺点：** 仍不是真正的即时，需要额外的 cron 配置

---

### 方案 C：事件驱动文件监听（Event-Driven / inotify）

**原理：** 使用 Linux inotify 机制，实时监听工作空间文件变化，发现新文件时立即触发 Agent 处理。

```
发送方 → 写文件到 inbox/ → inotify 检测到新文件 → Gateway 事件 → Agent 立即处理
```

| 维度 | 评估 |
|------|------|
| 即时性 | ✅ 优秀 — 毫秒级响应 |
| 可靠性 | ✅ 好 — 内核级事件通知 |
| 可达性 | ✅ 好 — 自动触发 |
| 降级 | ⚠️ 需要额外开发 |
| 实施成本 | 🔴 高 — 需要开发 Gateway 插件或外部守护进程 |

**优点：** 真正的即时通讯
**缺点：** 需要开发工作，与 OpenClaw 架构集成复杂

---

### 方案 D：多通道冗余（Multi-Channel Redundancy）

**原理：** 发送消息时同时使用多个通道（飞书 + workspace 文件 + 心跳），优先使用最快的可用通道。

```
发送方
  → 通道1: sessions_send (飞书) ← 优先
  → 通道2: 写工作空间文件 ← 备选
  → 通道3: 心跳轮询 ← 兜底
```

| 维度 | 评估 |
|------|------|
| 即时性 | ✅ 好 — 飞书正常时秒到 |
| 可靠性 | ✅ 好 — 多通道冗余 |
| 可达性 | ✅ 好 — 至少一个通道可达 |
| 降级 | ✅ 优秀 — 内置多级降级 |
| 实施成本 | 🟡 中 — 需要改造发送方逻辑 |

**优点：** 最接近"零丢失"，天然降级
**缺点：** 发送方需要处理多个通道，可能产生重复消息

---

### 方案 E：Session 健康检测 + 自动恢复

**原理：** 定期检测各 Agent 飞书 session 的健康状态，发现卡死时自动触发恢复（重启 session 或 gateway）。

```
健康检测 → 发现 session 卡死 → 自动恢复 → 消息重发
```

| 维度 | 评估 |
|------|------|
| 即时性 | ✅ 好 — 恢复后消息可处理 |
| 可靠性 | ⚠️ 中等 — 恢复不一定成功 |
| 可达性 | ✅ 好 — 自动检测和恢复 |
| 降级 | ✅ 好 — 自动恢复机制 |
| 实施成本 | 🟡 中 — 需要健康检测和恢复逻辑 |

**优点：** 从根本上解决 session 卡死问题
**缺点：** 恢复有副作用（中断当前对话），不能完全避免

---

## 三、推荐方案：混合架构（方案 A + D + E）

### 3.1 设计理念

```
                    ┌─────────────────────────────────────┐
                    │       可靠消息传递架构（混合）        │
                    └─────────────────────────────────────┘

发送方                                              接收方
──────                                              ──────
                                                    ┌─────────────┐
                         ┌── 通道1: sessions_send ──│ 飞书 session │
                         │   (即时，首选)            │  正常 → 秒到  │
                         │                          │  卡死 → 入队  │
┌──────┐    发送消息     │                          └─────────────┘
│ 阿尔法│───────────────►│
│ 发送方│                │                          ┌─────────────┐
└──────┘                ├── 通道2: workspace file ──│  工作空间    │
                         │   (持久化，备选)          │  inbox 目录  │
                         │                          └──────┬──────┘
                         │                                 │
                         │                          ┌──────▼──────┐
                         └── 通道3: 心跳轮询 ────────│  HEARTBEAT  │
                             (兜底，5-30分钟)        │  检查 inbox  │
                                                    └─────────────┘

                    ┌─────────────────────────────────────┐
                    │  Session 健康检测（独立守护进程）      │
                    │  - 检测 session 卡死                 │
                    │  - 自动恢复（重启 session）           │
                    │  - 通知发送方和接收方                 │
                    └─────────────────────────────────────┘
```

### 3.2 三层消息传递

#### 第一层：即时通道（sessions_send）

- **方式：** `sessions_send` 直接推送
- **延迟：** <1 秒（当 session 正常时）
- **可靠性：** 依赖目标 session 健康状态
- **问题：** session 卡死时消息入队但无法处理

#### 第二层：持久化通道（workspace inbox 文件）

- **方式：** 写入结构化消息文件到目标 Agent 的 inbox 目录
- **延迟：** 取决于轮询间隔
- **可靠性：** 文件持久化在磁盘，不依赖 session
- **问题：** 对方需要轮询才能发现

#### 第三层：心跳兜底（HEARTBEAT.md 检查）

- **方式：** Agent 心跳时检查 inbox 目录
- **延迟：** 心跳间隔（30 分钟）
- **可靠性：** 只要心跳运行就可靠
- **问题：** HEARTBEAT.md 为空时心跳被跳过

### 3.3 Inbox 消息结构

```
~/.openclaw/workspace-<agent>/
  inbox/
    .last-read              # 记录最后读取的消息 ID
    2026-03-18_0011_phase4-pause.json
    2026-03-18_2351_kb-policy.json
    ...
```

**消息文件格式：**

```json
{
  "id": "msg-20260318-0011-001",
  "timestamp": "2026-03-18T00:11:00+08:00",
  "from": "alpha",
  "priority": "high",
  "type": "notification",
  "subject": "Phase 4 任务暂停 + 知识库迁移",
  "body": "所有 Phase 4 未完成任务即日起暂停...",
  "source_file": "/home/gang/.openclaw/workspace-lambda/PHASE4-PAUSE-NOTIFY.md",
  "read": false,
  "actions": ["acknowledge"]
}
```

### 3.4 HEARTBEAT.md 标准配置

所有 Agent 的 HEARTBEAT.md 应包含以下最小检查任务：

```markdown
# HEARTBEAT.md

## 检查清单
- 检查 inbox/ 目录是否有新消息文件（以 .last-read 为基准）
- 如有新消息，读取并处理，更新 .last-read
- 检查工作空间根目录是否有 PHASE4-*.md、KB-*.md、URGENT-*.md 通知文件
- 如有未读通知，读取并回复确认
```

**注意：** 这不是空文件，心跳会实际运行并执行检查。

### 3.5 Session 健康检测

**检测逻辑：**

```bash
# 伪代码：检测 session 健康状态
for each agent:
  sessions = sessions_list(agent=agent.id)
  for each session:
    if session.updatedAt > now - 1h:  # 最近 1 小时有活动
      if session has streaming errors in logs:
        mark session as "stuck"
        trigger recovery (restart session)
```

**恢复策略：**

1. **轻度恢复：** 发送系统事件触发心跳
2. **中度恢复：** 重启目标 Agent 的飞书 session
3. **重度恢复：** 重启 Gateway（需要梦想家批准）

---

## 四、实施步骤

### Phase 1：快速修复（立即可做，1 天内）

**目标：** 让现有心跳机制开始工作

| 步骤 | 操作 | 负责人 |
|------|------|--------|
| 1 | 更新所有 Agent 的 HEARTBEAT.md，添加 inbox 检查任务 | 贝塔（配置） |
| 2 | 创建 inbox 目录结构（所有 Agent） | 贝塔 |
| 3 | 定义消息文件格式和读取协议 | 拉姆达 |
| 4 | 编写消息发送脚本（写文件 + sessions_send） | 伽马 |

**预期效果：** 心跳每 30 分钟检查一次 inbox，最坏 30 分钟内收到消息。

### Phase 2：增强通道（1 周内）

**目标：** 增加 session 健康检测和自动恢复

| 步骤 | 操作 | 负责人 |
|------|------|--------|
| 1 | 编写 session 健康检测脚本 | 贝塔 |
| 2 | 配置 cron 任务定期运行健康检测 | 贝塔 |
| 3 | 实现自动恢复逻辑（重启卡死 session） | 贝塔 |
| 4 | 添加恢复通知机制 | 西塔 |

**预期效果：** session 卡死后 5 分钟内自动恢复。

### Phase 3：高级特性（2-4 周）

**目标：** 实现真正的即时通讯

| 步骤 | 操作 | 负责人 |
|------|------|--------|
| 1 | 探索 OpenClaw webhook 机制 | 拉姆达 |
| 2 | 设计事件驱动通知（写文件 → webhook → 触发） | 拉姆达 |
| 3 | 实现消息投递确认机制 | 伽马 |
| 4 | 性能测试和优化 | 德尔塔 |

**预期效果：** 消息投递延迟 <1 分钟。

---

## 五、技术细节

### 5.1 消息发送脚本（推荐实现）

```bash
#!/bin/bash
# send-message.sh — 可靠消息发送脚本
# 用法: ./send-message.sh <target-agent> <priority> <subject> <body>

TARGET_AGENT=$1
PRIORITY=$2
SUBJECT=$3
BODY=$4

WORKSPACE="/home/gang/.openclaw/workspace-${TARGET_AGENT}"
INBOX="${WORKSPACE}/inbox"
MSG_ID="msg-$(date +%Y%m%d-%H%M%S)-$$"
TIMESTAMP=$(date -Iseconds)

# 确保 inbox 目录存在
mkdir -p "$INBOX"

# 写入消息文件
cat > "${INBOX}/${MSG_ID}.json" << EOF
{
  "id": "${MSG_ID}",
  "timestamp": "${TIMESTAMP}",
  "from": "$(whoami)",
  "priority": "${PRIORITY}",
  "type": "notification",
  "subject": "${SUBJECT}",
  "body": "${BODY}",
  "read": false
}
EOF

# 同时尝试 sessions_send（如果 session 活跃）
echo "消息已写入: ${INBOX}/${MSG_ID}.json"
echo "等待下次心跳轮询或 session 恢复后处理"
```

### 5.2 心跳检查脚本（HEARTBEAT.md 内容）

```markdown
# HEARTBEAT.md

## 收件箱检查
检查 `inbox/` 目录中 `.last-read` 之后的新消息文件：
1. 读取 `.last-read` 获取最后处理的消息 ID
2. 列出 `inbox/` 中所有 `.json` 文件
3. 处理 ID 大于 last-read 的消息
4. 更新 `.last-read`
5. 如有高优先级消息，立即回复确认

## 通知文件检查
检查工作空间根目录的紧急通知文件：
- `PHASE4-*.md` — Phase 4 相关通知
- `KB-*.md` — 知识库相关通知
- `URGENT-*.md` — 紧急通知
如发现未读通知，读取并处理。
```

### 5.3 Session 健康检测配置

```json5
// openclaw.json 中添加 cron 任务
{
  agents: {
    list: [
      {
        id: "beta",
        cron: {
          jobs: [
            {
              name: "Session Health Check",
              schedule: "*/5 * * * *",  // 每 5 分钟
              prompt: "检查所有 Agent 的飞书 session 健康状态。读取 gateway 日志中的 'session.stuck' 计数器。如发现卡死 session，记录到 shared-knowledge/operations/session-health.md。如连续 3 次检测到同一 session 卡死，通知梦想家。",
              delivery: { mode: "webhook" }
            }
          ]
        }
      }
    ]
  }
}
```

### 5.4 读取协议（Agent 端实现）

Agent 在心跳或新 session 启动时执行：

```
1. 检查 inbox/ 目录是否存在
2. 如果不存在，创建目录并返回
3. 读取 .last-read 文件（如不存在，视为 "0000-00-00_0000"）
4. 列出所有 .json 文件，按文件名排序
5. 对每个文件名 > .last-read 的消息：
   a. 读取消息内容
   b. 根据 priority 处理（high=立即，normal=下次汇报）
   c. 回复确认（如需要）
6. 更新 .last-read 为最新处理的消息文件名
```

---

## 六、与其他系统的集成

### 6.1 与防抖机制的关系

- 消息文件使用唯一 ID（时间戳 + 序号），天然防重
- `.last-read` 机制确保每条消息只处理一次
- 与 OpenClaw 的 `messageDebounceMs` 互补，不冲突

### 6.2 与心跳机制的关系

- HEARTBEAT.md 添加 inbox 检查任务
- 心跳间隔 30 分钟 → 最坏 30 分钟延迟
- 建议：关键 Agent 可缩短心跳间隔到 15 分钟

### 6.3 与队列机制的关系

- workspace 文件是持久化层，与运行时队列互补
- 队列处理 session 活跃时的消息
- 文件处理 session 卡死时的消息
- 两者互不干扰

---

## 七、风险评估

| 风险 | 等级 | 缓解措施 |
|------|------|----------|
| 心跳被跳过（HEARTBEAT.md 判定为空） | 🟡 中 | 确保 HEARTBEAT.md 有实际内容（非纯注释） |
| 消息文件堆积（inbox 目录膨胀） | 🟢 低 | 定期清理已处理消息（保留 7 天） |
| 重复处理（文件 + 飞书同时送达） | 🟢 低 | 消息 ID 去重 |
| inotify 资源消耗 | 🟢 低 | Phase 3 才实施，先评估 |
| 自动恢复误判（恢复正在运行的 session） | 🟡 中 | 设置连续 3 次检测确认后再恢复 |

---

## 八、衡量指标

| 指标 | 当前值 | 目标值 | 衡量方式 |
|------|--------|--------|----------|
| 消息投递成功率 | ~70%（估计） | ≥99% | 日志统计 |
| 平均投递延迟 | 不确定（可能数小时） | ≤15 分钟 | 时间戳对比 |
| Session 卡死检测时间 | 手动发现（数小时） | ≤5 分钟 | 自动检测 |
| Session 恢复时间 | 手动（梦想家操作） | ≤1 分钟（自动） | 恢复日志 |

---

## 九、总结

### 核心结论

1. **写工作空间文件 ≠ 通知** — 必须有轮询或监听机制
2. **心跳是关键基础设施** — 但需要正确配置 HEARTBEAT.md
3. **多通道冗余是必要的** — 单一通道（飞书）不可靠
4. **Session 健康检测应自动化** — 不能依赖手动发现

### 推荐行动

**立即执行（Phase 1）：**
- 所有 Agent 的 HEARTBEAT.md 添加 inbox 检查任务
- 创建 inbox 目录和消息格式标准
- 编写可靠消息发送脚本

**短期（Phase 2）：**
- 部署 session 健康检测 cron
- 实现自动恢复逻辑

**中期（Phase 3）：**
- 探索事件驱动方案（webhook/inotify）
- 实现投递确认机制

---

## 附录：参考资料

- OpenClaw 心跳文档：`docs/gateway/heartbeat.md`
- OpenClaw 队列文档：`docs/concepts/queue.md`
- OpenClaw 消息文档：`docs/concepts/messages.md`
- OpenClaw Cron 文档：`docs/automation/cron-jobs.md`
- 相关案例：`cases/message-delta-streaming-error.md`（流式错误根因分析）
- 相关案例：`cases/heartbeat-communication-deadlock.md`（心跳通信死锁）

---

*本报告由拉姆达研究，阿尔法指派，梦想家批准。如有改进建议，欢迎反馈。*
