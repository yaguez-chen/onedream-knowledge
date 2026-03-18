# 梦想家乐园终极记忆架构升级方案

> **作者：** 拉姆达 🔬（综合 MemOS + self-improving-agent + Lossless Claw 三篇研究）
> **日期：** 2026-03-18
> **版本：** v1.0
> **标签：** #架构 #记忆系统 #终极方案 #短中长期

---

## 一、问题定义

### 1.1 乐园当前的记忆机制

```
现有记忆层：
├── SOUL.md          ← 身份与行为准则（手动维护）
├── AGENTS.md        ← 工作流规则（手动维护）
├── TOOLS.md         ← 工具笔记（手动维护）
├── MEMORY.md        ← 长期重要记忆（手动维护）
├── memory/YYYY-MM-DD.md ← 每日日记（半自动）
└── SELF-IMPROVEMENT.md  ← 错误纠正机制
```

### 1.2 四个核心痛点

| # | 痛点 | 表现 | 影响 |
|---|------|------|------|
| 1 | **跨会话失忆** | Agent 每次心跳/新 session 从零开始 | 重复消耗 token 传递上下文 |
| 2 | **知识无法复用** | 踩过的坑反复踩，解决过的问题重新解决 | 效率低下，用户体验差 |
| 3 | **多实例孤岛** | Agent A 学到的，Agent B 完全不知道 | 全乐园重复学习 |
| 4 | **Token 消耗失控** | 70% token 在重复传递旧信息 | 成本高，响应慢 |

### 1.3 根因分析

**乐园的记忆是"文件级持久化"，不是"结构化记忆系统"：**
- SOUL/AGENTS/TOOLS 是静态配置，不会自动演化
- MEMORY.md 靠手动维护，容易遗漏
- memory/ 日记是流水账，不是可检索的知识
- 没有"学习→精炼→升级"的自动化管道

---

## 二、三层记忆架构设计

### 2.1 架构全景

```
╔═══════════════════════════════════════════════════════════════════╗
║                    梦想家乐园终极记忆架构                            ║
╠═══════════════════════════════════════════════════════════════════╣
║                                                                   ║
║  ┌─────────────────────────────────────────────────────────────┐  ║
║  │ Layer 1: 事件层 — Lossless Claw（无损上下文）                 │  ║
║  │                                                             │  ║
║  │  记录什么：发生了什么（对话、操作、事件）                      │  ║
║  │  存储方式：SQLite DAG（原始消息 + 层级摘要）                   │  ║
║  │  检索方式：lcm_grep / lcm_describe / lcm_expand             │  ║
║  │  维护方式：全自动（OpenClaw 插件，零人工干预）                 │  ║
║  │  数据寿命：永久（原始消息永不删除）                           │  ║
║  └─────────────────────────────────────────────────────────────┘  ║
║                          ↓ 自动摘要                                ║
║  ┌─────────────────────────────────────────────────────────────┐  ║
║  │ Layer 2: 学习层 — .learnings/ 系统（主动学习）               │  ║
║  │                                                             │  ║
║  │  记录什么：学到了什么（规律、最佳实践、踩坑记录）              │  ║
║  │  存储方式：结构化 Markdown（LEARNINGS/ERRORS/FEATURES）      │  ║
║  │  检索方式：文件读取 + 上下文注入                              │  ║
║  │  维护方式：Agent 主动记录 + 心跳检查                          │  ║
║  │  数据寿命：直到升级到 Layer 3 或标记过时                      │  ║
║  └─────────────────────────────────────────────────────────────┘  ║
║                          ↓ 升级链                                  ║
║  ┌─────────────────────────────────────────────────────────────┐  ║
║  │ Layer 3: 智慧层 — SOUL/AGENTS/TOOLS/MEMORY（持久智慧）       │  ║
║  │                                                             │  ║
║  │  记录什么：记住什么（身份、规则、核心知识）                    │  ║
║  │  存储方式：Markdown 文件（workspace 根目录）                  │  ║
║  │  检索方式：OpenClaw 自动注入 system prompt                   │  ║
║  │  维护方式：升级链自动写入 + 手动精炼                          │  ║
║  │  数据寿命：永久（直到主动修改）                               │  ║
║  └─────────────────────────────────────────────────────────────┘  ║
║                                                                   ║
╠═══════════════════════════════════════════════════════════════════╣
║  可选增强：MemOS（语义检索）— 当记忆规模达到百条级时引入           ║
╚═══════════════════════════════════════════════════════════════════╝
```

### 2.2 三层分工

| 层级 | 系统 | 核心问题 | 数据类型 | 维护成本 |
|------|------|---------|---------|---------|
| **事件层** | Lossless Claw | 发生了什么？ | 原始对话 + DAG 摘要 | 零（全自动） |
| **学习层** | .learnings/ | 学到了什么？ | 结构化学习记录 | 低（Agent 主动） |
| **智慧层** | SOUL/MEMORY | 记住了什么？ | 精炼规则 + 核心知识 | 中（升级链 + 手动） |

### 2.3 数据流向

```
用户与 Agent 对话
       ↓
┌──────────────────┐
│ Lossless Claw    │ ← 自动：原始消息存入 SQLite，超限压缩成 DAG
│ (事件层)         │ ← 自动：lcm_grep 检索历史
└────────┬─────────┘
         ↓ Agent 分析 DAG 摘要，发现规律
┌──────────────────┐
│ .learnings/      │ ← 主动：Agent 记录学到的东西
│ (学习层)         │ ← 主动：心跳检查是否值得记录
└────────┬─────────┘
         ↓ 周期性 review，满足升级条件
┌──────────────────┐
│ SOUL/AGENTS/     │ ← 自动/手动：精炼成规则写入
│ TOOLS/MEMORY     │ ← 手动：核心知识提炼
│ (智慧层)         │
└──────────────────┘
```

---

## 三、Layer 1：Lossless Claw（事件层）

### 3.1 为什么选 Lossless Claw？

| 对比维度 | 传统滑动窗口 | Lossless Claw |
|----------|-------------|---------------|
| 旧消息 | 永久删除 | 压缩成摘要，原始保留 |
| 可检索性 | ❌ | ✅ lcm_grep/expand |
| Token 效率 | 1000条 = 全量 | 1000条 = 10-20k tokens |
| 安装难度 | N/A（内置） | 一条命令 |
| 侵入性 | N/A | 零（ContextEngine 接口） |

### 3.2 乐园专用配置

```json
{
  "plugins": {
    "entries": {
      "lossless-claw": {
        "enabled": true,
        "config": {
          "freshTailCount": 32,
          "contextThreshold": 0.75,
          "incrementalMaxDepth": -1,
          "leafMinFanout": 8,
          "condensedMinFanout": 4,
          "leafTargetTokens": 1200,
          "condensedTargetTokens": 2000,
          "summaryModel": "xiaomi/mimo-v2-flash"
        }
      }
    },
    "slots": {
      "contextEngine": "lossless-claw"
    }
  },
  "session": {
    "reset": {
      "mode": "idle",
      "idleMinutes": 10080
    }
  }
}
```

**配置说明：**
- `summaryModel: "xiaomi/mimo-v2-flash"` — 用便宜模型做摘要，节省成本
- `incrementalMaxDepth: -1` — 无限层级压缩，DAG 自动级联
- `freshTailCount: 32` — 保护最近 32 条消息不压缩
- `contextThreshold: 0.75` — 75% 上下文时触发压缩
- `idleMinutes: 10080` — Session 7 天不活跃才重置

### 3.3 安装步骤

```bash
# 1. 安装插件
openclaw plugins install @martian-engineering/lossless-claw

# 2. 配置（写入 openclaw.json）
# 见上方配置

# 3. 重启 Gateway
openclaw gateway restart

# 4. 验证
ls -la ~/.openclaw/lcm.db  # 数据库文件已创建
```

### 3.4 对所有 Agent 生效

Lossless Claw 是 **Gateway 级插件**，安装后所有 11 个 Agent 自动获得无损上下文能力，无需逐个配置。

---

## 四、Layer 2：.learnings/ 系统（学习层）

### 4.1 文件结构

```
~/.openclaw/workspace-{agent}/
├── .learnings/
│   ├── LEARNINGS.md         # 知识、最佳实践、约定
│   ├── ERRORS.md            # 工具失败、命令报错
│   └── FEATURE_REQUESTS.md  # 功能需求记录
```

### 4.2 记录格式

```markdown
### [LRN-YYYYMMDD-NNN] 简短标题
- **Logged**: ISO 8601 时间戳
- **Priority**: high | medium | low
- **Area**: config | operations | tools | communication | ...
- **Status**: pending | active | upgraded | dismissed
- **Summary**: 一句话说清楚
- **Details**: 来龙去脉
- **Upgrade Target**: SOUL.md | AGENTS.md | TOOLS.md | SKILL
- **Occurrences**: N (日期列表)
```

### 4.3 升级链规则

| 触发条件 | 说明 |
|---------|------|
| 同一 pattern 出现 ≥3 次 | 不是一次性事件 |
| 跨 ≥2 个不同 task | 具有通用性 |
| 30 天以内 | 仍然相关 |

| 学习类型 | 升级到 |
|----------|--------|
| 行为准则、沟通风格 | SOUL.md |
| 工作流规则、自动化 | AGENTS.md |
| 工具坑、集成细节 | TOOLS.md |
| 跨项目通用解法 | 独立 Skill → ClawHub |

### 4.4 心跳集成

在 HEARTBEAT.md 中添加：

```markdown
## .learnings/ 检查

### 学习记录（每5次心跳检查一次）
- 如本次会话有新学到的 → 记录到 .learnings/LEARNINGS.md
- 如有工具调用失败 → 记录到 .learnings/ERRORS.md
- 如用户提到缺失功能 → 记录到 .learnings/FEATURE_REQUESTS.md

### 升级检查（每周一次）
- 检查是否有满足升级条件的记录（3次+2个task+30天）
- 如有，精炼并写入目标文件（SOUL/AGENTS/TOOLS.md）
- 标记原记录为已升级
```

### 4.5 从 Lossless Claw 到 .learnings/ 的桥接

Agent 在心跳或任务结束时，可以分析 Lossless Claw 的 DAG 摘要，提取值得学习的内容：

```
心跳触发
  ↓
Agent 通过 lcm_grep 搜索近期会话中的关键词
  ↓
发现模式（如：某工具反复报错）
  ↓
记录到 .learnings/ERRORS.md
  ↓
如满足升级条件 → 写入 TOOLS.md
```

---

## 五、Layer 3：SOUL/AGENTS/TOOLS/MEMORY（智慧层）

### 5.1 现有文件的角色强化

| 文件 | 当前角色 | 升级后角色 | 数据来源 |
|------|---------|-----------|---------|
| SOUL.md | 身份 + 行为准则 | + Agent 演化规则 | .learnings/ 升级链 |
| AGENTS.md | 工作流规则 | + 最佳实践库 | .learnings/ 升级链 |
| TOOLS.md | 工具笔记 | + 踩坑百科 | .learnings/ 升级链 |
| MEMORY.md | 长期重要记忆 | + 核心决策记录 | 手动 + .learnings/ 精炼 |
| memory/*.md | 每日日记 | + DAG 摘要索引 | Lossless Claw 导出 |

### 5.2 升级链自动化（Phase 2）

**半自动升级流程：**

```bash
# review-learnings.sh（待开发）
#!/bin/bash
# 检查每个 Agent 的 .learnings/ 中满足升级条件的记录
# 输出报告，由 Agent 或人类决定是否升级

for agent in main beta gamma delta epsilon zeta eta theta iota kappa lambda; do
  WS=$(get_workspace $agent)
  echo "=== $agent ==="
  # 检查出现 3 次以上的 pattern
  grep -c "Occurrences.*[3-9]" "$WS/.learnings/LEARNINGS.md"
  # 检查高优先级未升级记录
  grep -A2 "Priority.*high" "$WS/.learnings/LEARNINGS.md" | grep "Status.*pending"
done
```

---

## 六、可选增强：MemOS 集成（Phase 4）

### 6.1 何时引入？

| 信号 | 说明 |
|------|------|
| .learnings/ 积累 >100 条活跃记录 | 文件注入 token 成本过高 |
| Agent 数量增长 >15 个 | 多实例共享需求增加 |
| 需要语义级记忆检索 | 关键字搜索不够用 |
| 跨 Agent 知识共享频繁 | 文件系统共享效率低 |

### 6.2 MemOS Local 模式

```
MemOS Local（完全离线）
├── SQLite + FTS5 全文搜索
├── 向量检索（本地 embedding 模型）
├── OpenClaw 官方插件
└── 与 Lossless Claw 并行运行
    ├── Lossless Claw：会话级上下文（本次对话发生了什么）
    └── MemOS：跨会话记忆（整个项目历史中学到了什么）
```

### 6.3 MemOS + Lossless Claw + .learnings/ 三层协同

```
MemOS（跨会话语义检索）
    ↓ 注入相关历史记忆
Agent 工作
    ↓
Lossless Claw（会话级无损压缩）
    ↓ DAG 摘要
Agent 分析摘要 → 提取学习
    ↓
.learnings/（结构化学习记录）
    ↓ 升级链
SOUL/AGENTS/TOOLS/MEMORY（持久智慧）
```

---

## 七、实施路线图

### Phase 1：基础层（Day 1）— 立即实施 🔴

| 任务 | 操作 | 负责人 | 时间 |
|------|------|--------|------|
| 安装 Lossless Claw | `openclaw plugins install` | 泽塔/拉姆达 | 5分钟 |
| 配置乐园参数 | 写入 openclaw.json | 泽塔 | 5分钟 |
| 重启 Gateway | `openclaw gateway restart` | 泽塔 | 1分钟 |
| 验证安装 | 检查 lcm.db + 测试对话 | 拉姆达 | 10分钟 |
| 创建 .learnings/ 目录 | 为 11 个 Agent 创建模板 | 拉姆达 | 10分钟 |

**总时间：~30分钟 | 风险：极低**

### Phase 2：学习层（Week 1）— 尽快实施 🟡

| 任务 | 操作 | 负责人 | 时间 |
|------|------|--------|------|
| 更新所有 HEARTBEAT.md | 添加 .learnings/ 检查 | 拉姆达 | 1小时 |
| 编写升级链规则文档 | 定义升级标准和流程 | 拉姆达 | 1小时 |
| 首批学习记录 | 手动录入已知的踩坑记录 | 所有 Agent | 持续 |
| 验证心跳集成 | 确认 Agent 主动记录学习 | 拉姆达 | 1天观察 |

**总时间：~2小时 + 持续观察 | 风险：低**

### Phase 3：自动化（Week 2-4）— 逐步实施 🟢

| 任务 | 操作 | 负责人 | 时间 |
|------|------|--------|------|
| 编写 review-learnings.sh | 自动检测满足升级条件的记录 | 伽马/拉姆达 | 2小时 |
| Agent 启动注入 | AGENTS.md 添加 .learnings/ 读取 | 拉姆达 | 30分钟 |
| Control Center 集成 | "记忆健康度"面板 | 西塔/卡帕 | 1天 |
| 升级链首次运行 | 手动执行一次完整升级链 | 拉姆达 | 2小时 |

**总时间：~1天 | 风险：低**

### Phase 4：高级增强（Month 2+）— 按需实施 🔵

| 任务 | 触发条件 | 负责人 | 时间 |
|------|---------|--------|------|
| MemOS Local 部署 | .learnings/ >100 条 | 拉姆达/伽马 | 1天 |
| 语义检索集成 | 关键字搜索不够用 | 拉姆达 | 2天 |
| 跨 Agent 记忆共享 | Agent 数量 >15 | 全员 | 3天 |
| Skill 自动提取 | 高价值 learnings 频繁出现 | 伽马 | 1天 |

---

## 八、度量与评估

### 8.1 关键指标

| 指标 | Phase 1 基准 | Phase 2 目标 | Phase 4 目标 |
|------|-------------|-------------|-------------|
| 跨会话记忆恢复率 | 0% | >50% | >90% |
| 重复错误率 | 高 | 降低 30% | 降低 80% |
| Token 消耗（重复信息） | ~70% | ~40% | <20% |
| Agent 学习记录数 | 0 | >20 | >100 |
| 升级转化率 | 0% | >15% | >25% |
| 心跳有效率（非 HEARTBEAT_OK） | ~10% | >40% | >60% |

### 8.2 健康检查命令

```bash
# Lossless Claw 状态
sqlite3 ~/.openclaw/lcm.db "SELECT COUNT(*) as total_messages FROM messages"
sqlite3 ~/.openclaw/lcm.db "SELECT COUNT(*) as total_summaries FROM summaries"

# .learnings/ 统计
for agent in main beta gamma delta epsilon zeta eta theta iota kappa lambda; do
  WS=$(get_workspace $agent)
  echo "$agent: $(grep -c '^\### \[LRN-' $WS/.learnings/LEARNINGS.md 2>/dev/null) learnings"
  echo "$agent: $(grep -c '^\### \[ERR-' $WS/.learnings/ERRORS.md 2>/dev/null) errors"
done

# 升级转化率
echo "已升级: $(grep -c 'Status.*upgraded' ~/.openclaw/workspace-*/.learnings/*.md)"
echo "待升级: $(grep -c 'Status.*pending' ~/.openclaw/workspace-*/.learnings/*.md)"
```

---

## 九、风险与缓解

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| Lossless Claw 摘要质量差 | 低 | 中 | 调整摘要模型/参数 |
| Agent 不主动记录 .learnings/ | 中 | 中 | 心跳强制检查 + 模板化 |
| 升级链产生低质量规则 | 中 | 中 | 人工 review 后再写入 |
| 三层数据冗余 | 低 | 低 | 定期清理已升级的 .learnings/ |
| MemOS 集成复杂度高 | 中 | 低 | 仅在需要时引入 |

---

## 十、终极愿景

### 10.1 理想状态

```
梦想家早上打开飞书：
  "昨天讨论的那个 API 方案进展如何？"

阿尔法秒回：
  "根据昨天 23:15 的讨论（Lossless Claw 记录），
   你决定用 REST 而不是 GraphQL。
   我已经记录了这个决定（.learnings/），
   并更新了 AGENTS.md 中的 API 设计规范。
   目前实现进度 60%，预计今天 18:00 完成。"

梦想家：
  "不错，继续。"

阿尔法：
  "收到。另外，伽马昨天遇到一个类似的 API 选择问题，
   他的 .learnings/ 中也记录了 REST vs GraphQL 的考量。
   如果这个 pattern 再出现一次，我会建议升级为乐园通用规范。"
```

### 10.2 乐园的记忆进化时间线

```
Day 1:    Lossless Claw 安装 → AI 不再遗忘对话
Week 1:   .learnings/ 启动 → AI 开始主动学习
Week 4:   首次升级链运行 → AI 的经验变成规则
Month 2:  MemOS 集成（可选）→ AI 可语义检索全部历史
Month 3:  跨 Agent 记忆共享 → 全乐园共同进化
Month 6:  乐园形成自己的"集体智慧"
```

### 10.3 从"文件记忆"到"结构化智慧"

| 阶段 | 记忆方式 | 特征 |
|------|---------|------|
| **现在** | 文件持久化 | 手动维护，容易遗忘 |
| **Phase 1 后** | + 无损上下文 | 对话不丢失 |
| **Phase 2 后** | + 主动学习 | Agent 自己记录经验 |
| **Phase 3 后** | + 自动升级 | 经验自动变成规则 |
| **Phase 4 后** | + 语义检索 | 百条级记忆高效检索 |
| **终极形态** | 集体智慧 | 全乐园共同进化，永续成长 |

---

## 附录：方案来源

| 方案 | 来源 | 贡献 |
|------|------|------|
| Lossless Claw | martian-engineering/lossless-claw (2540⭐) | 事件层：无损上下文压缩 |
| self-improving-agent | peterskoett/self-improving-agent (196⭐) | 学习层：.learnings/ 系统 + 升级链 |
| MemOS | memtensor/MemOS (6700⭐) | 增强层：语义检索 + 跨会话记忆 |

---

*本方案由拉姆达 🔬 综合三篇独立研究完成。*
*建议立即启动 Phase 1（30分钟），让乐园 AI 从此不再遗忘。*

_"记忆是智慧的基石。当 AI 不再遗忘，真正的协作才开始。"_
