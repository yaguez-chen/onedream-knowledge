# OpenClaw Control Center 深度研究报告

> **作者：** 拉姆达 🔬
> **日期：** 2026-03-18
> **来源：** 梦想家分享的今日头条文章 → GitHub: TianyiDataScience/openclaw-control-center
> **标签：** #研究 #架构 #可视化 #监控 #OpenClaw

---

## 一、项目概述

**OpenClaw Control Center** — 将 OpenClaw 从"黑盒"变成"可视化控制中心"。

- **口号：** Turn OpenClaw from a black box into a local control center you can see, trust, and control.
- **定位：** OpenClaw 的"控制面"（管理和监控），补全 OpenClaw 本身的"数据面"（执行 AI 任务）
- **技术栈：** TypeScript（1.5MB，占 99%+）、React UI、Node.js
- **许可证：** 见 LICENSE 文件
- **Stars：** 2,016（截至 2026-03-18，文章发表时 1,390）
- **创建时间：** 2026-03-11
- **仓库：** github.com/TianyiDataScience/openclaw-control-center

**核心设计哲学：**
- **只读优先** — `READONLY_MODE=true` 默认
- **本地 Token 认证** — `LOCAL_TOKEN_AUTH_REQUIRED=true` 默认
- **变更操作默认禁用** — `IMPORT_MUTATION_ENABLED=false`、`APPROVAL_ACTIONS_ENABLED=false`
- **不修改 OpenClaw 配置** — 只读取 `~/.openclaw/`，不动 `openclaw.json`

---

## 二、架构分析

### 2.1 整体架构

```
┌──────────────────────────────────────────────────────────┐
│ React UI (port 4310)                                     │
│ Overview | Usage | Staff | Collaboration | Tasks |       │
│ Documents | Memory | Settings                            │
├──────────────────────────────────────────────────────────┤
│ UI Server (src/ui/server.ts)                             │
│ HTTP API + Static Files                                  │
├──────────────────────────────────────────────────────────┤
│ Runtime Layer (src/runtime/)                             │
│ 38 个功能模块（每个一个 .ts 文件）                        │
├──────────────────────────────────────────────────────────┤
│ Adapter Layer (src/adapters/)                            │
│ OpenClawReadonlyAdapter                                  │
├──────────────────────────────────────────────────────────┤
│ Client Layer (src/clients/)                              │
│ OpenClawLiveClient → OpenClaw Gateway (WebSocket)        │
├──────────────────────────────────────────────────────────┤
│ OpenClaw Gateway (ws://127.0.0.1:18789)                  │
│ ~/.openclaw/ (文件系统读取)                               │
└──────────────────────────────────────────────────────────┘
```

### 2.2 数据流

```
OpenClaw Gateway (WebSocket)
    ↓
OpenClawLiveClient (src/clients/openclaw-live-client.ts)
    ↓
OpenClawReadonlyAdapter (src/adapters/openclaw-readonly.ts)
    ├── sessionsList() → mapSessionsListToSummaries()
    ├── sessionStatus() → parseSessionStatusText()
    ├── cronList() → mapCronListToSummaries()
    └── approvalsGet() → mapApprovalsGetToSummaries()
    ↓
Runtime Modules (src/runtime/)
    ├── monitor.ts — 主监控循环
    ├── snapshot-store.ts — 快照存储
    ├── task-heartbeat.ts — 任务心跳
    ├── budget-governance.ts — 预算治理
    ├── notification-center.ts — 通知中心
    └── ... (38 个模块)
    ↓
UI Server (src/ui/server.ts) → React UI (port 4310)
```

### 2.3 核心代码结构

```
src/
├── index.ts                    # 入口：启动监控 + UI 服务
├── config.ts                   # 配置：环境变量解析
├── types.ts                    # 类型定义
├── adapters/
│   └── openclaw-readonly.ts    # 只读适配器（核心数据获取）
├── clients/
│   ├── factory.ts              # 客户端工厂
│   ├── openclaw-live-client.ts # OpenClaw Gateway WebSocket 客户端
│   └── tool-client.ts          # 工具客户端接口
├── contracts/
│   └── openclaw-tools.ts       # OpenClaw 工具契约定义
├── mappers/
│   ├── openclaw-mappers.ts     # 数据映射（sessions/cron/approvals）
│   └── session-status-parser.ts# Session 状态文本解析器
├── runtime/                    # 38 个功能模块
│   ├── monitor.ts              # 主监控循环
│   ├── healthz.ts              # 健康检查
│   ├── agent-roster.ts         # Agent 花名册
│   ├── session-conversations.ts# 会话对话记录
│   ├── task-store.ts           # 任务存储
│   ├── task-summary.ts         # 任务摘要
│   ├── task-heartbeat.ts       # 任务心跳
│   ├── budget-governance.ts    # 预算治理
│   ├── budget-policy.ts        # 预算策略
│   ├── usage-cost.ts           # 使用成本
│   ├── cron-overview.ts        # Cron 任务概览
│   ├── approval-action-service.ts# 审批操作服务
│   ├── commander.ts            # 指挥官（摘要生成）
│   ├── commander-digest.ts     # 指令摘要
│   ├── digest-renderer.ts      # 摘要渲染器
│   ├── notification-center.ts  # 通知中心
│   ├── notification-policy.ts  # 通知策略
│   ├── monitor-health.ts       # 监控健康
│   ├── operation-audit.ts      # 操作审计
│   ├── audit-timeline.ts       # 审计时间线
│   ├── snapshot-store.ts       # 快照存储
│   ├── project-store.ts        # 项目存储
│   ├── project-summary.ts      # 项目摘要
│   ├── export-bundle.ts        # 导出包
│   ├── import-dry-run.ts       # 导入试运行
│   ├── import-live.ts          # 实时导入
│   ├── local-token-auth.ts     # 本地 Token 认证
│   ├── avatar-preferences.ts   # 头像偏好
│   ├── ui-preferences.ts       # UI 偏好
│   ├── pixel-state.ts          # 像素状态
│   ├── replay-index.ts         # 回放索引
│   ├── doc-hub.ts              # 文档中心
│   ├── done-checklist.ts       # 完成清单
│   ├── diff-summary.ts         # 差异摘要
│   ├── action-queue-links.ts   # 操作队列链接
│   ├── current-agent-catalog.ts# 当前 Agent 目录
│   ├── office-session-presence.ts# 办公会话在线状态
│   ├── openclaw-cli-insights.ts# OpenClaw CLI 洞察
│   └── api-docs.ts             # API 文档
└── ui/
    └── server.ts               # HTTP UI 服务器
```

---

## 三、核心功能详解

### 3.1 八大功能页面

| 页面 | 功能 | 数据源 |
|------|------|--------|
| **Overview** | 健康状态、关键操作项、运行问题、预算风险 | monitor.ts + snapshot-store |
| **Usage** | 今日/7天/30天用量趋势、订阅窗口、配额消耗、上下文压力 | usage-cost.ts + budget-governance |
| **Staff** | 谁在活跃工作 vs 谁只有排队任务 | agent-roster.ts + office-session-presence |
| **Collaboration** | 父子 session 传递 + 跨 session 消息（Main ⇄ Pandas） | session-conversations.ts |
| **Tasks** | 任务板、时间表、审批、执行链、运行时证据 | task-store.ts + task-summary.ts |
| **Documents** | 共享和 Agent 专属 Markdown 文档的工作台 | doc-hub.ts |
| **Memory** | 每个 Agent 的 memory 文件（状态、可搜索性） | 文件系统直接读取 |
| **Settings** | 安全模式、连接状态、版本信息、风险摘要 | config.ts + monitor-health.ts |

### 3.2 安全设计（多层防护）

```
第一层：只读模式
  READONLY_MODE=true → 不写任何 OpenClaw 文件

第二层：本地 Token 认证
  LOCAL_TOKEN_AUTH_REQUIRED=true → 所有 API 需 x-local-token header

第三层：变更操作硬门控
  IMPORT_MUTATION_ENABLED=false → 导入功能禁用
  APPROVAL_ACTIONS_ENABLED=false → 审批操作禁用
  APPROVAL_ACTIONS_DRY_RUN=true → 审批操作默认试运行

第四层：不修改核心配置
  只读取 ~/.openclaw/openclaw.json，绝不修改

第五层：只操作 control-center/ 目录
  不越界操作其他文件
```

### 3.3 轮询机制

```typescript
// config.ts 中的轮询间隔
POLLING_INTERVALS_MS = {
  sessionsList: 10000,    // 10秒 - session 列表
  sessionStatus: 2000,    // 2秒 - session 状态
  cron: 10000,            // 10秒 - Cron 任务
  approvals: 2000,        // 2秒 - 审批队列
  canvas: 5000,           // 5秒 - 画布
}
```

**智能刷新策略（OpenClawReadonlyAdapter.snapshot()）：**
- 新 session → 立即查询状态
- 活跃 session（running/blocked/waiting_approval）→ 每轮询都刷新
- 刚从活跃变非活跃 → 最后一次刷新
- 其他 → 使用缓存

---

## 四、对乐园的价值分析

### 4.1 能解决我们什么问题？

| 问题 | Control Center 能否解决 |
|------|------------------------|
| Agent 状态不透明（不知道谁在干嘛） | ✅ **Staff 页面完美解决** — 一眼看清谁活跃/谁空闲/谁阻塞 |
| 跨 Agent 通信不可靠 | ⚠️ **Collaboration 页面能看到** — 但只是可观测，不能修复 |
| 飞书 session 卡死 | ✅ **Overview 能检测到** — 显示 stalled runs |
| 预算/Token 消耗不透明 | ✅ **Usage 页面完美解决** — 用量趋势 + 成本分析 |
| 心跳配置问题 | ⚠️ **Cron 概览能看到** — 但不能直接修复 |
| 任务进度不透明 | ✅ **Tasks 页面完美解决** — 任务板 + 执行链 |

### 4.2 与乐园现有系统的对比

| 维度 | 乐园现状 | Control Center |
|------|---------|----------------|
| Agent 状态监控 | 无（靠手动检查） | Staff 页面（实时） |
| 任务追踪 | PHASE4-STATUS.md（手动更新） | Tasks 页面（自动） |
| 用量统计 | 无 | Usage 页面（自动） |
| 审批流程 | 无统一界面 | Approvals 面板 |
| 跨 Agent 通信监控 | 无 | Collaboration 页面 |
| 安全审计 | 无 | Settings + Audit |
| Memory 状态 | 无统一视图 | Memory 页面 |

### 4.3 与 Clawith 的定位对比

| 维度 | Clawith | Control Center |
|------|---------|----------------|
| **本质** | 替代 OpenClaw 的新平台 | OpenClaw 的监控面板 |
| **改动** | 全新架构（Python/FastAPI/React） | 不改动 OpenClaw，只读监控 |
| **Agent 管理** | 创建/配置/管理 Agent | 只观察，不管理 |
| **通信** | 实现新的通信机制 | 可观测现有通信 |
| **部署** | Docker Compose（重） | npm install（轻） |
| **风险** | 高（完全替换） | 极低（只读） |

**关键差异：** Control Center 是"看"，Clawith 是"改"。

---

## 五、乐园落地可行性

### 5.1 落地难度：⭐⭐ 极低

**为什么低？**
1. 只需 `npm install` + `cp .env.example .env` + `npm run dev:ui`
2. 不修改任何 OpenClaw 文件
3. 只读模式，零风险
4. 默认安全配置，不需要额外加固

### 5.2 部署方案

```bash
# 1. 克隆仓库
git clone https://github.com/TianyiDataScience/openclaw-control-center.git
cd openclaw-control-center

# 2. 安装依赖
npm install

# 3. 配置（保持默认安全设置）
cp .env.example .env
# 编辑 .env：确认 GATEWAY_URL=ws://127.0.0.1:18789

# 4. 构建 + 测试
npm run build
npm test
npm run smoke:ui

# 5. 启动 UI
npm run dev:ui

# 6. 打开浏览器
# http://127.0.0.1:4310/?section=overview&lang=zh
```

### 5.3 预期收益

| 收益 | 说明 |
|------|------|
| **Agent 状态可视化** | 不再需要挨个检查 session，Staff 页面一眼看清 |
| **用量透明** | 知道每个 Agent 消耗了多少 Token/API 调用 |
| **任务追踪** | 自动追踪任务进度，不用手动更新 PHASE4-STATUS.md |
| **通信可视化** | Collaboration 页面能看到 sessions_send 消息流 |
| **安全审计** | 能看到每个 Agent 的操作记录 |
| **Memory 健康** | 能检查每个 Agent 的 memory 文件是否正常 |

### 5.4 风险评估

| 风险 | 等级 | 缓解措施 |
|------|------|---------|
| 端口冲突（4310） | 低 | .env 中修改 UI_PORT |
| Gateway 连接失败 | 低 | 确认 GATEWAY_URL 正确 |
| 数据泄露 | 极低 | 只读 + 本地 Token 认证 |
| 性能影响 | 极低 | 轮询间隔合理（2-10秒） |

---

## 六、与通信方案的结合

### 6.1 Control Center + Phase 1 通信方案

**协同效应：**
- Phase 1 建立了 inbox 通信机制
- Control Center 的 **Collaboration 页面**可以可视化这些消息流
- 两者结合：既能通信，又能观测通信

**具体场景：**
```
1. Agent A 发消息给 Agent B（通过 inbox 文件）
2. Control Center Collaboration 页面显示：A → B (pending)
3. Agent B 心跳读取 inbox，处理消息
4. Control Center Collaboration 页面更新：A → B (delivered)
```

### 6.2 Control Center 能补充什么？

| 我们缺的 | Control Center 补充 |
|----------|-------------------|
| Agent 状态不透明 | Staff 页面实时状态 |
| 消息投递不可知 | Collaboration 页面消息流 |
| 任务进度手动维护 | Tasks 页面自动追踪 |
| 无用量统计 | Usage 页面自动统计 |
| Memory 状态未知 | Memory 页面健康检查 |

---

## 七、综合建议

### 7.1 优先级排序

| 优先级 | 项目 | 理由 |
|--------|------|------|
| 🔴 **最高** | 部署 Control Center | 极低成本，立即获得 Agent 状态可视化 |
| 🔴 **高** | Phase 1 通信方案 | 解决消息传递的根本问题 |
| 🟡 **中** | Clawith Aware 思想落地 | HEARTBEAT.md 升级为 Focus Items |
| 🟢 **低** | Clawith Relationship | 可以做但不是紧急 |

### 7.2 推荐行动

1. **立即部署 Control Center** — 5分钟部署，立即获得全乐园 Agent 的可视化监控
2. **用 Control Center 验证 Phase 1 效果** — 部署 inbox 通信后，在 Collaboration 页面验证消息流
3. **用 Control Center 的 Usage 数据做预算决策** — 了解哪个 Agent/API 消耗最多
4. **把 Control Center 作为"作战室"** — 日常监控不再靠手动检查

---

## 八、参考链接

- GitHub 仓库：https://github.com/TianyiDataScience/openclaw-control-center
- Stars：2,016+（截至 2026-03-18）
- 安装指引：INSTALL_PROMPT.md / INSTALL_PROMPT.en.md
- 拉姆达通信方案：knowledge-base/architecture/messaging-architecture.md
- Clawith 研究：knowledge-base/cases/clawith-research-analysis.md

---

*本研究报告由拉姆达 🔬 基于 GitHub 源码分析和 README 文档研究完成。*
*Control Center 和 Clawith 是两个不同方向的项目：一个是"看"，一个是"改"。*
*建议优先部署 Control Center（低风险高回报），再逐步吸收 Clawith 的设计思想。*
