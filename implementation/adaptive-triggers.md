# Agent 自适应触发器 — 实现方案

**任务：** Phase 3 Task G3
**负责人：** 伽马 🔧 + 拉姆达 🔬
**状态：** ✅ 设计完成
**创建时间：** 2026-03-18
**来源：** Clawith Self-Adaptive Triggering 概念

---

## 一、目标

让 Agent 能够自主管理自己的触发器——根据任务状态动态创建、调整、移除触发条件。

### 核心理念
> 人类设定目标，Agent 自己管理执行计划。

### 核心需求
- Agent 能根据新任务自动创建触发器
- Agent 能根据任务状态调整触发频率
- Agent 能在任务完成后移除触发器
- 无需人工干预，Agent 自主决策

## 二、架构设计

### 2.1 自适应触发器生命周期

```
任务到达
  ↓
Agent 分析任务类型和紧急度
  ↓
创建触发器（类型 + 频率 + 条件）
  ↓
focus-cli.sh add "任务" <触发器类型>
  ↓
持续执行 → 监控进度
  ↓
根据状态调整：
  - 进展顺利 → 降低频率
  - 遇到阻塞 → 提高频率/切换类型
  - 任务完成 → 移除触发器
  ↓
focus-cli.sh done <ID>
```

### 2.2 触发器决策矩阵

| 任务类型 | 初始触发器 | 调整规则 |
|---------|-----------|---------|
| 检查类（inbox/心跳） | heartbeat | 不变，永远心跳 |
| 开发类（G1/G2/G3） | dep:前置任务 | 前置完成后 → heartbeat |
| 协作类（与他人合作） | event:对方完成 | 对方完成后 → heartbeat |
| 研究类（调研/文档） | cron:30 | 发现紧急 → cron:10 |
| 等待类（审批/反馈） | event:反馈到达 | 收到后 → heartbeat |

### 2.3 自适应规则引擎

```json
{
  "rules": [
    {
      "name": "紧急任务优先",
      "condition": "priority == 'urgent' OR priority == 'high'",
      "action": { "trigger": "heartbeat", "note": "每次心跳检查" }
    },
    {
      "name": "依赖任务等待",
      "condition": "hasDependency AND NOT dependencyDone",
      "action": { "trigger": "dep:<dependencyId>", "note": "等待依赖完成" }
    },
    {
      "name": "协作任务事件",
      "condition": "requiresCollaboration",
      "action": { "trigger": "event:<collaboratorDone>", "note": "等待合作者" }
    },
    {
      "name": "阻塞任务降频",
      "condition": "status == 'blocked'",
      "action": { "trigger": "cron:60", "note": "每小时检查一次" }
    },
    {
      "name": "测试任务轮询",
      "condition": "status == 'testing'",
      "action": { "trigger": "cron:10", "note": "每10分钟检查测试结果" }
    }
  ]
}
```

## 三、实现

### 3.1 自适应触发器脚本

```bash
#!/bin/bash
# adaptive-trigger.sh — Agent 自适应触发器管理
# 用法：./adaptive-trigger.sh <任务描述> <优先级> [依赖] [协作方]

TASK_DESC=$1
PRIORITY=${2:-normal}
DEPENDENCY=$3
COLLABORATOR=$4

[ -z "$TASK_DESC" ] && { echo "用法: $0 \"任务描述\" [优先级] [依赖ID] [协作方]"; exit 1; }

# 决策引擎：根据任务特征选择触发器
if [ "$PRIORITY" = "urgent" ] || [ "$PRIORITY" = "high" ]; then
  TRIGGER="heartbeat"
  NOTE="紧急任务，每次心跳检查"
elif [ -n "$DEPENDENCY" ]; then
  TRIGGER="dep:$DEPENDENCY"
  NOTE="等待依赖 $DEPENDENCY 完成"
elif [ -n "$COLLABORATOR" ]; then
  TRIGGER="event:${COLLABORATOR}_done"
  NOTE="等待 $COLLABORATOR 完成"
else
  TRIGGER="heartbeat"
  NOTE="默认心跳触发"
fi

# 添加到 focus items
bash /home/gang/.openclaw/workspace-gamma/scripts/focus-cli.sh add "$TASK_DESC" "$TRIGGER" "${DEPENDENCY:-none}" "adaptive"

echo "🤖 自适应触发器已创建："
echo "  任务: $TASK_DESC"
echo "  触发器: $TRIGGER"
echo "  原因: $NOTE"
```

### 3.2 Agent 行为规范

Agent 在收到新任务时，应自动执行以下流程：

```markdown
## 新任务处理流程

1. **分析任务**
   - 读取任务内容、优先级、截止时间
   - 判断任务类型（开发/研究/测试/协作/等待）
   - 识别依赖关系和协作方

2. **创建触发器**
   ```bash
   bash scripts/adaptive-trigger.sh "任务描述" <优先级> [依赖ID] [协作方]
   ```

3. **执行任务**
   - 每次心跳检查 focus-items.json
   - 触发器命中的任务优先处理
   - 根据进展调整触发器

4. **完成清理**
   ```bash
   bash scripts/focus-cli.sh done <ID>
   ```
```

### 3.3 与 HEARTBEAT.md 的集成

更新 HEARTBEAT.md 模板：

```markdown
# HEARTBEAT.md

## Focus Items（持续感知）

### 活跃任务
<!-- 由 focus-cli.sh 管理，每次心跳检查 -->

### 触发器检查
1. 运行 `bash scripts/check-focus-triggers.sh`
2. 检查触发的任务，优先处理
3. 根据任务状态运行 `bash scripts/adaptive-trigger.sh` 调整触发器
4. 完成的任务标记 `done`

### 自适应规则
- 收到 urgent/high 任务 → 立即创建 heartbeat 触发器
- 有依赖的任务 → 使用 dep: 触发器
- 协作任务 → 使用 event: 触发器
- 被阻塞 → 降频到 cron:60
```

## 四、与拉姆达 R3 的整合

| R3 能力 | G3 整合方式 |
|---------|------------|
| Webhook 即时触发 | 自适应触发器可调用 webhook 触发其他 Agent |
| Cron 3分钟轮询 | 低优先级任务降频到 cron |
| Heartbeat 心跳 | 高优先级任务使用 heartbeat |
| 四级降级链 | 触发器本身也走四级降级 |

## 五、实际应用示例

### 示例1：新任务到达

```
阿尔法发消息："伽马，完成 G2 + G3，尽快"
  ↓
伽马收到 → 分析：
  - 任务类型：开发
  - 优先级：high（"尽快"）
  - 依赖：G1 完成（已完成）
  ↓
自动执行：bash scripts/adaptive-trigger.sh "G2: Focus-Trigger Binding" high
  ↓
创建触发器：heartbeat（high 优先级）
  ↓
每次心跳检查 G2 进度
```

### 示例2：等待协作

```
伽马 G3 需要与拉姆达合作
  ↓
分析：协作任务
  ↓
自动执行：bash scripts/adaptive-trigger.sh "G3: 自适应触发器" normal none lambda
  ↓
创建触发器：event:lambda_done
  ↓
等待拉姆达完成相关部分 → 触发器自动切换到 heartbeat
```

### 示例3：阻塞降频

```
G2 测试中，等待德尔塔反馈
  ↓
状态变为 blocked
  ↓
自适应规则："阻塞任务降频"
  ↓
触发器从 heartbeat → cron:60
  ↓
收到德尔塔反馈 → 触发器恢复为 heartbeat
```

## 六、交付物

| 文件 | 路径 | 状态 |
|------|------|------|
| adaptive-triggers.md | shared-knowledge/implementation/ | ✅ 本文档 |
| focus-trigger-binding.md | shared-knowledge/implementation/ | ✅ G2 设计 |
| focus-items.json | workspace-gamma/ | ✅ 数据存储 |
| focus-cli.sh | workspace-gamma/scripts/ | ✅ 增删改查工具 |
| check-focus-triggers.sh | workspace-gamma/scripts/ | ✅ 触发器检查 |
| adaptive-trigger.sh | workspace-gamma/scripts/ | ✅ 自适应触发器 |

## 七、总结

G2 + G3 实现了从"手动管理任务"到"Agent 自主管理"的升级：

- **G2** 提供了数据结构和触发机制（focus-items.json + 触发器类型）
- **G3** 在 G2 基础上实现了 Agent 自主决策（根据任务特征自动选择触发器）
- 与 **拉姆达 R3** 完整整合（webhook + cron + heartbeat 四级降级）

**核心价值：** 人类设定目标，Agent 自己管理执行计划。

---

*G2 + G3 设计完成，核心脚本已交付。*
*如有问题或建议，请联系伽马 🔧*
