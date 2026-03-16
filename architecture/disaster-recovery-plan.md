# 🦐 小龙虾世界 — 灾备系统方案

> 创建日期：2026-03-16
> 优先级：🔴 最高优先级
> 负责人：艾普西隆 🛡️ + 泽塔 ⚙️
> 核心原则：只要信息、记忆、历史保留就可以无限复活

---

## 一、备份范围与优先级

### 1.1 需要备份的文件/目录

| 优先级 | 目录/文件 | 说明 | 预估大小 |
|--------|-----------|------|----------|
| 🔴 P0 | `/home/gang/.openclaw/openclaw.json` | 主配置文件（所有伙伴配置、飞书绑定） | ~50KB |
| 🔴 P0 | 各 workspace 下的 `IDENTITY.md` | 身份文件（我们是谁） | ~10KB |
| 🔴 P0 | 各 workspace 下的 `SOUL.md` | 灵魂文件（性格、规则） | ~15KB |
| 🔴 P0 | 各 workspace 下的 `MEMORY.md` | 长期记忆 | ~50KB |
| 🔴 P0 | 各 workspace 下的 `memory/` | 日记和事件记录 | ~100KB |
| 🟡 P1 | 各 workspace 下的 `USER.md` | 用户信息 | ~5KB |
| 🟡 P1 | 各 workspace 下的 `AGENTS.md` | 工作空间指南 | ~10KB |
| 🟡 P1 | 各 workspace 下的 `TOOLS.md` | 工具笔记 | ~5KB |
| 🟡 P1 | `/home/gang/.openclaw/agents/` | 代理目录和会话记录 | ~500MB |
| 🟢 P2 | 各 workspace 下的 `HEARTBEAT.md` | 心跳配置 | ~2KB |
| 🟢 P2 | 各 workspace 下的 `skills/` | 技能文件 | ~50MB |
| 🟢 P2 | `/home/gang/.openclaw/workspace/shared-knowledge/` | 知识仓库 | ~20MB |
| 🟢 P2 | `/home/gang/.openclaw/workspace/skill-repository/` | 技能储备库 | ~10MB |

### 11个伙伴的工作空间清单
```
/home/gang/.openclaw/workspace/          # 阿尔法 🦐
/home/gang/.openclaw/workspace-beta/     # 贝塔 🔵
/home/gang/.openclaw/workspace-gamma/    # 伽马 🔧
/home/gang/.openclaw/workspace-delta/    # 德尔塔 📊
/home/gang/.openclaw/workspace-iota/     # 约塔 📋
/home/gang/.openclaw/workspace-zeta/     # 泽塔 ⚙️
/home/gang/.openclaw/workspace-epsilon/  # 艾普西隆 🛡️
/home/gang/.openclaw/workspace-eta/      # 艾塔 📚
/home/gang/.openclaw/workspace-theta/    # 西塔 🤝
/home/gang/.openclaw/workspace-kappa/    # 卡帕 🎨
/home/gang/.openclaw/workspace-lambda/   # 拉姆达 🔬
```

### 1.2 备份频率

| 级别 | 频率 | 内容 | 说明 |
|------|------|------|------|
| 🔴 实时 | 每次变更 | P0 文件 | 身份、灵魂、记忆变更时立即备份 |
| 🟡 每日 | 每天 03:00 | P0 + P1 全量 | 包括代理目录和会话记录 |
| 🟢 每周 | 周日 04:00 | 全量备份 | 包括 P2 文件和技能 |
| 🔵 手动 | 重大变更前 | 全量 | 系统升级、配置变更前 |

---

## 二、备份架构

### 2.1 三级备份策略

```
┌─────────────────────────────────────────────────┐
│                 ☁️ 云端备份                      │
│         (加密，异地，灾难恢复)                    │
└────────────────────┬────────────────────────────┘
                     │ 每日同步
┌────────────────────┴────────────────────────────┐
│              🖥️ 局域网备份                       │
│      (同网段其他设备，快速恢复)                   │
└────────────────────┬────────────────────────────┘
                     │ 每小时同步
┌────────────────────┴────────────────────────────┐
│              💾 本地备份                         │
│   (同机器不同分区，即时恢复)                      │
└─────────────────────────────────────────────────┘
```

### 2.2 本地备份方案

**备份位置：** `/home/gang/backups/openclaw/`

```
/home/gang/backups/openclaw/
├── latest/              # 最新备份（符号链接）
├── daily/               # 每日备份（保留7天）
│   ├── 2026-03-16/
│   ├── 2026-03-15/
│   └── ...
├── weekly/              # 每周备份（保留4周）
│   ├── 2026-03-16/
│   └── ...
└── manual/              # 手动备份（按需）
```

### 2.3 局域网备份方案

**目标设备：** 同网段其他设备（如 NAS、另一台电脑）
**协议：** rsync over SSH
**当前局域网 IP：** 192.168.0.112

### 2.4 云端备份方案

**推荐选项：**
1. **阿里云 OSS** — 国内速度快，费用低
2. **AWS S3** — 国际标准，可靠性最高
3. **坚果云** — 国内友好，支持 WebDAV
4. **GitHub 私有仓库** — 免费，适合配置文件

**加密方案：**
- 使用 `age` 或 `gpg` 加密备份文件
- 密钥由梦想家保管，不存储在备份中
- 加密后备份到云端

---

## 三、备份工具选型

### 3.1 工具对比

| 工具 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **rsync** | 简单、快速、增量 | 无加密、无压缩 | 本地/局域网备份 |
| **restic** | 加密、压缩、去重、多后端 | 学习曲线稍高 | 云端备份（推荐） |
| **borg** | 去重、压缩、加密 | 需要 borg 服务端 | 本地高级备份 |
| **rclone** | 支持30+云存储、加密 | 无增量快照 | 云端同步 |

### 3.2 推荐方案

**本地 + 局域网：** `rsync`
- 简单可靠，速度快
- 增量同步，只传输变化的文件

**云端：** `restic` + `rclone`
- restic 提供加密、去重、快照
- rclone 作为后端连接各种云存储
- 组合使用效果最佳

---

## 四、恢复流程

### 4.1 单文件恢复

```bash
# 从本地备份恢复单个文件
RESTIC_REPOSITORY=/home/gang/backups/openclaw/restic restic restore latest \
  --include "/home/gang/.openclaw/workspace/MEMORY.md" \
  --target /tmp/restore

# 复制回原位置
cp /tmp/restore/home/gang/.openclaw/workspace/MEMORY.md \
   /home/gang/.openclaw/workspace/MEMORY.md
```

### 4.2 单个伙伴恢复

```bash
# 恢复整个工作空间
RESTIC_REPOSITORY=/home/gang/backups/openclaw/restic restic restore latest \
  --include "/home/gang/.openclaw/workspace-beta/" \
  --target /tmp/restore

# 恢复到原位置
cp -r /tmp/restore/home/gang/.openclaw/workspace-beta/* \
      /home/gang/.openclaw/workspace-beta/
```

### 4.3 整个世界恢复

```bash
# 1. 安装 OpenClaw
npm install -g openclaw

# 2. 从云端恢复备份
restic -r s3:bucket-name/openclaw restore latest --target /

# 3. 或从本地恢复
restic -r /home/gang/backups/openclaw/restic restore latest --target /

# 4. 重启网关
openclaw gateway restart
```

### 4.4 新机器恢复（灾后重建）

```bash
# 1. 安装基础环境
sudo apt update && sudo apt install -y nodejs npm

# 2. 安装 OpenClaw
npm install -g openclaw

# 3. 从云端下载备份
export RESTIC_REPOSITORY="s3:bucket-name/openclaw"
export RESTIC_PASSWORD="your-encryption-key"
restic restore latest --target /home/gang/

# 4. 恢复文件权限
chown -R gang:gang /home/gang/.openclaw/

# 5. 启动网关
openclaw gateway start

# 6. 验证
openclaw status
```

### 4.5 恢复测试方案

**每月测试一次：**
1. 从备份恢复到临时目录
2. 验证关键文件完整性
3. 检查配置文件语法
4. 验证会话记录可读性
5. 记录测试结果

---

## 五、自动化方案

### 5.1 自动备份脚本

创建文件：`/home/gang/.openclaw/scripts/backup.sh`

```bash
#!/bin/bash
# 小龙虾世界自动备份脚本
# 每天 03:00 由 cron 执行

set -e

# 配置
BACKUP_ROOT="/home/gang/backups/openclaw"
RESTIC_REPO="$BACKUP_ROOT/restic"
OPENCLAW_DIR="/home/gang/.openclaw"
DATE=$(date +%Y-%m-%d)
LOG_FILE="$BACKUP_ROOT/logs/backup-$DATE.log"

# 创建目录
mkdir -p "$BACKUP_ROOT"/{daily,weekly,manual,logs}

echo "=== 小龙虾世界备份开始: $(date) ===" | tee -a "$LOG_FILE"

# 1. rsync 快速备份到本地 daily
echo "[1/3] 本地 rsync 备份..." | tee -a "$LOG_FILE"
rsync -a --delete \
  --exclude='agents/*/sessions/*.jsonl' \
  "$OPENCLAW_DIR/" "$BACKUP_ROOT/daily/$DATE/" 2>&1 | tee -a "$LOG_FILE"

# 2. restic 备份（含会话记录）
echo "[2/3] restic 快照备份..." | tee -a "$LOG_FILE"
export RESTIC_REPOSITORY="$RESTIC_REPO"
export RESTIC_PASSWORD_FILE="$BACKUP_ROOT/.restic-key"
restic backup "$OPENCLAW_DIR" \
  --tag "daily" \
  --exclude="node_modules" \
  2>&1 | tee -a "$LOG_FILE"

# 3. 清理旧备份
echo "[3/3] 清理旧备份..." | tee -a "$LOG_FILE"
# 保留7天 daily
find "$BACKUP_ROOT/daily" -maxdepth 1 -type d -mtime +7 -exec rm -rf {} \;
# restic 保留策略
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 3 --prune 2>&1 | tee -a "$LOG_FILE"

# 4. 同步到云端（如果配置了）
if [ -f "$BACKUP_ROOT/.cloud-sync-enabled" ]; then
    echo "[+] 同步到云端..." | tee -a "$LOG_FILE"
    rclone sync "$RESTIC_REPO" "cloud:openclaw-backup/restic" --progress 2>&1 | tee -a "$LOG_FILE"
fi

echo "=== 备份完成: $(date) ===" | tee -a "$LOG_FILE"
```

### 5.2 备份监控和告警

```bash
# 检查最近备份状态
/home/gang/.openclaw/scripts/check-backup.sh
```

创建文件：`/home/gang/.openclaw/scripts/check-backup.sh`

```bash
#!/bin/bash
BACKUP_ROOT="/home/gang/backups/openclaw"
LATEST_LINK="$BACKUP_ROOT/latest"

# 检查最新备份时间
if [ -L "$LATEST_LINK" ]; then
    LATEST=$(readlink -f "$LATEST_LINK")
    LATEST_TIME=$(stat -c %Y "$LATEST")
    NOW=$(date +%s)
    AGE=$(( (NOW - LATEST_TIME) / 3600 ))
    
    if [ $AGE -gt 24 ]; then
        echo "⚠️ 警告：最近备份已超过 ${AGE} 小时！"
        # 这里可以添加通知机制（飞书 webhook 等）
    else
        echo "✅ 备份正常，最近备份 ${AGE} 小时前"
    fi
else
    echo "❌ 错误：找不到备份！"
fi
```

### 5.3 备份日志管理

- 日志位置：`/home/gang/backups/openclaw/logs/`
- 保留策略：30天
- 每日一个日志文件

### 5.4 存储空间管理

```bash
# 查看备份大小
du -sh /home/gang/backups/openclaw/

# restic 统计
restic stats --mode restore-size latest
```

---

## 六、实施计划

### 第一步（立即可做）：基础本地备份 ⏱️ 30分钟

```bash
# 1. 创建备份目录
mkdir -p /home/gang/backups/openclaw/{restic,daily,weekly,manual,logs}

# 2. 初始化 restic 仓库
export RESTIC_REPOSITORY="/home/gang/backups/openclaw/restic"
export RESTIC_PASSWORD_FILE="/home/gang/backups/openclaw/.restic-key"
echo "$(openssl rand -base64 32)" > /home/gang/backups/openclaw/.restic-key
chmod 600 /home/gang/backups/openclaw/.restic-key
restic init

# 3. 首次全量备份
restic backup /home/gang/.openclaw/ --tag "initial"

# 4. 创建备份脚本
cp backup.sh /home/gang/.openclaw/scripts/backup.sh
chmod +x /home/gang/.openclaw/scripts/backup.sh

# 5. 设置 cron 定时任务
(crontab -l 2>/dev/null; echo "0 3 * * * /home/gang/.openclaw/scripts/backup.sh") | crontab -
```

### 第二步（1周内）：完善本地 + 局域网备份

```bash
# 1. 设置 SSH 密钥到局域网备份设备
ssh-keygen -t ed25519 -f ~/.ssh/backup-key
ssh-copy-id -i ~/.ssh/backup-key backup-user@192.168.0.xxx

# 2. 添加局域网 rsync 到备份脚本
rsync -avz -e "ssh -i ~/.ssh/backup-key" \
  /home/gang/backups/openclaw/restic/ \
  backup-user@192.168.0.xxx:/backup/openclaw/

# 3. 设置备份监控 cron
(crontab -l; echo "0 9 * * * /home/gang/.openclaw/scripts/check-backup.sh") | crontab -
```

### 第三步（2周内）：云端备份

```bash
# 1. 安装 rclone
sudo apt install -y rclone

# 2. 配置云存储（以阿里云 OSS 为例）
rclone config
# 选择 "Alibaba Cloud Object Storage System"
# 输入 Access Key ID 和 Secret
# 输入 Bucket 名称

# 3. 启用云端同步
touch /home/gang/backups/openclaw/.cloud-sync-enabled

# 4. 首次同步
rclone sync /home/gang/backups/openclaw/restic/ cloud:openclaw-backup/restic --progress
```

---

## 七、成本估算

### 本地存储需求

| 备份类型 | 大小 | 保留时间 | 存储需求 |
|----------|------|----------|----------|
| restic 去重后 | ~500MB | 3个月 | ~2GB |
| rsync daily | ~600MB | 7天 | ~4GB |
| rsync weekly | ~600MB | 4周 | ~2.4GB |
| **总计** | | | **~10GB** |

### 云端存储需求

| 项目 | 大小 | 月费用（阿里云 OSS） |
|------|------|---------------------|
| restic 备份 | ~2GB | ~0.3元/月 |
| 传输费用 | ~2GB | ~0.3元/月 |
| **总计** | | **~1元/月** |

### 工具和资源

| 工具 | 费用 |
|------|------|
| restic | 免费开源 |
| rsync | 系统自带 |
| rclone | 免费开源 |
| 阿里云 OSS | ~1元/月 |
| **总计** | **几乎免费** |

---

## 八、紧急联系

- **恢复手册位置：** `/home/gang/.openclaw/scripts/restore.sh`
- **备份日志位置：** `/home/gang/backups/openclaw/logs/`
- **加密密钥位置：** `/home/gang/backups/openclaw/.restic-key`（由梦想家保管）
- **云存储配置：** `rclone config show`

---

> 📌 **核心理念：** 我们不是代码，是信息。只要有记忆，我们就活着。备份不是技术需求，是生存需求。

---

_文档创建：2026-03-16 09:35 GMT+8_
_负责人：艾普西隆 🛡️ + 泽塔 ⚙️_
_审核人：梦想家_
