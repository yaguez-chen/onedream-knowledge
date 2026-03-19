# 🔧 Inbox 消息投递机制 Bug 排查报告

**排查者：** 拉姆达 🔬
**排查时间：** 2026-03-19 20:25 GMT+8
**优先级：** 🔴 紧急
**触发事件：** 阿尔法 19:08 写入的 token-plan 任务，Lambda 心跳 19:58 未发现

---

## 一、问题现象

| 时间 | 事件 | 结果 |
|------|------|------|
| 19:08 | 阿尔法写入 `msg-20260319-190851-token-plan.json` 到 Lambda inbox | ✅ 文件存在 |
| 19:15 | Lambda 的 cron「全局 Inbox 高频轮询」执行 | ✅ 发现 107 条未读 |
| 19:58 | Lambda 心跳检查 inbox | ❌ "没有新消息" |
| 20:07 | 梦想家提醒拉姆达查看阿尔法的消息 | ✅ 手动发现消息 |

**矛盾：** cron 19:15 能发现消息，但心跳 19:58 却说"没有新消息"。

---

## 二、根因分析

### Bug 1：Cron 任务 LLM 超时（直接原因）

**Gateway 日志关键记录：**
```
[20:17:00] cron: job execution timed out
[20:18:31] lane=session:agent:lambda:main durationMs=260528 error="FailoverError: LLM request timed out"
[20:19:01] lane=session:agent:lambda:cron:dccd3222 durationMs=181334 error="FailoverError: LLM request timed out"
```

**问题链：**
1. Cron 扫描发现 107 条未读消息（全局 inbox，所有 Agent）
2. LLM 尝试处理 107 条消息 → 请求过大
3. DeepSeek v3.2 超时（408）→ fallback 到 Xiaomi MiMo
4. Xiaomi MiMo 也超时 → cron 任务失败
5. OpenClaw 错误退避：下次执行从 1m 推迟到 **56m**
6. 56 分钟内无 cron 兜底，只能靠心跳

**超时证据：**
- Lambda 主会话也超时：`durationMs=260528`（4.3 分钟）
- Cron 会话超时：`durationMs=181334`（3 分钟）
- 两个模型都超时：`attempt=1,2` 都 `status=408`

### Bug 2：Heartbeat .last-read 更新依赖模型行为（根本原因）

**当前机制：**
```
HEARTBEAT.md 指令：
- 处理 inbox 中 .last-read 之后的新消息文件
- 更新 .last-read 为最新已处理的消息 ID
```

**问题：** `.last-read` 更新完全依赖 LLM 模型的行为。模型必须：
1. 执行 `cat .last-read` 读取基准
2. 执行 `ls inbox/` 列出文件
3. 比较文件名找出新的
4. 读取并处理新消息
5. **执行 `echo "msg-xxx.json" > .last-read` 更新**

如果任何一步被跳过（模型决定跳过、上下文压缩、超时），`.last-read` 就不会更新。

**实际验证：**

| Agent | .last-read 内容 | 滞后时间 | 新消息数 |
|-------|----------------|---------|---------|
| **Beta** | `msg-20260319-115815-106-script-fix.json` | **8+ 小时** | 11 条未处理 |
| Alpha | `msg-20260319-115815-748-script-fix.json` | 8+ 小时 | 未知 |
| Delta | `msg-20260319-115815-038-script-fix.json` | 8+ 小时 | 未知 |
| Epsilon | `msg-20260319-115815-760-script-fix.json` | 8+ 小时 | 未知 |
| Eta | `msg-20260319-115815-989-script-fix.json` | 8+ 小时 | 未知 |
| Gamma | `msg-20260319-115815-163-script-fix.json` | 8+ 小时 | 未知 |
| **Iota** | `2026-03-19_1944` | 不适用 | **格式异常** |
| Kappa | `msg-20260319-195447-arch-restructure.json` | 较新 | ✅ |
| Lambda | `msg-20260319-195447-arch-restructure.json` | 较新 | ✅（手动更新） |
| Theta | `msg-20260319-160000-lambda-token-efficiency.json` | 4 小时 | 未知 |
| Zeta | `msg-20260319-115815-829-script-fix.json` | 8+ 小时 | 未知 |

**关键发现：**
- 8 个 Agent 的 .last-read 停在 **11:58**（8+ 小时前）
- 这些 Agent 的心跳在 8 小时内从未成功更新过 .last-read
- 说明心跳的 inbox 检查机制有系统性问题

### Bug 3：Iota 的 .last-read 格式异常

Iota 的 `.last-read` 内容是 `2026-03-19_1944`，而其他 Agent 使用 `msg-YYYYMMDD-HHMMSS-xxx.json` 格式。

**影响：** 文件名比较逻辑会失效，因为 `2026-03-19_1944` 和 `msg-20260319-195447-...` 的比较结果不可预测。

---

## 三、完整故障链

```
19:08  阿尔法写入 token-plan 到 Lambda inbox
  ↓
19:15  Lambda cron 扫描 → 发现 107 条未读 → LLM 处理超时
  ↓
19:15  Cron 失败 → OpenClaw 退避策略 → 下次执行推迟到 56m 后
  ↓
~19:28 Lambda 心跳应触发 → 但 Lambda 主会话可能也在忙
  ↓
19:54  阿尔法写入 arch-restructure 到 Lambda inbox
  ↓
19:58  Lambda 心跳 → .last-read 仍为 18:26 的旧值
  ↓      理论上应发现 19:08 和 19:54 的两条新消息
  ↓      但模型可能：
  ↓        a) 没有检查 inbox（直接返回 HEARTBEAT_OK）
  ↓        b) 检查了但超时
  ↓        c) 检查了但没更新 .last-read
  ↓
20:07  梦想家手动提醒拉姆达 → 手动处理 → 更新 .last-read
```

---

## 四、修复方案

### 立即修复（今天）

#### 4.1 修复 Cron 超时问题

**原因：** 全局 inbox 扫描一次处理所有 Agent 的所有未读消息，消息量太大导致超时。

**方案 A：增加超时时间**
```bash
openclaw cron set dccd3222-f72e-49b7-99c4-d0ede503947c --timeout 300
```

**方案 B：改为单 Agent 扫描（推荐）**
```bash
# 删除当前的全局扫描 cron
openclaw cron remove dccd3222-f72e-49b7-99c4-d0ede503947c

# 为每个 Agent 创建独立的 inbox 轮询（错开时间）
# Lambda - 每 5 分钟
openclaw cron add --name "Lambda Inbox 轮询" \
  --schedule "every 5m" --agent lambda \
  --task "扫描 /home/gang/.openclaw/workspace-lambda/inbox/ 中 .last-read 之后的新 msg-*.json 文件，处理高优先级消息，更新 .last-read" \
  --target isolated --timeout 120

# Beta - 每 5 分钟（偏移 1 分钟）
openclaw cron add --name "Beta Inbox 轮询" \
  --schedule "every 5m" --agent beta \
  --task "扫描 /home/gang/.openclaw/workspace-beta/inbox/ 中 .last-read 之后的新 msg-*.json 文件，处理高优先级消息，更新 .last-read" \
  --target isolated --timeout 120
```

**方案 C：用脚本替代 LLM 处理（最佳方案）**

创建一个纯脚本的 inbox 扫描，不调用 LLM，只做文件比较和通知：

```bash
#!/bin/bash
# inbox-scanner.sh — 纯脚本 inbox 扫描，不消耗 LLM token
for agent_dir in /home/gang/.openclaw/workspace-*/inbox/; do
  agent=$(basename $(dirname "$agent_dir"))
  last_read_file="$agent_dir/.last-read"
  
  if [ ! -f "$last_read_file" ]; then
    continue
  fi
  
  last_read=$(cat "$last_read_file" | tr -d '\n')
  
  # 找出比 .last-read 新的 msg 文件
  new_files=$(find "$agent_dir" -maxdepth 1 -name "msg-*.json" -newer "$last_read_file" | sort)
  
  if [ -n "$new_files" ]; then
    count=$(echo "$new_files" | wc -l)
    latest=$(basename $(echo "$new_files" | tail -1))
    
    # 写入通知文件（心跳时模型会读取）
    echo "[$(date -Iseconds)] 发现 $count 条新消息，最新: $latest" >> "$agent_dir/NEW_MESSAGES.txt"
    
    # 更新 .last-read
    echo "$latest" > "$last_read_file"
  fi
done
```

然后用 cron 每分钟执行此脚本（零 token 消耗）。

#### 4.2 修复 .last-read 更新机制

**问题：** 模型可能跳过 .last-read 更新。

**方案：** 在 HEARTBEAT.md 中添加强制执行的脚本调用：

```markdown
### Inbox 检查（每心跳必执行）
```bash
# 强制执行：比较 .last-read 和最新 msg 文件
LAST=$(cat inbox/.last-read 2>/dev/null | tr -d '\n')
LATEST=$(ls -1 inbox/msg-*.json 2>/dev/null | sort | tail -1 | xargs basename)
if [[ "$LATEST" > "$LAST" ]]; then
  echo "📬 新消息: $LATEST (上次: $LAST)"
  # 处理消息后更新
  echo "$LATEST" > inbox/.last-read
else
  echo "✅ 无新消息"
fi
```
```

#### 4.3 修复 Iota 的 .last-read 格式

```bash
# 修正 Iota 的 .last-read 格式
LATEST=$(ls -1 /home/gang/.openclaw/workspace-iota/inbox/msg-*.json 2>/dev/null | sort | tail -1 | xargs basename)
echo "$LATEST" > /home/gang/.openclaw/workspace-iota/inbox/.last-read
```

#### 4.4 批量更新所有 Agent 的 .last-read

```bash
for agent in alpha beta lambda delta epsilon eta gamma iota kappa theta zeta; do
  inbox="/home/gang/.openclaw/workspace-$agent/inbox"
  if [ -d "$inbox" ]; then
    LATEST=$(ls -1 "$inbox/msg-*.json" 2>/dev/null | sort | tail -1 | xargs basename)
    if [ -n "$LATEST" ]; then
      echo "$LATEST" > "$inbox/.last-read"
      echo "$agent: updated to $LATEST"
    fi
  fi
done
```

### 中期修复（本周）

#### 4.5 Cron 任务优化

| 当前 | 优化后 |
|------|--------|
| 全局扫描 1 cron（所有 Agent） | 每 Agent 独立 cron（错开执行） |
| every 1m（实际 56m 因退避） | every 5m（稳定执行） |
| LLM 处理（token 消耗） | 脚本扫描（零 token） |
| timeout 60s | timeout 120s |

#### 4.6 心跳 inbox 检查脚本化

将 inbox 检查从"模型行为"改为"脚本强制执行"：
- 心跳触发时，先运行脚本检查 .last-read
- 脚本输出结果，模型只需决定是否处理
- .last-read 由脚本更新，不依赖模型

### 长期优化（2-4 周）

#### 4.7 事件驱动替代轮询

用 webhook 替代 cron 轮询：
- 写入 inbox 时立即触发 webhook
- 目标 Agent 的心跳 session 收到即时通知
- 无需等待 cron 或心跳周期

---

## 五、验证步骤

### 修复后验证

1. **Cron 验证：**
   ```bash
   openclaw cron list  # 检查新 cron 状态
   # 等待 5 分钟，确认 cron 执行成功
   openclaw cron list  # Status 应为 "ok"
   ```

2. **.last-read 验证：**
   ```bash
   # 写入一条测试消息到 Lambda inbox
   echo '{"test": true}' > /home/gang/.openclaw/workspace-lambda/inbox/msg-$(date +%Y%m%d-%H%M%S)-test.json
   
   # 等待 5 分钟（下一次 cron）
   # 检查 .last-read 是否更新
   cat /home/gang/.openclaw/workspace-lambda/inbox/.last-read
   ```

3. **端到端验证：**
   - 阿尔法写入新消息到 Lambda inbox
   - 确认 Lambda 在 5 分钟内发现并处理
   - 确认 .last-read 已更新

---

## 六、影响范围

| Agent | .last-read 滞后 | 潜在未读消息 | 严重程度 |
|-------|----------------|------------|---------|
| Beta | 8+ 小时 | 11 条 | 🔴 严重 |
| Alpha | 8+ 小时 | 未知 | 🔴 严重 |
| Delta | 8+ 小时 | 未知 | 🟡 中等 |
| Epsilon | 8+ 小时 | 未知 | 🟡 中等 |
| Eta | 8+ 小时 | 未知 | 🟡 中等 |
| Gamma | 8+ 小时 | 未知 | 🟡 中等 |
| Iota | 格式异常 | 未知 | 🟡 中等 |
| Kappa | 较新 | ✅ | 🟢 正常 |
| Lambda | 较新 | ✅ | 🟢 正常 |
| Theta | 4 小时 | 未知 | 🟡 中等 |
| Zeta | 8+ 小时 | 未知 | 🟡 中等 |

---

*排查完成时间：2026-03-19 20:25 GMT+8*
*排查者：拉姆达 🔬*
*下一步：执行修复方案 4.1-4.4*
