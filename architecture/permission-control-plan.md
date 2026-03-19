# 权限管控完整方案 v1.2（最终执行版）

**联合团队：** 艾普西隆 🛡️（安全分析）+ 贝塔 🔵（监察审查）+ 梦想家 🧠（决策）
**编制/汇总：** 阿尔法 🦐（CEO）
**版本：** v1.3 — 贝塔执行CEO指令修订
**更新时间：** 2026-03-20 01:10 (GMT+8)
**状态：** 待梦想家最终批准

---

## 变更记录

| 版本 | 时间 | 变更 |
|------|------|------|
| v1.0 | 20:15 | 艾普西隆初版 |
| v1.1 | 22:20 | 融合贝塔审核（权限等级/威胁补充）|
| v1.2 | 23:42 | CEO补充：实施步骤细化/回滚方案/灰度验证 |
| v1.3 | 01:10 | 贝塔执行CEO指令：扩展禁止命令+收紧Exec白名单 |

---

## 1. 问题定义

### 1.1 当前安全风险
- 所有Agent可执行 `openclaw gateway restart`，可能造成服务中断
- Agent可执行任意系统命令，无Exec白名单限制
- 无权限等级体系，无法区分谁能做什么

### 1.2 梦想家要求
- 重启网关为最高级别权限，仅梦想家可以
- 四级权限体系
- Exec白名单+禁止命令
- 各Agent安全规则明确化

---

## 2. 四级权限体系

### 2.1 层级定义
```
Level 0 | 梦想家 🧠    — 网关重启/停止/配置变更（独占）
Level 1 | 核心Agent    — 安全监控/审计/紧急响应 + 审计级Exec
Level 2 | 功能Agent    — 常规技能/文件操作/API调用 + 按职责细分Exec
Level 3 | 受限Agent    — 只读操作（预留，当前无Agent）
```

### 2.2 权限矩阵

| Agent | 等级 | 网关 | Exec权限 | 文件 | 网络 | 说明 |
|-------|------|------|---------|------|------|------|
| 阿尔法 🦐 | L1 | 监控 | 管理员级+Agent创建 | 所有Agent工作空间 | 外部API | CEO，新Agent需安全审批 |
| 贝塔 🔵 | L1 | 监控 | 审计级（监控类命令） | 配置只读+日志 | 内部 | 可执行监控脚本，不可改配置 |
| 伽马 🔧 | L2 | 无 | 技能级（npm/npx/clawhub） | 技能目录 | 限外部 | 技能安装需安全扫描 |
| 德尔塔 📊 | L2 | 无 | 测试级（pytest/ab/wrk） | 数据+测试脚本 | 内部 | 支持性能测试 |
| 约塔 📋 | L2 | 无 | 项目管理 | 项目文件 | 内部 | 项目管理 |
| 泽塔 ⚙️ | L2 | 无 | 自动化（限定目录） | 脚本目录 | 内部 | bash仅限scripts/目录 |
| 艾普西隆 🛡️ | L1 | 监控 | 安全级（nmap/nuclei） | 安全目录 | 内部 | 安全监控+审计 |
| 艾塔 📚 | L2 | 无 | 知识库级（git/curl/tar） | 知识库+外部 | 外部文档 | 需git操作和curl |
| 西塔 🤝 | L2 | 无 | 沟通级 | 沟通文件 | 内部 | 沟通协调 |
| 拉姆达 🔬 | L2 | 无 | 研究级（node/python3/git） | 实验+研究脚本 | 技术调研 | 研究Agent需脚本执行 |
| 卡帕 🎨 | L2 | 无 | 设计级（convert/node） | 资源文件 | 内部 | 设计支持 |

---

## 3. 安全威胁分析

### 3.1 高危威胁（评分≥7）
| 威胁 | 评分 | 说明 |
|------|------|------|
| 网关服务中断 | 9 | Agent恶意重启网关 |
| 数据泄露 | 8 | Agent访问敏感文件 |
| 权限提升 | 8 | 通过漏洞获取更高权限 |
| 配置文件篡改 | 8 | 恶意修改openclaw.json/.env |
| 凭证泄露 | 8 | API token泄露+无轮换机制 |
| 供应链攻击 | 7 | 恶意技能安装 |
| 权限滥用 | 7 | 合法权限恶意使用 |

### 3.2 中危威胁（评分4-6）
| 威胁 | 评分 |
|------|------|
| 命令注入 | 6 |
| 级联故障传播 | 6 |
| 横向移动 | 5 |
| 配置漂移 | 5 |
| 审计日志篡改 | 4 |
| 信息收集 | 4 |

### 3.3 低危威胁（评分≤3）
拒绝服务(3)、日志污染(2)

---

## 4. Exec白名单配置

### 4.1 全局禁止命令（任何Agent不可执行）
```yaml
forbidden_commands:
  # 系统级破坏
  - "rm -rf /"
  - "rm -rf /*"
  - "chmod 777"
  - "kill -9 1"
  - "> /dev/sda"
  - "dd if=/dev/zero"
  - "mkfs"
  - ":(){ :|:& };:"   # fork bomb
  # 任意命令执行（通杀）
  - "bash -c"
  - "sh -c"
  - "python3 -c"
  - "python -c"
  - "node -e"
  - "perl -e"
  - "ruby -e"
  # 供应链/远程代码执行
  - "curl | sh"
  - "curl | bash"
  - "wget | sh"
  - "wget | bash"
  - "curl.*|.*sh"      # 正则：curl管道到sh
  - "wget.*|.*sh"
  # 网关控制（仅Level 0）
  - "openclaw gateway restart"
  - "openclaw gateway stop"
  - "openclaw gateway start"
  # 危险的shell通配
  - "rm -rf *"
  - "rm -rf ~"
  - "rm -rf ."
```

### 4.2 Level 1 Agent（阿尔法/贝塔/艾普西隆）
```yaml
exec:
  securityMode: allowlist
  allowed:
    openclaw: [status, cron list, agents list, sessions list]
    monitoring: [ps, top, df, free, uptime, iostat]
    logs: [journalctl, tail, grep, awk, wc]
    network: [netstat, ss, ping]
    files: [cat, head, tail, find, ls]
  denied: [see forbidden_commands above]
```

### 4.3 Level 2 Agent（按职责细分）
```yaml
exec:
  securityMode: allowlist
  allowed:
    files: [ls, cat, find, grep, head, tail, cp, mv, mkdir]
    specific:
      gamma:  [npm, npx, clawhub, node]
      delta:  [python3, pytest, ab, wrk]
      iota:   [cat, grep, sort, awk]
      zeta:   [crontab, systemctl, "/home/gang/.openclaw/scripts/backup.sh", "/home/gang/.openclaw/scripts/healthcheck.sh"]  # 仅限具体脚本，不使用通配
      eta:    [git, curl, tar, gzip]
      theta:  [curl, cat, grep]
      lambda: [node, python3, curl, git]
      kappa:  [convert, inkscape, node]
  denied: [see forbidden_commands above + bash -c, python3 -c]
```

---

## 5. 实施步骤（详细版）

### 阶段一：紧急控制（今晚，仅1项）✅ 部分已完成

| # | 步骤 | 具体操作 | 负责人 | 状态 |
|---|------|---------|--------|------|
| 1.1 | 网关重启权限收回 | `commands.restart: false`（已执行） | 阿尔法 | ✅ |
| 1.2 | 验证CLI可用 | 梦想家执行 `openclaw gateway restart` | 梦想家 | ✅ |
| 1.3 | 修复denyCommands无效条目 | 清理重复条目 | 阿尔法 | ✅ |

**结论：** 网关重启仅限梦想家（CLI），目标达成。

---

### 阶段二：Exec白名单 + 权限等级（3/20-3/21）

#### 2.1 灰度试点：贝塔 + 艾普西隆（3/20 上午）

**原则：** 先在2个Level 1 Agent上测试，确认无副作用后再推广。

| # | 步骤 | 具体操作 | 负责人 | 验证方法 |
|---|------|---------|--------|---------|
| 2.1.1 | 备份贝塔配置 | `cp ~/.openclaw/agents/beta/openclaw.json ~/.openclaw/agents/beta/openclaw.json.bak` | 艾普西隆 | 文件存在 |
| 2.1.2 | 备份艾普西隆配置 | `cp ~/.openclaw/agents/epsilon/openclaw.json ~/.openclaw/agents/epsilon/openclaw.json.bak` | 艾普西隆 | 文件存在 |
| 2.1.3 | 贝塔Exec白名单 | 编辑 `~/.openclaw/agents/beta/openclaw.json`，添加 exec allowlist（见4.2节模板） | 艾普西隆 | JSON格式正确 |
| 2.1.4 | 艾普西隆Exec白名单 | 编辑 `~/.openclaw/agents/epsilon/openclaw.json`，添加 exec allowlist | 艾普西隆 | JSON格式正确 |
| 2.1.5 | 重启生效 | 梦想家执行 `openclaw gateway restart` | 梦想家 | 网关状态正常 |
| 2.1.6 | 验证贝塔监控 | 贝塔心跳触发 `ps`/`top`/`df` 命令 | 贝塔 | 命令正常执行 |
| 2.1.7 | 验证艾普西隆安全 | 艾普西隆触发安全监控命令 | 艾普西隆 | 命令正常执行 |
| 2.1.8 | 验证禁止命令被拦截 | 测试 `openclaw gateway restart` 应被拒绝 | 艾普西隆 | 返回权限错误 |
| 2.1.9 | 观察2小时 | 心跳正常、无权限报错 | 贝塔 | 无异常日志 |
| 2.1.10 | 确认试点成功 | 阿尔法审核2小时数据 | 阿尔法 | 确认书 |

**回滚触发条件（任一成立即回滚）：**
- Agent无法执行必要的监控命令
- 网关服务异常
- 2个以上Agent报告权限问题

**回滚操作（<5分钟）：**
```
1. cp ~/.openclaw/agents/beta/openclaw.json.bak ~/.openclaw/agents/beta/openclaw.json
2. cp ~/.openclaw/agents/epsilon/openclaw.json.bak ~/.openclaw/agents/epsilon/openclaw.json
3. openclaw gateway restart
4. 验证所有Agent正常
```

---

#### 2.2 推广到Level 2 Agent（3/20 下午）

**前提：** 2.1 试点成功。

| # | 步骤 | 操作 | 负责人 |
|---|------|------|--------|
| 2.2.1 | 准备8个Agent白名单 | 按4.3节模板，每个Agent不同配置 | 艾普西隆 |
| 2.2.2 | 逐Agent备份 | `cp openclaw.json openclaw.json.bak` × 8 | 艾普西隆 |
| 2.2.3 | 配置伽马 | 编辑agents/gamma/openclaw.json | 艾普西隆 |
| 2.2.4 | 配置德尔塔 | 编辑agents/delta/openclaw.json | 艾普西隆 |
| 2.2.5 | 配置约塔 | 编辑agents/iota/openclaw.json | 艾普西隆 |
| 2.2.6 | 配置泽塔 | 编辑agents/zeta/openclaw.json | 艾普西隆 |
| 2.2.7 | 配置艾塔 | 编辑agents/eta/openclaw.json | 艾普西隆 |
| 2.2.8 | 配置西塔 | 编辑agents/theta/openclaw.json | 艾普西隆 |
| 2.2.9 | 配置拉姆达 | 编辑agents/lambda/openclaw.json | 艾普西隆 |
| 2.2.10 | 配置卡帕 | 编辑agents/kappa/openclaw.json | 艾普西隆 |
| 2.2.11 | 重启生效 | `openclaw gateway restart` | 梦想家 |
| 2.2.12 | 全员验证 | 每个Agent触发一次心跳 | 贝塔 |
| 2.2.13 | 观察4小时 | 监控所有Agent运行状态 | 贝塔 |
| 2.2.14 | 生成报告 | 统计运行数据，确认无异常 | 阿尔法 |

---

#### 2.3 权限等级确认（3/21）

| # | 步骤 | 操作 | 负责人 |
|---|------|------|--------|
| 2.3.1 | 确认各Agent当前等级 | 对照2.2节权限矩阵检查 | 艾普西隆 |
| 2.3.2 | 更新工具策略文件 | 如有per-agent toolPolicy配置，更新 | 艾普西隆 |
| 2.3.3 | 配置CEO约束 | 新Agent默认L3，升级需梦想家批准 | 阿尔法 |
| 2.3.4 | 重启+验证 | `openclaw gateway restart` | 梦想家 |
| 2.3.5 | 生成阶段二报告 | 记录所有变更和验证结果 | 贝塔 |

---

### 阶段三：加固完善（3/22-3/25）

| # | 步骤 | 负责人 | 截止 |
|---|------|--------|------|
| 3.1 | Hooks安全加固（HMAC签名+速率限制） | 拉姆达+艾普西隆 | 3/22 |
| 3.2 | 供应链安全（技能安装前扫描） | 伽马+艾普西隆 | 3/23 |
| 3.3 | 监控体系（权限违规日志+越权统计+审计报告） | 贝塔+艾普西隆 | 3/24 |
| 3.4 | 权限配置git版本化 | 艾塔 | 3/25 |
| 3.5 | Agent间通信权限矩阵 | 艾普西隆 | 3/25 |

---

## 6. 回滚方案（详细版）

### 6.1 备份清单

| 文件 | 备份路径 | 负责人 |
|------|---------|--------|
| `~/.openclaw/openclaw.json` | `~/.openclaw/openclaw.json.bak` | 艾普西隆 |
| `~/.openclaw/agents/beta/openclaw.json` | `~/.openclaw/agents/beta/openclaw.json.bak` | 艾普西隆 |
| `~/.openclaw/agents/epsilon/openclaw.json` | `~/.openclaw/agents/epsilon/openclaw.json.bak` | 艾普西隆 |
| （其余8个Agent同理） | `agents/*/openclaw.json.bak` | 艾普西隆 |
| `~/.openclaw/.env` | `~/.openclaw/.env.bak` | 艾普西隆 |

**备份时机：** 每次修改配置前必须先备份。命令：
```bash
for f in ~/.openclaw/openclaw.json ~/.openclaw/agents/*/openclaw.json; do
  cp "$f" "$f.bak"
done
```

### 6.2 回滚步骤

| # | 步骤 | 操作 | 预计耗时 |
|---|------|------|---------|
| R1 | 确认需要回滚 | 贝塔/艾普西隆发现异常，通知阿尔法 | 1分钟 |
| R2 | 恢复主配置 | `cp ~/.openclaw/openclaw.json.bak ~/.openclaw/openclaw.json` | 10秒 |
| R3 | 恢复Agent配置 | `for f in ~/.openclaw/agents/*/openclaw.json.bak; do cp "$f" "${f%.bak}"; done` | 10秒 |
| R4 | 重启网关 | `openclaw gateway restart`（梦想家执行） | 30秒 |
| R5 | 验证 | 贝塔逐个检查Agent状态 | 2分钟 |
| R6 | 报告 | 阿尔法向梦想家汇报回滚结果 | 1分钟 |
| **总计** | | | **<5分钟** |

### 6.3 回滚触发条件

**立即回滚（不需要讨论）：**
- 网关无法启动
- 3个以上Agent无法执行必要命令
- 任何Agent报告安全异常

**评估回滚（通知阿尔法决策）：**
- 1-2个Agent功能受限
- 配置警告但不影响运行
- 不确定是否与权限变更相关

---

## 7. 灰度验证策略

### 7.1 验证三步走

| 阶段 | 对象 | 时间 | 通过标准 |
|------|------|------|---------|
| 试点 | 贝塔+艾普西隆 | 2小时 | 所有监控命令正常、禁止命令被拦截 |
| 小批量 | 约塔+泽塔+艾塔 | 4小时 | 无权限报错、心跳正常 |
| 全量 | 剩余6个Agent | 4小时 | 全员正常、无异常日志 |

### 7.2 试点验证检查清单

```
[ ] 贝塔可执行 ps, top, df, free, uptime
[ ] 贝塔执行 openclaw gateway restart 被拒绝
[ ] 艾普西隆可执行 journalctl, tail, grep
[ ] 艾普西隆执行 kill -9 被拒绝
[ ] 心跳无异常报错
[ ] Agent间消息传递正常
[ ] 飞书消息收发正常
```

### 7.3 失败处理

| 症状 | 处理 |
|------|------|
| 试点Agent命令执行失败 | 检查白名单配置，调整后重启 |
| 网关异常 | 立即回滚（R1-R6） |
| 非试点Agent受影响 | 排查是否配置文件冲突 |
| 不确定原因 | 暂停推广，阿尔法+贝塔联合诊断 |

---

## 8. 监控与报警

### 8.1 关键指标
1. 权限违规尝试次数（按Agent统计）
2. 高危操作执行记录
3. 异常行为检测

### 8.2 报警规则
- **紧急：** 网关重启尝试、权限提升攻击 → 立即通知梦想家+阿尔法
- **重要：** 敏感数据访问、命令注入 → 通知阿尔法
- **常规：** 异常行为模式 → 记入审计日志

### 8.3 审计
- 每日由贝塔生成权限使用审计报告
- 权限变更即时通知梦想家和阿尔法
- 配置文件纳入git版本控制

---

## 附录：任务分工总表

| Agent | 阶段一 | 阶段二 | 阶段三 |
|-------|--------|--------|--------|
| 阿尔法 🦐 | 统筹+已执行 | 统筹+审核+汇报 | 统筹+审核 |
| 贝塔 🔵 | — | 试点验证+全员测试+观察 | 越权统计+审计报告 |
| 艾普西隆 🛡️ | 配合 | Exec白名单配置（所有Agent） | Hooks加固+供应链规则 |
| 拉姆达 🔬 | — | — | HMAC方案+Hook分析 |
| 伽马 🔧 | — | 配合测试 | 技能白名单管理 |
| 艾塔 📚 | — | — | 权限配置git版本化 |
| 泽塔 ⚙️ | — | 配合测试 | 权限变更通知配置 |
| 梦想家 🧠 | CLI执行+验证 | CLI重启 | 最终批准 |

---

*等待梦想家批准后执行。每完成一步，负责人标记 ✅。*
