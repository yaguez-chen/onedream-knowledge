# .learnings/ 系统乐园落地方案

> **作者：** 拉姆达 🔬
> **日期：** 2026-03-18
> **版本：** v1.0
> **来源：** self-improving-agent 项目 + 乐园适配
> **标签：** #实施 #记忆系统 #学习记录

---

## 一、系统概述

### 1.1 什么是 .learnings/ 系统？

让每个 Agent 在工作过程中**主动记录学到的东西**，通过升级链把零散的学习记录精炼成持久化规则（写入 SOUL.md / AGENTS.md / TOOLS.md）。

### 1.2 核心价值

- **跨会话记忆** — 新 session 不会忘记之前学到的
- **错误复用** — 踩过的坑不再踩
- **知识积累** — Agent 随时间越来越聪明
- **社区共享** — 高价值学习提取成 Skill 分享

### 1.3 与现有系统的关系

```
乐园现有记忆层：
├── SOUL.md          ← 升级链目标（行为准则）
├── AGENTS.md        ← 升级链目标（工作流规则）
├── TOOLS.md         ← 升级链目标（工具细节）
├── MEMORY.md        ← 长期记忆（手动维护）
├── memory/*.md      ← 每日日记（自动/半自动）
└── SELF-IMPROVEMENT.md ← 错误纠正（已有）

新增 .learnings/ 系统：
├── .learnings/
│   ├── LEARNINGS.md         ← 日常学习记录
│   ├── ERRORS.md            ← 错误记录（补充 SELF-IMPROVEMENT）
│   └── FEATURE_REQUESTS.md  ← 功能需求记录
└── 升级链 → SOUL.md / AGENTS.md / TOOLS.md / 独立 Skill
```

**关键：** `.learnings/` 不替代现有系统，是**补充层**。它连接"日常经验"和"持久化规则"。

---

## 二、文件结构

### 2.1 每个 Agent 工作空间

```
~/.openclaw/workspace-{agent}/
├── .learnings/
│   ├── LEARNINGS.md         # 日常学习记录
│   ├── ERRORS.md            # 错误记录
│   ├── FEATURE_REQUESTS.md  # 功能需求
│   └── .stats               # 统计数据（可选）
├── SOUL.md                  # 升级链目标 ✅ 已有
├── AGENTS.md                # 升级链目标 ✅ 已有
├── TOOLS.md                 # 升级链目标 ✅ 已有
├── SELF-IMPROVEMENT.md      # 错误纠正 ✅ 已有
└── memory/
    └── YYYY-MM-DD.md        # 每日日记 ✅ 已有
```

### 2.2 记录格式标准

**LEARNINGS.md 格式：**
```markdown
# .learnings/LEARNINGS.md — 学习记录

> Agent 在工作中学到的知识、最佳实践、约定。
> 定期 review，符合条件的升级到 SOUL.md / AGENTS.md / TOOLS.md。

## 活跃记录

### [LRN-20260318-001] 飞书 session 卡死后恢复方法
- **Logged**: 2026-03-18T01:30:00+08:00
- **Priority**: high
- **Area**: operations
- **Status**: pending
- **Summary**: 飞书 session 卡死（流式错误导致）时，需要在飞书里给 bot 发消息重新激活
- **Details**: sessions_send timeout 不代表消息未送达，而是 session 无法处理新请求。恢复方法：在飞书客户端给对应 bot 发任意消息。
- **Upgrade Target**: TOOLS.md
- **Occurrences**: 2 (2026-03-17, 2026-03-18)

### [LRN-20260318-002] 空 HEARTBEAT.md 会导致心跳被跳过
- **Logged**: 2026-03-18T01:35:00+08:00
- **Priority**: high
- **Area**: config
- **Status**: pending
- **Summary**: HEARTBEAT.md 只有注释时，OpenClaw 认为无任务，跳过心跳
- **Details**: 这是设计行为（节省 API 调用），不是 bug。解决：至少添加一个实际任务。
- **Upgrade Target**: AGENTS.md
- **Occurrences**: 3 (2026-03-15, 2026-03-16, 2026-03-18)

## 已升级

### [LRN-20260314-001] 三条铁律仅对梦想家生效 ✅
- **Logged**: 2026-03-14T16:50:00+08:00
- **Upgraded**: 2026-03-15 → SOUL.md
- **Rule**: "这三条规则仅适用于梦想家下达的指令"
```

**ERRORS.md 格式：**
```markdown
# .learnings/ERRORS.md — 错误记录

> 工具调用失败、命令报错、异常情况。
> 用于避免重复犯错。

## 活跃错误

### [ERR-20260318-001] web_search 缺少 Brave API Key
- **Logged**: 2026-03-18T04:19:00+08:00
- **Command**: web_search({ query: "..." })
- **Error**: `missing_brave_api_key`
- **Fix**: 运行 `openclaw configure --section web` 或用 DuckDuckGo 替代
- **Workaround**: 使用 web-search 技能（DuckDuckGo）
- **Status**: workaround_active
- **Occurrences**: 5

### [ERR-20260318-002] web_fetch 无法抓取今日头条
- **Logged**: 2026-03-18T04:22:00+08:00
- **Command**: web_fetch({ url: "https://m.toutiao.com/..." })
- **Error**: `Readability and Firecrawl returned no content`
- **Fix**: 使用 Agent Browser 代替
- **Workaround**: `agent-browser open <url>` + `agent-browser snapshot -c`
- **Status**: permanent_limitation
- **Occurrences**: 3
```

**FEATURE_REQUESTS.md 格式：**
```markdown
# .learnings/FEATURE_REQUESTS.md — 功能需求

> 用户提到的缺失功能、期望改进。
> 定期评估是否可实现。

## 待评估

### [FEAT-20260318-001] Agent 间即时通信
- **Logged**: 2026-03-18T02:00:00+08:00
- **Requested By**: 梦想家（通过研究发现）
- **Description**: Agent A 发消息给 Agent B 时，B 能秒级响应，不用等心跳
- **Current Workaround**: 心跳轮询（最坏 30 分钟延迟）
- **Priority**: high
- **Related**: Phase 1 通信方案 / on_message 触发器

### [FEAT-20260318-002] Agent 状态可视化
- **Logged**: 2026-03-18T05:05:00+08:00
- **Requested By**: 梦想家
- **Description**: 一个面板看清所有 Agent 的运行状态、任务进度、消息流
- **Current Workaround**: 手动检查 session 列表
- **Priority**: high
- **Related**: OpenClaw Control Center 适配方案
```

---

## 三、升级链规则

### 3.1 升级触发条件

| 条件 | 说明 |
|------|------|
| **同一 pattern 出现 ≥3 次** | 不是一次性事件 |
| **跨 ≥2 个不同 task** | 具有通用性 |
| **30 天以内** | 仍然相关 |

### 3.2 升级目标映射

| 学习类型 | 升级到 | 示例 |
|----------|--------|------|
| 行为准则、沟通风格 | SOUL.md | "对梦想家指令必须先确认再执行" |
| 工作流规则、自动化 | AGENTS.md | "心跳必须包含实际任务，否则被跳过" |
| 工具坑、集成细节 | TOOLS.md | "飞书 session 卡死后在飞书发消息激活" |
| 项目约定 | 项目 CLAUDE.md | "这个项目用 pnpm，不用 npm" |
| 跨项目通用解法 | 独立 Skill | "网页抓取失败时用 Agent Browser" |

### 3.3 升级流程

```
1. Review 触发（手动或自动）
   ↓
2. 检查是否满足升级条件（3次 + 2个task + 30天）
   ↓
3. 精炼：把冗长记录浓缩成 1-2 句规则
   ↓
4. 写入目标文件（SOUL.md / AGENTS.md / TOOLS.md）
   ↓
5. 标记原记录为 "已升级"
   ↓
6. （可选）提取成 Skill → ClawHub
```

### 3.4 升级示例

**原始记录（.learnings/LEARNINGS.md）：**
```markdown
### [LRN-20260318-002] 空 HEARTBEAT.md 会导致心跳被跳过
- **Logged**: 2026-03-18T01:35:00+08:00
- **Priority**: high
- **Area**: config
- **Summary**: HEARTBEAT.md 只有注释时，OpenClaw 认为无任务，跳过心跳
- **Details**: 这是设计行为（节省 API 调用）。解决：至少添加一个实际任务。
- **Occurrences**: 3 (2026-03-15, 2026-03-16, 2026-03-18)
```

**升级后（AGENTS.md 添加一行）：**
```markdown
## 心跳配置注意事项
- HEARTBEAT.md 必须包含实际任务（不只是注释），否则心跳被跳过
- 这是设计行为，目的是节省 API 调用
- 最低配置：至少一个需要检查的项目
```

---

## 四、心跳集成

### 4.1 HEARTBEAT.md 新增检查项

```markdown
## .learnings/ 检查（每5次心跳检查一次）

### 学习记录检查
- 如本次会话中有新学到的东西 → 记录到 .learnings/LEARNINGS.md
- 如有工具调用失败 → 记录到 .learnings/ERRORS.md
- 如用户提到缺失功能 → 记录到 .learnings/FEATURE_REQUESTS.md

### 升级检查（每周一次）
- 检查 .learnings/ 中是否有满足升级条件的记录
- 如有，精炼并写入目标文件
- 标记原记录为已升级
```

### 4.2 Agent 启动时的学习注入

在每个 Agent 的 AGENTS.md 中添加：

```markdown
## 启动时
1. 读取 SOUL.md / USER.md / memory/YYYY-MM-DD.md（已有）
2. **新增：读取 .learnings/LEARNINGS.md（前 20 条活跃记录）**
3. **新增：读取 .learnings/ERRORS.md（前 10 条活跃错误）**
4. 这些学习记录注入上下文，让 Agent 知道之前踩过的坑
```

---

## 五、实施步骤

### Phase 1：基础搭建（立即可做）

```bash
# 为所有 Agent 创建 .learnings/ 目录
for agent in main beta gamma delta epsilon zeta eta theta iota kappa lambda; do
  if [ "$agent" = "main" ]; then
    WS="/home/gang/.openclaw/workspace"
  else
    WS="/home/gang/.openclaw/workspace-$agent"
  fi
  
  mkdir -p "$WS/.learnings"
  
  # 创建模板文件
  cat > "$WS/.learnings/LEARNINGS.md" << 'EOF'
# .learnings/LEARNINGS.md — 学习记录

> Agent 在工作中学到的知识、最佳实践、约定。
> 定期 review，符合条件的升级到 SOUL.md / AGENTS.md / TOOLS.md。

## 活跃记录

（暂无记录）

## 已升级

（暂无升级记录）
EOF

  cat > "$WS/.learnings/ERRORS.md" << 'EOF'
# .learnings/ERRORS.md — 错误记录

> 工具调用失败、命令报错、异常情况。
> 用于避免重复犯错。

## 活跃错误

（暂无错误记录）

## 已解决

（暂无已解决记录）
EOF

  cat > "$WS/.learnings/FEATURE_REQUESTS.md" << 'EOF'
# .learnings/FEATURE_REQUESTS.md — 功能需求

> 用户提到的缺失功能、期望改进。

## 待评估

（暂无需求记录）

## 已实现

（暂无已实现记录）
EOF

  echo "✅ $agent: .learnings/ 已创建"
done
```

### Phase 2：心跳集成（更新 HEARTBEAT.md）

在所有 Agent 的 HEARTBEAT.md 中添加 .learnings/ 检查。

### Phase 3：升级链自动化（待 Phase 1 稳定后）

- 编写 review 脚本，自动检测满足升级条件的记录
- 半自动升级：检测到 → 提示 Agent → Agent 精炼并写入目标文件

### Phase 4：Control Center 集成（与 Control Center 适配同步）

- 新增"记忆健康度"面板
- 显示每个 Agent 的 .learnings/ 统计
- 追踪升级链转化率

---

## 六、统计与度量

### 6.1 关键指标

| 指标 | 说明 | 目标 |
|------|------|------|
| 学习记录数 | .learnings/LEARNINGS.md 活跃记录 | 持续增长 |
| 错误记录数 | .learnings/ERRORS.md 活跃记录 | 逐步降低 |
| 升级转化率 | 已升级 / 总记录 | >20% |
| 重复错误率 | 同一 ERR 出现次数 | 逐步降低 |
| Skill 提取数 | 从 learnings 提取的 Skill | 按需 |

### 6.2 .stats 文件格式（可选）

```json
{
  "agent": "lambda",
  "created": "2026-03-18",
  "total_learnings": 0,
  "total_errors": 0,
  "total_upgrades": 0,
  "last_review": null,
  "last_learning": null,
  "last_error": null
}
```

---

## 七、最佳实践

### 7.1 什么值得记录？

**✅ 记录：**
- 用户纠正了你的理解
- 工具调用失败及解决方法
- 发现的系统性规律
- 项目特定约定
- 之前不知道的 API/功能

**❌ 不记录：**
- 一次性的临时信息
- 已经在 SOUL.md/AGENTS.md 中的规则
- 明显的常识
- 临时的调试信息

### 7.2 记录质量标准

- **简洁** — Summary 一句话说清楚
- **可操作** — Suggested Action 明确具体
- **可搜索** — 用关键词，不用模糊描述
- **有上下文** — Details 说明来龙去脉

### 7.3 定期 Review

| 频率 | 操作 |
|------|------|
| 每次心跳 | 检查是否有新学习值得记录 |
| 每周 | Review 活跃记录，标记满足升级条件的 |
| 每月 | 清理过时记录，升级成熟规则 |

---

## 八、与现有系统的分工

| 系统 | 用途 | 维护方式 |
|------|------|---------|
| `.learnings/` | 零散学习、错误、需求 | Agent 主动记录 |
| `memory/YYYY-MM-DD.md` | 每日事件和上下文 | 每日/心跳更新 |
| `MEMORY.md` | 长期重要记忆 | 手动维护 |
| `SOUL.md` | 行为准则 | 升级链写入 |
| `AGENTS.md` | 工作流规则 | 升级链写入 |
| `TOOLS.md` | 工具细节 | 升级链写入 |
| `SELF-IMPROVEMENT.md` | 错误纠正机制 | Agent 自主维护 |

**信息流：**
```
日常经验 → .learnings/ → 升级链 → SOUL/AGENTS/TOOLS.md
                ↓
          memory/ 日记（记录事件）
                ↓
          MEMORY.md（精炼长期记忆）
```

---

*本方案由拉姆达 🔬 编写，基于 self-improving-agent 项目适配乐园环境。*
*Phase 1 可立即实施，零成本零风险。*
