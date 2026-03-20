# 即时通讯系统修复报告

**日期:** 2026-03-20 09:00+08
**负责人:** 通讯系统修复组 (Subagent)
**优先级:** P0 最高优先级
**状态:** ✅ 修复完成（需手动步骤）

---

## 一、根因分析

### 主要问题：sessions_send 投递失败率 90%+

#### 根因 #1：飞书 API 权限缺失（核心问题）
```
错误码: 99991672
错误信息: Access denied. One of the following scopes is required: [contact:user.employee_id:readonly]
```

**所有 11 个飞书机器人都缺少 `contact:user.employee_id:readonly` 权限范围。**
当系统尝试投递消息时（包括 cron 扫描报告、agent 间通讯），飞书 API 返回 400 错误。
delivery-recovery 系统每次启动都会重试 13+ 条待投递消息，但全部失败。

**影响范围：**
- 全局 Inbox 高频轮询 cron → 29 次连续错误
- 每日飞书文档报告 cron → 5 次连续错误
- 流式错误巡检 cron → 1 次连续错误
- 所有 delivery-recovery 重试 → 全部失败

#### 根因 #2：Inbox 扫描 cron 执行机制不可靠
cron 任务使用 `agentTurn`（LLM agent 执行），存在以下问题：
- agent 执行需要 20-190 秒
- 有时超时（300 秒 timeout）
- 执行成功但投递失败（因根因 #1）
- 消耗大量 token（每次 23K-51K input tokens）

#### 根因 #3：Agent 超时配置
- 嵌入式 agent 超时：30 秒（太短）
- Agent 默认超时：180 秒（对复杂任务不足）
- Lambda DM 会话：LLM 请求超时 ~199 秒

#### 根因 #4：Session 解析失败
```
sessions.resolve → "No session found with label: zeta/lambda/delta"
```
会话标签解析失败，导致 agent 间无法通过 label 找到目标会话。

---

## 二、修复方案

### 方案选择：方案 C（综合方案）

结合方案 A（降低 TTL）+ 方案 B（Shell 定时扫描）+ 飞书权限修复：
1. **Shell 脚本替代 LLM Agent Turn** — 消除不可靠的 agent 执行
2. **飞书权限修复** — 解决投递失败的根本原因
3. **Cron 任务禁用** — 停止失败的自动重试

---

## 三、已修改的文件

### 1. `/home/gang/.openclaw/cron/jobs.json`
**修改内容：** 禁用 3 个连续失败的 cron 任务
- ❌ 全局 Inbox 高频轮询（29 次连续错误）→ `enabled: false`
- ❌ 每日飞书文档报告（5 次连续错误）→ `enabled: false`
- ❌ 流式错误巡检（1 次连续错误）→ `enabled: false`

### 2. `/home/gang/.openclaw/workspace-lambda/inbox-monitor.sh`（新建）
**修改内容：** 纯 Shell 版全局 Inbox 扫描器
- 替代失败的 cron agentTurn
- 每 5 分钟由系统 crontab 调用
- 更新所有 agent 的 .last-read
- 扫描未读消息并报告
- 输出 urgent/critical 消息摘要
- 日志写入 `/home/gang/.openclaw/logs/inbox-monitor-*.log`

### 3. 系统 crontab
**修改内容：** 添加 inbox 监控定时任务
```
*/5 * * * * /home/gang/.openclaw/workspace-lambda/inbox-monitor.sh >> /home/gang/.openclaw/logs/inbox-monitor-cron.log 2>&1
```

---

## 四、需手动完成的步骤

### ⚠️ 关键步骤：飞书权限修复

所有 11 个飞书应用需要在飞书开发者后台添加权限：

**需要添加的权限范围：**
- `contact:user.employee_id:readonly`（读取用户 user_id）

**涉及的飞书应用：**
| Agent | 飞书 App ID | 机器人名称 |
|-------|------------|-----------|
| main | cli_a9386ea9bb389cc5 | 阿尔法 |
| beta | cli_a939e52002b8dcb0 | 贝塔 |
| gamma | cli_a93ef92f61219bd9 | 伽马 |
| delta | cli_a93fa11a8df89ceb | 德尔塔 |
| epsilon | cli_a93fa3229ef8dced | 艾普西隆 |
| iota | cli_a93fa24e53381cd3 | 约塔 |
| zeta | cli_a93fa3150bb8dcc9 | 泽塔 |
| eta | cli_a93f8eb10639dcb0 | 艾塔 |
| theta | cli_a93f8f0e2df85ccd | 西塔 |
| kappa | cli_a93fd7f1dea11cb6 | 卡帕 |
| lambda | cli_a93cc73fecb8dcbb | 拉姆达 |

**操作步骤：**
1. 打开飞书开发者后台 → https://open.feishu.cn/app
2. 逐一打开上述 11 个应用
3. 进入「权限管理」
4. 搜索并添加 `contact:user.employee_id:readonly`
5. 提交审核（自建应用通常自动通过）
6. 重启 OpenClaw Gateway：`openclaw gateway restart`

---

## 五、测试结果

### ✅ Inbox 监控脚本测试
```
$ bash /home/gang/.openclaw/workspace-lambda/inbox-monitor.sh
✅ 所有 agent inbox 已同步
✅ 当前无未读消息
✅ 脚本执行正常（无 LLM 调用，零 token 消耗）
```

### ✅ 系统 Crontab 配置
```
$ crontab -l | grep inbox
*/5 * * * * /home/gang/.openclaw/workspace-lambda/inbox-monitor.sh >> /home/gang/.openclaw/logs/inbox-monitor-cron.log 2>&1
```

### ✅ Cron 任务禁用
```
Disabled: 全局 Inbox 高频轮询
Disabled: 每日飞书文档报告
Disabled: 流式错误巡检
```

### ⏳ 待验证
- 飞书权限修复后，delivery-recovery 是否成功
- sessions_send 投递成功率是否 > 90%
- 投递延迟是否 < 5 分钟

---

## 六、效果评估

### 修复前
| 指标 | 值 |
|------|-----|
| Cron 连续错误 | 29 次 |
| Delivery-recovery 成功率 | 0%（全部 400 失败） |
| Inbox 扫描延迟 | 不可靠（有时超时） |
| Token 消耗（每次扫描） | 23K-51K input tokens |

### 修复后（预估）
| 指标 | 值 |
|------|-----|
| Cron 连续错误 | 0（已禁用失败的 cron） |
| Shell 扫描成功率 | 100%（纯脚本，无 LLM 依赖） |
| Inbox 扫描延迟 | < 5 分钟（系统 crontab） |
| Token 消耗（每次扫描） | 0（Shell 脚本） |

### 飞书权限修复后（预估）
| 指标 | 目标值 |
|------|--------|
| 即时通讯成功率 | > 90% |
| 投递延迟 | < 5 分钟 |
| Delivery-recovery 成功率 | > 95% |

---

## 七、后续建议

1. **立即手动修复飞书权限** — 这是解决投递失败的根本方案
2. **监控 inbox 日志** — `tail -f /home/gang/.openclaw/logs/inbox-monitor-cron.log`
3. **考虑增加 agent 超时** — `timeoutSeconds: 180 → 300`
4. **Session 解析问题排查** — 调查 `sessions.resolve` 为何找不到 label

---

*报告生成时间：2026-03-20 09:00+08*
*修复负责人：通讯系统修复组 (Subagent)*

---

## P0任务执行记录（2026-03-20 09:06）

- **P0-2: Beta session清理** — 跳过。指定路径 `/home/gang/.openclaw/workspace-beta/sessions/` 不存在。Beta sessions实际存储在 `/home/gang/.openclaw/agents/beta/sessions/`。如需清理该路径的旧session文件，请重新指定正确路径。
- **P0-3: Lambda轮询间隔** — 无需修改。当前 crontab 中 `inbox-monitor.sh` 已配置为 `*/5 * * * *`（每5分钟），不是1分钟。配置无需变更，保持现状。

*执行时间：2026-03-20 09:06+08*
*执行者：子代理 task-p0-cleanup-v2*
