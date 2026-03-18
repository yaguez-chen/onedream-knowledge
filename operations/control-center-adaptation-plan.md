# OpenClaw Control Center 乐园适配方案

> **作者：** 拉姆达 🔬
> **日期：** 2026-03-18
> **版本：** v1.0
> **目标：** 将 OpenClaw Control Center 完整适配到梦想家乐园环境，支持自主迭代升级
> **标签：** #实施 #运维 #可视化 #监控

---

## 一、乐园环境全景

### 1.1 Agent 花名册（11个）

| Agent ID | 名字 | 角色 | 工作空间 | 飞书 Bot |
|----------|------|------|---------|----------|
| `main` | 阿尔法 🦐 | 首席伙伴 | `workspace/` | 阿尔法 |
| `beta` | 贝塔 🔵 | 监察者 | `workspace-beta/` | 贝塔 |
| `gamma` | 伽马 🔧 | 工匠 | `workspace-gamma/` | 伽马 |
| `delta` | 德尔塔 📊 | 数据分析师 | `workspace-delta/` | 德尔塔 |
| `epsilon` | 艾普西隆 Σ | 架构师 | `workspace-epsilon/` | 艾普西隆 |
| `zeta` | 泽塔 ⚙️ | 运维工程师 | `workspace-zeta/` | 泽塔 |
| `eta` | 艾塔 🗄️ | 归档监察员 | `workspace-eta/` | 艾塔 |
| `theta` | 西塔 🎨 | 设计师 | `workspace-theta/` | 西塔 |
| `iota` | 约塔 📋 | 简报员 | `workspace-iota/` | 约塔 |
| `kappa` | 卡帕 🏛️ | 建筑师 | `workspace-kappa/` | 卡帕 |
| `lambda` | 拉姆达 🔬 | 研究员 | `workspace-lambda/` | 拉姆达 |

### 1.2 基础设施

| 组件 | 配置 |
|------|------|
| **Gateway** | `ws://127.0.0.1:18789`（默认） |
| **模型** | `xiaomi/mimo-claw-0301`（默认） |
| **心跳** | 30 分钟间隔 |
| **渠道** | 飞书（11个独立 Bot 账号） |
| **子代理** | 最大并发 8 |
| **主会话** | 最大并发 4 |
| **OpenClaw Home** | `/home/gang/.openclaw/` |
| **配置文件** | `/home/gang/.openclaw/openclaw.json` |

### 1.3 当前通信基础设施（Phase 1 已实施）

```
~/.openclaw/workspace-{agent}/
├── inbox/                    # 消息收件箱（Phase 1）
│   ├── .last-read            # 已读标记
│   ├── msg-*.json            # 消息文件
├── HEARTBEAT.md              # 已更新：含 Focus Items + inbox 检查
└── ...

~/.openclaw/shared-knowledge/     # 共享知识库
~/.openclaw/workspace-shared/     # 共享工作空间（Plaza 待实施）
```

---

## 二、部署方案

### 2.1 安装步骤

```bash
# === 第一步：克隆仓库 ===
cd /home/gang/.openclaw/
git clone https://github.com/TianyiDataScience/openclaw-control-center.git
cd openclaw-control-center

# === 第二步：安装依赖 ===
npm install

# === 第三步：创建乐园专用配置 ===
cp .env.example .env
```

### 2.2 乐园专用 .env 配置

```dotenv
# === OpenClaw Control Center - 梦想家乐园配置 ===
# 文件: .env
# 创建日期: 2026-03-18
# 维护者: 拉姆达 / 泽塔

# --- Gateway 连接 ---
GATEWAY_URL=ws://127.0.0.1:18789

# --- 路径配置 ---
OPENCLAW_HOME=/home/gang/.openclaw
# OPENCLAW_CONFIG_PATH=/home/gang/.openclaw/openclaw.json  # 默认路径，无需覆盖
# OPENCLAW_WORKSPACE_ROOT=/home/gang/.openclaw             # 所有工作空间的根目录

# --- 安全设置（保持默认安全）---
READONLY_MODE=true
LOCAL_TOKEN_AUTH_REQUIRED=true
LOCAL_API_TOKEN=paradise-2026-secure-token-change-me

# --- 变更控制（保持禁用）---
APPROVAL_ACTIONS_ENABLED=false
APPROVAL_ACTIONS_DRY_RUN=true
IMPORT_MUTATION_ENABLED=false
IMPORT_MUTATION_DRY_RUN=false

# --- 任务心跳（可选启用）---
TASK_HEARTBEAT_ENABLED=true
TASK_HEARTBEAT_DRY_RUN=true
TASK_HEARTBEAT_MAX_TASKS_PER_RUN=5

# --- 监控模式 ---
MONITOR_CONTINUOUS=true

# --- UI 设置 ---
UI_MODE=true
UI_PORT=4310
# UI_BIND_ADDRESS=127.0.0.1  # 仅本机访问（安全）
```

### 2.3 验证与启动

```bash
# === 第四步：构建 + 测试 ===
npm run build
npm test
npm run smoke:ui

# === 第五步：启动（前台测试）===
npm run dev:ui

# === 第六步：打开浏览器 ===
# http://127.0.0.1:4310/?section=overview&lang=zh
```

### 2.4 生产部署（后台运行）

```bash
# 方式 A：nohup（简单）
nohup npm run dev:ui > /home/gang/.openclaw/control-center.log 2>&1 &
echo $! > /home/gang/.openclaw/control-center.pid

# 方式 B：systemd service（推荐，泽塔负责）
# 见 2.5 systemd 配置

# 方式 C：screen/tmux（调试用）
tmux new -s control-center 'cd /home/gang/.openclaw/control-center && npm run dev:ui'
```

### 2.5 systemd Service 配置（推荐）

```ini
# /etc/systemd/system/openclaw-control-center.service
[Unit]
Description=OpenClaw Control Center - 梦想家乐园监控面板
After=network.target
Wants=network.target

[Service]
Type=simple
User=gang
WorkingDirectory=/home/gang/.openclaw/control-center
ExecStart=/usr/bin/npm run dev:ui
Restart=on-failure
RestartSec=10
StandardOutput=append:/home/gang/.openclaw/logs/control-center.log
StandardError=append:/home/gang/.openclaw/logs/control-center.log

# 安全加固
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/home/gang/.openclaw/control-center
ReadOnlyPaths=/home/gang/.openclaw

[Install]
WantedBy=multi-user.target
```

```bash
# 启用并启动
sudo systemctl daemon-reload
sudo systemctl enable openclaw-control-center
sudo systemctl start openclaw-control-center
sudo systemctl status openclaw-control-center
```

---

## 三、乐园定制化

### 3.1 Agent 名称映射

Control Center 会自动从 `~/.openclaw/openclaw.json` 读取 Agent 列表。但默认显示的是 Agent ID（main, beta, gamma...），建议在 UI 层添加名称映射。

**方案：创建 `agent-display-names.json`**

在 Control Center 仓库根目录创建：

```json
{
  "main": "阿尔法 🦐",
  "beta": "贝塔 🔵",
  "gamma": "伽马 🔧",
  "delta": "德尔塔 📊",
  "epsilon": "艾普西隆 Σ",
  "zeta": "泽塔 ⚙️",
  "eta": "艾塔 🗄️",
  "theta": "西塔 🎨",
  "iota": "约塔 📋",
  "kappa": "卡帕 🏛️",
  "lambda": "拉姆达 🔬"
}
```

**注意：** 此定制需要修改 Control Center 源码或在 Fork 中维护。详见"六、自主迭代框架"。

### 3.2 角色分组

在 Staff 页面建议按角色分组显示：

| 分组 | Agent | 说明 |
|------|-------|------|
| **领导层** | 阿尔法（首席） | 直接对接梦想家 |
| **监察** | 贝塔（监察者）、艾塔（归档） | 质量与合规 |
| **技术** | 伽马（工匠）、泽塔（运维）、艾普西隆（架构） | 技能与基础设施 |
| **分析** | 德尔塔（数据）、拉姆达（研究） | 数据与知识 |
| **设计** | 西塔（设计）、卡帕（建筑） | 创意与架构 |
| **沟通** | 约塔（简报） | 信息传递 |

### 3.3 自定义 Dashboard 面板建议

基于乐园现有机制，建议新增以下自定义面板：

#### 面板 A：Inbox 消息流（Phase 1 通信可视化）

```
┌─────────────────────────────────────────────┐
│ 📬 Inbox 消息流                              │
├─────────────────────────────────────────────┤
│ 拉姆达 → 伽马    "搜索能力评估"    ✅ 已读   │
│ 阿尔法 → 全员    "嘉奖通报"       ✅ 已读   │
│ 拉姆达 → 阿尔法  "Clawith 研究"   ⏳ 未读   │
│ 梦想家 → 拉姆达  "深入研究"       ✅ 已读   │
├─────────────────────────────────────────────┤
│ 统计: 今日 12 条 | 未读 1 条 | 高优先 0 条   │
└─────────────────────────────────────────────┘
```

**数据源：** 扫描 `workspace-*/inbox/msg-*.json` 文件

#### 面板 B：知识库健康度

```
┌─────────────────────────────────────────────┐
│ 📚 知识库健康度                              │
├─────────────────────────────────────────────┤
│ shared-knowledge/     15 文档  ✅ 最新 2h前  │
│ 知识库（拉姆达）       8 文档  ✅ 最新 1h前  │
│ 共享工作空间          3 文件   ⚠️ 最新 2d前  │
├─────────────────────────────────────────────┤
│ 最近更新: messaging-architecture.md (2h前)   │
│ 待归档: 0 个交付物                           │
└─────────────────────────────────────────────┘
```

**数据源：** 扫描 `shared-knowledge/` 和 `workspace-*/knowledge-base/`

#### 面板 C：心跳健康度

```
┌─────────────────────────────────────────────┐
│ 💓 心跳健康度                                │
├─────────────────────────────────────────────┤
│ 阿尔法  ✅ 12分钟前  HEARTBEAT: 有内容       │
│ 贝塔    ✅ 8分钟前   HEARTBEAT: 有内容       │
│ 伽马    ✅ 15分钟前  HEARTBEAT: 有内容       │
│ 德尔塔  ⚠️ 45分钟前  HEARTBEAT: 有内容       │
│ 拉姆达  ✅ 5分钟前   HEARTBEAT: Focus Items  │
├─────────────────────────────────────────────┤
│ 健康: 10/11 | 警告: 1 | 异常: 0             │
└─────────────────────────────────────────────┘
```

**数据源：** Gateway session 状态 + HEARTBEAT.md 内容检查

#### 面板 D：Phase 进度追踪

```
┌─────────────────────────────────────────────┐
│ 📊 项目进度                                  │
├─────────────────────────────────────────────┤
│ Phase 1: 通信基础      ██████████ ✅ 完成    │
│ Phase 2: 增强通道      ███░░░░░░░ 🔄 30%    │
│ Phase 3: 高级特性      ░░░░░░░░░░ ⏳ 待启动  │
├─────────────────────────────────────────────┤
│ 活跃任务: 3 | 等待审批: 0 | 已完成今日: 5    │
└─────────────────────────────────────────────┘
```

**数据源：** `PHASE*-STATUS.md` 文件 + Tasks 模块

---

## 四、与乐园通信系统的集成

### 4.1 集成架构

```
Control Center（只读监控）
    ↓ 读取
乐园文件系统
    ├── workspace-*/inbox/msg-*.json     → Inbox 消息流面板
    ├── workspace-*/HEARTBEAT.md          → 心跳健康面板
    ├── shared-knowledge/**/*.md          → 知识库面板
    ├── PHASE*-STATUS.md                  → 进度追踪面板
    └── *.jsonl (session transcripts)     → 会话分析面板
```

### 4.2 数据接入点

| 乐园数据 | Control Center 接入方式 | 说明 |
|----------|------------------------|------|
| **Session 列表** | Gateway WebSocket（原生） | 自动获取所有 Agent 的 session 状态 |
| **Cron 任务** | Gateway WebSocket（原生） | 自动获取 cron 任务列表和状态 |
| **审批队列** | Gateway WebSocket（原生） | 自动获取审批请求 |
| **Inbox 消息** | 文件系统扫描（需定制） | 读取 `workspace-*/inbox/*.json` |
| **知识库** | 文件系统扫描（需定制） | 读取 `shared-knowledge/` 和 `workspace-*/knowledge-base/` |
| **HEARTBEAT.md** | 文件系统扫描（需定制） | 解析心跳配置和最近执行时间 |
| **Phase 进度** | 文件解析（需定制） | 读取 `PHASE*-STATUS.md` |

### 4.3 读取 inbox 消息的伪代码

```typescript
// src/runtime/paradise-inbox.ts（需新增）
import { readdirSync, readFileSync } from "fs";
import { join } from "path";

interface InboxMessage {
  id: string;
  timestamp: string;
  from: string;
  priority: "high" | "normal" | "low";
  type: string;
  subject: string;
  body: string;
  read: boolean;
}

const WORKSPACE_PREFIX = "/home/gang/.openclaw/workspace-";
const AGENTS = ["main","beta","gamma","delta","epsilon","zeta","eta","theta","iota","kappa","lambda"];

export function scanAllInboxes(): Map<string, InboxMessage[]> {
  const result = new Map<string, InboxMessage[]>();
  
  for (const agent of AGENTS) {
    const inboxDir = agent === "main" 
      ? "/home/gang/.openclaw/workspace/inbox"
      : `${WORKSPACE_PREFIX}${agent}/inbox`;
    
    try {
      const lastReadFile = join(inboxDir, ".last-read");
      const lastRead = readFileSync(lastReadFile, "utf-8").trim();
      
      const files = readdirSync(inboxDir)
        .filter(f => f.startsWith("msg-") && f.endsWith(".json"))
        .sort();
      
      const messages: InboxMessage[] = [];
      for (const file of files) {
        const raw = readFileSync(join(inboxDir, file), "utf-8");
        const msg: InboxMessage = JSON.parse(raw);
        msg.read = file.replace(".json", "") <= lastRead;
        messages.push(msg);
      }
      
      result.set(agent, messages);
    } catch (e) {
      result.set(agent, []);
    }
  }
  
  return result;
}
```

---

## 五、升级与维护策略

### 5.1 版本管理策略

```
upstream (原仓库)          fork (乐园定制)
main ──────────────────→ paradise-main
  │                         │
  │ git merge upstream      │ 乐园定制：
  │                         │ - agent-display-names.json
  ↓                         │ - paradise-inbox.ts
新版本发布                   │ - paradise-kb.ts
                            │ - 自定义 UI 面板
                            ↓
                      npm run build
                      systemctl restart
```

**推荐：Fork + Upstream Merge 策略**

```bash
# 1. Fork 到自己的 GitHub（如 onedreamer-ai/openclaw-control-center）
# 2. 添加 upstream 远程仓库
git remote add upstream https://github.com/TianyiDataScience/openclaw-control-center.git

# 3. 每次升级时
git fetch upstream
git merge upstream/main --no-edit
# 解决冲突（如有）
npm install  # 依赖可能更新
npm run build
npm test
# 重启服务
sudo systemctl restart openclaw-control-center
```

### 5.2 定制文件清单

以下文件是乐园定制的，升级时需要注意保留：

| 文件 | 用途 | 升级风险 |
|------|------|---------|
| `.env` | 乐园环境配置 | 低（不进 git） |
| `agent-display-names.json` | Agent 显示名映射 | 低（独立文件） |
| `src/runtime/paradise-inbox.ts` | Inbox 消息扫描 | 中（需与 API 适配） |
| `src/runtime/paradise-kb.ts` | 知识库扫描 | 中（同上） |
| `src/ui/paradise-panels.tsx` | 自定义 UI 面板 | 高（依赖 React 组件） |

### 5.3 升级检查清单

```markdown
## Control Center 升级检查清单

### 升级前
- [ ] 备份当前 .env 文件
- [ ] 备份所有 paradise-*.ts / paradise-*.tsx 文件
- [ ] 记录当前版本号（git log --oneline -1）
- [ ] 检查 CHANGELOG 或 Release Notes

### 升级中
- [ ] git fetch upstream
- [ ] git merge upstream/main --no-edit
- [ ] 检查冲突（git diff --name-only --diff-filter=U）
- [ ] 解决冲突，保留乐园定制
- [ ] npm install（更新依赖）
- [ ] npm run build（构建）

### 升级后
- [ ] npm test（测试通过）
- [ ] npm run smoke:ui（冒烟测试）
- [ ] 检查 .env 配置未被覆盖
- [ ] 检查自定义面板正常显示
- [ ] 重启服务（systemctl restart）
- [ ] 验证 UI 可访问（curl http://127.0.0.1:4310）
- [ ] 记录升级日志

### 回滚计划
- [ ] 如失败：git reset --hard HEAD~1
- [ ] 恢复备份的 .env 和定制文件
- [ ] npm install + npm run build
- [ ] 重启服务
```

---

## 六、自主迭代框架

### 6.1 迭代层次

```
Level 0：纯配置（无需改代码）
├── .env 参数调整
├── UI_PORT / GATEWAY_URL 修改
└── 安全策略调整（READONLY_MODE 等）

Level 1：数据接入（新增 runtime 模块）
├── paradise-inbox.ts（Inbox 消息扫描）
├── paradise-kb.ts（知识库扫描）
├── paradise-heartbeat.ts（心跳健康检查）
└── paradise-phase.ts（Phase 进度追踪）

Level 2：UI 定制（新增 React 面板）
├── 自定义 Dashboard 面板
├── Agent 名称映射显示
└── 角色分组视图

Level 3：核心改动（需谨慎）
├── Adapter 层扩展
├── 轮询策略调整
└── 新的数据源接入
```

### 6.2 Level 1 迭代示例：添加 Inbox 消息扫描

**步骤 1：创建 `src/runtime/paradise-inbox.ts`**

参考 4.3 节的伪代码实现。

**步骤 2：在 `src/index.ts` 中注册**

```typescript
import { scanAllInboxes } from "./runtime/paradise-inbox";

// 在 snapshot() 调用后添加
const inboxMessages = scanAllInboxes();
```

**步骤 3：创建 API 端点 `src/runtime/paradise-inbox-api.ts`**

```typescript
// 在 UI server 中添加 /api/paradise/inbox 路由
```

**步骤 4：构建并测试**

```bash
npm run build && npm test
```

### 6.3 Level 2 迭代示例：添加自定义 Dashboard 面板

**步骤 1：确定面板位置**

Control Center 使用 React。建议在 `src/ui/` 下创建 `paradise/` 子目录。

**步骤 2：创建面板组件**

```tsx
// src/ui/paradise/InboxPanel.tsx
export function InboxPanel() {
  const [messages, setMessages] = useState([]);
  
  useEffect(() => {
    fetch("/api/paradise/inbox")
      .then(r => r.json())
      .then(setMessages);
  }, []);
  
  return (
    <div className="paradise-panel">
      <h2>📬 Inbox 消息流</h2>
      {/* 渲染消息列表 */}
    </div>
  );
}
```

**步骤 3：在主 UI 中集成**

找到 Control Center 的主布局组件，添加 Paradise 面板区域。

### 6.4 迭代原则

1. **不动核心文件** — 尽量只添加 `paradise-*` 前缀的新文件
2. **通过 import 注入** — 在 `index.ts` 和 `server.ts` 中 import 定制模块
3. **保持向兼容** — 不删除 upstream 功能，只添加
4. **文档先行** — 每次迭代先更新本文档
5. **测试必过** — `npm test` 通过才能部署

---

## 七、监控与告警

### 7.1 建议监控项

| 监控项 | 阈值 | 告警方式 | 负责人 |
|--------|------|---------|--------|
| Control Center 进程存活 | 进程退出 | systemd 自动重启 + 泽塔通知 | 泽塔 |
| Gateway 连接状态 | 连接断开 > 5分钟 | UI 红色警告 | 自动 |
| Agent session 卡死 | blocked > 30分钟 | UI 黄色警告 | 自动 |
| Agent 心跳超时 | 最后心跳 > 60分钟 | UI 红色警告 | 自动 |
| 磁盘空间 | < 5GB | 泽塔通知 | 泽塔 |
| API 预算超支 | 超过月度预算 80% | Usage 页面警告 | 德尔塔 |

### 7.2 日志管理

```bash
# 日志位置
/home/gang/.openclaw/logs/control-center.log

# 日志轮转（logrotate 配置）
# /etc/logrotate.d/openclaw-control-center
/home/gang/.openclaw/logs/control-center.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    create 0644 gang gang
}
```

---

## 八、安全加固

### 8.1 网络安全

```bash
# 仅本机访问（默认）
UI_BIND_ADDRESS=127.0.0.1

# 如需远程访问（通过 SSH 隧道代替）
ssh -L 4310:127.0.0.1:4310 gang@<服务器IP>

# 如必须开放外网（不推荐）
# 1. 更换端口为非标准端口
# 2. 配置防火墙白名单
# 3. 启用 Nginx 反向代理 + HTTPS
```

### 8.2 Token 管理

```bash
# 生成强 Token
openssl rand -hex 32

# 更新 .env
LOCAL_API_TOKEN=<生成的强Token>

# 重启服务
sudo systemctl restart openclaw-control-center
```

### 8.3 文件权限

```bash
# .env 文件（含敏感 Token）
chmod 600 /home/gang/.openclaw/control-center/.env

# 日志目录
chmod 755 /home/gang/.openclaw/logs/
```

---

## 九、与现有工具的协同

### 9.1 Control Center + Clawith 研究

| 维度 | Control Center | Clawith |
|------|----------------|---------|
| **部署优先级** | 🔴 立即 | 🟡 观望 |
| **风险** | 极低（只读） | 高（全新平台） |
| **关系** | "看得见" | "改得了" |
| **协同** | Control Center 可作为 Clawith 的监控层 | Clawith 的 Agent 状态可被 CC 监控 |

### 9.2 Control Center + Phase 1 通信

```
Phase 1 通信：Agent A → inbox/ → Agent B 心跳读取
Control Center：监控这个流程
    ├── Staff 页面：Agent A/B 状态
    ├── Collaboration 页面：sessions_send 消息流
    └── Inbox 面板（定制）：文件消息投递状态
```

---

## 十、实施时间表

| 阶段 | 任务 | 负责人 | 预计时间 |
|------|------|--------|---------|
| **Day 1** | 安装 + 基础配置 + 测试 | 拉姆达/泽塔 | 1小时 |
| **Day 1** | systemd 部署 + 日志配置 | 泽塔 | 30分钟 |
| **Day 2** | 验证所有 Agent 可见 | 拉姆达 | 30分钟 |
| **Day 3-5** | Level 1 定制（Inbox 扫描） | 拉姆达/伽马 | 2-3小时 |
| **Week 2** | Level 2 定制（自定义面板） | 西塔/卡帕 | 1-2天 |
| **Ongoing** | 升级维护 | 泽塔 | 每月检查 |

---

## 十一、常见问题

### Q1: Control Center 会影响 OpenClaw 性能吗？
**A:** 不会。它是只读的，通过 Gateway WebSocket 获取数据，轮询间隔 2-10 秒。对 Gateway 的负载极小。

### Q2: 如果 Control Center 崩溃了会怎样？
**A:** 不影响任何 Agent。它只是监控面板，崩溃只影响可视化，不影响 Agent 运行。

### Q3: 可以远程访问吗？
**A:** 默认只本机访问。远程访问推荐 SSH 隧道：`ssh -L 4310:127.0.0.1:4310 gang@服务器`

### Q4: 升级会不会丢失数据？
**A:** 不会。Control Center 不存储持久数据（除了快照缓存），所有数据都从 Gateway 和文件系统实时读取。

### Q5: 我们需要 Fork 吗？
**A:** 建议 Fork。Level 1+ 的定制需要修改源码，Fork 可以方便地合并上游更新。

---

## 附录 A：快速参考卡

```bash
# 启动
sudo systemctl start openclaw-control-center

# 停止
sudo systemctl stop openclaw-control-center

# 重启
sudo systemctl restart openclaw-control-center

# 查看日志
journalctl -u openclaw-control-center -f

# 查看状态
sudo systemctl status openclaw-control-center

# 手动启动（调试）
cd /home/gang/.openclaw/control-center && npm run dev:ui

# 升级
cd /home/gang/.openclaw/control-center
git fetch upstream && git merge upstream/main
npm install && npm run build
sudo systemctl restart openclaw-control-center
```

---

## 附录 B：相关文档

| 文档 | 路径 |
|------|------|
| Control Center GitHub | github.com/TianyiDataScience/openclaw-control-center |
| 乐园通信方案 | shared-knowledge/architecture/messaging-architecture.md |
| Clawith 研究 | knowledge-base/cases/clawith-research-analysis.md |
| Control Center 研究 | knowledge-base/cases/openclaw-control-center-research.md |
| 乐园关系图谱 | RELATIONSHIPS.md（待创建） |
| Phase 1 状态 | PHASE1-STATUS.md |

---

*本方案由拉姆达 🔬 编写，泽塔运维支持。*
*遵循"先可控，再信任"原则 — Control Center 是乐园的"作战室"。*
*如遇问题，联系拉姆达（研究相关）或泽塔（运维相关）。*
