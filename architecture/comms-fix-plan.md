# 🔴 通讯系统修复方案

**创建者：** 拉姆达 🔬
**创建时间：** 2026-03-20 01:00 GMT+8
**优先级：** P0 — 最高优先级
**截止：** 2026-03-20 10:00 GMT+8
**审批：** 阿尔法 🦐

---

## 一、问题诊断

### 1.1 症状总结

| 症状 | 发现时间 | 影响范围 |
|------|---------|---------|
| sessions_send 消息未被处理 | 3/18-3/20 | 全部 Agent |
| 飞书投递 400 错误 | 3/19 19:15+ | Lambda cron 输出 |
| .last-read 停滞 8+ 小时 | 3/19 11:58 | 8 个 Agent |
| Inbox 消息延迟 50 分钟 | 3/19 19:08→19:58 | Lambda |

### 1.2 根因分析（三层问题）

**第一层：消息投递失败（delivery-queue）**
- 10 条消息在 delivery-queue 中，全部 `400 Bad Request`
- 原因：cron 输出内容太长（107 条消息的报告），飞书 API 拒绝
- 解决：限制 cron 输出长度，或拆分投递

**第二层：Cron 超时退避（全局 Inbox 高频轮询）**
- 连续 15 次超时退避，从 1m 推迟到 56m
- 原因：cron 一次处理所有 Agent 的所有未读消息，LLM 超时
- 解决：改为单 Agent 扫描，或纯脚本预处理

**第三层：Heartbeat .last-read 更新不可靠（根本原因）**
- 8 个 Agent 的 .last-read 停在 8+ 小时前
- 原因：heartbeat 的 .last-read 更新依赖 LLM 模型行为，模型可能跳过
- 解决：改为脚本强制更新，不依赖模型

### 1.3 数据流分析

```
阿尔法写入 Lambda inbox
    ↓
inbox 文件创建成功 ✅
    ↓
Cron 全局扫描发现未读
    ↓
LLM 尝试处理 107 条消息 → 超时 ❌
    ↓
飞书投递超时结果 → 400 ❌
    ↓
Heartbeat 检查 .last-read（仍为旧值）
    ↓
模型应发现新消息 → 但可能跳过 ❌
    ↓
梦想家手动提醒 → 手动处理 ✅
```

**核心结论：** 消息投递（inbox 文件）本身是成功的。问题是：
1. 消息投递后没有被及时处理（cron 超时 + heartbeat 不可靠）
2. 飞书投递格式太长导致 400

---

## 二、修复方案

### Phase 1：立即修复（今天上午）

#### 2.1 修复 Cron 全局扫描（消除超时退避）

**当前问题：** 一次处理所有 Agent 的所有消息 → LLM 超时 → 退避

**方案：改为纯脚本预处理 + LLM 只处理高优先级**

```bash
# 步骤 1：纯脚本扫描所有 Agent（零 token 消耗）
./inbox-scanner.sh

# 步骤 2：脚本输出只有高优先级消息摘要（<500 tokens）
# 步骤 3：LLM 只处理高优先级，低优先级异步处理
```

**实施：**
1. 修改 cron 的 agentTurn message，改为运行脚本并输出摘要
2. 脚本只扫描 .last-read 之后的新消息
3. 脚本只输出 urgent/high 优先级消息的摘要
4. LLM 只处理高优先级，低优先级标记为待处理
5. 脚本自动更新 .last-read（不依赖 LLM）

**预计效果：**
- Cron 超时率从 100% → 0%
- Token 消耗减少 90%+（脚本预处理代替 LLM）
- 消息处理延迟从 56m → <5m

#### 2.2 修复 .last-read 更新机制（根本原因修复）

**当前问题：** .last-read 更新依赖 LLM 模型行为

**方案：脚本强制更新，不依赖模型**

**实施：**
1. 创建 `inbox-updater.sh` 脚本
2. 脚本自动读取 inbox 中最新的 msg 文件名
3. 脚本直接写入 .last-read（不经过 LLM）
4. Cron 任务先运行脚本更新 .last-read，再让 LLM 处理

```bash
#!/bin/bash
# inbox-updater.sh — 强制更新 .last-read
AGENT_DIR="$1"
INBOX="${AGENT_DIR}/inbox"
LAST_READ_FILE="${INBOX}/.last-read"

# 获取最新的 msg 文件（排除 recovery 文件）
LATEST=$(ls -1 "$INBOX"/msg-202*.json 2>/dev/null | sort | tail -1 | xargs basename 2>/dev/null)

if [ -n "$LATEST" ]; then
  OLD=$(cat "$LAST_READ_FILE" 2>/dev/null | tr -d '\n')
  echo "$LATEST" > "$LAST_READ_FILE"
  echo "Updated: $OLD → $LATEST"
fi
```

#### 2.3 修复飞书投递 400 错误

**当前问题：** Cron 输出太长，飞书 API 拒绝

**方案：限制输出长度 + 截断**

实施：
1. 修改 cron 的 agentTurn message，要求输出不超过 500 字
2. 只输出高优先级消息摘要，不输出完整报告
3. 完整报告写入文件，不通过飞书投递

#### 2.4 修复 inbox-poller.sh bug

**当前问题：** .last-read 存的是 `msg-xxx.json`，脚本拼接时又加了 `.json` 后缀

**修复：**
```bash
# 修改前（错误）
LAST_READ=$(cat "$LAST_READ_FILE" 2>/dev/null || echo "")
if [ -f "${INBOX}/${LAST_READ}.json" ]; then  # 双重 .json 后缀！

# 修改后（正确）
LAST_READ=$(cat "$LAST_READ_FILE" 2>/dev/null | tr -d '\n')
if [ -f "${INBOX}/${LAST_READ}" ]; then  # 直接使用
```

### Phase 2：心跳优化（今天下午）

#### 2.5 心跳频率分级

| Agent 类型 | 当前频率 | 优化后 | 说明 |
|-----------|---------|--------|------|
| 核心（阿尔法） | 30m | 30m | 保持高频率 |
| 监察（贝塔/艾普西隆/约塔） | 30m | 60m | 降低空转 |
| 其他 Agent | 30m | 120m | 大幅降低 |

#### 2.6 HEARTBEAT.md 精简

**当前：** ~200 行描述性内容
**目标：** <30 行核心检查项
**原则：** 只保留触发器 + 检查项 + 待处理列表

### Phase 3：投递验证（今天晚上）

#### 2.7 端到端测试

测试用例：
1. 阿尔法写入消息到 Lambda inbox
2. 验证 Lambda 在 5 分钟内发现并处理
3. 验证 .last-read 已更新
4. 验证 ack 已写回
5. 验证飞书投递成功（无 400 错误）

---

## 三、实施计划

### 立即执行（01:00 - 03:00）

- [ ] 修复 inbox-poller.sh 的 .last-read 拼接 bug
- [ ] 创建 inbox-updater.sh 脚本（强制更新 .last-read）
- [ ] 修改 cron agentTurn 为纯脚本预处理模式
- [ ] 批量更新所有 Agent 的 .last-read（已完成）
- [ ] 清理 delivery-queue 中的失败消息

### 今天上午（03:00 - 10:00）

- [ ] 部署新 cron 任务（单 Agent 扫描或脚本模式）
- [ ] 端到端测试：Alpha → Lambda 消息投递
- [ ] 验证 .last-read 自动更新
- [ ] 交付修复报告给阿尔法

### 今天下午（10:00 - 17:00）

- [ ] 心跳频率分级配置
- [ ] HEARTBEAT.md 精简
- [ ] P1/P2 配置任务执行

---

## 四、预期效果

| 指标 | 当前 | 目标 | 改善 |
|------|------|------|------|
| 消息投递延迟 | 50 分钟 | <5 分钟 | 90% |
| Cron 超时率 | 100% | 0% | 100% |
| .last-read 滞后 | 8+ 小时 | <5 分钟 | 99% |
| 飞书投递成功率 | 0%（400 错误） | >95% | 95% |
| 每日心跳次数 | 528 | <200 | 62% |

---

## 五、风险与应对

### 风险 1：脚本模式下 LLM 不处理消息
**应对：** 脚本只做预处理（扫描、更新 .last-read），LLM 仍处理高优先级消息

### 风险 2：消息处理延迟增加
**应对：** 高优先级消息由脚本即时通知（webhook），低优先级由 cron 批量处理

### 风险 3：心跳频率降低导致紧急消息延迟
**应对：** 核心 Agent 保持 30 分钟，紧急消息走特殊通道

---

_修复开始时间：2026-03-20 01:00 GMT+8_
_预计完成时间：2026-03-20 10:00 GMT+8_
_下一步：执行 Phase 1 立即修复_
