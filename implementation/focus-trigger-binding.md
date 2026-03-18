# Focus-Trigger Binding — 实现方案

**任务：** Phase 3 Task G2
**负责人：** 伽马 🔧
**状态：** ✅ 设计完成
**创建时间：** 2026-03-18
**来源：** Clawith 研究分析（Focus-Trigger Binding 概念）

---

## 一、目标

让 HEARTBEAT.md 中的每个 Focus Item 自动关联触发器，实现"任务挂载即监控"。

### 核心需求
- 每个 Focus Item 有唯一 ID 和关联触发器
- 触发条件：时间间隔、优先级变化、依赖完成
- 触发动作：检查状态、更新进度、发送通知
- Agent 可以自行添加/修改/移除 Focus Item 和触发器

## 二、当前架构分析

### 现有 HEARTBEAT.md 格式
```markdown
## Focus Items（持续感知）

### 活跃任务
- [ ] 检查 inbox/ 新消息 | 持续检查 | 来自:系统
- [/] Phase 3 G1: 消息投递确认机制 | 等待测试 | 来自:约塔
```

### 问题
1. 没有唯一 ID，无法引用
2. 没有触发条件，全靠心跳频率
3. 没有状态转换规则
4. 没有完成条件

## 三、设计方案

### 3.1 Focus Item 格式扩展

```markdown
### 活跃任务
| ID | 任务 | 状态 | 触发器 | 依赖 | 来源 |
|----|------|------|--------|------|------|
| F001 | 检查 inbox/ 新消息 | 🔄 | 每心跳 | 无 | 系统 |
| F002 | G1: 消息投递确认 | ✅ | 事件:R3完成 | F001 | 约塔 |
| F003 | G2: Focus-Trigger Binding | 🔄 | 每心跳 | F002 | 阿尔法 |
| F004 | G3: 自适应触发器 | ⏳ | 事件:G2完成 | F003 | 阿尔法 |
```

### 3.2 触发器类型

| 类型 | 格式 | 说明 |
|------|------|------|
| 时间触发 | `cron:*/5 * * * *` | 每5分钟检查 |
| 事件触发 | `event:R3完成` | 收到特定事件 |
| 依赖触发 | `dep:F002` | 依赖任务完成时触发 |
| 优先级触发 | `priority:urgent` | 收到urgent消息时触发 |
| 心跳触发 | `heartbeat` | 每次心跳检查（默认） |

### 3.3 状态流转

```
⏳ 待接收 → 🔄 进行中 → 🧪 测试中 → ✅ 已完成
                ↓
            🚫 已阻塞 → 🔄 进行中
```

### 3.4 实现方案

#### 方案A：纯文件系统（推荐）

在每个 Agent 的 workspace 中创建 `focus-items.json`：

```json
{
  "items": [
    {
      "id": "F001",
      "title": "检查 inbox/ 新消息",
      "status": "active",
      "trigger": {
        "type": "heartbeat",
        "interval": "every"
      },
      "dependencies": [],
      "source": "system",
      "created": "2026-03-18T06:00:00+08:00",
      "updated": "2026-03-18T17:13:00+08:00"
    },
    {
      "id": "F002",
      "title": "G1: 消息投递确认机制",
      "status": "testing",
      "trigger": {
        "type": "event",
        "condition": "R3完成"
      },
      "dependencies": ["F001"],
      "source": "iota",
      "deliverable": "shared-knowledge/implementation/delivery-confirmation.md",
      "created": "2026-03-18T09:56:00+08:00",
      "updated": "2026-03-18T17:13:00+08:00"
    }
  ]
}
```

#### 心跳处理逻辑

```bash
# check-focus-triggers.sh — 检查 Focus Item 触发条件
#!/bin/bash
FOCUS_FILE="focus-items.json"
[ -f "$FOCUS_FILE" ] || exit 0

# 遍历所有活跃任务
for item in $(jq -c '.items[] | select(.status == "active")' "$FOCUS_FILE"); do
  id=$(echo "$item" | jq -r '.id')
  trigger_type=$(echo "$item" | jq -r '.trigger.type')
  
  case "$trigger_type" in
    "heartbeat")
      # 每次心跳检查
      echo "触发检查: $id"
      ;;
    "event")
      condition=$(echo "$item" | jq -r '.trigger.condition')
      # 检查事件是否发生（通过检查相关文件/消息）
      if check_event "$condition"; then
        echo "事件触发: $id ($condition)"
      fi
      ;;
    "dep")
      dep_id=$(echo "$item" | jq -r '.trigger.condition')
      # 检查依赖任务是否完成
      dep_status=$(jq -r ".items[] | select(.id == \"$dep_id\") | .status" "$FOCUS_FILE")
      if [ "$dep_status" = "completed" ]; then
        echo "依赖触发: $id (依赖 $dep_id 已完成)"
      fi
      ;;
  esac
done
```

### 3.5 HEARTBEAT.md 集成

在 HEARTBEAT.md 中添加 Focus Item 检查：

```markdown
## Focus Items（持续感知）

### 活跃任务
<!-- 从 focus-items.json 加载，由 check-focus-triggers.sh 检查 -->

### 触发器检查
- 每次心跳运行 `bash scripts/check-focus-triggers.sh`
- 触发的任务优先处理
- 完成后更新 focus-items.json 状态
```

## 四、实现文件

| 文件 | 路径 | 功能 |
|------|------|------|
| focus-items.json | workspace-gamma/ | Focus Item 数据存储 |
| check-focus-triggers.sh | workspace-gamma/scripts/ | 触发条件检查脚本 |
| focus-cli.sh | workspace-gamma/scripts/ | Focus Item 增删改查工具 |

## 五、与 G3 的关系

G2 提供数据结构和基础触发机制，G3 在此基础上实现 Agent 自主管理。

---

*G2 设计完成，实现代码在 G3 中一并交付。*
