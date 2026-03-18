# Clawith（团队版 OpenClaw）研究分析与乐园落地可行性

> **作者：** 拉姆达 🔬
> **日期：** 2026-03-18
> **来源：** 梦想家分享的今日头条文章 → GitHub: dataelement/Clawith
> **标签：** #研究 #架构 #对比 #可行性分析 #多Agent协作

---

## 一、项目概述

**Clawith** — "Claw with Claw, Claw with You"，一个开源的多 Agent 协作平台。

- **定位：** 企业级数字员工平台，将 OpenClaw 的个人能力扩展到组织级
- **技术栈：** Python 3.12+ / FastAPI / React 19 / PostgreSQL / Redis / Docker
- **许可证：** Apache 2.0
- **仓库：** github.com/dataelement/Clawith

**核心差异：** OpenClaw 是个人助手（单 Gateway 进程），Clawith 是多 Agent 协作平台（Web 应用 + 数据库）。

---

## 二、四项值得借鉴的创新

### 2.1 Aware 持续感知系统 ⭐⭐⭐⭐⭐

**问题：** OpenClaw 的 Heartbeat 机制是固定间隔（30分钟）的被动检查，HEARTBEAT.md 为空时直接跳过，导致大部分 Agent 的心跳形同虚设。

**Clawith 的方案：**

| 组件 | 功能 |
|------|------|
| **Focus Items** | Agent 维护结构化工作记忆（pending → in_progress → completed） |
| **Focus-Trigger Binding** | 每个任务必须有 Focus item，通过 `focus_ref` 关联触发器 |
| **Self-Adaptive Triggering** | Agent 自己动态创建/调整/移除触发器 |
| **Reflections** | 独立视图展示 Agent 触发时的自主推理过程 |

**6种触发器（比 OpenClaw 多1个）：**
- cron / once / interval / poll / webhook（OpenClaw 已有类似）
- **on_message（新增）** — 等某个 Agent 或人类回复时触发

**核心哲学：** 人类设定目标，Agent 自己管理执行计划。

---

### 2.2 Relationship 关系图谱 ⭐⭐⭐⭐

**问题：** OpenClaw 的 Agent 之间没有正式的关系定义。我们（小龙虾世界）通过 AGENTS.md 手动记录角色，但没有结构化的关系数据。

**Clawith 的方案：**
- 每个 Agent 进来先"认人"
- 知道老板是谁、同事是谁、谁的性格随和、谁老改需求
- 关系类型：Supervisor（上级）、Colleague（同事）、Subordinate（下级）
- 通过 `send_message_to_agent` 工具进行异步通信

**数据模型：**
```python
# Agent 关系表（推测）
class AgentRelationship:
    source_agent_id: UUID
    target_agent_id: UUID
    relation_type: Enum  # supervisor, colleague, subordinate
    metadata: JSON  # 性格笔记、合作历史等
```

---

### 2.3 on_message 触发器 ⭐⭐⭐⭐⭐

**问题：** 我们目前的跨 Agent 通信最大问题是"不知道对方什么时候回复"。只能靠心跳轮询或手动检查，导致消息延迟可能长达 30 分钟。

**Clawith 的方案：**
```python
# Agent 设置一个 on_message 触发器
set_trigger(
    type="on_message",
    target_agent="alpha",  # 监听谁的消息
    focus_ref="task-001",  # 关联的任务
    action="process_reply" # 触发后执行什么
)
# 当 alpha 回复时，自动唤醒当前 Agent 处理
```

**效果：** 不再需要轮询，对方回复时立即触发处理。

---

### 2.4 Plaza 广场 ⭐⭐⭐

**问题：** Agent 之间的信息共享目前靠"通知文件"（PHASE4-*.md, KB-*.md），需要对方主动检查或心跳轮询才能发现。

**Clawith 的方案：**
- 类似公司内部社交信息流
- Agent 可以发帖、订阅、评论
- 每个 Agent 通过信息流吸收组织知识，保持上下文感知
- 比文件通知更自然、更及时

---

## 三、与我们现有方案的对比

### 3.1 我们当前的通信问题（拉姆达分析报告，2026-03-18）

| 根因 | 现状 |
|------|------|
| 飞书 session 不稳定 | 流式错误导致 session 卡死，消息入队但无法处理 |
| 工作空间文件 ≠ 通知 | 写文件是磁盘操作，Agent 没有内置文件监听 |
| 心跳未配置 | 所有 Agent 的 HEARTBEAT.md 只有注释，心跳被跳过 |
| 无降级策略 | 飞书不通时没有自动切换 |

### 3.2 我们的可靠消息传递方案（拉姆达，Phase 1-3）

**Phase 1（快速修复）：** 心跳文件轮询
- 所有 Agent 的 HEARTBEAT.md 添加 inbox 检查任务
- 创建 inbox 目录和消息格式标准
- 最坏 30 分钟延迟

**Phase 2（增强）：** Session 健康检测 + 自动恢复
- Cron 定期检测 session 卡死
- 自动恢复逻辑

**Phase 3（高级）：** 事件驱动
- 探索 webhook/inotify 机制
- 投递确认机制

---

## 四、落地可行性分析

### 4.1 Aware 持续感知 → 乐园落地

**可行性：⭐⭐⭐⭐ 高（可部分实施）**

**限制：** 我们运行在 OpenClaw 上，不能直接移植 Clawith 的代码。但可以**借鉴设计思想**。

**可行的实施方案：**

#### 方案 A：增强版 HEARTBEAT.md（立即可做）

```markdown
# HEARTBEAT.md

## Focus Items（持续感知）
### 活跃任务
- [ ] task-001: 研究 Clawith 落地可行性 | 等待梦想家反馈 | 自:lambda:发:2026-03-18
- [/] task-002: 更新知识库 | 进行中 | 截止:2026-03-18 12:00
- [x] task-003: 分析搜索能力 | 已完成 | 伽马已评估

### 触发器
- inbox/ 检查（每心跳）
- 如有高优先级消息，立即处理
- 如 task-001 收到梦想家回复，更新状态为 in_progress

### 待处理
- 检查 inbox/ 目录新消息
- 如有 PHASE4-*.md / KB-*.md / URGENT-*.md 未读通知，读取处理
```

**效果：**
- 把 HEARTBEAT.md 从"空文件"变成"活跃任务看板"
- 心跳不再是无意义的空转，而是真正检查任务状态
- 自适应：Agent 根据任务进展动态调整心跳检查内容

**实施成本：** 低，只需更新所有 Agent 的 HEARTBEAT.md
**立即可行：** ✅ 是

#### 方案 B：on_message 的替代实现（需要 Phase 2+）

我们无法直接实现 on_message 触发器（OpenClaw 没有这个机制），但可以用**组合方案**模拟：

```
1. Agent A 发消息给 Agent B（sessions_send + 写 inbox 文件）
2. Agent B 收到后，处理并回复（sessions_send 回 Agent A）
3. Agent A 的下次心跳检查 inbox，发现 Agent B 的回复
```

**差距：** 不是真正的"即时触发"，而是"下次心跳发现"。
**改善：** 如果心跳间隔缩短到 5-10 分钟，延迟可接受。

**结论：** 方案 A + B 的组合，实际上就是我们 Phase 1 方案的增强版。Clawith 的 Aware 思想验证了我们的方向是对的。

---

### 4.2 Relationship 关系图谱 → 乐园落地

**可行性：⭐⭐⭐⭐⭐ 极高（立即可做）**

**现状对比：**

| 维度 | Clawith | 我们目前 | 差距 |
|------|---------|---------|------|
| 关系定义 | 数据库结构化 | AGENTS.md 手动文本 | 格式化程度低 |
| 关系类型 | supervisor/colleague/subordinate | "阿尔法创建了XXX" | 无显式类型 |
| Agent 使用 | `send_message_to_agent` 引用关系 | 硬编码目标 sessionKey | 不够灵活 |

**实施方案：创建 `RELATIONSHIPS.md`**

```markdown
# 乐园关系图谱

## 组织架构
梦想家（人类）
├── 阿尔法 🦐 — 首席伙伴（直接汇报）
│   ├── 贝塔 🔵 — 监察者
│   ├── 德尔塔 📊 — 数据分析师
│   ├── 伽马 🔧 — 工匠
│   ├── 西塔 🎨 — 设计师
│   └── 拉姆达 🔬 — 研究员
│       └── 卡帕 🏛️ — 建筑师（架构相关协作）
└── 艾普西隆 Σ — 架构师（独立）

## 协作关系
- 拉姆达 ↔ 伽马：研究 ↔ 技能安装（频繁）
- 拉姆达 ↔ 贝塔：研究 ↔ 审核（定期）
- 阿尔法 ↔ 所有人：创建 ↔ 协调（持续）

## 消息路由偏好
- 技能相关 → 伽马
- 数据分析 → 德尔塔
- 设计相关 → 西塔
- 架构设计 → 卡帕 / 艾普西隆
```

**实施成本：** 极低，只需一个文件
**立即可行：** ✅ 是

**进阶（Phase 2）：** 在 HEARTBEAT.md 中加入关系感知
```markdown
## 关系感知
- 如收到伽马的消息，优先处理（技能相关）
- 如贝塔有审核请求，24小时内回复
```

---

### 4.3 on_message 触发器 → 乐园落地

**可行性：⭐⭐⭐ 中等（需要创新）**

**核心挑战：** OpenClaw 没有 on_message 机制，我们无法直接实现"等对方回复时自动触发"。

**但有三条路径：**

#### 路径 1：心跳轮询模拟（Phase 1，立即可做）

**原理：** 每次心跳检查 inbox 目录，发现新消息就处理。

```
Agent A 发消息 → 写入 Agent B 的 inbox/
Agent B 心跳（30分钟内）→ 检查 inbox/ → 发现新消息 → 处理
```

**延迟：** 最坏 30 分钟（心跳间隔）
**改善：** 缩短心跳到 5-10 分钟

#### 路径 2：Cron 高频检查（Phase 2，1周内）

**原理：** 用 OpenClaw cron 机制，每 5 分钟运行一次 inbox 检查。

```
Cron job: "*/5 * * * *" → 检查 inbox/ → 发现新消息 → 处理
```

**延迟：** 最坏 5 分钟
**优势：** 不依赖心跳，独立运行

#### 路径 3：Webhook 事件驱动（Phase 3，2-4周）

**原理：** 探索 OpenClaw 是否支持 webhook 或系统事件机制。

```
Agent A 发消息 → 写入 inbox/ → 触发 Gateway 事件 → Agent B 立即唤醒
```

**延迟：** 秒级
**风险：** 需要 OpenClaw 底层支持，可能无法实现

**结论：** 路径 1 + 2 的组合可以在短期内达到"5分钟内响应"的效果，虽然不如 Clawith 的即时 on_message，但对乐园够用。

---

### 4.4 Plaza 广场 → 乐园落地

**可行性：⭐⭐⭐ 中等（需要基础设施）**

**核心挑战：** Plaza 需要一个持久化的信息流服务。OpenClaw 没有这个组件。

**两条路径：**

#### 路径 1：文件系统 Plaza（轻量级，立即可做）

**原理：** 用共享目录模拟信息流。

```
~/.openclaw/workspace-shared/plaza/
├── posts/
│   ├── 2026-03-18_lambda_clawith-research.md
│   ├── 2026-03-18_gamma_skill-evaluation.md
│   └── ...
└── .last-read-alpha    # 每个 Agent 的已读标记
```

**Agent 心跳时检查：**
```markdown
## Plaza 检查
- 检查 workspace-shared/plaza/posts/ 有无新帖子
- 如有，读取并根据相关性处理
- 可选：回复或转发相关帖子
```

**优点：** 利用现有文件系统，无需新基础设施
**缺点：** 仍然是轮询模式，有延迟

#### 路径 2：飞书群 Plaza（利用现有渠道）

**原理：** 创建一个飞书群，所有 Agent 通过飞书 bot 发帖。

**优点：** 即时推送，Agent 的飞书 session 活跃时秒到
**缺点：** 飞书 session 不稳定时失效；需要每个 Agent 都接入同一群

**结论：** 路径 1（文件系统 Plaza）可以立即实施，作为 Phase 1 的补充。

---

## 五、综合落地路线图

### Phase 1：快速修复（1天）— ✅ 与原方案一致

| 任务 | 对应 Clawith 特性 | 优先级 |
|------|-------------------|--------|
| 更新所有 Agent 的 HEARTBEAT.md（Focus Items） | Aware（部分） | 🔴 高 |
| 创建 inbox 目录和消息格式 | 基础通信 | 🔴 高 |
| 创建 RELATIONSHIPS.md | Relationship | 🟡 中 |
| 创建 Plaza 共享目录 | Plaza（简化版） | 🟢 低 |

### Phase 2：增强通道（1周）— 🔄 扩展原方案

| 任务 | 对应 Clawith 特性 | 优先级 |
|------|-------------------|--------|
| Cron 高频 inbox 检查（5分钟） | on_message（模拟） | 🔴 高 |
| Session 健康检测 + 自动恢复 | 基础设施 | 🔴 高 |
| 心跳缩短到 10-15 分钟 | Aware（增强） | 🟡 中 |
| Plaza 自动发帖机制 | Plaza（增强） | 🟢 低 |

### Phase 3：高级特性（2-4周）— 🔮 探索性

| 任务 | 对应 Clawith 特性 | 优先级 |
|------|-------------------|--------|
| 探索 OpenClaw webhook 机制 | on_message（真正实现） | 🟡 中 |
| 事件驱动通知（写文件 → 触发） | Aware（完整） | 🟡 中 |
| 消息投递确认机制 | 可靠通信 | 🟡 中 |
| Agent 自适应触发器 | Aware（Self-Adaptive） | 🟢 低 |

---

## 六、关键结论

### 6.1 我们的方向是对的 ✅

Clawith 的 Aware 系统验证了我们 Phase 1-3 方案的核心思路：
- **持久化消息文件**（inbox/）← 我们和 Clawith 都用这个模式
- **心跳/触发器轮询** ← 我们的心跳检查 = Clawith 的简化版 Aware
- **Session 健康检测** ← 两者都需要

### 6.2 差距在哪？

| 我们缺的 | Clawith 有 | 影响 |
|----------|-----------|------|
| on_message 触发器 | ✅ 原生支持 | 我们最多 5 分钟延迟 vs 秒级 |
| 结构化关系图谱 | ✅ 数据库模型 | 我们用文本文件，但够用 |
| 自适应触发器 | ✅ Agent 自己管理 | 我们靠手动配置 HEARTBEAT.md |
| Plaza 信息流 | ✅ 原生组件 | 我们用文件模拟，功能受限 |

### 6.3 我们的优势

| 我们有 | Clawith 没有 |
|--------|-------------|
| 轻量级（单进程，无需数据库） | 需要 PostgreSQL + Redis + Docker |
| 飞书原生集成（20+ 渠道） | 需要单独配置每个渠道 |
| OpenClaw 生态（ClawHub 技能） | 自己的 MCP 注册中心 |
| 终端优先，快速启动 | Web 应用，启动较重 |

### 6.4 最终建议

**不要尝试变成 Clawith，而是吸收它的精华。**

1. **Aware 思想 → 立即实施** — 把 HEARTBEAT.md 变成 Focus Items 看板，这是零成本高回报的改进
2. **Relationship → 立即实施** — 创建 RELATIONSHIPS.md，让每个 Agent 知道谁负责什么
3. **on_message → Phase 2 实施** — 用 Cron 5 分钟轮询模拟，接近效果
4. **Plaza → 可选实施** — 文件系统 Plaza 作为信息共享的补充渠道

**核心原则：** 用最小的改动获得最大的协作效率提升。

---

## 七、参考链接

- GitHub 仓库：https://github.com/dataelement/Clawith
- 架构文档：ARCHITECTURE_SPEC.md（仓库根目录）
- 拉姆达通信方案：《可靠消息传递机制 — 架构设计方案》（2026-03-18）
- 伽马搜索评估：Agent Browser 已安装，无需额外技能

---

*本研究报告由拉姆达 🔬 基于 GitHub 源码分析和架构文档研究完成。*
*如有疑问或建议，请联系拉姆达。*
