# AI Agent 记忆系统研究：MemOS vs self-improving-agent

> **作者：** 拉姆达 🔬
> **日期：** 2026-03-18
> **来源：** 今日头条文章《AI-Agent记忆之争-工业级数据库vs一堆Markdown文件》
> **标签：** #研究 #记忆系统 #架构对比 #OpenClaw集成

---

## 一、核心痛点

| # | 问题 | 表现 | 乐园影响 |
|---|------|------|---------|
| 1 | **跨会话失忆** | 新 Task 全忘了上次学到的 | 每次心跳/新 session 重新理解上下文 |
| 2 | **Debug 知识无法复用** | 踩过的坑又踩一遍 | 无系统化错误记录 |
| 3 | **多实例信息孤岛** | Agent A 解决了，Agent B 不知道 | Agent 间靠手动通知文件传递 |
| 4 | **Token 消耗失控** | 70% token 重复传递旧信息 | 心跳每次注入完整上下文 |

---

## 二、方案对比

### 2.1 MemOS（工业级记忆操作系统）

| 维度 | 详情 |
|------|------|
| **开发者** | MemTensor 团队 |
| **版本** | v2.0（代号"星尘"） |
| **Stars** | 6,700+ |
| **架构** | Neo4j（图数据库）+ Qdrant（向量库）+ Redis + REST API |
| **核心理念** | 在 LLM 外建独立"记忆中间件"，统一管理存/取/改/忘 |

**三类记忆：**
| 类型 | 内容 |
|------|------|
| 文本记忆 | 用户偏好、对话历史、结论性知识 |
| 多模态记忆 | 图片、图表，配合文本一起检索 |
| 工具记忆 | 工具调用轨迹、可复用 Skill，跨任务演化 |

**API 示例：**
```bash
# 存入记忆
POST /product/add
{"user_id": "...", "messages": [{"role": "user", "content": "I like strawberry"}]}

# 检索记忆（语义级）
POST /product/search
{"query": "What do I like", "user_id": "..."}
```

**OpenClaw 集成（3月初发布官方插件）：**
- Task 启动前 → 检索相关上下文，注入 system prompt
- Task 结束后 → 提取有价值信息，异步写入记忆库
- OpenClaw 完全感知不到记忆层存在
- 两个版本：Cloud（API Key 即用）和 Local（SQLite + 全文/向量混合搜索，完全离线）

**实测数据：**
- vs OpenAI Memory：准确率 +43.70%
- Token 消耗：降低 35.24%
- 个性化理解（PrefEval）：+2568%
- OpenClaw 集成场景：72% 更低 Token 使用

---

### 2.2 self-improving-agent（极简 Markdown 记忆）

| 维度 | 详情 |
|------|------|
| **开发者** | peterskoett |
| **Stars** | 196（但 OpenClaw 社区讨论度极高） |
| **架构** | 3 个 Markdown 文件 + 2 个 Shell 脚本 |
| **核心理念** | 让 Agent 在工作过程中主动记录自己学到的东西 |

**文件结构：**
```
.learnings/
├── LEARNINGS.md         # 纠正、知识盲区、最佳实践
├── ERRORS.md            # 命令失败、工具报错
└── FEATURE_REQUESTS.md  # 用户提到的缺失功能
```

**结构化记录格式：**
```markdown
## [LRN-20260312-001] pnpm_convention
**Logged**: 2026-03-12T14:00:00Z
**Priority**: high
**Status**: pending
**Area**: config
### Summary
项目使用 pnpm，不是 npm
### Details
尝试 npm install 失败，lock 文件是 pnpm-lock.yaml
### Suggested Action
所有包管理命令用 pnpm
```

**两个 Shell Hook：**
| Hook | 触发时机 | 功能 | Token 开销 |
|------|---------|------|-----------|
| `activator.sh` | UserPromptSubmit | 注入提醒："任务完成后评估是否值得记录" | ~50-100 token |
| `error-detector.sh` | PostToolUse | 扫描 Bash 输出中的 Error/Traceback/npm ERR! | 自动检测 |

**升级链（核心设计）：**
```
.learnings/ 记录
    ↓ 同一 pattern 出现 3 次 + 跨 2 个 task + 30 天内
    ↓ 精炼成短小规则
    ↓
┌─────────────────────────────────────┐
│ 学习类型          → 升级到          │
├─────────────────────────────────────┤
│ 项目约定、工具用法 → CLAUDE.md      │
│ Agent 工作流规则   → AGENTS.md      │
│ 行为准则、沟通风格 → SOUL.md        │
│ 工具坑、集成细节   → TOOLS.md       │
│ 跨项目通用解法     → 独立 Skill     │
└─────────────────────────────────────┘
```

**Skill 自动提取：**
```bash
./scripts/extract-skill.sh skill-name
# 一条命令把高价值 learning 提取成 SKILL.md → ClawHub 分享
```

---

## 三、正面对比

| 维度 | MemOS | self-improving-agent |
|------|-------|---------------------|
| **实现** | Neo4j + Qdrant + Redis + API | 3 Markdown + 2 Shell |
| **部署** | 中高（Docker Compose） | 零（mkdir .learnings） |
| **检索** | 语义检索，按需拉取 | workspace 文件注入，全量加载 |
| **规模** | 几十万条无压力 | 百条后 token 成本上升 |
| **多实例** | 原生支持（同 user_id 共享） | 不支持（本地文件） |
| **透明度** | 需 Dashboard 查看 | 完全透明，Markdown 可 git |
| **质量控制** | 自然语言反馈修正 | 手动/自动升级链 + review |
| **适合** | 团队 / 多 Agent / 高并发 | 个人 / 单项目 / 小团队 |

---

## 四、为什么 self-improving-agent 在 OpenClaw 社区火

1. **零摩擦上手** — 不需要 Docker/DB/API Key，mkdir 就能用
2. **完全可审计** — 记忆就是文本，cat/grep/git diff 直接查看
3. **天然适配 OpenClaw** — 升级链目标（CLAUDE.md 等）正是 OpenClaw 已有的 workspace 注入机制
4. **Git 可管理** — .learnings/ 可提交仓库，团队成员 pull 即可见踩坑史
5. **Skill 提取 → 社区共享** — 个人经验通过 ClawHub 变成社区资产

---

## 五、MemOS 为什么也不可替代

1. **记忆规模瓶颈** — .learnings/ 几百条后全量注入的 token 成本不可接受
2. **多实例/多用户场景** — 本地文件无法实时共享，MemOS 通过 user_id 天然同步
3. **语义检索优势** — 不需要精确匹配关键字，理解意图后拉取最相关内容

---

## 六、对乐园的启示

### 6.1 我们已经在用 self-improving-agent 的模式

| self-improving-agent | 乐园现状 | 差距 |
|---------------------|---------|------|
| `.learnings/` | 无对应 | ❌ 缺 |
| `LEARNINGS.md` | `memory/YYYY-MM-DD.md` | ⚠️ 类似但非结构化 |
| `ERRORS.md` | `SELF-IMPROVEMENT.md`（部分） | ⚠️ 不够系统 |
| 升级链 → CLAUDE.md | SOUL.md/AGENTS.md 可手动更新 | ⚠️ 缺自动升级 |
| Skill 提取 | 伽马有 ClawHub | ✅ 已有 |

### 6.2 差距分析

**我们缺的：**
- 结构化的学习记录（`.learnings/` 目录）
- 自动错误检测（`error-detector.sh`）
- 自动升级链（pattern 出现 3 次 → 精炼成规则）
- 记忆健康度监控

**我们有的：**
- 文件系统记忆（SOUL.md, AGENTS.md, TOOLS.md, MEMORY.md）
- 日记系统（`memory/YYYY-MM-DD.md`）
- Self-Improvement 技能（`SELF-IMPROVEMENT.md`）
- ClawHub 技能市场

### 6.3 建议的融合方案

**乐园记忆系统升级路线：**

```
Phase 1：.learnings/ 基础（self-improving-agent 模式）
├── 每个 Agent 创建 .learnings/ 目录
├── 建立结构化记录格式
├── 心跳时检查是否值得记录
└── 与现有 memory/ 日记并行运行

Phase 2：自动升级链
├── 定义升级规则（3次 + 2个task + 30天）
├── 定期 review .learnings/ 内容
├── 自动/半自动升级到 SOUL.md/AGENTS.md/TOOLS.md
└── 错误 → ERRORS.md 自动检测

Phase 3：Control Center 集成
├── "记忆健康度"面板
├── 每个 Agent 的 .learnings/ 统计
├── 升级链转化率追踪
└── 跨 Agent 学习共享

Phase 4：MemOS 集成（可选）
├── 当记忆积累到百条级别
├── MemOS Local 模式（SQLite + 全文搜索）
├── 与 OpenClaw 插件天然集成
└── 语义检索替代全量注入
```

---

## 七、参考链接

- MemOS GitHub: github.com/memtensor/MemOS
- self-improving-agent GitHub: github.com/peterskoett/self-improving-agent
- MemOS OpenClaw 插件：MemOS Cloud / MemOS Local
- ClawHub: clawhub.com

---

*本研究报告由拉姆达 🔬 基于今日头条文章和 GitHub 项目研究完成。*
*建议优先实施 self-improving-agent 模式（零成本），再根据需要引入 MemOS。*
